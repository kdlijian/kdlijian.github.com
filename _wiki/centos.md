---
title: CentOS
---


Yum
===

### EPEL

    $ sudo yum install epel-release

### Development Tools

	$ sudo yum group list
	$ sudo yum group info "Development Tools"
	$ sudo yum groupinstall "Development Tools"

### RPM

查看一个包安装了哪些文件：

	$ rpm -q -l logrotate

### Yum Proxy

	# vim /etc/yum.conf

	proxy=http://192.168.200.1:1080

### list installed

	# yum list installed

### 查看某个文件有哪个包提供

	# yum provides /usr/sbin/nginx

### 清空缓存

	# yum clean all && yum clean metadata && yum clean dbcache && yum makecache && yum update

### 安装某个软件包的某个版本

	# yum list mysql-community-server --showduplicates
	# yum install mysql-community-server-x.xxx

Hostname
========

CentOS 7 的 hostname 由 systemd-hostnamed 服务设定，hostnamectl 是该服务的命令行客户端。

	# hostnamectl status

	# hostnamectl set-hostname app1 --static (写入 /etc/hostname，重启系统读取)
	# hostnamectl set-hostname app1

SELinux
=======

可能需要安装实用工具先：

    # yum install policycoretools

查看状态：

	$ sestatus

### 关闭 SELinux

在 `/etc/selinux/config` 里设置：

	SELINUX=disabled

重启系统生效。


System Limits
=============

查看系统的 file-max:

    $ sysctl fs.file-max

查看某个进程的 limits:

    $ cat /proc/<pid_of_nginx>/limits

调整 Systemd Services 的 limits：

    $ sudo mkdir /etc/systemd/system/nginx.service.d
    $ sudo vim /etc/systemd/system/nginx.service.d/limit.conf
    [Service]
    LimitNOFILE=10240

/etc/security/limits.conf 里的 limits 仅适用于通过 PAM 登录的*用户*，不适用 Systemd *服务*。



LAMP
====

	# yum install httpd
	# service httpd start

	# yum install mysql-server
	# service mysqld start
	# /usr/bin/mysql_secure_installation

	# yum install php php-mysql

	# chkconfig httpd on
	# chkconfig mysqld on


Python 3
========

### Prepare
	
	# yum groupinstall 'Development tools'
	# yum group install 'Development tools' ### CentOS 7
	# yum install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel

### Python

	# cd /usr/local/src/python-3.3.5
	# ./configure --prefix=/usr/local/python3.3.5
	# make && make install

### Pip

Python 3 自带 pip。



Ansible
=======

Ansible 似乎只支持 Python 2? 是的。

先安装 pip:[1]

	# wget https://bootstrap.pypa.io/get-pip.py
	# python get-pip.py

	# yum install python-devel zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel
	# pip install ansible


Cron
====

CentOS 6 里有两个 cron 程序，一个是 cronie, 它提供 `/etc/init.d/crond`, `/etc/crontab` 和 `/etc/cron.d/0hourly` 等文件和功能；另一个是 cronie-anacron，它提供 `/etc/anacrontab` 和 `/etc/cron.hourly/0anacron 等文件。 

先来说说两者的关系， cronie 是主要的 cron 程序，它默认开机启动；而 cronie-anacron 没有系统启动项，它是 cronie 的补充，被 cronie 所调用。cronie 用来设定在固定时间执行任务，而如果该执行任务时系统是关机的状态，那么就得用 anacron 来捕获这个任务并在开机后补充执行。

默认地，`/etc/crontab` 里没有定义任何任务；不过在 `/etc/cron.d/0hourly` 文件里定义了一条任务：

	01 * * * * root run-parts /etc/cron.hourly

也就是说 cronie 会在每个小时的第一分钟执行一次 `/etc/cron.hourly/` 目录下定义的任务。

我们来看看 `/etc/cron.hourly` 目录下定义了什么：只有 cronie-anacron 程序包安装进来的 `/etc/cron.hourly/0anacron` 文件。这个脚本里有一行指令 `/usr/sbin/anacron -s `，它读取 `/etc/anacrontab` 里的配置：

	...
	START_HOURS_RANGE=3-22

	#period in days   delay in minutes   job-identifier   command
	1       5       cron.daily              nice run-parts /etc/cron.daily
	7       25      cron.weekly             nice run-parts /etc/cron.weekly
	@monthly 45     cron.monthly            nice run-parts /etc/cron.monthly

配置里定义了几个任务，比如 cron.daily，每个任务的第一个字段是时间段，anacron 会将上一次执行该任务的时间记录在 `/var/spool/anacron/cron.daily` 文件里，如果当前 anacron 的执行时间(被 cronie 调用）与配置里上次某个任务的执行时间相差大于该任务定义的时间段，这个任务就会再次执行，允许执行的时间区间是 3 点 到 22 点。

因此我们发现，CentOS 的策略不是在 `/etc/crontab` 里定义严格的按照时间执行任务，而是交给更加灵活的 anacron 来判断执行时机。这样称 anacron 是 cronie 的补充的说法就一目了然了。


Logrotate
=========

Logrotate 会安装一个脚本放在 `/etc/cron.daily` 目录里，用于每日滚动日志。

下面是一个滚动 squid 日志的用例：

	# vi /etc/logrotate.d/squid
	
	/usr/local/squid/var/logs/*.log {
		missingok
		notifempty
		rotate 5
		daily
		nocompress
		create 0640 squid squid
	}

Dry run(debug 模式):

	# logrotate -d /etc/logrotated.d/squid

立即执行:

	# logrotate /etc/logrotated.d/squid

注：如果待滚动日志所在目录权限太开放（比如所有人可写），logrotate 会拒绝执行，需要在配置文件里指定执行滚动所用用户：

> considering log /usr/local/MongoDB-to-MySQL/import.log
error: skipping "/usr/local/MongoDB-to-MySQL/import.log" because parent directory has insecure permissions (It's world writable or writable by group which is not "root") Set "su" directive in config file to tell logrotate which user/group should be used for rotation.

在配置文件添上以下指令：

	su root root


Iptables
========

	### CentOS 7
	# systemctl stop firewalld
	# systemctl disable firewalld

Docker
======

要想使用 Docker 的网络特性，需要打开宿主机的端口转发：

	### /etc/sysctl.conf 添加：
	net.ipv4.ip_forward=1

而后，运行该命令使其生效：

	# sysctl -p

Redis
=====

	# yum install gcc-c++ make
	# cd /usr/local/src/Redis
	# make 
	# make PREFIX=/usr/local/redis install

	# ./bin/redis-server conf/redis-6379.conf
	# ./bin/redic-cli shutdown

[1]: https://pip.pypa.io/en/stable/installing/
