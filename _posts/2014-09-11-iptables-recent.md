---
layout: post
title: "iptables模块介绍：recent"
---

##简介
* recent模块可以看作iptables里面维护了一个地址列表.这个地址列表可以通过”–set”、”–update”、”–rcheck”、”–remove”四种方法来修改或检测列表，每次使用时只能选用一种。
* ”–name”参数来指定列表的名字（默认为DEFAULT）
* “–rsource”、“–rdest”指示当前方法应用到数据包的源地址还是目的地址（默认是前者）。
四个基本方法的作用：
* –set将地址添加进列表，并更新信息，包含地址加入的时间戳。–rcheck检查地址是否在列表。–update跟rcheck一样，但会刷新时间戳。–remove就是在列表里删除地址，如果要删除的地址不存在就会返回假。

##例子1
限制无法ssh直接连接服务器，需先用较大包ping一下，此时在15秒内才可以连接上：

	iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
	iptables -A INPUT -p icmp --icmp-type 8 -m length --length 128 -m recent --set --name SSHOPEN --rsource -j ACCEPT
	iptables -A INPUT -p icmp --icmp-type 8 -j ACCEPT
	iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --rcheck --seconds 15 --name SSHOPEN --rsource -j ACCEPT
	iptables -A INPUT -j DROP

1. 对于已建立连接或是与已连接相关的包都接受，服务器对外连接回来的包一般都走这条；基本环境已经配好了，现在开始要为连接到服务器的ssh打开通路。
2. icmp类型8是ping包；指定包大小为128字节；recent用的列表名称为SSHOPEN，列表记录源地址。符合上述条件的数据包都接收。如果ping包内容为100字节，则加上IP头、ICMP头的28字节，总共128字节。
3. 接受一般的ping包；
4. 对连接ssh 22端口的连接进行处理，来源于SSHOPEN源地址列表并且在列表时间小于等于15秒的才放行。
5. 将INPUT链默认策略置为DROP，当包走完INPUT链而没被拿走时就会丢弃掉；

测试：无法ssh直接连接服务器，使用一下命令后，15秒内就可以ssh连接服务器了。

	ping -s 100 服务器ip

##例子2

对连接到本机的SSH连接进行限制，每个IP每小时只限连接5次

	iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
	iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --name SSHPOOL --rcheck --seconds 3600 --hitcount 5 --rttl -j DROP
	iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --name SSHPOOL --set -j ACCEPT

这里rcheck可以换成update，rcheck和update的区别：

rcheck从第1个包开始计算时间，update是在rcheck的基础上增加了从最近的DROP包开始计算阻断时间，具有准许时间和阻断时间，帮助中说update会更新last-seen时间戳。

放在案例中，rcheck是接收到第1个数据包时开始计时，一个小时内仅限5次连接，后续的包丢弃，直到一小时过后又可以继续连接。update则是接收到第1个数据包时计算准许时间，在一个小时的准许时间内仅限5次连接，当有包被丢弃时，从最近的丢弃包开始计算阻断时间，在一个小时的阻断时间内没有接收到包，才可以继续连接。所以rcheck类似令牌桶，一小时给你5次，用完了抱歉等下个小时吧。update类似网银，连续输错5次密码，停止一小时，只不过update更严格，阻断时间是从最近的一次输错时间开始算，比如输错了5次，过了半个小时又输错一次，这时阻断时间不是剩半小时，而是从第6次重新计算，剩一小时。

rttl的含义是ttl值要和使用set添加到列表中地址的ttl一致，目的是防止有人伪造源地址来进行DOS。

###其他的限制ssh登陆次数方式和分析

	iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --name SSHPOOL --set -j ACCEPT
	iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --name SSHPOOL --rcheck --seconds 60 --hitcount 3 -j DROP
	iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT

以上配置限制每个ip60秒内登录2次，原因是第3次时和第二条规则匹配就被drop了。而上一种情况匹配5次是由于第5次ssh包到达时第二条规则判断目前列表中只有4次，所以可以登陆5次。

	iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
	iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --name SSHPOOL --set -j ACCEPT
	iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --name SSHPOOL --rcheck --seconds 60 --hitcount 3 -j DROP

这种情况是不能限制ssh的登陆次数的，上一种配置可以限制的原因是ssh登陆还有established包，那么和第二条规则匹配时就被drop了。这就是包含RELATED,ESTABLISHED的规则位置不同，结果也不同的原因。

可以通过加log来看一下：

	iptables -A INPUT -i eth0 -p tcp -m tcp --sport 22 -j ACCEPT
	iptables -A INPUT -i eth0 -p tcp -m tcp --dport 22 -j ACCEPT
	iptables -A INPUT -i eth0 -p tcp -m tcp --dport 80 -m recent --rcheck --seconds 60 --hitcount 3 --rttl --name SSH --mask 255.255.255.255 --rsource -j DROP
	iptables -A INPUT -i eth0 -p tcp -m tcp --dport 80 -m conntrack --ctstate NEW -m recent --set --name SSH --mask 255.255.255.255 --rsource -j ACCEPT
	iptables -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
	iptables -A INPUT -i eth0 -m conntrack --ctstate RELATED -j LOG --log-prefix "iptables related"
	iptables -A INPUT -i eth0 -m conntrack --ctstate RELATED -j ACCEPT
	iptables -A INPUT -i eth0 -m conntrack --ctstate ESTABLISHED -j LOG --log-prefix "iptables established"
	iptables -A INPUT -i eth0 -m conntrack --ctstate ESTABLISHED -j ACCEPT
	iptables -A INPUT -i eth0 -j DROP

这里使用80端口，避免ssh登陆产生的log，使用 “curl 服务器ip”命令测试，用“tail -f /var/log/kern.log”命令查看日志，可以看到登陆成功的前两次都有established的日志产生，而登陆失败的第3次没有日志产生，这说明被recent给drop了。
