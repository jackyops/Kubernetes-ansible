# Kubernetes-ansible

## 全部重构,步骤和kubeadm一样,九成九配置文件和路径一样,管理组件扣成了systemd二进制,并且强制bind网卡支持多网卡部署和更细的配置

## 更新
 * 2018/06/08 - 解决2.5.3的ansible使用脚本报错
 * 2018/06/09 - 添加增加node剧本
 * 2019/03/08 - 推倒重来,仿照kubeadm,但是支持多网卡下部署,etcd支持单独的机器跑
 * 2019/03/14 - 修复setup里yum cache报错,优化etcd非master成员场景下的逻辑
 * 2019/03/18 - 改善未在fstab关闭swap和模板渲染回车问题,增加更多可选设置参数

## ansible部署Kubernetes

系统可采用`Ubuntu 16.x`与`CentOS 7.x`(CentOS建议使用最新的)
本次安裝的版本：
> * Kubernetes v1.13.4 (HA高可用)
> * CNI plugins v0.7.1
> * Etcd v3.3.12
> * flanneld v0.11.0
> * Calico v3.0.4(未写完)
> * Docker CE 18.06.03


**hosts文件写ip来支持多网卡部署**

**因为kubeadm扣的步骤,路径九成九一致,剧本里路径不给自定义**

**HA是基于VIP,无法用于云上(测试过青云默认解除ip和mac绑定可以飘VIP,阿里不行,存在ip和mac绑定,所以flannel的host-gw应该也用不了),阿里四层SLB也有问题**

