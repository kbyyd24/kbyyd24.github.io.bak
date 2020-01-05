---
title: 服务器配置小记
date: 2016-11-17
updated: 2016-11-17
---

最近一个项目要上线了,需要搭服务器,本来是交给同学搭的,结果遇到了大坑,还得自己来,今天把这些坑记一下.

服务器有好几台,都是`CentOS6.X`,两台6.8,一台6.4.

项目需要的环境是`Java`+`Gradle`+`MySql`+`Redis`+`Nginx`

# 太长不看总结版

## 顺利的安装

```shell
# 准备下载工具
wget http://ftp.tu-chemnitz.de/pub/linux/dag/redhat/el6/en/x86_64/rpmforge/RPMS/axel-2.4-1.el6.rf.x86_64.rpm
yum localinstall axel-2.4-1.el6.rf.x86_64.rpm -y

# 安装MySql
wget http://dev.mysql.com/get/mysql57-community-release-el6-9.noarch.rpm
yum localinstall mysql57-community-release-el6-9.noarch.rpm -y
yum replist all | grep mysql
yum install yum-utils -y
yum-config-manager --disable mysql57-community
yum-config-manager --enalble mysql56-community
yum install mysql mysql-server mysql-devel -y

# 安装Nginx
echo -e '[nginx]\nname=nginx repo\nbaseurl=http://nginx.org/packages/centos/6/$basearch/\ngpgcheck=0\nenabled=1' > /etc/yum.repos.d/nginx.repo
yum makecache 
yum install nginx -y

# 安装Redis
yum install -y tcl
axel -n 5 http://download.redis.io/releases/redis-2.8.24.tar.gz
cd /usr/local
tar zxf ~/redis-2.8.24.tar.gz
mv redis-2.8.24/ redis/
cd redis 
make 
make test
cd src/
make install

# 安装Java
wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u111-b14/jdk-8u111-linux-x64.tar.gz
mkdir /usr/java
cd /usr/java
tar zxf ~/jdk-8u111-linux-x64.tar.gz
echo 'export JAVA_HOME=/usr/java/jdk1.8.0_111' >> /etc/profile
echo 'export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar' >> /etc/profile
echo 'export PATH=$PATH:$JAVA_HOME/bin' >> /etc/profile
source /etc/profile
java -version

# 安装Gradle
axel -n 5 https://services.gradle.org/distributions/gradle-2.14-bin.zip
mkdir /usr/local/gradle
cd /usr/local/gradle 
unzip -q ~/gradle-2.14-bin.zip
echo 'export PATH=$PATH:/usr/local/gradle/gradle-2.14/bin' >> /etc/profile
source /etc/profile
gradle -version
```

## 坑

### Docker的莫名错误

- 在`CentOS6.4`上面试过`Docker`,启动容器后报错,找不到容器
- 在`CentOS6.8`上面,`Docker`的守护进程总是过一会儿自己就挂掉
- 第二台`CentOS6.8`上面,因为`iptables`的配置问题,导致`127.0.0.1`无法访问,引起`Docker`映射端口没有作用
- 第二台`CentOS6.8`上面使用`MySql`的镜像,启动配置`MYSQL_ROOT_PASSWORD`环境变量后面的启动配置不能生效

### MySql的安装中的坑

