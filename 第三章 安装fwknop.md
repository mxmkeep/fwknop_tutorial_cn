# 第三章 安装fwknop
原文地址 [https://www.cipherdyne.org/fwknop/docs/fwknop-tutorial.html](https://www.cipherdyne.org/fwknop/docs/fwknop-tutorial.html)



### 3.1 介绍依赖库：
一个是此项目自己生成的libfko库。另一个是libpcap库，默认采用此库来获取UDP包。当然，也可以不使用pcap，而是直接起UDP监听服务。虽然UDP监听服务会绑定一个UDP端口，但两种模式都不会对SPA包进行响应，所以黑客也扫不到此端口。

另外，如果使用GPG验证功能，则需要安装libgpgme库。iptables或ipfw或pf防火墙也要安装。

![image](images/f1sC_OUXhRN4Td8sP-5D2iVNnEdQEfWDsYICe5sP3B0.png)

### 3.2 源码下载
地址 [http://www.cipherdyne.org/fwknop/download/](https://www.cipherdyne.org/fwknop/download/) .

### 3.3 支持平台列表
![image](images/HIZREMrnsvXlg6Ut6BoQA_YqgR88R4JShZLhvxAdIiw.png)

### 3.4 介绍各平台注意事项
具体事项，有需要看原文

### 3.5 介绍如何从源码安装
```bash
#下载安装包
wget http://www.cipherdyne.org/fwknop/download/fwknop-2.6.8.tar.gz
#如果想验证签名，则下载这个，需要先导入签名密钥 
#  https://www.cipherdyne.org/signing_key    gpg --import <key file>
wget http://www.cipherdyne.org/fwknop/download/fwknop-2.6.8.tar.gz.asc
 gpg --verify fwknop-2.6.8.tar.gz.asc

#编译安装
tar xfz fwknop-2.6.8.tar.gz
cd fwknop-2.6.8
 ./configure --prefix=/usr --sysconfdir=/etc && make
make install
fwknop -V
```
Ubuntu直接可以用apt安装

```bash
apt-get install fwknop-client
apt-get install fwknop-server
```


### 3.6 介绍程序的测试套件
在make install之前，可以用test测试有没有编译正确

```bash
cd fwknop-2.6.8/test/
./test-fwknop.pl --enable-all

如果有失败，可以用以下命令看看
./test-fwknop.pl --diff

#支持valgrind工具做内存泄漏检测
./test-fwknop.pl --enable-valgrind
```
如果进行代码开发，开发之前先跑一遍检测，开发后再跑一遍进行对比

```bash
[spaserver]# ./test-fwknop.pl --enable-recompile --enable-valgrind

--- make code changes, and then:
 
[spaserver]# ./test-fwknop.pl --enable-recompile --enable-valgrind
[spaserver]# ./test-fwknop.pl --diff
```




















