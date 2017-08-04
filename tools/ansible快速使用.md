

### 1、安装及了解

CentOs和Ubuntu等大多数系统均可以一键安装。

CentOs：
```
yum search ansible
yum install -y ansible
```
Ubuntu:
```
apt-get update
apt-get install ansible
```


如果想快速使用的话，只需看[Ansible中文权威指南](http://www.ansible.com.cn/docs/intro_getting_started.html)总体介绍下的安装和新手上路两小节就可以了。

剩下的边用边了解吧。先用起来。

### 2、编辑/etc/ansible/hosts文件

该文件是ansible的核心文件，里面放置一些你要控制的机器ip或者机器ip组。比如：

```
[kube-nodes]
10.2.26.50
10.2.26.51
10.2.26.52
10.2.26.53
10.2.26.54
10.2.26.55

[kube-master]
10.2.26.45
```

### 3、配置SSH

 将控制机的公钥添加到被控制机器的authorized_keys

 公钥用于加密存于服务器，私钥用于解密存于客户机，这些不多说了。 

 * `ssh-keygen -t rsa`，一路回车；
 * 将~/.ssh下的id_rsa.pub中的内容添加到被控制机器的authorized_keys；
 * 修改权限，`chmod 644 ~/.ssh/authorized_keys； chmod 700 ~/.ssh `
 
  [ssh配置](http://blog.csdn.net/qq_35613461/article/details/51941680)，[原理](http://shihlei.iteye.com/blog/2064677)

### 4、使用

ping所有机器：

```
ansible all -m ping
```

ping某个机器组比如kube-nodes内的机器：

```
ansible kube-nodes -m ping
```

查看某一个host的docker服务的状态：
```
ansible 10.21.106.157 -m command -a 'systemctl status docker.service'
```

重启所有hosts的docker服务：

```
ansible all -m command -a 'systemctl restart docker'
```

如果你修改了docker的镜像代理，需要重启才能生效，这时你一条ansible指令就全部重启了，简直是神器啊。

刚才提到修改k8s集群下所有node上的docker镜像代理，这里你修改好一个，复制就可以了，比如：

```
ansible kube-master -m copy -a 'src=/tmp/demo.txt dest=/tmp'
```

执行脚本：
```
ansible kube-master -m command -a 'bin/start-slaves.sh'
```

sudo权限执行：
```
ansible kube-master -m command -a 'service elasticsearch start' --sudo --ask-sudo-pass
```


参数说明：

 * -m 表示使用什么模块，常用的有command shell copy
 * -a 表示模块的具体指令

   [ansible使用](http://www.jianshu.com/p/aaae4cd19930)

以上基本的操作差不多够用了。要学习更深的东西，需要学习一下ansible的playbook了。

