# SOMEIP configuration
1. 没看到相关文档

# vSOMEP configuration实现
## vSOMEIP configuration代码目录
    # 函数参数形式
    # vsomeip/implementation/configuration/include/configuration_element.hpp
    void load_tcp_restart_settings(const configuration_element &_element);

    # vsomeip/implementation/configuration/include/eventgroup.hpp
    std::shared_ptr<eventgroup> find_eventgroup(service_t _service,
            instance_t _instance, eventgroup_t _eventgroup) const;

    # 非函数参数形式
    # vsomeip/implementation/security/include/policy_manager_impl.hpp
    std::shared_ptr<policy_manager_impl> policy_manager_;

    # vsomeip/implementation/security/include/security.hpp
    std::shared_ptr<security> security_;

    # vsomeip/implementation/configuration/include/application_configuration.hpp
    std::map<
        std::string,
        application_configuration
    > applications_;

    # vsomeip/implementation/configuration/include/service.hpp
    std::map<service_t,
        std::map<instance_t,
            std::shared_ptr<service> > > services_;

    # vsomeip/implementation/configuration/include/service.hpp
    std::map<std::string, // IP
        std::map<std::uint16_t, // port
            std::map<service_t,
                std::shared_ptr<service>>>> services_by_ip_port_;

    # vsomeip/implementation/configuration/include/configuration_impl.hpp
    # vsomeip/implementation/configuration/include/event.hpp
    std::set<suppress_t> suppress_events_;

    # vsomeip/implementation/configuration/include/client.hpp
    std::list< std::shared_ptr<client> > clients_;

    # vsomeip/implementation/configuration/include/routing.hpp
    routing_t routing_;

    # vsomeip/implementation/configuration/include/trace.hpp
    std::shared_ptr<trace> trace_;

    # vsomeip/implementation/configuration/include/watchdog.hpp
    std::shared_ptr<watchdog> watchdog_;

    # vsomeip/implementation/configuration/include/service_instance_range.hpp
    std::vector<service_instance_range> internal_service_ranges_;

    # vsomeip/implementation/configuration/include/e2e.hpp
    std::map<e2exf::data_identifier_t, std::shared_ptr<cfg::e2e>> e2e_configuration_;

    # vsomeip/implementation/configuration/include/debounce_filter_impl.hpp
    debounce_configuration_t debounces_;

## vSOMEIP configuration类关系图
![configuration类图](ap-2311/pic/vsomeip-configuration_impl-class.png)

## vSOMEIP configuration实现思路
1. 解析json文件
    1. 使用boost提供的boost::property_tree::ptree类解析json文件
2. 通过key，如"events"等关键字，获取对于的value
3. value存到configration_impl类内的变量中

## configuration_impl类解析
### eventgroup类
    struct eventgroup {
        eventgroup_t id_;
        std::set<std::shared_ptr<event> > events_;
        std::string multicast_address_;
        uint16_t multicast_port_;
        uint8_t threshold_;
    };
#### json中的eventgroup
    "eventgroups" :
    [
        {
            "eventgroup" : "0x1111",
            "threshold" : "8",
            "is_multicast" : "true",
            "events" : [ "0x778", "0x77A" ]
        }
    ]
#### load_eventgroup函数
    void load_eventgroup(std::shared_ptr<service> &_service,
            const boost::property_tree::ptree &_tree);

1. 遍历_tree，获取key值，如eventgroup相关key值："eventgroup"、"threshold"、"events"等
2. 获去的key填充到eventgroup的临时变量中
    a. _service->events[its_event_id] = its_event;
3. 上面的临时变量和eventgroup_id做成key-value存到_service中
    a. _service->eventgroups[its_eventgroup->id_] = its_eventgroup;

### configuration_element类
1. name_是地址
2. tree_是解析json文件后的数据结构

        struct configuration_element {
            std::string name_;
            boost::property_tree::ptree tree_;

            configuration_element(const std::string &_name, const boost::property_tree::ptree &_tree) noexcept
                : name_(_name),
                tree_(_tree) {
            }

            configuration_element(configuration_element &&_source) noexcept
                : name_(std::move(_source.name_)),
                tree_(std::move(_source.tree_)) {
            }

            bool operator<(const configuration_element &_other) const {
                return (name_ < _other.name_);
            }
        };
