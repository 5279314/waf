## 项目背景介绍

### 需求产生

由于原生态的Nginx的一些安全防护功能有限，就研究能不能自己编写一个WAF，参考Kindle大神的ngx_lua_waf，自己尝试写一个了，使用两天时间，边学Lua，边写。不过不是安全专业，只实现了一些比较简单的功能：

### 功能列表：

1.	支持IP白名单和黑名单功能，直接将黑名单的IP访问拒绝。
2.	支持URL白名单，将不需要过滤的URL进行定义。
3.	支持User-Agent的过滤，匹配自定义规则中的条目，然后进行处理（返回403）。
4.	支持CC攻击防护，单个URL指定时间的访问次数，超过设定值，直接返回403。
5.	支持Cookie过滤，匹配自定义规则中的条目，然后进行处理（返回403）。
6.	支持URL过滤，匹配自定义规则中的条目，如果用户请求的URL包含这些，返回403。
7.	支持URL参数过滤，原理同上。
8.	支持日志记录，将所有拒绝的操作，记录到日志中去。
9.	日志记录为JSON格式，便于日志分析，例如使用ELK进行攻击日志收集、存储、搜索和展示。项目背景介绍

### WAF实现

WAF一句话描述，就是解析HTTP请求（协议解析模块），规则检测（规则模块），做不同的防御动作（动作模块），并将防御过程（日志模块）记录下来。所以本文中的WAF的实现由五个模块(配置模块、协议解析模块、规则模块、动作模块、错误处理模块）组成。

## 安装部署

以下方案选择其中之一即可：

- 选择1: 可以选择使用原生的Nginx，增加Lua模块实现部署。
- 选择2: 直接使用OpenResty

### Nginx + Lua源码编译部署

#### 准备必备Nginx环境

```shell
mkdir /server/tools -p
cd /server/tools/
wget http://nginx.org/download/nginx-1.20.1.tar.gz
yum -y install openssl-devel pcre-devel
useradd -s /sbin/nolgin -M -u 1010 www
#其次，下载当前最新的luajit和ngx_devel_kit (NDK)，以及春哥（章）编写的lua-nginx-module
wget http://luajit.org/download/LuaJIT-2.0.5.tar.gz
wget https://github.com/vision5/ngx_devel_kit/archive/refs/tags/v0.3.1.tar.gz
wget https://github.com/openresty/lua-nginx-module/archive/refs/tags/v0.10.10.zip
```

#### 解压NDK和lua-nginx-module

```shell
tar zxvf v0.3.1.tar.gz
unzip -q v0.10.10.zip
```

#### 安装LuaJIT

```shell
tar zxvf LuaJIT-2.0.5.tar.gz 
cd LuaJIT-2.0.5
make && make install
```

#### 安装Nginx并加载模块

```shell
tools]# tar xf nginx-1.20.1.tar.gz
tools]# cd nginx-1.20.1/
nginx-1.20.1]# export LUAJIT_LIB=/usr/local/lib
nginx-1.20.1]# export LUAJIT_INC=/usr/local/include/luajit-2.0
nginx-1.20.1]# ./configure --user=www --group=www --prefix=/application/nginx-1.20 --with-http_stub_status_module --with-http_ssl_module --with-http_sub_module --with-http_gzip_static_module --add-module=/server/tools/ngx_devel_kit-0.3.1/ --add-module=/server/tools/lua-nginx-module-0.10.10
nginx-1.20.1]# make && make install
```

**报错：**

```shell
如果不创建符号链接，可能出现以下异常：
error while loading shared libraries: libluajit-5.1.so.2: cannot open shared object file: No such file or directory
nginx-1.20.1]# ln -s /usr/local/lib/libluajit-5.1.so.2 /lib64/libluajit-5.1.so.2
```

#### 测试安装

安装完毕后，下面可以测试安装了，修改nginx.conf 增加第一个配置。

```shell
        location /hello {
                default_type 'text/plain';
                content_by_lua 'ngx.say("hello,lua")';
        }
```

#### 启动nginx

```shell
]# /application/nginx-1.20/sbin/nginx -t
]# /application/nginx-1.20/sbin/nginx
```



#### 浏览器访问

然后访问http://10.1.1.10/hello 如果出现hello,lua。表示安装完成,然后就可以。

![](https://images.boysec.cn/image-20210911224244895.png)

### WAF部署

```shell
git clone https://github.com/5279314/waf.git
cp -r ./waf/waf /application/nginx-1.20/conf/
vim /application/nginx-1.20/conf/nginx.conf

#在http{}中增加，注意路径，同时WAF日志默认存放在/tmp/日期_waf.log
#WAF配置
    lua_shared_dict limit 50m;
    lua_package_path "/application/nginx-1.20/conf/waf/?.lua";
    init_by_lua_file "/application/nginx-1.20/conf/waf/init.lua";
    access_by_lua_file "/application/nginx-1.20/conf/waf/access.lua";
```

#### 重启

```shell
]# /application/nginx-1.20/sbin/nginx -t
]# /application/nginx-1.20/sbin/nginx -s reload
```
### 实现效果

![](https://images.boysec.cn/image-20210911225715210.png)

攻击日志存放目录
```shell
cat /tmp/2022-08-15_waf.log 
{"user_agent":"python-httpx\/0.23.0","rule_tag":"(HTTrack|harvest|audit|dirbuster|pangolin|nmap|sqln|-scan|hydra|Parser|libwww|BBBike|sqlmap|w3af|owasp|Nikto|fimap|havij|PycURL|zmeu|BabyKrokodil|netsparker|httperf|bench|python)","req_url":"\/","client_ip":"42.194.241.91","local_time":"2022-08-15 00:05:36","attack_method":"Deny_USER_AGENT","req_data":"-","server_name":"boysec.cn"}
{"user_agent":"python-requests\/2.26.0","rule_tag":"(HTTrack|harvest|audit|dirbuster|pangolin|nmap|sqln|-scan|hydra|Parser|libwww|BBBike|sqlmap|w3af|owasp|Nikto|fimap|havij|PycURL|zmeu|BabyKrokodil|netsparker|httperf|bench|python)","req_url":"\/","client_ip":"94.102.61.8","local_time":"2022-08-15 03:43:38","attack_method":"Deny_USER_AGENT","req_data":"-","server_name":"boysec.cn"}
{"user_agent":"Mozilla\/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit\/537.36 (KHTML, like Gecko) Chrome\/99.0.4844.51 Safari\/537.36","rule_tag":"-","req_url":"\/","client_ip":"93.212.150.104","local_time":"2022-08-15 03:59:25","attack_method":"CC_Attack","req_data":"-","server_name":"boysec.cn"}
{"user_agent":"Mozilla\/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit\/537.36 (KHTML, like Gecko) Chrome\/99.0.4844.51 Safari\/537.36","rule_tag":"-","req_url":"\/","client_ip":"93.212.150.104","local_time":"2022-08-15 03:59:26","attack_method":"CC_Attack","req_data":"-","server_name":"boysec.cn"}
{"user_agent":"Mozilla\/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit\/537.36 (KHTML, like Gecko) Chrome\/99.0.4844.51 Safari\/537.36","rule_tag":"-","req_url":"\/","client_ip":"93.212.150.104","local_time":"2022-08-15 03:59:26","attack_method":"CC_Attack","req_data":"-","server_name":"boysec.cn"}
{"user_agent":"Mozilla\/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit\/537.36 (KHTML, like Gecko) Chrome\/99.0.4844.51 Safari\/537.36","rule_tag":"-","req_url":"\/","client_ip":"93.212.150.104","local_time":"2022-08-15 03:59:27","attack_method":"CC_Attack","req_data":"-","server_name":"boysec.cn"}
```

### OpenResty安装

#### Yum安装OpenResty（推荐）

源码安装和Yum安装选择其一即可，默认均安装在/usr/local/openresty目录下。

```
[root@opsany ~]# vim /etc/yum.repo.d/openresty.repo
[openresty]
name=OfficialOpenRestyOpenSourceRepositoryforCentOS
baseurl=https://openresty.org/package/centos/$releasever/$basearch
skip_if_unavailable=False
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://openresty.org/package/pubkey.gpg
enabled=1
enabled_metadata=1

[root@opsany ~]# sudo yum install -y openresty
```

#### 测试OpenResty和运行Lua

```
[root@opsany ~]# vim /usr/local/openresty/nginx/conf/nginx.conf
#在默认的server配置中增加
        location /hello {
            default_type text/html;
            content_by_lua_block {
                ngx.say("<p>hello, world</p>")
            }
        }
[root@opsany ~]# /usr/local/openresty/nginx/sbin/nginx -t
nginx: the configuration file /usr/local/openresty-1.19.9.1/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/openresty-1.19.9.1/nginx/conf/nginx.conf test is successful
[root@opsany ~]# /usr/local/openresty/nginx/sbin/nginx
