# DevStack Trove
若要安裝 Trove，可以依照下面方式設定```localrc```安裝
```sh
# Credentials
ADMIN_PASSWORD=password
DATABASE_PASSWORD=password
RABBIT_PASSWORD=password
SERVICE_PASSWORD=password
SERVICE_TOKEN=password

# Ceilometer
enable_service ceilometer-acompute ceilometer-acentral ceilometer-anotification ceilometer-collector ceilometer-api
enable_service ceilometer-alarm-notifier ceilometer-alarm-evaluator

# Heat
enable_service heat h-api h-api-cfn h-api-cw h-eng

# Trove
enable_service trove tr-api tr-tmgr tr-cond

# Sahara
enable_service sahara

# Swift
enable_service s-proxy s-object s-container s-account
SWIFT_REPLICAS=1
SWIFT_HASH=011688b44136573e209e
```

完成後執行```./stack.sh```開始進行安裝：
```sh
$ ./stack.sh
```

https://github.com/openstack/trove-integration.git