#### load_services函数为例
以load_services函数讲解类configuration_element的用法
1. read_data将json配置地址读取到ptree类型的数据中

        boost::property_tree::ptree its_tree;
        try {
            boost::property_tree::json_parser::read_json(_input, its_tree);
            _elements.push_back({ _input, its_tree });
        }
        catch (boost::property_tree::json_parser_error&) {
            _failed.insert(_input);
        }
2. 然后把地址和ptree数据存到_elements

#### read_data函数
    void configuration_impl::read_data(const std::set<std::string> &_input,
            std::vector<configuration_element> &_elements, std::set<std::string> &_failed,
            bool _mandatory_only, bool _read_second_level)

#### load_data函数
1. 遍历std::vector<configuration_element> &_elements

        bool configuration_impl::load_data(const std::vector<configuration_element> &_elements,
                bool _load_mandatory, bool _load_optional)
        {
            ...
                for (const auto& e : _elements) {
                    has_routing = load_routing(e) || has_routing;
                    has_applications = load_applications(e) || has_applications;
                    load_network(e);
                    load_diagnosis_address(e);
                    load_shutdown_timeout(e);
                    load_payload_sizes(e);
                    load_endpoint_queue_sizes(e);
                    load_tcp_restart_settings(e);
                    load_permissions(e);
                    load_security(e);
                    load_tracing(e);
                    load_udp_receive_buffer_size(e);
                    load_services(e);
                }
            ...
        }
#### load_services函数
1. 根据关键字"services"、"servicegroups"分为两种加载情况
    a. load_service
    b. load_servicegroup

        void configuration_impl::load_services(const configuration_element &_element) {
            std::lock_guard<std::mutex> its_lock(services_mutex_);
            try {
                auto its_services = _element.tree_.get_child("services");
                for (auto i = its_services.begin(); i != its_services.end(); ++i)
                    load_service(i->second, default_unicast_);
            } catch (...) {
                try {
                    auto its_servicegroups = _element.tree_.get_child("servicegroups");
                    for (auto i = its_servicegroups.begin(); i != its_servicegroups.end(); ++i)
                        load_servicegroup(i->second);
                } catch (...) {
                    // intentionally left empty
                }
            }
        }

#### load_servicegroup函数
    void configuration_impl::load_servicegroup(
            const boost::property_tree::ptree &_tree) 

1. 遍历_tree节点，查找key值，如key值："delays"、"services"等
2. load_delays(i->second)
    a. 未保存到service类中
3. load_service(j->second, its_unicast_address);
    a. services_[its_service->service_][its_service->instance_] = its_service;

