# Cetus 读写分离版本管理手册

## 前言

**有配置修改均能动态生效，配置更改后请务必修改原始配置文件，以确保下次重启时配置能够保留。**

## 查看帮助

`SELECT * FROM help`或
`select help`

查看管理端口用法

| Command                                  | Description                              |
| :--------------------------------------- | :--------------------------------------- |
| select conn_details from backend         | display the idle conns                   |
| select * from backends                   | list the backends and their state        |
| show connectionlist [\<num>]             | show \<num> connections                  |
| show allow_ip \<module>                  | show allow_ip rules of module, currently admin |
| add allow_ip \<module> \<address>        | add address to white list of module      |
| delete allow_ip \<module> \<address>     | delete address from white list of module |
| set reduce_conns (true\|false)           | reduce idle connections if set to true   |
| set maintain (true\|false)               | close all client connections if set to true |
| show status [like '%pattern%']           | show select/update/insert/delete statistics |
| show variables [like '%pattern%']        |                                          |
| select version                           | cetus version                            |
| select conn_num from backends where backend_ndx=\<index> and user='\<name>') |                                          |
| select * from user_pwd [where user='\<name>'] |                                          |
| select * from app_user_pwd [where user='\<name>'] |                                          |
| update user_pwd set password='xx' where user='\<name>' |                                          |
| update app_user_pwd set password='xx' where user='\<name>' |                                          |
| delete from user_pwd where user='\<name>' |                                          |
| delete from app_user_pwd where user='\<name>' |                                          |
| insert into backends values ('\<ip:port>', '(ro\|rw)', '\<state>') | add mysql instance to backends list      |
| update backends set (type\|state)=x where (backend_ndx=\<index>\|address=\<'ip:port'>) | update mysql instance type or state      |
| delete from backends where (backend_ndx=\<index>\|address=\<'ip:port'>) |                                          |
| remove backend where (backend_ndx=\<index>\|address=\<'ip:port'>) |                                          |
| add master \<'ip:port'>                  |                                          |
| add slave \<'ip:port'>                   |                                          |
| stats get [\<item>]                      | show query statistics                    |
| config get [\<item>]                     | show config                              |
| config set \<key>=\<value>               |                                          |
| stats reset                              | reset query statistics                   |
| save settings                            | not implemented                          |
| select * from help                       | show this help                           |
| select help                              | show this help                           |
| cetus                                    | Show overall status of Cetus             |

结果说明：

读写分离版本管理端口提供了32条语句对cetus进行管理，具体用法见以下说明。

## 后端配置

### 查看后端

`SELECT * FROM backends`

查看后端信息。

| backend_ndx | address        | state | type | slave delay | uuid | idle_conns | used_conns | total_conns |
| :---------- | :------------- | :---- | :--- | :---------- | :--- | :--------- | :--------- | :---------- |
| 1           | 127.0.0.1:3306 | up    | rw   | NULL        | NULL | 100        | 0          | 100         |
| 2           | 127.0.0.1:3307 | up    | ro   | 0           | NULL | 100        | 0          | 100         |

结果说明：

* backend_ndx: 后端序号，按照添加顺序排列；
* address: 后端地址，IP:PORT格式；
* state: 后端状态(unknown|up|down|maintaining|delete)；
* type: 读写类型(rw|ro)；
* slave delay: 主从延迟时间(单位：毫秒)；
* uuid: 暂时无用；
* idle_conns: 空闲连接数；
* used_conns: 正在使用的连接数；
* total_conns: 总连接数。

```
状态说明
unknown:     后端初始状态，还未建立连接;
up:          能与后端正常建立连接；
down:        与后端无法联通(如果开启后端状态检测，能连通后自动变为UP);
maintaining: 后端正在维护，无法建立连接或自动切换状态(此状态由管理员手动设置);
delete:      后端已被删除，无法再建立连接。
```

### 查看后端连接状态

`SELECT CONN_DETAILS FROM backends`

查看每个用户占用和空闲的后端连接数。

| backend_ndx | username | idle_conns | used_used_conns | total_used_conns |
| :---------- | :------- | :--------- | :-------------- | ---------------- |
| 1           | test1    | 2          | 0               | 0                |
| 2           | test2    | 11         | 0               | 0                |

结果说明：

* backend_ndx: 后端序号；
* username: 用户名；
* idle_conns: 空闲连接数；
* used_used_conns：正在使用的连接数。
* total_used_conns: 总的连接数。

### 添加后端

`ADD MASTER '<ip:port>'`

添加一个读写类型的后端。

