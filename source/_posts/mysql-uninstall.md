---
title: Mac卸载mysql教程
date: 2018-12-28 16:30:09
copyright: true
tags:
 - mysql
categories:
 - mysql
---

{% cq %} 
mysql卸载教程
{% endcq %}
<!-- more -->

### 卸载教程

> mysql没有卸载器，因此我们再mac上安装的mysql需要通过命令行卸载具体的卸载命令如下：
```shell
sudo rm /usr/local/mysql
sudo rm -rf /usr/local/mysql*
sudo rm -rf /Library/StartupItems/MySQLCOM
sudo rm -rf /Library/PreferencePanes/My*
edit /etc/hostconfig (删掉MYSQLCOM=-YES-这一行)
rm -rf ~/Library/PreferencePanes/My*
sudo rm -rf /Library/Receipts/mysql*
sudo rm -rf /Library/Receipts/MySQL*
sudo rm -rf /private/var/db/receipts/*mysql*
```

> 以上mysql卸载完成