# mysql连接超时问题

## 问题
线上玩家日志记录在mysql，查询log时经常发现有报错：`{failed_to_recv_packet_header,closed}`，猜测可能时数据库空闲的连接失效所导致。

## 排查
环境：
+ 腾讯云mysql-5.7
+ emysql

链接上mysql，查看数据库设置的超时时间：
```bash
mysql> show global variables like 'wait_timeout';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| wait_timeout  | 3600  |
+---------------+-------+
```

数据库默认设置了3600秒。

再看看emysql源码：
```erlang
-module(emysql_tcp).

recv_packet_header(Sock, Timeout, Buff) when erlang:byte_size(Buff) < 4 ->
        case gen_tcp:recv(Sock, 0, Timeout) of
            {ok, Data} ->
                recv_packet_header(Sock, Timeout, <<Buff/binary, Data/binary>>);
            {error, Reason} ->
                exit({failed_to_recv_packet_header, Reason})
        end;
```

找到了问题产生的地方，由于`recv`了一个已关闭的连接导致。再向上翻：
```erlang
-module(emysql_tcp).

send_and_recv_packet(Sock, Packet, SeqNum) ->
    case gen_tcp:send(Sock, [<<(size(Packet)):24/little, SeqNum:8>>, Packet]) of
        ok -> ok;
        {error, closed} ->
            %% If we can't communicate on the socket since it has been closed, we exit the process
            %% at this point. The exit reason is caught by `emysql:monitor_work/3` and since it is
            %% with the atom `conn_tcp_closed` we special-case that and rehandle it properly
            exit(tcp_connection_closed)
    end,
    DefaultTimeout = emysql_app:default_timeout(),
    case response_list(Sock, DefaultTimeout, ?SERVER_MORE_RESULTS_EXIST) of
        % This is a bit murky. It's compatible with former Emysql versions
        % but sometimes returns a list, e.g. for stored procedures,
        % since an extra OK package is sent at the end of their results.
        [Record] -> Record;
        List -> List
    end.
```

这里发送一个数据包到mysql，然后去`recv`(response_list函数调用)回包。这里可以看出，即使超时了，向mysql发送数据时，该tcp连接依然可用，等到mysql接收到后，认为该连接已超时，才断开连接，所以才导致`send`是ok的而`recv`错误。

再看看`emysql:execute/3`的逻辑，这是执行sql语句的接口：
```erlang
-module(emysql).

execute(PoolId, Query, Args, Timeout) ->
    Connection = emysql_conn_mgr:wait_for_connection(PoolId),
    monitor_work(Connection, Timeout, [Connection, Query, Args]);
    ......

monitor_work(Connection0, Timeout, Args) when is_record(Connection0, emysql_connection) ->
    Connection = if
                     Connection0#emysql_connection.alive =:= false ->
                         case emysql_conn:reset_connection(emysql_conn_mgr:pools(), Connection0, keep) of
                             NewConn when is_record(NewConn, emysql_connection) ->
                                 NewConn;
                             {error, FailedReset0} ->
                                 exit({connection_down, {and_conn_reset_failed, FailedReset0}})
                         end;
                     true ->
                         case emysql_conn:need_test_connection(Connection0) of %% 问题所在
                             true ->
                                 emysql_conn:test_connection(Connection0, keep);
                             false ->
                                 Connection0
                         end
                 end,
    ......
```

emysql使用一条连接前，会先检测该连接是否已超时(`emysql_conn:need_test_connection`)，如果已超时，则重新建立建立(`emysql_conn:test_connection`)。再看看它的检测接口：

```erlang
-module(emysql_conn).

need_test_connection(Conn) ->
   (Conn#emysql_connection.test_period =:= 0) orelse
     (Conn#emysql_connection.last_test_time =:= 0) orelse
     (Conn#emysql_connection.last_test_time + Conn#emysql_connection.test_period < now_seconds()).
```

是否超时，emysql单纯根据时间来确定，`Conn#emysql_connection.test_period`就是超时时间，其默认是28000，emysql.hrl头文件定义了`-define(CONN_TEST_PERIOD, 28000).`。所以，一条连接空闲了3600秒后，mysql认为它已经是超时连接，但emysql认为还是有效连接，所以正常去使用，所以导致了该问题。


## 结论

1. emysql的超时时间应设置为于mysql的一致，application的evn参数可以设置`conn_test_period`，单位秒。
2. 连接池大小再设计前应该考虑好，如果业务量少，不需要太多连接。
3. 数据库操作接口报错后，需要有个容错措施。比如可以把sql语句缓存到dets，等下次再有写入操作时一并执行，确保日志数据不丢失。