- CentOS6默认的`yum`安装的`MySql`版本是5.1,会出现中文乱码的问题,需要下载[mysql57-community-release-el6-9.noarch.rpm](http://dev.mysql.com/get/mysql57-community-release-el6-9.noarch.rpm)来添加`yum`源
- 添加后的仓库默认激活`MySql5.7`,`CentOS6`并不支持,需要激活`MySql5.6`
- enable与disable哪个包可以使用`yum-config-manager`进行配置,这个命令包含在`yum-utils`包中,可通过`yum`安装

### localhost,127.0.0.1&iptables

- 现在的主机默认`localhost`会解析到`127.0.0.1`,而`127.0.0.1`会发送到环回的虚拟网卡,回到主机上
- `MySql`是个特立独行的家伙,默认的`mysql`连接会访问`localhost`,但不会映射到`127.0.0.1`上,而是访问配置文件中描述的`sock`文件,不会经历网络层的传输
- `127.0.0.1`虽然不会离开主机,但也会经过网卡(虽然是虚拟的),所以也会收到`iptables`的影响
- 不要随意删除`iptables`里面的规则,否则真的会后悔的

## tips

- `linux`上的命令行多线程下载工具:`axel`,地址:[https://pkgs.org/centos-6/repoforge-x86_64/axel-2.4-1.el6.rf.x86_64.rpm.html](https://pkgs.org/centos-6/repoforge-x86_64/axel-2.4-1.el6.rf.x86_64.rpm.html)
- `nginx`可通过`yum`下载最新稳定版:[http://nginx.org/en/linux_packages.html](http://nginx.org/en/linux_packages.html)
- 在旧的发行版上安装`mysql`,优先考虑更新源,而不是下载安装包
- 在旧的发行版上,就不要尝试`docker`了,遇到坑伤不起
- 更新你的服务器,大清亡了

---

详细版

# 在CentOS6.4上的折腾

## 关于Mysql的折腾

我通过`MySql`官网下载的`yum`源安装的`MySql`,在`CentOS6.4`的系统上执行

```shell
yum install mysql mysql-server mysql-devel
```

没有遇到任何依赖错误,于是顺利安装,然后启动服务

```shell
servcie mysqld start
```

但是得到了失败的结果.于是查看配置文件,找到日志,发现了如下错误信息

```
2016-11-17T21:44:32.121190Z 0 [ERROR] Fatal error: mysql.user table is damaged. Please run mysql_upgrade.
2016-11-17T21:44:32.121315Z 0 [ERROR] Aborting
```

于是按照他的提示,执行`mysql_upgrade`,却又提示没有启动`mysqld`服务..这是一个死锁啊 :dizzy_face:

然后移除了这个`MySql5.7`,向系统妥协,用`yum`装了个`5.1`的版本,导入建库语句,没有错误产生,暗道庆幸,还以为是自己多虑了.
**然而,等我放入数据之后,该来的还是来了.**

我在本地开发环境中是用的是`docker`(后面会讲[为什么不在服务器上用`docker`](#莫名挂掉的Docker))拉取的`MySql5.7`,本地测试一切正常.然后放到服务器上,当我查询一个中文的记录的时候,杀机终于显现——**乱码啦** :scream_cat:

作为一个对自己代码拥有充分自信的家伙,怎么能是我的问题呢,都怪`MySql`的版本太低太弱智.

# 在第一台CentOS6.8上的折腾

所以我决定换一台服务器,从头再来.还好实验室服务器多,够我折腾

## 先来折腾MySql

基于在上一台服务器上的惨痛教训,这次我决定先装`MySql`.

首先确定发行版本

```shell
$ cat /etc/issue
CentOS release 6.8 (Final)
Kernel \r on an \m
```

准备下载工具`axel`

```shell
wget http://ftp.tu-chemnitz.de/pub/linux/dag/redhat/el6/en/x86_64/rpmforge/RPMS/axel-2.4-1.el6.rf.x86_64.rpm
yum localinstall -y axel-2.4-1.el6.rf.x86_64.rpm
```

竟然连`wget`也要安装一下 :joy:

然后安装`MySql`的`yum`源并安装`MySql`,这个文件只有9k,直接用`wget`就行了

```shell
wget http://dev.mysql.com/get/mysql57-community-release-el6-9.noarch.rpm
yum localinstall mysql57-community-release-el6-9.noarch.rpm -y
yum install -y mysql mysql-server mysql-devel
```

然后出现依赖报错

```
Error: Package: mysql-community-server-5.7.34-2.el7.x86_64 (mysql56-community)
           Requires: libc.so.6(GLIBC_2.17)(64bit)
Error: Package: mysql-community-server-5.7.34-2.el7.x86_64 (mysql56-community)
           Requires: systemd
Error: Package: mysql-community-client-5.7.34-2.el7.x86_64 (mysql56-community)
           Requires: libc.so.6(GLIBC_2.17)(64bit)
Error: Package: mysql-community-libs-5.7.34-2.el7.x86_64 (mysql56-community)
           Requires: libc.so.6(GLIBC_2.17)(64bit)
Error: Package: mysql-community-server-5.7.34-2.el7.x86_64 (mysql56-community)
           Requires: libstdc++.so.6(GLIBCXX_3.4.15)(64bit)
 You could try using --skip-broken to work around the problem
** Found 1 pre-existing rpmdb problem(s), 'yum check' output follows:
tzdata-2016i-1.el6.noarch is a duplicate with tzdata-2012j-1.el6.noarch
```

我网上找了半天也没找到怎么解决这个错误的解决办法,应该是`CentOS6.8`不支持,于是放弃自己安装,准备上`docker`

## 莫名挂掉的Docker

我平常使用的是[daocloud.io](http://daocloud.io/)的`docker`镜像,这里也使用这个安装

```shell
curl -sSL https://get.daocloud.io/docker | sh
chkconfig docker on
servcie docker start
```

然后拉取`MySql`的镜像、运行,一切正常.然后拉取`redis`和`nginx`,如果一切正常的话,很快就能搞定了.

然而可怕的情况出现了,拉取过程挂掉了... :scream: 于是查看`docker`守护进程,发现守护进程竟然挂掉了!我可是什么都没做啊!而且三更半夜,谁会和我一起搞服务器?! :question:

对于这种莫名的原因,我感觉很是害怕.因为之前这台主机是被动过的,所以我不知道是不是别人动过什么.决定再换一台没动过的主机来搞.

# 在第二台CentOS6.8上的折腾

我没有在这台主机上尝试`docker`,我已经不信任在`CentOS6.X`环境中的`docker`了,直接来安装`MySql`吧

## 还是先来MySql

这次我找到了这篇文章[https://segmentfault.com/a/1190000003049498](https://segmentfault.com/a/1190000003049498)

什么,竟然`yum`源还有激活这个事情?! :scream: 我还是太天真,太孤陋寡闻.于是赶紧查看一下`MySql`源的激活情况

```shell
yum repolist all | grep mysql
```

果然默认激活`5.7`,于是我选择降低一个版本,使用`5.6`

```shell
yum install -y yum-utils
yum-config-manager --disable mysql57-community
yum-config-manager --enalbe mysql56-community
yum install -y mysql mysql-server mysql-devel
```

欣喜的看到安装成功了,但是有之前在`CentOS6.4`上的惨痛教训,我还是不敢高兴的太早,至少`mysql`先要能运行起来

```shell
service mysqld start
```

然后根据提示,可以执行

```shell
mysqladmin -u root password 'new_root_pass'
mysql_secure_installation
```

然后就是少有的顺利安装完成,然后登录,创建一个数据库和用户并授权,再用新的用户登录建库,都没有问题.我想应该是OK了

## 用yum安装Nginx-1.10.2

接下来就是`nginx`了,官网就有教程教你怎么用`yum`安装[http://nginx.org/en/linux_packages.html](http://nginx.org/en/linux_packages.html)

于是我就照着教程的样子创建了一个`/etc/yum.repos.d/nginx.repo`,内容如下:

```
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/6/$basearch/
gpgcheck=0
enabled=1
```

然后安装、配置、启动、测试都正常通过.按照我"多年"的`nginx`使用经验,这个家伙是安装完成了.

## 安装Redis--异常初现

接着是`Redis`,只有编译安装这一个方法.因为使用的是`Spring`框架连接`Redis`所以下载了一个`Redis-2.8`

```shell
yum install -y tcl
# tcl 是 redis 的依赖,可以通过执行 {redis_home}/runtest 了解 redis 需要安装的依赖
axel -n 5 http://download.redis.io/releases/redis-2.8.24.tar.gz
cd /usr/local
tar zxf ~/redis-2.8.24.tar.gz
mv redis-2.8.24/ redis/
cd redis 
make 
make test
cd src/
make install
```

然后在`make test`的时候出现了错误

```
Executing test client: couldn't open socket: host is unreachable. 
couldn't open socket: host is unreachable     
while executing
```

google了好久也没找到解决方案,索性就不管它,直接启动`redis-server`,然后使用客户端测试连接,却得到错误

```
Could not connect to Redis at 127.0.0.1:6379: No route to host
```

这我就想不通了,但是没关系,之前那台`CentOS6.4`上面还有`redis`服务,配置一下防火墙就行了,先测试`mysql`乱不乱码要紧.

## 异常真凶浮出水面

于是我急匆匆的把代码clone下来,修改一下`redis-server`地址就跑起来了.
然后访问一下试试.然后`Spring Boot`内置的`Tomcat`的数据库连接池就报错了.

```
java.net.NoRouteToHostException: 没有到主机的路由
```

这次我就真懵逼了:`Mysql`不是在运行吗?不是能连上吗?这怎么就不行了?难道我必须得用`Docker`,还要解决`daemon`离奇死亡的问题?

**咦?`Docker`?`MySql`?**

于是我想到了什么,便在服务器的命令行连接了一次`MySql`

```
$ mysql -h 127.0.0.1 -u user -ppassword
ERROR 2003 (HY000): Can't connect to MySQL server on '127.0.0.1' (113)
```

果然,和`redis`遇到的错误是同一个原因:**无法访问127.0.0.1上的服务**

> 我是怎么想到的?
>
> 因为我本地的`mysql`是使用`docker`运行的,所以我要连接`mysql`,访问地址必须得是`127.0.0.1`,所以项目的配置就写的`127.0.0.1`
>
> 为什么呢?因为`mysql`对`localhost`的请求做了优化,直接走`socket`的路线,没有通过`127.0.0.1`去连接,所以`docker`运行的`mysql`如果没做`sock`文件的映射,是不能通过`localhost`访问的.而`mysql`客户端默认使用的地址正是`localhost`
>
> 这就是为什么一开始我能通过`mysql`命令连接上`mysql-server`,而加上`-h 127.0.0.1`参数就不行了的原因.

知道了原因,那就很好对症下药了.

## 菜鸡的防火墙配置--服务器血崩

`127.0.0.1`虽然是访问的本机,但实际请求也是会经过网卡(虚拟网卡)的.所以应该是防火墙拦截了我的请求.那么开始与`iptables`做斗争吧.

对一个安全菜鸡来说,搞防火墙是一件需要小心谨慎的事情;但对一个熬夜的菜鸡来说,理智什么的都已经走远了 :joy:

这是一开始我看到的`iptables`

```
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination         
1    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           tcp dpt:9000 
2    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           state RELATED,ESTABLISHED 
3    ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0           
4    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           state NEW tcp dpt:22 
5    REJECT     all  --  0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited 

Chain FORWARD (policy ACCEPT)
num  target     prot opt source               destination         
1    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           ctstate RELATED,ESTABLISHED 
2    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
3    REJECT     all  --  0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited 
4    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination
```

* 第一条规则是我设置的,确定没问题(这时我还保有理智)
* 第二条什么鬼?删了!

于是一条命令执行出去了

```shell
iptables -D INPUT 2
```

然后这台主机今晚就变成一条废咸鱼了,我也像吃了鲱鱼罐头一样清醒了.

我竟然把`RELATED`和`ESTABLISHED`的连接都给拒绝了!!! :scream:

经过半个小时的头(zi)脑(wo)风(feng)暴(ci), 我想起第一台`CentOS6.8`的主机,也许我激活一下`mysql5.6`还有救

# 回到第一台CentOS6.8的再次折腾

有了前几次的经验总结,这一次搭建变得顺利起来

## 顺利安装MySql

先是安装`MySql`

```shell
yum-config-manager --disable mysql57-community
yum-config-manager --enable mysql56-community
yum install -y mysql mysql-server mysql-devel
service mysqld start
mysqladin -u root password 'new_root_pass'
mysql_secure_installation
```

## 成功安装并运行的Redis

接下来是`Redis`

```shell
yum install -y tcl
axel -n 5 http://download.redis.io/releases/redis-2.8.24.tar.gz
cd /usr/local
tar zxf ~/redis-2.8.24.tar.gz
mv redis-2.8.24/ redis/
cd redis 
make 
make test
cd src/
make install
```

对于`Redis`还修改了一些默认配置

```
daemonize yes
bind 0.0.0.0
logfile "/var/log/redis.log"
```

然后使用这个配置文件启动

```shell
redis-server /usr/local/redis/redis.conf
```

## 同样顺利的Nginx

然后是`Nginx`,先按照官网的方法设置`yum`仓库

```shell
yum makecache
yum install -y nginx
service nginx start
```

这个方法安装的`Nginx`是默认开机启动的.测试一下,能够正常运行,然后配置了一下,做了个反向代理绕过跨域,重启.

## 安装Java

接着是`Java`

```shell
wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u111-b14/jdk-8u111-linux-x64.tar.gz
mkdir /usr/java
cd /usr/java
tar zxf ~/jdk-8u111-linux-x64.tar.gz
echo 'export JAVA_HOME=/usr/java/jdk1.8.0_111' >> /etc/profile
echo 'export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar' >> /etc/profile
echo 'export PATH=$PATH:$JAVA_HOME/bin' >> /etc/profile
source /etc/profile
java -version
```

## 安装Gradle

最后是`Gradle`

```shell
axel -n 5 https://services.gradle.org/distributions/gradle-2.14-bin.zip
mkdir /usr/local/gradle
cd /usr/local/gradle 
unzip -q ~/gradle-2.14-bin.zip
echo 'export PATH=$PATH:/usr/local/gradle/gradle-2.14/bin' >> /etc/profile
source /etc/profile
gradle -version
```

终于,把所有的东西都装好了,赶快把代码拉下来跑一跑,终于没有乱码了,接口都能正常使用.而此时,天都开始亮了..