#### json中的servicegroups
    "servicegroups" :
    [
        {
            "name" : "default",
            "unicast" : "local",
            "delays" :
            {
                "initial" : { "minimum" : "10", "maximum" : "100" },
                "repetition-base" : "200",
                "repetition-max" : "7",
                "cyclic-offer" : "2132",
                "cyclic-request" : "2001",
                "ttl" : "5"
            },
            "services" :
            [
                {
                    "service" : "0x1234",
                    "instance" : "0x0022",
                    "reliable" : { "port" : "30506", "enable-magic-cookies" : "true" },
                    "unreliable" : "31000",
                    "events" :
                    [
                        {
                            "event" : "0x0778",
                            "is_field" : "false"
                        },
                        {
                            "event" : "0x779",
                            "is_field" : "true"
                        },
                        {
                            "event" : "0x77A",
                            "is_field" : "false"
                        }
                    ],
                    "eventgroups" :
                    [
                        {
                            "eventgroup" : "0x4567",
                            "multicast" : "225.226.227.228",
                            "events" : [ "0x778", "0x779" ]
                        },
                        {
                            "eventgroup" : "0x4569",
                            "multicast" : "225.227.227.228",
                            "events" : [ "0x779", "0x77A" ]
                        },
                        {
                            "eventgroup" : "0x4569",
                            "multicast" : "225.222.227.228",
                            "events" : [ "0x778", "0x77A" ]
                        }
                    ]
                },
                {
                    "service" : "0x1234",
                    "instance" : "0x0023",
                    "reliable" : "30503"
                },
                {
                    "service" : "0x7809",
                    "instance" : "0x1",
                    "multicast" :
                    {
                        "address" : "224.212.244.225",
                        "port" : "1234"
                    },
                    "eventgroups" :
                    [
                        {
                            "eventgroup" : "0x1111",
                            "threshold" : "8",
                            "is_multicast" : "true",
                            "events" : [ "0x778", "0x77A" ]
                        }
                    ]
                }
            ]
        },
        {
            "name" : "extra",
            "unicast" : "local",
            "delays" :
            {
                "initial" : { "minimum" : "10", "maximum" : "100" },
                "repetition-base" : "200",
                "repetition-max" : "7",
                "cyclic-offer" : "2132",
                "cyclic-request" : "2001",
                "ttl" : "5"
            },
            "services" :
            [
                {
                    "service" : "0x2277",
                    "instance" : "0x0022",
                    "reliable" : { "port" : "30505" },
                    "unreliable" : "31001"
                },
                {
                    "service" : "0x2266",
                    "instance" : "0x0022",
                    "reliable" : "30505",
                    "unreliable" : "30507"
                },
                {
                    "service" : "0x3333",
                    "instance" : "0x1"
                },
                {
                    "service" : "0x3555",
                    "instance" : "0x1",
                    "protocol" : "other"
                }
            ]
        },
        {
            "name" : "remote",
            "unicast" : "10.0.2.23",
            "services" :
            [
                {
                    "service" : "0x4466",
                    "instance" : "0x0321",
                    "reliable" : "30506",
                    "unreliable" : "30444"
                }
            ]
        }
    ],

#### load_service函数

#### json中的"services"
    "services":
    [
        {
            "service":"0x1000",
            "instance":"0x0001",
            "unreliable":"30000",
            "reliable":
            {
                "port":"40000",
                "enable-magic-cookies":"false"
            }
        },
        {
            "service":"0x2000",
            "instance":"0x0001",
            "unreliable":"30000",
            "reliable":
            {
                "port":"40000",
                "enable-magic-cookies":"false"
            }
        },
        {
            "service":"0x3000",
            "instance":"0x0001",
            "unreliable":"30000",
            "reliable":
            {
                "port":"40000",
                "enable-magic-cookies":"false"
            }
        }
    ],
### service类
1. vsomeip/implementation/configuration/include/service.hpp

        struct service {
            service_t service_;
            instance_t instance_;

            std::string unicast_address_;

            uint16_t reliable_;
            uint16_t unreliable_;

            std::string multicast_address_;
            uint16_t multicast_port_;

            std::string protocol_;

            // [0] = debounce_time
            // [1] = retention_time
            typedef std::map<method_t, std::array<std::chrono::nanoseconds, 2>> npdu_time_configuration_t;
            npdu_time_configuration_t debounce_times_requests_;
            npdu_time_configuration_t debounce_times_responses_;

            std::map<event_t, std::shared_ptr<event> > events_;
            std::map<eventgroup_t, std::shared_ptr<eventgroup> > eventgroups_;

            // SOME/IP-TP
            std::map<method_t, std::pair<uint16_t, uint32_t> > tp_client_config_;
            std::map<method_t, std::pair<uint16_t, uint32_t> > tp_service_config_;
        };
2. 见configuration_element中的load_service函数

### policy_manager_impl类
1. vsomeip/implementation/security/include/policy_manager_impl.hpp
#### load_security函数
    void configuration_impl::load_security(const configuration_element &_element) {
        ...
    }
1. 查找关键字“security”，获取其value

        auto its_security = element.tree.get_child_optional("security");

2. 若有value不为空，加载policy
    
        policy_manager_->load(_element);
#### json中的"security"
    "security" :
    {
        "check_credentials" : "true",
        "policies" :
        [
            {
                "client" : { "first" : "0x1343", "last" : "0x1346" },
                "credentials" : { "uid" : "0", "gid" : "0" },
                "allow" :
                {
                    "requests":
                    [
                        {
                            "service"  : "0x1234",
                            "instance" : "0x5678"
                        }
                    ]
                }
            }
        ]
    },

