---
layout: post
title: 使用ubuntu虚拟机搭建spark环境遇到的坑
category: project
description: 写攻略的同学一定要用心啊！
---
使用的环境是Ubuntu 17.04，一主两从共三个虚拟机。

1、创建第一台虚拟机，即master机器。将java、scala、hadoop、spark的压缩包拷进去。解压java和scala的压缩包，配置java和scala的环境变量。

2、赋予master机器登录用户root权限。修改``/etc/passwd``文件，将master机器登录用户名后的两个1000修改为0即可。重启电脑后，设置root用户密码：``passwd root``。

2、配置虚拟机的静态ip。一般vmware会勾选“使用本地DHCP服务将IP地址分配给虚拟机”，这会在未来会导致虚拟机ip发生变动，导致spark和hadoop无法正常连接slave机器。具体配置方法见[为VMware虚拟机内安装的Ubuntu 16.04设置静态IP地址](https://www.linuxidc.com/Linux/2017-04/143102.htm)。

3、修改``/etc/hostname``文件，将主机名修改为``master``。

4、修改/etc/hosts文件，根据网关地址，将三台机器ip分配在一个子网下。以网关为192.168.21.2，子网掩码为225.225.225.0为例，ip可以配置为：   
``192.168.21.10   master``   
``192.168.21.11   slave1``   
``192.168.21.12   slave2``

5、安装ssh服务：``apt-get install openssh-server``

6、克隆主机。以配置好的master机器克隆出两台slave机器，完成之后修改两台slave机器的``hostname``和``ip``地址，与第四步中配置的ip地址和主机名相对应。配置完成之后可以使用``ifconfig``查看配置是否完成，并互ping看是否可以ping通。   
7、配置ssh，这是坑最大的一步，最开始照攻略配置完成后无法使用``ssh slave1``连接，始终需要输入slave1的root用户密码，而且输入后也会被拒绝连接。至少卡了我3个小时啊啊啊啊啊。   

首先在每台主机的/root文件夹下生成密钥仓库。在每台主机上都生成私钥公钥。命令是``ssh-keygen -t rsa``，一直按回车就行。    
修改``/etc/ssh/sshd_config``，将``PermitRootLogin``项的值设为``yes``   
将slave1与slave2上的id_rsa.pub用scp命令发送给master：   
``scp /root/.ssh/id_rsa.pub master@master:/root/.ssh/id_rsa.pub.slave1``   
``scp /root/.ssh/id_rsa.pub master@master:/root/.ssh/id_rsa.pub.slave2``   
在master上，将所有公钥加到用于认证的公钥文件authorized_keys中：   
``cat /root/.ssh/id_rsa.pub* >> /root/.ssh/authorized_keys ``   
将公钥文件authorized_keys分发给每台slave：   
``scp /root/.ssh/authorized_keys master@slave1:/root/.ssh/``   
``scp /root/.ssh/authorized_keys master@slave2:/root/.ssh/``    

需要注意的是，一定要将.ssh文件夹放在``root``文件夹下，这样才可以实现使用ssh免密登录root用户(``ssh slave1``为使用root用户登录，``ssh master@slave1``为使用master用户登录，两者权限区别很大。)。这也是前面给予用户root权限和设置root密码的原因，有了root权限后可以才可以访问和修改``root``文件夹。   
当然这并不是攻略的问题，攻略使用的是centOS，centOS与Ubuntu的用户组策略不同，centOS允许root用户登录系统，Ubuntu则不允许，这是导致问题的核心（我猜）。   

最后在每台主机上，用SSH命令，检验下是否能免密码登录。   
``ssh slave1``   
``ssh slave2``

8、后面基本就没什么坑了，照攻略配置就好，需要注意的是，如果需要修改的文件不存在，只有对应的template文件的话，直接去掉``template``文件的``template``后缀，修改之即可。   
如需要修改``core-site.xml``文件，直接将``core-site.xml.template``修改为``core-site.xml``。

9、我说的攻略是[使用虚拟机从小白开始搭建Spark集群](http://blog.csdn.net/wy250229163/article/details/52729608)，真心感谢作者。

10、不要使用ubuntu，请使用centOS:)