安装过程是参考的[Kubernetes v1.13.4 HA全手动苦工安装教学](https://zhangguanzhang.github.io/2019/03/03/kubernetes-1-13-4)

**下面是我的配置,电脑配置低就两个Node节点,master节点建议也跑kubelet但是请打上污点,聚合路由有空测试试试**

| IP    | Hostname   |  CPU  |   Memory | 
| :----- |  :----:  | :----:  |  :----:  |
| 172.16.1.3 |k8s-m1|  2   |   2G    |
| 172.16.1.6 |k8s-m2|  2   |   2G    |
| 172.16.1.10 |k8s-m3|  2   |   2G    |
| 172.16.1.15 |k8s-n1|  2   |   2G    |
| 172.16.1.19 |k8s-n1|  2   |   2G    |

# 使用前提和注意事项
> * 每台主机端口和密码最好一致(不一致最好懂点ansible修改hosts文件)


# 使用(在master1的主机上使用且master1安装了ansible,剧本也可以支持非部署的机器运行剧本)

centos通过yum安装ansible或者pip
```
#可以yum安装指定版本或者离线安装 yum install -y  https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/ansible-2.7.8-1.el7.ans.noarch.rpm
yum install -y wget epel-release && yum install -y python-pip git sshpass
pip install ansible
# 嫌弃慢就下面的
# pip install --no-cache-dir ansible -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
```
#离线安装ansible的话可以先下载下来用yum解决依赖
```
yum install wget -y 1 > /dev/null
wget https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/ansible-2.7.8-1.el7.ans.noarch.rpm
yum localinstall ansible-2.5.4-1.el7.ans.noarch.rpm -y
```

**1 git clone**
```
git clone https://github.com/zhangguanzhang/Kubernetes-ansible.git
cd Kubernetes-ansible
```

**2 配置脚本属性**

 * 修改当前目录ansible的`hosts`分组成员文件

 * 修改`group_vars/all.yml`里面的参数
 1. ansible_ssh_pass为ssh密码(如果每台主机密码不一致请注释掉`all.yml`里的`ansible_ssh_pass`后按照的`hosts`文件里的注释那样写上每台主机的密码）
 2. `VIP`为高可用HA的虚ip,和master在同一个网段没有被使用过的ip即可,`NETMASK`为VIP的掩码
 3. `INTERFACE_NAME`为各机器的ip所在网卡名字Centos可能是`ens33`或`eth0`,如果业务做了bond是bond后的网卡,看情况自行修改.测试vip可用度是某台机器手动添加`ip addr add $VIP/$MASK dev $interface`后其他机器手动ping下
 4. 其余的参数按需修改,不熟悉最好别乱改
 5. 涉及到一些集群更新状态的参数参考 https://github.com/kubernetes-sigs/kubespray/blob/master/docs/kubernetes-reliability.md 
 6. `nodeStatusUpdate`变量对应下面的三种+默认值任选其一,单独选项的值优先级高`nodeStatusUpdate`,

`--node-status-update-frequency`为`kubelet`上传自身状态

`--node-monitor-period`为`kube-controller`多久查看一次node状态

`--node-monitor-grace-period`为`kube-controller`认为node多久无响应后是`NotReady`

`--pod-eviction-timeout`为`kube-controller`认为node无响应后多久驱逐该node上的pod
- Fast update and Fast Reaction
  
| 参数 | 值 | 默认值 |
| --- | ---: | ---: |
| --node-status-update-frequency | 4s | 10s |
| --node-monitor-period | 2s | 5s |
| --node-monitor-grace-period | 20s | 40s |
| --pod-eviction-timeout | 30s | 5m |
- Medium Update and Average Reaction 
  
| 参数 | 值 | 默认值 |
| --- | ---: | ---: |
| --node-status-update-frequency | 20s | 10s |
| --node-monitor-period | 5s | 5s |
| --node-monitor-grace-period | 2m | 40s |
| --pod-eviction-timeout | 1m | 5m |
- Low Update and Show reaction
  
| 参数 | 值 | 默认值 |
| --- | ---: | ---: |
| --node-status-update-frequency | 1m | 10s |
| --node-monitor-period | 5s | 5s |
| --node-monitor-grace-period | 5m | 40s |
| --pod-eviction-timeout | 1m | 5m |

 7. 运行下`ansible all -m ping`测试连通性
----------

**3 开始运行安装，下面是用法**
 * -----   setup.yml     -------
 * setup: 机器设置(关闭swap安装一些依赖+ntp)+内核升级并重启,例如`ansible-playbook setup.yml`,带上`-e 'kernel=false'`不会升级内核.如果是升级了内核,那重新连上后再`ansible all -m ping`看看连通性
 * -----    deploy.yml   -------
 * docker: 安装docker,默认是18.06,其他版本自行修改`group_vars/all.yml`, 运行命令为`ansible-playbook deploy.yml --tags docker`
 * 这步不是标签,手动运行`bash get-binaries.sh all`: 通过docker下载k8s和etcd的二进制文件还有cni插件,觉得不信任可以自己其他方式下载,cni插件可能不好下载.如果是运行剧本机器不是master[0]请自行下载相关文件到`/usr/local/bin`
 * tls: 生成证书和管理组件的kubeconfig,kubeconfig生成依赖kubectl命令,此步确保已经下载有.运行命令为`ansible-playbook deploy.yml --tags tls`,下面的也是tag改tags后面标签为下面的即可
 * etcd: 部署etcd,etcd可以非master上跑,按照提示设置好hosts文件即可,生成alias别名脚本存放在目录`/etc/profile.d/etcd.sh`,命令为`etcd_v2`和`etcd_v3`方便操作etcd
 * HA: keepalived+haproxy
 * master: 管理组件,检测apiserver端口那个`curl -sk https://master[0]:6443/healthz`的uri模块写法不知道是不是没调对经常报错,测试apiserver端口这步会出现带有`...ignoring`报错请忽略,执行完运行下`kubectl get cs`有输出就不用管
 * bootstrap: 给kubelet注册用
 * node: kubelet
 * addon: kube-proxy,flannel,coredns.metrics-server flannel二进制跑请运行前下载二进制文件`bash get-binaries.sh flanneld`,否则提前拉取镜像使用命令拉取`ansible Allnode -m shell -a 'curl -s https://zhangguanzhang.github.io/bash/pull.sh | bash -s -- quay.io/coreos/flannel:v0.11.0-amd64'`

运行方式为下面,xxx为上面标签
ansible-playbook deploy.yml --tags xxx  
运行到master后查看下管理组件状态
```
$ kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
```

运行完node后`kubectl get node -o wide`查看注册上来否
运行完addon后查看node状态是否为ready,以及kube-system命名空间下pod是否运行
不想master跑可以按照输出的命令打污点
```
kubectl taint nodes ${node_name} node-role.kubernetes.io/master="":NoSchedule
```
master的`ROLES`字段默认是none,它显示的值是来源于一个label
```
kubectl label node ${node_name} node-role.kubernetes.io/master=""
kubectl label node ${node_name} node-role.kubernetes.io/node=""
```


![k8s](https://raw.githubusercontent.com/zhangguanzhang/Image-Hosting/master/k8s/kube-ansible.png)

**4 后续添加Node节点(未完成)**
 1. 在当前的ansible目录改hosts,添加[newNode]分组写上成员
 2. 后执行以下命令添加node
 3. 然后查看是否添加上
```
$ kubectl get node -o wide
NAME     STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION              CONTAINER-RUNTIME
k8s-m1   Ready    <none>   27h   v1.13.4   172.16.1.4    <none>        CentOS Linux 7 (Core)   5.0.0-2.el7.elrepo.x86_64   docker://18.6.3
k8s-m2   Ready    <none>   27h   v1.13.4   172.16.1.5    <none>        CentOS Linux 7 (Core)   5.0.0-2.el7.elrepo.x86_64   docker://18.6.3
k8s-m3   Ready    <none>   27h   v1.13.4   172.16.1.8    <none>        CentOS Linux 7 (Core)   5.0.0-2.el7.elrepo.x86_64   docker://18.6.3
k8s-n1   Ready    <none>   27h   v1.13.4   172.16.1.12   <none>        CentOS Linux 7 (Core)   5.0.0-2.el7.elrepo.x86_64   docker://18.6.3
k8s-n2   Ready    <none>   27h   v1.13.4   172.16.1.13   <none>        CentOS Linux 7 (Core)   5.0.0-2.el7.elrepo.x86_64   docker://18.6.3
```

后面的一些Extraaddon后续更新