### security类
1. 这里的语法很有意思，但不知道设计意图

        class VSOMEIP_IMPORT_EXPORT security {
        public:
            security(std::shared_ptr<policy_manager_impl> _policy_manager);
            bool load();

            std::function<decltype(vsomeip_sec_policy_initialize)>                         initialize;
            std::function<decltype(vsomeip_sec_policy_authenticate_router)>                authenticate_router;
            std::function<decltype(vsomeip_sec_policy_is_client_allowed_to_offer)>         is_client_allowed_to_offer;
            std::function<decltype(vsomeip_sec_policy_is_client_allowed_to_request)>       is_client_allowed_to_request;
            std::function<decltype(vsomeip_sec_policy_is_client_allowed_to_access_member)> is_client_allowed_to_access_member;
            std::function<decltype(vsomeip_sec_sync_client)>                               sync_client;

        private:

            decltype(vsomeip_sec_policy_initialize)                         default_initialize;
            decltype(vsomeip_sec_policy_authenticate_router)                default_authenticate_router;
            decltype(vsomeip_sec_policy_is_client_allowed_to_offer)         default_is_client_allowed_to_offer;
            decltype(vsomeip_sec_policy_is_client_allowed_to_request)       default_is_client_allowed_to_request;
            decltype(vsomeip_sec_policy_is_client_allowed_to_access_member) default_is_client_allowed_to_access_member;
            decltype(vsomeip_sec_sync_client)                               default_sync_client;

            std::shared_ptr<policy_manager_impl> policy_manager_;
        };
#### check_routing_credentials函数
1. 校验路由资格

        configuration_impl::check_routing_credentials(
                client_t _client, const vsomeip_sec_client_t *_sec_client) const {
            ...
        }   
2. 由policy_manager_impl的函数check_routing_credentials完成

        policy_manager_impl::check_routing_credentials(
                const vsomeip_sec_client_t *_sec_client) const {
            ...
        }

### application_configuration类
    struct application_configuration {
        client_t client_;
        std::size_t max_dispatchers_;
        std::size_t max_dispatch_time_;
        std::size_t max_detach_thread_wait_time_;
        std::size_t thread_count_;
        std::size_t request_debouncing_;
        std::map<plugin_type_e, std::set<std::string> > plugins_;
        int nice_level_;
        debounce_configuration_t debounces_;
        bool has_session_handling_;
    };
#### load_applications函数
1. 根据关键字"applications"查找对应的value
2. 保存value值到applications_

        applications_[its_name] = {its_id,
                                its_max_dispatchers,
                                its_max_dispatch_time,
                                its_max_detached_thread_wait_time,
                                its_io_thread_count,
                                its_request_debounce_time,
                                plugins,
                                its_io_thread_nice_level,
                                its_debounces,
                                has_session_handling};

#### json中的"applications"
    "applications" :
    [
        {
            "name" : "event_test_service",
            "id" : "0x1210",
            "max_dispatch_time" : "1000"
        }
    ],

### event类
    struct event {
        event(event_t _id, bool _is_field, reliability_type_e _reliability,
                std::chrono::milliseconds _cycle, bool _change_resets_cycle,
                bool _update_on_change)
            : id_(_id),
            is_field_(_is_field),
            reliability_(_reliability),
            cycle_(_cycle),
            change_resets_cycle_(_change_resets_cycle),
            update_on_change_(_update_on_change) {
        }

        event_t id_;
        bool is_field_;
        reliability_type_e reliability_;
        std::vector<std::weak_ptr<eventgroup> > groups_;

        std::chrono::milliseconds cycle_;
        bool change_resets_cycle_;
        bool update_on_change_;
    };

#### load_event函数
1. 根据关键字查找"event"，获取value
2. 将vlaue存入
    a. _service->events[its_event_id] = its_event;
3. value中的有些关键字和load_event函数中的处理不一致

        void configuration_impl::load_event(
                std::shared_ptr<service> &_service,
                const boost::property_tree::ptree &_tree) {
            ...
        }
#### json中的event
    "events" :
    [
        {
            "event" : "0x0777",
            "is_field" : "true",
            "update-cycle" : 2000
        },
        {
            "event" : "0x0778",
            "is_field" : "true",
            "update-cycle" : 0
        },
        {
            "event" : "0x0779",
            "is_field" : "true"
        }
    ],

