# event-tests
    cd xxxx/build/test/network_tests/event_tests
## service
    gedit event_test_slave_tcp.json
    # 输入service端IP
    "unicast":"XXX.XXX.XXX.XXX" => "unicast":"173.20.14.127"

    # 执行
    ./event_test_slave_starter.sh TCP
## client
    gedit event_test_master.json
    # 输入client端IP
    "unicast":"XXX.XXX.XXX.XXX" => "unicast":"173.20.14.111"

    #执行
    ./event_test_master_starter.sh PAYLOAD_FIXED TCP
## 结果
![service端测试截图](pic/event-tests-service.png)

![client端测试截图](pic/event-tests-client.png)