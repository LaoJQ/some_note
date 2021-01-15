# ranch

Ranch是一个tcp连接池，用于服务端创建和管理tcp连接。

## 使用

ranch使用简单，首先启动ranch的application：

```erlang
application:ensure_all_started(ranch).
```

然后启动`listener`：
```erlang
{ok, _} = ranch:start_listener(tcp_echo, ranch_tcp, [{port, 5555}], echo_protocol, []).
```

`demo_protocol`是自定义的模块，需要实现`ranch_protocol`behaviour，作为处理客户端连接的进程：
```erlang
%% ranch/examples/tcp_echo/src/echo_protocol.erl
-module(echo_protocol).
-behaviour(ranch_protocol).

-export([start_link/4]).
-export([init/3]).

start_link(Ref, _Socket, Transport, Opts) ->
	Pid = spawn_link(?MODULE, init, [Ref, Transport, Opts]),
	{ok, Pid}.

init(Ref, Transport, _Opts = []) ->
	{ok, Socket} = ranch:handshake(Ref),
	loop(Socket, Transport).

loop(Socket, Transport) ->
	case Transport:recv(Socket, 0, 5000) of
		{ok, Data} when Data =/= <<4>> ->
			Transport:send(Socket, Data),
			loop(Socket, Transport);
		_ ->
			ok = Transport:close(Socket)
	end.
```


## ranch_server

该进程在ranch应用启动时启动，用于操作ets记录listener的信息，以及监控listener和conns_sup进程。

```erlang
{{max_conns, Ref}, MaxConns}
{{trans_opts, Ref}, TransOpts}
{{proto_opts, Ref}, ProtoOpts}
{{listener_start_args, Ref}, StartArgs}
{{listener_sup, Ref}, Pid} %% listener_sup进程pid
{{addr, Ref}, Addr} %% tcp监听的ip端口
{{conns_sup, Ref}, Pid} %% conns_sup进程pid
```


## 几个问题

### why not gen_server?

从上面的`echo_protocol`例子看出，`init`函数调用了`ranch:handshake(Ref)`获取socket，但翻源码可知，该函数是堵塞式的：
```erlang
-module(ranch).

handshake(Ref, Opts) ->
	receive {handshake, Ref, Transport, CSocket, HandshakeTimeout} ->
        ......
```
那这个message什么时候会发送给protocol进程呢？

再翻翻tcp accept流程，实际上调用tcp accept的是`ranch_acceptor`模块：
```erlang
loop(LSocket, Transport, Logger, ConnsSup) ->
	_ = case Transport:accept(LSocket, infinity) of
		{ok, CSocket} ->
			case Transport:controlling_process(CSocket, ConnsSup) of
				ok ->
					%% This call will not return until process has been started
					%% AND we are below the maximum number of connections.
					ranch_conns_sup:start_protocol(ConnsSup, CSocket);
				{error, _} ->
					Transport:close(CSocket)
			end;
        ......
```

accept完后，调用了`ranch_conns_sup:start_protocol/2`，通知`ranch_conns_sup`进程创建protocol进程：
```erlang
-module(ranch_conns_sup).

loop(State=#state{parent=Parent, ref=Ref, conn_type=ConnType,
		transport=Transport, protocol=Protocol, opts=Opts,
		max_conns=MaxConns, logger=Logger}, CurConns, NbChildren, Sleepers) ->
	receive
		{?MODULE, start_protocol, To, Socket} ->
			try Protocol:start_link(Ref, Socket, Transport, Opts) of
				{ok, Pid} ->
					handshake(State, CurConns, NbChildren, Sleepers, To, Socket, Pid, Pid);
                ......

handshake(State=#state{ref=Ref, transport=Transport, handshake_timeout=HandshakeTimeout,
		max_conns=MaxConns}, CurConns, NbChildren, Sleepers, To, Socket, SupPid, ProtocolPid) ->
	case Transport:controlling_process(Socket, ProtocolPid) of
		ok ->
			ProtocolPid ! {handshake, Ref, Transport, Socket, HandshakeTimeout},
            ......
```

最终`ranch_conns_sup`进程会在，protocol进程创建完成后才发送。

