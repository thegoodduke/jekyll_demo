---
layout: post
title: "ubuntu 安装指定版本的包"
---

##安装apache2在ubuntu上

首先查看apache2有哪些版本

<pre>
root@ubuntu2:~/downloads# apt-cache policy apache2
apache2:
  Installed: (none)
  Candidate: 2.4.6-2ubuntu2.2
  Version table:
     2.4.6-2ubuntu2.2 0
        500 http://security.ubuntu.com/ubuntu/ saucy-security/main amd64 Packages
        100 /var/lib/dpkg/status
     2.4.6-2ubuntu2 0
        500 http://mirrors.163.com/ubuntu/ saucy/main amd64 Packages
</pre>
可以看到有2.4.6-2ubuntu2.2和2.4.6-2ubuntu2两个版本,而且缺省安装2.4.6-2ubuntu2.2这个版本

使用"=版本号"，就可以安装指定版本了

	apt-get install apache2=2.4.6-2ubuntu2