### suppress_t类
1. 这是个可选配置

        struct suppress_t {
            service_t service;
            instance_t instance;
            event_t event;

            inline bool operator<(const suppress_t& entry_) const {
                if(service != entry_.service) {
                    return service < entry_.service;
                }
                if(instance != entry_.instance) {
                    return instance < entry_.instance;
                }
                if(event != entry_.event) {
                    return event < entry_.event;
                }
                return false;
            }
        };
#### load_suppress_events函数
1. 不知什么意思

### client类
1. vsomeip/implementation/configuration/include/client.hpp

    struct client {
        client() : service_(ANY_SERVICE),
                instance_(ANY_INSTANCE) {
        }

        // ports for specific service / instance
        service_t service_;
        instance_t instance_;
        std::map<bool, std::set<uint16_t> > ports_;
        std::map<bool, uint16_t> last_used_specific_client_port_;

        // client port ranges mapped to remote port ranges
        std::map<bool, std::pair<uint16_t, uint16_t> > remote_ports_;
        std::map<bool, std::pair<uint16_t, uint16_t> > client_ports_;
        std::map<bool, uint16_t> last_used_client_port_;
    };
#### json中的events
    "clients" :
    [
        {
            "reliable_remote_ports"   : { "first" : "30500", "last" : "30599" },
            "unreliable_remote_ports" : { "first" : "30500", "last" : "30599" },
            "reliable_client_ports"   : { "first" : "30491", "last" : "30499" },
            "unreliable_client_ports" : { "first" : "30491", "last" : "30499" }
        },
        {
            "reliable_remote_ports"   : { "first" : "31500", "last" : "31599" },
            "unreliable_remote_ports" : { "first" : "31500", "last" : "31599" },
            "reliable_client_ports"   : { "first" : "31491", "last" : "31499" },
            "unreliable_client_ports" : { "first" : "31491", "last" : "31499" }
        },
        {
            "reliable_remote_ports"   : { "first" : "32500", "last" : "32599" },
            "unreliable_remote_ports" : { "first" : "32500", "last" : "32599" },
            "reliable_client_ports"   : { "first" : "32491", "last" : "32499" },
            "unreliable_client_ports" : { "first" : "32491", "last" : "32499" }
        },
        {
            "service" : "0x8888",
            "instance" : "0x1",
            "unreliable" : [ "0x11", "0x10" ],
            "reliable" : [ "0x11", "0x10" ]
        },
        {
            "service" : "8888",
            "instance" : "1",
            "unreliable" : [ 40000, 40001 ],
            "reliable" : [ 40000, 40001 ]
        }
    ],
#### load_clients函数
1. "clients" => value
2. 存入clients_.push_back(its_client);

        void configuration_impl::load_clients(const configuration_element &_element) {
            ...
        }

        void configuration_impl::load_client(const boost::property_tree::ptree &_tree) {
            ...
        }

### routing类
1. vsomeip/implementation/configuration/include/routing.hpp

        struct routing_t {
            bool is_enabled_;

            routing_host_t host_;
            routing_guests_t guests_;

            routing_t() : is_enabled_(true) {}

            routing_t &operator=(const routing_t &_other) {
                is_enabled_ = _other.is_enabled_;
                host_ = _other.host_;
                guests_ = _other.guests_;

                return *this;
            }
        };
#### json中的"routing"
    "routing":
    {
        "host" :
        {
            "name" : "big_payload_test_client",
            "unicast" : "XXX.XXX.XXX.XXX",
            "port" : "31490"
        },
        "guests" :
        {
            "unicast" : "XXX.XXX.XXX.XXX"
        }
    },
#### load_routing函数
    bool
    configuration_impl::load_routing(const configuration_element &_element)
    {
        ...
    }

    configuration_impl::load_routing_host(const boost::property_tree::ptree &_tree,
            const std::string &_name)
    {
        ...
    }

    bool
    configuration_impl::load_routing_guests(const boost::property_tree::ptree &_tree) {
        ...
    }

### trace类
1. vsomeip/implementation/configuration/include/trace.hpp

        struct trace {
            trace()
                : is_enabled_(false),
                is_sd_enabled_(false),
                channels_(),
                filters_() {
            }

            bool is_enabled_;
            bool is_sd_enabled_;

            std::vector<std::shared_ptr<trace_channel>> channels_;
            std::vector<std::shared_ptr<trace_filter>> filters_;
        };
