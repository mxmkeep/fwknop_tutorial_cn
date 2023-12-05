# 第四章 fwknop配置 - 2
## 4.6 多用户处理
本章介绍了如何配置多用户

bob，alice，john三人配置如下

里边提到需要加-a或-R参数，就是把本机IP，或公网IP，放到消息里加密发出去

服务端会针对此IP进行放行，防止中间人攻击(就是中间人截获这消息，再从他那发送，进而放行他的IP，而不是原来客户端的IP)

```cpp
[spaserver]# cat /etc/fwknop/access.conf
 
### for bob   使用对称加密
SOURCE                     ANY
OPEN_PORTS                 tcp/22, tcp/993
REQUIRE_USERNAME           bob
REQUIRE_SOURCE_ADDRESS     Y
FW_ACCESS_TIMEOUT          30
KEY_BASE64                 kgohbCga6D5a4YZ0dtbL8SEVbjI1A5KYrRvj0oqcKEk=
HMAC_KEY_BASE64            Zig9ZYcqj5gYl2S/UpFNp76RlD7SniyN5Ser5WoIKM7zXS28eptWtLcuxCbnh/9R+MjVfUqmqVHqbEyWtHTj4w==
 
### for alice   使用非对称加密
SOURCE                     ANY
GPG_REMOTE_ID              7234ABCD
GPG_DECRYPT_ID             EBCD1234
GPG_ALLOW_NO_PW            Y
REQUIRE_SOURCE_ADDRESS     Y
REQUIRE_USERNAME           alice
FW_ACCESS_TIMEOUT          30
HMAC_KEY_BASE64            STQ9m03hxj+WXwOpxMuNHQkTAx/EtfAKaXQ3tK8+Azcy2zZpimzRzo4+I53cNZvPJaMBfXjZ9NsB98iOpHY7Tg==
 
### for john
SOURCE                     3.3.3.0/24, 4.4.0.0/16
OPEN_PORTS                 tcp/80
REQUIRE_USERNAME           john
REQUIRE_SOURCE_ADDRESS     Y
FW_ACCESS_TIMEOUT          300
KEY_BASE64                 bOx25a5kjXf8/TmNQO1IRD3s/E9iLoPaqUbOv8X4VBA=
HMAC_KEY_BASE64            i0mIhR//1146/T+IMxDVZm1gosNVatvpqpCfkv4X6Xzv4E3SHR6AivCCWk/K/uLDpymyJr95KdEkagfGU4o5yw==
```
按照代码逻辑，是按顺序匹配source配置，前两个是配ANY，所以，程序逻辑是遍历配置，用hmac验证摘要，不符合再拿下一个配置，性能可能不好

第三个john，配置了source，则其外网IP要符合才能命中

对NAT网内多用户的处理，没提及好方式，看着只能遍历或改造



## 4.7 客户端自动化
介绍依靠配置文件 .fwknoprc来预设简化SPA包发送指令，包括保存各种密码在文件里，当然，密码放配置文件可能不安全，如果不放，则需要在执行命令的时候填参数进去。

.fwknoprc文件配置包含两部分

\[default\] section的配置 和 指定的目标IP section配置

可以使用 --save-rc-stanza 参数，将执行的指令参数保存到此文件

```bash
[spaclient]$ fwknop -A tcp/22 -a 1.1.1.1 -D spaserver.domain.com --key-gen --use-hmac --save-rc-stanza
[+] Wrote Rijndael and HMAC keys to rc file: /home/mbr/.fwknoprc

[spaclient]$ cat /home/mbr/.fwknoprc
# .fwknoprc
##############################################################################
#
# Firewall Knock Operator (fwknop) client rc file.
#
# This file contains user-specific fwknop client configuration default
# and named parameter sets for specific invocations of the fwknop client.
#
# Each section (or stanza) is identified and started by a line in this
# file that contains a single identifier surrounded by square brackets.
# It is this identifier (or name) that is used from the fwknop command line
# via the '-n <name>' argument to reference the corresponding stanza.
#
# The parameters within the stanza typically match corresponding client
# command-line parameters.
#
# The first one should always be `[default]' as it defines the global
# default settings for the user. These override the program defaults
# for these parameters. If a named stanza is used, its entries will
# override any of the default values. Command-line options will trump them
# all.
#
# Subsequent stanzas will have only the overriding and destination
# specific parameters.
#
# Lines starting with `#' and empty lines are ignored.
#
# See the fwknop.8 man page for a complete list of valid parameters
# and their values.
#
##############################################################################
#
# We start with the 'default' stanza. Uncomment and edit for your
# preferences. The client will use its built-in default for those items
# that are commented out.
#

[default]
#DIGEST_TYPE         sha256
#FW_TIMEOUT          30
#SPA_SERVER_PORT     62201
#SPA_SERVER_PROTO    udp
#ALLOW_IP            <ip addr>
#SPOOF_USER          <username>
#SPOOF_SOURCE_IP     <IPaddr>
#TIME_OFFSET         0
#USE_GPG             N
#GPG_HOMEDIR         /path/to/.gnupg
#GPG_SIGNER          <signer ID>
#GPG_RECIPIENT       <recipient ID>
## User-provided named stanzas:
## Example for a destination server of 192.168.1.20 to open access to
# SSH for an IP that is resolved externally, and one with a NAT request
# for a specific source IP that maps port 8088 on the server
# to port 88 on 192.168.1.55 with timeout.
#
#[myssh]
#SPA_SERVER          192.168.1.20
#ACCESS              tcp/22
#ALLOW_IP            resolve
#
#[mynatreq]
#SPA_SERVER          192.168.1.20
#ACCESS              tcp/8088
#ALLOW_IP            10.21.2.6
#NAT_ACCESS          192.168.1.55,88
#CLIENT_TIMEOUT      60

