# 第四章 fwknop配置 - 3
## 4.12 SPA over Tor
介绍了如何使用匿名Tor，通过TCP发送SPA包，Tor是一种匿名协议，通过各种方式来隐藏，也有浏览器。

使用tcp发SPA，那服务端也需要起tcp服务，在/etc/fwknop/fwknopd.conf 里配置 ENABLE\_TCP\_SERVER Y;  参数 `TCPSERV_PORT 可以指定tcp端口`

客户端操作

```cpp
#配置好客户端和服务端后
[spaclient]$ fwknop -n spa.tor -A tcp/22 -a 1.1.1.1 -D 127.0.0.1 -P tcp --key-gen --use-hmac --save-rc-stanza

#建立代理TCP链接，打通到SPA服务2.2.2.2
[spaclient]$ socat TCP-LISTEN:62201 SOCKS4A:127.0.0.1:2.2.2.2:62201,socksport=9050 &

#敲门
[spaclient]$ fwknop -n spa.tor

```
服务端日志

```cpp
Jun 22 23:11:56 spaserver fwknopd[12624]: (stanza #1) SPA Packet from IP: 96.47.N.N received with access source match
Jun 22 23:11:56 spaserver fwknopd[12624]: Added Rule to FWKNOP_INPUT for 1.1.1.1, tcp/22 expires at 1349403326
```
可以看到，tcp链接到服务的IP为96.47.N.N，这是tor网络的一个出口路由，并不是真实客户端IP。

但服务端放行的是SPA包指定的1.1.1.1 IP，需要从此IP访问22服务



## 4.13 SPA with ipset
介绍2.6.8 版本后，spa服务端如何支持ipset功能

增加了支持SPA验证通过后执行自定义指令的功能，也就是可以在SPA验证通过后，配置自定义的ipset命令

服务端配置：创建ipset，增加iptables规则

```bash
[spaserver]# ipset create fwknop_allow hash:ip,port timeout 30
[spaserver]# iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
[spaserver]# iptables -A INPUT -m set --match-set fwknop_allow src,dst -j ACCEPT
[spaserver]# iptables -A INPUT -j DROP
```
然后配置 */etc/fwknop/access.conf*

```bash
[spaserver]# cat /etc/fwknop/access.conf
SOURCE            ANY
KEY_BASE64        <base64 string>
HMAC_KEY_BASE64   <base64 string>
CMD_CYCLE_OPEN    ipset add fwknop_allow $SRC,$PROTO:$PORT timeout $CLIENT_TIMEOUT
CMD_CYCLE_CLOSE   NONE

[spaserver]# service fwknopd start
```
客户端敲门配置和指令不变

```bash
[spaclient]$ fwknop -A tcp/22 -a 1.1.1.1 -f 30 -n spaserver
[spaclient]$ ssh user@spaserver
```
服务端执行SPA操作后，会增加IP到ipset里

```bash
[spaserver]# grep fwknopd /var/log/syslog |tail -n 2
Dec 23 15:38:06 ubuntu fwknopd[13537]: (stanza #1) SPA Packet from IP: 1.2.3.4 received with access source match
Dec 23 15:38:06 ubuntu fwknopd[13537]: [1.2.3.4] (stanza #1) Running CMD_CYCLE_OPEN command: /sbin/ipset add fwknop_allow 1.1.1.1,6:22 timeout 30

[spaserver]# ipset list
Name: fwknop_allow
Type: hash:ip,port
Revision: 5
Header: family inet hashsize 1024 maxelem 65536 timeout 30
Size in memory: 224
References: 0
Members:
1.1.1.1,tcp:22 timeout 27
```
## 4.14 向后兼容
介绍如何兼容2.5及以前旧版本的加密问题

服务端配置指令 ENCRYPTION\_MODE          legacy

客户端加参数 -M

```bash
[spaclient]$ fwknop -A tcp/22 -M legacy -a 1.1.1.1 -D spaserver.domain.com
```






