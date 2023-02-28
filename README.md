<h2><center style="color:red;font-family:Noto Mono">OpenSIPS 集群</center></h2>

### 目标：

```shell
用户随机注册到不同opensips实例上，并且usrloc数据是共享的，可被lookup的
```

### 环境准备：

```bash
CentOS 7 物理机IP: 10.10.18.20 （版本：7.9.2009） 
一个 keepalived 实例 (版本：v2.0.7)「 物理机安装 」
三个 opensips 实例（版本：v3.3.3）「 容器化安装 」
一个 MySQL 实例 （版本：v5.7.31）「 容器化安装 」
一个 mongodb 实例（版本：v4.1）「 容器化安装 」
```

### 编排文件:

#### `mysql` & `mongodb` 实例

```yaml
version: "3"
services:
  mysql:
    container_name: mysql
    image: mysql:5.7.31
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=MyNewPass4!
      - MYSQL_DATABASE=opensips
      - MYSQL_USER=opensips
      - MYSQL_PASSWORD=Opensipsrw4!
      - MYSQL_ROOT_HOST=%
      - TZ=Asia/Shanghai
    ports:
      - 3306:3306
    networks:
      sipnetwork:
        ipv4_address: 192.168.1.100
    volumes:
      - $PWD/mysql/data:/var/lib/mysql
      - $PWD/mysql/config:/etc/mysql/conf.d
      - $PWD/mysql/init:/docker-entrypoint-initdb.d
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
      --max_allowed_packet=128M
  mongo:
    container_name: mongo
    image: mongo:4.1
    privileged: true
    restart: always
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=Opensipsrw4!
      - MONGO_INITDB_DATABASE=opensips
      - TZ=Asia/Shanghai
    volumes:
      - $PWD/mongodb/setup/:/docker-entrypoint-initdb.d/
      - $PWD/mongodb/data/db:/data/db
    ports:
      - 27017:27017
    command: mongod --bind_ip 0.0.0.0 --auth --wiredTigerCacheSizeGB 2
    networks:
      sipnetwork:
        ipv4_address: 192.168.1.104
networks:
  sipnetwork:
    external:
      name: sipnet
```

> `mongodb`用户，默认使用`admin`库创建`opensips`库再创建`opensips`用户

#### `opensips`实例：

```yaml
version: '3'
services:
  opensips-1:
    container_name: opensips-1
    image: opensips:v3.3
    privileged: true
    restart: always
    volumes:
      - $PWD/opensips-1/etc/opensips.cfg:/etc/opensips/opensips.cfg
      - $PWD/opensips-1/etc/opensips-cli.cfg:/etc/opensips-cli.cfg
      - $PWD/opensips-1/etc/pi_framework.xml:/usr/share/opensips/pi_http/pi_framework.xml
    network_mode: host
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
  opensips-2:
    container_name: opensips-2
    image: opensips:v3.3
    privileged: true
    restart: always
    volumes:
      - $PWD/opensips-2/etc/opensips.cfg:/etc/opensips/opensips.cfg
      - $PWD/opensips-2/etc/opensips-cli.cfg:/etc/opensips-cli.cfg
      - $PWD/opensips-2/etc/pi_framework.xml:/usr/share/opensips/pi_http/pi_framework.xml
    network_mode: host
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
  opensips-3:
    container_name: opensips-3
    image: opensips:v3.3
    privileged: true
    restart: always
    volumes:
      - $PWD/opensips-3/etc/opensips.cfg:/etc/opensips/opensips.cfg
      - $PWD/opensips-3/etc/opensips-cli.cfg:/etc/opensips-cli.cfg
      - $PWD/opensips-3/etc/pi_framework.xml:/usr/share/opensips/pi_http/pi_framework.xml
    network_mode: host
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
```

##### 端口分配：

> `opensips-1`:	`sip: 10.10.18.20:1060`    `bin:10.10.18.20:5551`
>
> `opensips-2`:	`sip: 10.10.18.20:2060`	`bin:10.10.18.20:5552`
>
> `opensips-3`:	`sip: 10.10.18.20:3060`	`bin:10.10.18.20:5553`

##### `mongo`模块加载：

```bash
#### MongoDB module                                                                                                                                                                                           
loadmodule "cachedb_mongodb.so"
modparam("cachedb_mongodb", "compat_mode_3.0", 1)
```

##### `Clusterer`模块加载：

```
#### Cluster module
loadmodule "clusterer.so"          
modparam("clusterer", "my_node_id", 1)
modparam("clusterer", "seed_fallback_interval", 5)
modparam("clusterer", "my_node_info", "cluster_id=1, url=bin:eth0:5551")
modparam("clusterer", "db_url", "mysql://opensips:Opensipsrw4!@127.0.0.1/opensips")
```

##### `Usrloc`模块加载：

```
#### USeR LOCation module
loadmodule "usrloc.so"
modparam("usrloc", "nat_bflag", "NAT")
modparam("usrloc", "location_cluster", 1)
modparam("usrloc", "cachedb_url", "mongodb:cluster://opensips:Opensipsrw4!@127.0.0.1:27017/opensips.usrloc")
modparam("usrloc", "cluster_mode", "full-sharing-cachedb")
```

#### `keepalived`实例

```bash
yum install -y keepalived
```

##### `keepalived配置`：

> `/etc/keepalived/keepalived.conf`

```json
! Configuration File for keepalived
global_defs {
   router_id opensips_dev
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 130
    priority 160
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {
        10.10.18.25/24 dev eth0
    }
}

virtual_server 10.10.18.25 5060 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol UDP

    real_server 10.10.18.20 1060 {
        weight 1
    }

    real_server 10.10.18.20 2060 {
        weight 1
    }

    real_server 10.10.18.20 3060 {
        weight 1
    }

}
```

> 物理机`IP: 10.10.18.20`, 虚拟`VIP: 10.10.18.25`

### 注册验证：

#### 对外注册端口： 

`10.10.18.25:5060`

#### 软电话注册：

<img src="C:\Users\elegant\AppData\Roaming\Typora\typora-user-images\image-20230210133854695.png" alt="image-20230210133854695" style="zoom:50%;" />

#### `WebRTC`注册：

<img src="C:\Users\elegant\AppData\Roaming\Typora\typora-user-images\image-20230210134244969.png" alt="image-20230210134244969" style="zoom:50%;" />

<img src="C:\Users\elegant\AppData\Roaming\Typora\typora-user-images\image-20230210134300381.png" alt="image-20230210134300381" style="zoom:50%;" />

#### `MongoDB` 验证`Usrloc`数据：

<img src="C:\Users\elegant\AppData\Roaming\Typora\typora-user-images\image-20230210134929395.png" alt="image-20230210134929395" style="zoom:50%;" />