例如

>add master '127.0.0.1:3307'

`ADD SLAVE '<ip:port>'`

添加一个只读类型的后端。

例如

>add slave '127.0.0.1:3360'

`INSERT INTO backends VALUES ('<ip:port>', '(ro|rw)', '<state>')`

添加一个后端，同时指定读写类型。

例如

>insert into backends values ('127.0.0.1:3306', 'rw', 'up');

### 删除后端

`REMOVE BACKEND <backend_ndx>` 或
`DELETE FROM BACKENDS where backend_ndx = <backend_ndx>`

删除一个指定序号的后端。

例如

>remove backend 1

`DELETE FROM BACKENDS where address = '<ip:port>'`

删除一个指定地址的后端。

例如

>delete from backends where address = '127.0.0.1:3306'

### 修改后端

`UPDATE BACKENDS SET (type|state)=<value> WHERE (backend_ndx=<index>|address=<'ip:port'>)`

修改后端类型或状态。

例如

>update backends set type="rw" where address="127.0.0.1:3306"

>update backends set state="up" where backend_ndx=1

## 基本配置

### 查看连接池/通用配置

`config get [<item>]`

`config get`查看支持的配置类型
   * `pool`连接池配置
   * `common`通用配置

`config get common`查看通用配置
   * `common.check_slave_delay` 是否需要检测从库延迟
   * `common.slave_delay_down_threshold_sec` 若延迟大于此值(秒)，后端状态置为DOWN
   * `common.slave_delay_recover_threshold_sec` 若延迟小于此值(秒)，后端状态置为UP

`config get pool`查看连接池配置
   * `pool.default_pool_size` 默认连接池大小
   * `pool.max_pool_size` 最大连接数量
   * `pool.max_resp_len` 最大结果集长度
   * `pool.master_preferred` 是否只允许走主库

### 修改连接池/通用配置

`config set [<item>]`

`config set common.[option] = [value]`修改基本配置

例如

>config set common.slave_delay_down = 3

`config set pool.[option] = [value]`修改连接池配置

例如

>config set pool.max_pool_size = 200

### 查看参数配置

`show variables [like '%pattern%']`

查看的参数均为启动配置选项中的参数，详见[Cetus 启动配置选项说明](https://github.com/Lede-Inc/cetus/blob/master/doc/cetus-configuration.md)。

## 查看/设置连接信息

### 查看当前连接的详细信息

`SHOW CONNECTIONLIST`

将当前全部连接的详细内容按表格显示出来。

| User  | Host           | db   | Command | Time | Trans | PS   | State      | Server | Info |
| ----- | -------------- | ---- | ------- | ---- | ----- | ---- | ---------- | ------ | ---- |
| test1 | 127.0.0.1:3306 | test | Sleep   | 0    | N     | N    | READ_QUERY | NULL   | NULL |
| test2 | 127.0.0.1:3307 | test | Sleep   | 0    | N     | N    | READ_QUERY | NULL   | NULL |

结果说明：

* User: 用户名;
* Host: 客户端的IP和端口;
* db: 数据库名称;
* Command: 执行的sql，"Sleep"代表当前空闲;
* Time: 已执行的时间;
* Trans: 是否在事务中;
* PS：是否存在prepare;
* State: 连接当前的状态，"READ_QUERY"代表在等待获取命令;
* Server: 后端地址;
* Info: 暂未知。

### 查看某用户对某后端的连接数

`select conn_num from backends where backend_ndx=<index> and user='<name>')`

例如

>select conn_num from backends where backend_ndx=2 and user='root');

### 设置是否减少空闲连接

`set reduce_conns (true|false)`

例如

>set reduce_conns true;

减少空闲连接。

### 设置是否关闭所有客户端连接

`set maintain (true|false)`

例如

>set maintain true;

关闭所有客户端连接。

## 用户/密码管理

### 密码查询

`SELECT * FROM user_pwd [where user='<name>']`

查询某个用户的后端密码。
**注意由于密码是非明文的，仅能显示字节码。**

>select * from user_pwd where user="root";

`SELECT * FROM app_user_pwd [where user='<name>']`

查询某个用户连接proxy的密码，同样是非明文。

例如

>select * from app_user_pwd where user="test";

### 密码添加/修改

`UPDATE user_pwd SET password='<password>' where user='<name>'`

添加或修改特定用户的后端密码(如果该用户不存在则添加，已存在则覆盖)。

例如

>update user_pwd set password="123456" where user="test"

`UPDATE app_user_pwd SET password='<password>' where user='<name>'`

