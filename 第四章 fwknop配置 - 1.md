# 第四章 fwknop配置 - 1
假定操作环境：Ubuntu系统， 客户端 `spaclient (IP: 1.1.1.1)` ，服务端`spaserver (IP: 2.2.2.2) eth0网口`

## 4.1 配置 HMAC和对称密钥进行SPA认证
服务端配置，隐藏22和993服务

```bash
grep PCAP_INTF /etc/fwknop/fwknopd.conf
PCAP_INTF eth0;

cat /etc/fwknop/access.conf
SOURCE                     ANY
OPEN_PORTS                 tcp/22, tcp/993
REQUIRE_SOURCE_ADDRESS     Y
KEY_BASE64                 Sz80RjpXOlhH2olGuKBUamHKcqyMBsS9BTgLaMugUsg=
HMAC_KEY_BASE64            c0TOaMJ2aVPdYTh4Aa25Dwxni7PrLo2zLAtBoVwSepkvH6nLcW45Cjb9zaEC2SQd03kaaV+Ckx3FhCh5ohNM5Q==

#防火墙配置
[spaserver]# iptables -I INPUT 1 -i eth0 -p tcp --dport 22 -j DROP
[spaserver]# iptables -I INPUT 1 -i eth0 -p tcp --dport 993 -j DROP
[spaserver]# iptables -I INPUT 1 -i eth0 -p tcp -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

最后启动服务
```
客户端敲门，访问端口

```bash
[spaclient]$ fwknop -A tcp/22,tcp/993 -n spaserver.domain.com
[spaclient]$ ssh mbr@spaserver
```


## 4.2 使用HMAC和GPG非对称密钥进行配置
服务端创建证书密钥

```bash
创建密钥
[spaserver]# gpg --gen-key
[spaserver]# gpg --list-keys
pub   1024D/ABCD1234 2012-05-01
uid                  fwknop server key <fwknopd@spaserver>
sub   2048g/EFGH1234 2012-05-01

#导出证书到文件
[spaserver]# gpg -a --export ABCD1234 > server.asc
```
客户端

```bash
#创建密钥
[spaclient]$ gpg --gen-key
[spaclient]$ gpg --list-keys
pub   1024D/1234ABCD 2012-05-01
uid                  fwknop client key <fwknop@spaclient>
sub   2048g/1234EFGH 2012-05-01
 
#导出证书到文件
[spaclient]$ gpg -a --export 1234ABCD > client.asc
```
接下来需要把各自证书给另一方，并在系统上导入对方证书并签名认可

```bash
[spaclient]$ scp client.asc root@spaserver:
[spaclient]$ scp root@spaserver:server.asc 

#服务端
[spaserver]# gpg --import client.asc
[spaserver]# gpg --edit-key 1234ABCD
Command> sign 
 
#客户端
[spaclient]$ gpg --import server.asc
[spaclient]$ gpg --edit-key ABCD1234
Command> sign 

```


接下来配置服务端fwknop配置文件

```bash
[spaserver]# cat /etc/fwknop/access.conf
SOURCE                     ANY
OPEN_PORTS                 tcp/22
REQUIRE_SOURCE_ADDRESS     Y
#客户端证书ID
GPG_REMOTE_ID              1234ABCD;
#服务端证书ID
GPG_DECRYPT_ID             ABCD1234;
#服务端密钥
GPG_DECRYPT_PW             <your decryption password>       ### or set GPG_ALLOW_NO_PW and remove the passphrase
HMAC_KEY_BASE64            c0TOaMJ2aVPdYTh4Aa25Dwxni7PrLo2zLAtBoVwSepkvH6nLcW45Cjb9zaEC2SQd03kaaV+Ckx3FhCh5ohNM5Q==
GPG_HOME_DIR               /root/.gnupg
FW_ACCESS_TIMEOUT          30
 
#重启服务
[spaserver]# service fwknop restart
```


客户端SPA认证操作

注意，如果之前用对称加密认证过，则需要修改 .fwknoprc 文件，去掉对称密钥 KEY\_BASE64

```bash
#使用gpg执行SPA认证，并保存到文件
[spaclient]$ fwknop -A tcp/22 --gpg-recip ABCD1234 --gpg-sign 1234ABCD -a 1.1.1.1 -D spaserver.domain.com --save-rc-stanza

#连服务
[spaclient]$ ssh -l mbr spaserver
```


## 4.3 做service启停操作
在ubuntu下，拷贝脚本到 /etc/init下即可，使用脚本做为service启停的好处是，会有守护进程存在，当fwknop没了，会重新拉起来

centos得用systemd里的脚本

```bash
[spaserver]# cd fwknop.git/extras/upstart
[spaserver]# cp fwknop.conf /etc/init

[spaserver]# service fwknop status
[spaserver]# service fwknop stop
[spaserver]# service fwknop start
```
## 没有4.4
## 4.5 隐藏多服务能力
就是OPEN\_PORTS   支持多端口配置



















