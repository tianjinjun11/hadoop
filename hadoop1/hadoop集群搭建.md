#Hadoop云计算集群搭建
####环境
三台阿里云ECS，均为单核CPU，一个2G内存和两个1G内存。宽带1M。其中一台作为master/namenode节点，另外两台作为slave/datanode节点。安装CentOS 6.5(64)操作系统。
依次分别为公网ip， 内网ip，hostname

182.92.198.101 (公)   10.171.78.74 (内) master.hdp

182.92.204.211 (公)   10.171.78.69 (内) slave1.hdp

123.57.84.249 (公)    10.173.41.177 (内) slave2.hdp

####1：修改hostname(以master.hdp为例）
vim /etc/sysconfig/network

修改：HOSTNAME=master.hdp

重启即可！！！
####2：修改/etc/hosts（三个节点皆修改）
vim /etc/hosts

10.171.78.74 master.hdp

10.171.78.69 slave1.hdp

10.173.41.177 slave2.hdp
####3:ssh免密登录(单向）
节点1 : 182.92.198.101 节点2 : 182.92.204.211

步骤一：测试节点1到节点2的连接和访问

ssh root@182.92.204.211

需要输入密码才能进入！！

步骤二：使用 ssh-key-gen 命令生成公钥和私钥（在两个节点都需要生成）

ssh-keygen

步骤三：用 ssh-copy-id 命令将公钥复制或上传到远程主机，并将身份标识文件追加到节点2的 ~/.ssh/authorized_keys 中

ssh-copy-id -i /root/.ssh/id_rsa.pub 182.92.204.211

步骤四：验证免密码 SSH 登录节点2

ssh root@182.92.204.211

不输入 密码即可登录成功！！

###Ambari 的安装
####安装准备
关于 Ambari 的安装，目前网上能找到两个发行版，一个是 Apache 的 Ambari，另一个是 Hortonworks 的，两者区别不大。这里就以 Apache 的 Ambari 2.0.1 作为示例。本文使用三台 Redhat 6.6 作为安装环境（目前测试验证结果为 Ambari 在 Redhat 6.6 的版本上运行比较稳定），三台机器分别为 master.hdp,slave1.hdp,slave2.hdp。master.hdp 计划安装为 Ambari 的 Server，另外两台为 Ambari Agent。

安装 Ambari 最方便的方式就是使用公共的库源（public repository）。有兴趣的朋友可以自己研究一下搭建一个本地库（local repository）进行安装。这个不是重点，所以不在此赘述。在进行具体的安装之前，需要做几个准备工作。

1.SSH 的无密码登录；

Ambari 的 Server 会 SSH 到 Agent 的机器，拷贝并执行一些命令。因此我们需要配置 Ambari Server 到 Agent 的 SSH 无密码登录。在这个例子里，master.hdp可以 SSH 无密码登录 slave1.hdp 和 slave2.hdp。

2.确保 Yum 可以正常工作；

通过公共库（public repository），安装 Hadoop 这些软件，背后其实就是应用 Yum 在安装公共库里面的 rpm 包。所以这里需要您的机器都能访问 Internet。

3.确保 home 目录的写权限。

Ambari 会创建一些 OS 用户。

4.确保机器的 Python 版本大于或等于 2.6.（Redhat6.6，默认就是 2.6 的）。

以上的准备工作完成后，便可以真正的开始安装 Ambari 了。
####安装过程
首先需要获取 Ambari 的公共库文件（public repository）。登录到 Linux 主机并执行下面的命令（也可以自己手工下载）：

`wget http://public-repo-1.hortonworks.com/ambari/centos6/2.x/updates/2.0.1/ambari.repo`

将下载的 ambari.repo 文件拷贝到 Linux 的系统目录/etc/yum.repos.d/。拷贝完后，我们需要获取该公共库的所有的源文件列表。依次执行以下命令。

`yum clean all`

`yum list|grep ambari`

如图 1 所示：

图 1. 获取公共库源文件列表

![photos](https://github.com/tianjinjun11/hadoop/blob/%E5%AF%B9%E4%BB%98%E6%96%B9%E6%B3%95%E5%AF%B9%E4%BB%98/photos/img001.jpg?raw=true)

如果可以看到 Ambari 的对应版本的安装包列表，说明公共库已配置成功。然后就可以安装 Ambari 的 package 了。执行下面的命令安装 Ambari Server 到该机器。

`yum install ambari-server`

待安装完成后，便需要对 Ambari Server 做一个简单的配置。执行下面的命令。

`ambari-server setup`

在这个交互式的设置中，采用默认配置即可。Ambari 会使用 Postgres 数据库，默认会安装并使用 Oracle 的 JDK。默认设置了 Ambari GUI 的登录用户为 admin/admin。并且指定 Ambari Server 的运行用户为 root。

> JDK安装（三个节点皆需）
> 
> 在 /etc/profile文件中插入
> 
> export JAVA_HOME=/usr/jdk64/jdk1.7.0_67
> 
>export PATH=$JAVA_HOME/bin:$PATH
>
>export CLASSPATH=.:$JAVA_HOME/lib:$JAVA_HOME/jre/lib

简单的 setup 配置完成后。就可以启动 Ambari 了。运行下面的命令。

`ambari-server start`

当成功启动 Ambari Server 之后，便可以从浏览器登录，默认的端口为 8080。以本文环境为例，在浏览器的地址栏输入 http://182.92.198.101:8080，登录密码为 admin/admin。登入 Ambari 之后的页面如下图。

**图 2. Ambari 的 welcome 页面**

![photos](https://www.ibm.com/developerworks/cn/opensource/os-cn-bigdata-ambari/img002.jpg)

至此，Ambari Server 就安装完成了。
***
##部署一个 Hadoop2.x 集群
到这一节，我们将可以真正地体验到 Ambari 的用武之地，以及它所能带来的方便之处。
登录 Ambari 之后，点击按钮“Launch Install Wizard”，就可以开始创建属于自己的大数据平台。

第一步，命名集群的名字。本环境为 bigdata。

第二步，选择一个 Stack，这个 Stack 相当于一个 Hadoop 生态圈软件的集合。Stack 的版本越高，里面的软件版本也就越高。这里我们选择 HDP2.2，里面的对应的 Hadoop 版本为 2.6.x。

第三步，指定 Agent 机器（如果配置了域，必须包含完整域名，例如本文环境的域为 example.com），这些机器会被安装 Hadoop 等软件包。还记得在安装章节中提到的 SSH 无密码登陆吗，这里需要指定当时在 Ambari Server 机器生成的私钥（ssh-keygen 生成的，公钥已经拷贝到 Ambari Agent 的机器，具体的 SSH 无密码登录配置，可以在网上很容易找到配置方法，不在此赘述）。另外不要选择“Perform manual registration on hosts and do not use SSH“。因为我们需要 Ambari Server 自动去安装 Ambari Agent。具体参见下图示例。

**图 3. 安装配置页面**
![](https://www.ibm.com/developerworks/cn/opensource/os-cn-bigdata-ambari/img003.jpg)