添加或修改特定用户连接Proxy的密码(如果该用户不存在则添加，已存在则覆盖)。

例如

>update app_user_pwd set password="123456" where user="root"

### 密码删除

`DELETE FROM user_pwd where user='<name>'`

删除特定用户的后端密码。

例如

>delete from user_pwd where user="root"

`DELETE FROM app_user_pwd where user='<name>'`

删除特定用户连接Proxy的密码。

例如

>delete from app_user_pwd where user="root"

## IP白名单

### 查看IP白名单

`SHOW ALLOW_IP <module>`

\<module\>：admin|proxy

查看admin／proxy模块的IP白名单。
若列表为空，则代表没有任何限制。

### 增加IP白名单

`ADD ALLOW_IP <module> <address>`

向白名单增加一个IP许可。（IP不要加引号）

\<module\>：admin|proxy

\<address\>：[[user@]IP]

```
说明
Admin: 仅能配置IP，不能限制用户(Admin有效用户只有一个)；
Proxy: 仅配置IP，代表允许该IP来源所有用户的访问；配置User@IP，代表允许该IP来源的特定用户访问。
```

例如

>add allow_ip admin 127.0.0.1

>add allow_ip proxy test@127.0.0.1

### 删除IP白名单

`DELETE ALLOW_IP <module> <address>`

删除白名单中的一个IP许可。（IP不要加引号）

\<module\>：admin|proxy

\<address\>：[[user@]IP]

例如

>delete allow_ip admin 127.0.0.1

>delete allow_ip proxy test@127.0.0.1

## IP黑名单

### 查看IP黑名单

`SHOW DENY_IP <module>`

\<module\>：admin|proxy

查看admin／proxy模块的IP黑名单。
若列表为空，则代表没有任何限制。

### 增加IP黑名单

`ADD DENY_IP <module> <address>`

向黑名单中增加一个IP限制。（IP不要加引号）

\<module\>：admin|proxy

\<address\>：[[user@]IP]

```
说明
Admin: 仅能配置IP，不能限制用户(Admin有效用户只有一个)；
Proxy: 仅配置IP，代表限制该IP来源所有用户的访问；配置User@IP，代表限制该IP来源的特定用户访问。
```

例如

>add deny_ip admin 127.0.0.1

>add deny_ip proxy test@127.0.0.1

### 删除IP黑名单

`DELETE DENY_IP <module> <address>`

删除黑名单中的一个IP限制。（IP不要加引号）

\<module\>：admin|proxy

\<address\>：[[user@]IP]

例如

>delete deny_ip admin 127.0.0.1

>delete deny_ip proxy test@127.0.0.1

**注意：IP白名单的优先级高于IP黑名单**

### 保存配置到本地文件

`SAVE SETTINGS [FILE]`

保存当前配置到指定路径的本地文件中。

例如

>save settings /tmp/proxy.cnf

## 查看整体信息

### 查看统计信息

`stats get [<item>]`

`stats get`查看支持的统计类型
   * `client_query` 客户发来的SQL数量
   * `proxyed_query` 发往后端的SQL数量
   * `query_time_table` 查询时间直方图
   * `server_query_details` 每个后端接收的SQL数量
   * `query_wait_table` 等待时间直方图

`stats get client_query` `stats get proxyed_query`查看读/写SQL数量

`stats get server_query_details`查看各个后端读/写SQL数量

`stats get query_time_table` `stats get query_wait_table` 查看各时间值对应的SQL数量，如：

| name               | value |
| :----------------- | :---- |
| query_time_table.1 | 3     |
| query_time_table.2 | 5     |
| query_time_table.5 | 1     |

表示用时1秒的SQL有3条，用时2秒的SQL有5条，用时5秒的SQL有1条

```
说明
stats reset：重置统计信息 
```

### 查看总体状态

`cetus`

包括程序版本、连接数量、QPS、TPS等信息

### 查看各类SQL统计

`show status [like '%pattern%']`

```
参数说明
Com_select         总的SELECT数量
Com_insert         总的INSERT数量
Com_update         总的UPDATE数量
Com_delete         总的DELETE数量
Com_select_shard   走多个节点的SELECT数量
Com_insert_shard   走多个节点的INSERT数量
Com_update_shard   走多个节点的UPDATE数量
Com_delete_shard   走多个节点的DELETE数量
Com_select_gobal   仅涉及公共表的SELECT数量
Com_select_bad_key 分库键未识别导致走全库的SELECT数量
```

### 查看当前cetus版本

`select version`