#### json中的"trace"
    "tracing" :
    {
        "enable" : "false"
    },
#### load_tracing函数
    void configuration_impl::load_tracing(const configuration_element &_element) {
        ...
    }

### watchdog类
1. vsomeip/implementation/configuration/include/watchdog.hpp

        struct watchdog {
            watchdog()
                : is_enabeled_(false),
                timeout_in_ms_(VSOMEIP_DEFAULT_WATCHDOG_TIMEOUT),
                missing_pongs_allowed_(VSOMEIP_DEFAULT_MAX_MISSING_PONGS) {
            }

            bool is_enabeled_;
            uint32_t timeout_in_ms_;
            uint32_t missing_pongs_allowed_;
        };
#### json中的watchdog
    "watchdog" :
    {
        "enable" : "true",
        "timeout" : "1234",
        "allowed_missing_pongs" : "7"
    },
#### load_watchdog函数
    void configuration_impl::load_watchdog(const configuration_element &_element) {
        ...
    }

#### service_instance_range类
1. vsomeip/implementation/configuration/include/service_instance_range.hpp

        struct service_instance_range {
                service_t first_service_;
                service_t last_service_;
                instance_t first_instance_;
                instance_t last_instance_;
        };
#### json中的internal_services
    "internal_services" :
    [
        {
            "first" : "0xF100",
            "last" : "0xF109"
        },
        {
            "first" : {
                "service" : "0xF300",
                "instance" : "0x1"
            },
            "last" : {
                "service" : "0xF300",
                "instance" : "0x10"
            }
        }
    ],
#### load_internal_services函数
1. "internal_services"  => 获取value
2. 存入internal_service_ranges_.push_back(range);

        void configuration_impl::load_internal_services(const configuration_element &_element) {
            ...
        }

### e2e类
1. vsomeip/implementation/configuration/include/e2e.hpp

        struct e2e {
            typedef std::map<std::string, std::string> custom_parameters_t;

            e2e() :
                variant(""),
                profile(""),
                service_id(0),
                event_id(0) {
            }

            e2e(const std::string &_variant, const std::string &_profile, service_t _service_id,
                event_t _event_id, custom_parameters_t&& _custom_parameters) :
                variant(_variant),
                profile(_profile),
                service_id(_service_id),
                event_id(_event_id),
                custom_parameters(_custom_parameters) {
            }

            // common config
            std::string variant;
            std::string profile;
            service_t service_id;
            event_t event_id;

            // custom parameters
            custom_parameters_t custom_parameters;
        };
#### json中的e2e
    "e2e" :
    {
        "e2e_enabled" : "true",
        "protected" :
        [
            {
                "service_id" : "0xd025",
                "event_id" : "0x0001",
                "profile" : "P04",
                "variant" : "checker",
                "crc_offset" : "64",
                "data_id" : "0x2d"
            },
            {
                "service_id" : "0xd025",
                "event_id" : "0x8001",
                "profile" : "P04",
                "variant" : "checker",
                "crc_offset" : "64",
                "data_id" : "0x2d"
            }
        ]
    },
#### load_e2e函数
1. "e2e" => 获取value
2. value存入e2e_configuration_
    
        e2e_configuration_[std::make_pair(service_id, event_id)] = std::make_shared<cfg::e2e>(
            variant,
            profile,
            service_id,
            event_id,
            std::move(custom_parameters)
        );

### debounce_configuration_t类
1. vsomeip/implementation/configuration/include/debounce_filter_impl.hpp

        using debounce_configuration_t =
            std::map<service_t,
                std::map<instance_t,
                    std::map<event_t,
                        std::shared_ptr<debounce_filter_impl_t>>>>;json中的"debounce"

#### load_debounce函数
1. "debounce" => 获取value
2. 存入_debounces[its_service][its_instance] = its_debounces;

        void
        configuration_impl::load_debounce(const configuration_element &_element) {
            ...
        }

        void
        configuration_impl::load_service_debounce(
                const boost::property_tree::ptree &_tree,
                debounce_configuration_t &_debounces) {
            ...
        }