[spaserver.domain.com]
ACCESS                      tcp/22
ALLOW_IP                    1.1.1.1
SPA_SERVER                  spaserver.domain.com
KEY_BASE64                  Sz80RjpXOlhH2olGuKBUamHKcqyMBsS9BTgLaMugUsg=
HMAC_KEY_BASE64             c0TOaMJ2aVPdYTh4Aa25Dwxni7PrLo2zLAtBoVwSepkvH6nLcW45Cjb9zaEC2SQd03kaaV+Ckx3FhCh5ohNM5Q==
USE_HMAC                    Y
```


## 4.8 SPA加NAT网关
介绍使用SPA包，让服务端配置NAT网关，以便访问服务端后面的内网服务

例如访问 spaserver 后面的内网服务 10.2.1.10，服务端配置如下

```bash
#启用NAT转发
[spaserver]# grep ENABLE_IPT_FORWARDING /etc/fwknop/fwknopd.conf
ENABLE_IPT_FORWARDING      Y

#重启服务
[spaserver]# service fwknop restart
```


客户端在执行SPA发送命令时，加参数 -N

```bash
[spaclient]$ fwknop -N 10.2.1.10:22 -n spaserver.domain.com

[spaclient]$ ssh -l mbr spaserver
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
```


服务端可以查看下iptables规则，自动添加了DNAT规则

```bash
[spaserver]# fwknopd --fw-list
Listing rules in fwknopd iptables chains...
 
Chain FWKNOP_INPUT (1 references)
num  target     prot opt source               destination
 
Chain FWKNOP_FORWARD (1 references)
num  target     prot opt source               destination
1    ACCEPT     tcp  --  1.1.1.1              10.1.2.10            tcp dpt:22 /* _exp_1349634445 */
 
Chain FWKNOP_PREROUTING (1 references)
num  target     prot opt source               destination
1    DNAT       tcp  --  1.1.1.1              0.0.0.0/0            tcp dpt:22 /* _exp_1349634445 */ to:10.1.2.10:22
```


### 4.8.1 SPA Ghost services  幽灵服务
介绍了什么是幽灵服务，例如，用Apache在spaserver上起一个80端口服务，平时访问spaserver的80端口，会是一个http服务，返回http页面，但如果用SPA+NAT方式，给敲门的IP设置NAT转换到内网的22服务，那么客户端访问80端口，则会真实访问到内网主机的22 ssh服务。

```bash
#客户端敲门指令，对目标80端口做NAT转换
[spaclient]$ fwknop -A tcp/80 -N 10.2.1.10:22 -n spaserver.domain.com

#服务端iptables规则
[spaserver]# fwknopd --fw-list
Listing rules in fwknopd iptables chains...
 
Chain FWKNOP_INPUT (1 references)
num  target     prot opt source               destination
 
Chain FWKNOP_FORWARD (1 references)
num  target     prot opt source               destination
1    ACCEPT     tcp  --  1.1.1.1              10.1.2.10            tcp dpt:22 /* _exp_1349638012 */
 
Chain FWKNOP_PREROUTING (1 references)
num  target     prot opt source               destination
1    DNAT       tcp  --  1.1.1.1              0.0.0.0/0            tcp dpt:80 /* _exp_1349638012 */ to:10.1.2.10:22
```


## 4.9 用户界面
介绍了客户端在各平台上的界面和设置

![image](https://github.com/mxmkeep/fwknop_tutorial_cn/assets/20048552/14845209-1ab5-44e2-9333-35af530a5194)


![image](https://github.com/mxmkeep/fwknop_tutorial_cn/assets/20048552/d7275406-932f-4cf1-8f35-3cd01e80d0a7)


跨平台界面开源项目 ： [https://github.com/jp-bennett/fwknop-gui](https://github.com/jp-bennett/fwknop-gui)



## 4.10 SPA数据包欺骗
介绍如何使用客户端发送伪造来源IP的SPA

```bash
#用原始socket，发送伪造来源IP为4.4.4.4的SPA包
#服务端看到的来源IP为4.4.4.4，而不是真实IP
[spaclient]$ fwknop -P udpraw -Q 4.4.4.4 -n spaserver.domain.com
```
服务端可以启用 `REQUIRE_SOURCE_ADDRESS 参数，来强制要求填写IP到加密数据包里`

客户端则需要使用 -a 参数来填写来源IP



## 4.11 阻止包重放攻击
如果攻击者在网络上获取到了SPA包，那就可以进行重放攻击

客户端参数 --verbose 可以把发送包的数据打印出来，然后可以用简单的命令进行重放

```bash
[attacker]$ echo -n "8rqjaO4SdW4Yaf1grhi0f3pcdcWO6ZpRxnduxHwJ/5zcqne0VCrZ8oibPYiwazGCOJ7HhGy+f2ju2mVz//DnIr4qKYf48gqe0QJz14Y6Z7BQCYdJdwyYz+hTo9Z41tOA788VcK7sK2XJjdPQL0C794QqEFAqCOyno1VTsf4E2vlPP7Kum94SfyRJzAYwl86LVMQtt7H0/zP0" | nc -u spaserver.domain.com 62201
```
spaserver会保存请求包的摘要，从而对后续进入的包进行比对，看是否为重放攻击

















































