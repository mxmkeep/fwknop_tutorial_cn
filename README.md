# fwknop_tutorial_cn
fwknop(SPA)项目帮助文档的中文阅读笔记

原项目在 https://github.com/mrash/fwknop

原帮助文档在 http://www.cipherdyne.org/fwknop/docs/fwknop-tutorial.html

工作中需要用到SPA，所以在阅读原文档的时候顺便做了笔记，方便快速阅读

另外，此项目源码存在三个问题，使用的话，还需要进行改造
1. 单线程，需改造为多线程提高速度
2. 默认采用iptables规则，性能太低，可以启用ipset替代
3. 保存报文摘要使用文件，量越大越慢，可以改用内存数据库替代
