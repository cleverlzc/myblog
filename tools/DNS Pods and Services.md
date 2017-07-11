

##kube-dns配置注意及问题排查

基础的k8s集群可以通过flannel的网络ip地址工作，但要扩展addons的service，都要通过域名来连通，因为各个镜像的配置文件中是不可能把ip写死在文件中的，但域名是可以不变的。因此，一个k8s集群中kube-dns的配置是必要的。

下面介绍配置kube-dns，需要注意的地方，验证及问题排查过程。

在/etc/resolv.conf文件中配置短域补齐，比如cluster.local，这里必须要与skydns-rc.yaml.sed文件中的domain参数一致。

###1、看kube-dns的pod内3个容器是否全部running

```
kubectl get pods --namespace=kube-system -l k8s-app=kube-dns
```

###2、如果全部running，看日志是否有异常

可以通过命令，也可以通过kube dashboard。

###3、若日志无明显异常，验证healthz是否能够解析完整域和短域

 * 完整域

```
kubectl exec -n kube-system -ti kube-dns-v20-xxxxx -c healthz -- nslookup kube-dns.kube-system.svc.cluster.local
```

 * 短域

```
kubectl exec -n kube-system -ti kube-dns-v20-xxxxx -c healthz -- nslookup kube-dns
```

###4、如果完整域可以解析，短域不可以解析
查看/etc/resolv.conf文件，是否补齐cluster.local

```
kubectl exec -n kube-system -ti kube-dns-v20-xxxxx -c healthz cat /etc/resolv.conf
```

一般文件内容为：


```
search default.svc.cluster.local svc.cluster.local cluster.local 
nameserver 172.17.26.52 #kube-dns service clusterIp
options ndots:5
```

这个是从宿主机的/etc/resolv.conf文件中继承的。


这些都可以通过官网doc可以查看到[Troubleshooting Tips](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)


###5、在node上的修改

修改/etc/kubernetes/kubelet文件：

```
KUBELET_ARGS="--cluster-dns=172.17.26.52 --cluster-domain=cluster.local --log_dir=/var/log/kubernetes"
```
修改之后要重启kubelet
```
systemctl restart kubelet
```

修改宿主机的/etc/resolv.conf：
```
search default.svc.cluster.local svc.cluster.local cluster.local 
nameserver 172.17.26.52 #kube-dns service clusterIp
```

###5、遇到的问题，这里也比较关键

####<1>、手动指定的clusterIp不能解析

在实际操作中，由于kube-dns的特殊性，需要手动指定kube-dns service的clusterIp（在各种“教程”中全都在说手动指定，他们竟然都没遇到问题），然后创建svc，结果使用healthz的nslookup验证时却不能解析。

幸好suzhen经验丰富，尝试把clusterIp注释掉，让k8s自动为kube-dns分配一个ip。然后再手动指定这个ip，重新创建svc。或者直接拿这个ip用就可以了。

按道理说指定的clusterIp在k8s限定的范围之内都是可以的，但是不知道随机指定了一个ip就不行。。。就这么不巧。。。具体原因目前还不清楚。


####<2>、命令可解析，healthz自带的参数不能解析

这个问题也是非常诡异的，把healthz的cmd参数单独拿出来用命令解析可以，但是它自己却不能解析，命令一模一样，完全没道理。。。

命令：
```
kubectl exec -n kube-system -ti kube-dns-v20-xxxxx -c healthz -- nslookup kubernetes.default.svc.cluster.local 127.0.0.1
```

日志：
```
can't resolve "kubernetes.default.svc.cluster.local"
```

healthz是负责dns的健康的，根据创建rc时的yaml文件，healthz容器定时向kubedns容器查询
`kubernetes.default.svc.cluster.local 127.0.0.1`及`kubernetes.default.svc.cluster.lcoal 127.0.0.1:10053`,

即(因为这个奇怪的问题把原args都注释了，暂时这样解决了)：
```
- name: healthz
        image: gcr.io/google_containers/exechealthz-amd64:1.2
        ...
        args:
        #- --cmd=nslookup kubernetes.default.svc.cluster.local 127.0.0.1 >/dev/null
        - --cmd=ping 127.0.0.1
        #- --url=/healthz-dnsmasq
        #- --cmd=nslookup kubernetes.default.svc.cluster.lcoal 127.0.0.1:10053 >/dev/null
        #- --url=/healthz-kubedns
        #- --port=8080
        #- --quiet
```

如果一段时间healthz一直解析不过，就会发送一个terminated信号给kubedns，这时即使kubedns本身正常（可以通过kubedns直接执行nslookup进行解析验证），也会自毁，此时该pod就会变的不正常。


####<3>、BTW

官网太简单，网上教程太杂，经验很重要，谨慎。


附skydns-rc.yaml.sed修改的地方：
```
spec:
      #nodeName: k8s-nod5
      containers:
      - name: kubedns
        image: gcr.io/google_containers/kubedns-amd64:1.8
        ...
        args:
        # command = "/kube-dns"
        - --domain=cluster.local.
        - --dns-port=10053
        - --kube-master-url=http://100.101.69.252:8080
...
- name: healthz
        image: gcr.io/google_containers/exechealthz-amd64:1.2
        ...
        args:
        #- --cmd=nslookup kubernetes.default.svc.cluster.local 127.0.0.1 >/dev/null
        - --cmd=ping 127.0.0.1
        #- --url=/healthz-dnsmasq
        #- --cmd=nslookup kubernetes.default.svc.cluster.lcoal 127.0.0.1:10053 >/dev/null
        #- --url=/healthz-kubedns
        #- --port=8080
        #- --quiet
```