假如用`gen_server`作为protocol进程入口，这就无法在`init`回调函数中调用`ranch:handshake(Ref)`了，因为此时`gen_server`进程还未算创建完，`ranch_conns_sup`进程也不会发送消息，这样就造成了死锁。因此，可以绕过`gen_server`的启动流程，解决这一问题：
```erlang
start_link(Ref, _Socket, Transport, Opts) ->
	{ok, proc_lib:spawn_link(?MODULE, init, [Ref, Transport, Opts])}.

init(Ref, Transport, Opts) ->
    ok = proc_lib:init_ack({ok, self()}),
	{ok, Socket} = ranch:handshake(Ref),
    %% do some state init
    State = {},
    gen_server:enter_loop(?MODULE, [], State, infinity).
```

其实`gen_server`底层也是调用`proc_lib:spawn_link`来创建新进程的，而该函数是堵塞等待进程创建完成：
```erlang
start_link(M,F,A,Timeout,SpawnOpts) when is_atom(M), is_atom(F), is_list(A) ->
    Pid = ?MODULE:spawn_opt(M, F, A, ensure_link(SpawnOpts)),
    sync_wait(Pid, Timeout).

sync_wait(Pid, Timeout) ->
    receive
	{ack, Pid, Return} ->
	    Return;
    ......
```

即收到`{ack, Pid, Return}`消息就认为新进程创建完成，然后`start_link`返回。`proc_lib:init_ack`函数就是用于发送该消息。而`gen_server`是在`init`回调函数结束后才调用该函数，所以就会出现上述的问题。那么，可以通过提前调用`init_ack`使得`ranch_conns_sup`提前结束堵塞，发送handshake消息。这样就能顺利调用`ranch:handshake`函数完成tcp进程初始化。最后通过`gen_server:enter_loop`进入正常的`gen_server`逻辑。

### ranch_tcp.erl

`ranch:start_listener`启动函数需要传参一个传输层模块，并实现了`ranch_transport`行为模式。ranch默认提供了两个传输层模块，`ranch_tcp`和`ranch_ssl`。可以看到`ranch_tcp`模块源码的`listen`函数，默认丢弃了`header`参数，并且`packet`参数默认为`raw`。当我们自定义应用层协议（例如：`size/4,cmd/2,payload`类似这种格式）时，就没法利用到`gen_tcp`模块提供的包头处理功能了。

最简单粗暴的解决方法，可以用`ranch_tcp`拷贝一个新的自定义传输层模块，修改`disallowed_listen_options`函数内容，即可：

```erlang
%% 简单粗暴把所有选项都去掉
disallowed_listen_options() ->
	[].
```

这样启动`listener`时传入新的传输层模块，就能使用到`header`和`packet`的功能了。

或者利用`Transport:setopts/2`（即`ranch_tcp:setopts/2`）函数手动修改socket参数：

```erlang
Transport:setopts(Socket, [{packet, 4}, {header, 1}])， %% 包头长度为4，消息头1字节
```



### 高并发时的瓶颈

在ranch 1.x源码可以看到，每个listener只有一个`ranch_conn_sup`进程用来新建和管理连接进程，`ranch_acceptor`接收到tcp连接后，统一发送消息给`ranch_conn_sup`进程处理，然后acceptor堵塞等待返回值：

```erlang
-module(ranch_acceptor).
loop(LSocket, Transport, Logger, ConnsSup, MonitorRef) ->
	_ = case Transport:accept(LSocket, infinity) of
		{ok, CSocket} ->
			case Transport:controlling_process(CSocket, ConnsSup) of
				ok ->
					ranch_conns_sup:start_protocol(ConnsSup, MonitorRef, CSocket);
                	......

-module(ranch_conns_sup).
start_protocol(SupPid, MonitorRef, Socket) ->
	SupPid ! {?MODULE, start_protocol, self(), Socket},
	receive %% 发送给ranch_conn_sup进程后，堵塞等待返回值
		SupPid ->
			ok;
		{'DOWN', MonitorRef, process, SupPid, Reason} ->
			error(Reason)
	end.
```

高并发情况下，`ranch_conn_sup`进程消息队列就会积压消息，导致系统整体的accept效率不足，因为这时系统相当于一个单进程工作状态。

解决方法是增加`ranch_conn_sup`进程数量。ranch 2.0版本增加了`num_conns_sups`选项用于listener启动时定义`ranch_conn_sup`进程数量，默认等于acceptor的数量。这样就可以解决单进程处理新建连接的问题。