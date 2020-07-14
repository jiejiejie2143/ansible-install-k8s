# Kubernetes v1.16 高可用集群自动部署（离线版）
>### 确保所有节点系统时间一致
>### 所有节点请手动升级内核
### 手动升级内核（不好自动化）
一：升级内核
1.源

```
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
```
2、列出可用的系统内核相关包
```
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
```
3、安装最新的主线稳定内核
```
yum --enablerepo=elrepo-kernel install kernel-ml kernel-ml-devel -y
```
4、 查看默认启动顺序
```
awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg
```
5、 默认启动的顺序是从0开始，但我们新内核是从头插入（目前位置在1，而4.0.2的是在0），所以需要选择0，如果想生效最新的内核，需要
```
grub2-set-default 0
grub2-mkconfig -o /boot/grub2/grub.cfg
cat /boot/grub2/grub.cfg
yum remove kernel-3.10.0-327.el7.x86_64 kernel-devel-3.10.0-327.el7.x86_64 -y （删除自己本机对应的）
reboot
uname -r
```
### 1、下载所需文件

下载Ansible部署文件：

```
 git clone https://github.com/jiejiejie2143/ansible-install-k8s.git
cd ansible-install-k8s
```

下载软件包并解压：

链接：https://pan.baidu.com/s/1w0qRjb5MD72j7jFnF65ZUA 
提取码：e49g
```
tar zxf binary_pkg.tar.gz
```
### 2、修改Ansible文件

修改hosts文件，根据规划修改对应IP和名称。

```
vi hosts
```
修改group_vars/all.yml文件，修改软件包目录(binary_pkg提前解压)和证书可信任IP。

```
vim group_vars/all.yml
software_dir: '/root/binary_pkg'
...
cert_hosts:
  k8s:
  etcd:
```
## 3、一键部署
### 架构图
单Master架构
![avatar](https://raw.githubusercontent.com/jiejiejie2143/ansible-install-k8s/master/single-master.jpg)
多Master架构

![avatar](https://raw.githubusercontent.com/jiejiejie2143/ansible-install-k8s/master/multi-master.jpg)

### 部署命令
单Master版：（阿里云环境，先部署单master，再做slb的高可用）
```
ansible-playbook -i hosts single-master-deploy.yml -uroot -k
```
多Master版：
```
ansible-playbook -i hosts multi-master-deploy.yml -uroot -k
```

## 4、部署控制
如果安装某个阶段失败，可针对性测试.

例如：只运行部署插件
```
ansible-playbook -i hosts single-master-deploy.yml -uroot -k --tags addons
```
## 5、添加node
```
ansible-playbook -i hosts add-node.yml -uroot -k 
```

## 6、其他配置说明（为安装prometheus监控提前增加的）
增加了roles/master/templates/kube-apiserver.conf.j2的配置

1、开启了apiserver的agg聚会层注册
```
--requestheader-client-ca-file={{ k8s_work_dir }}/ssl/ca.pem \
--proxy-client-cert-file={{ k8s_work_dir }}/ssl/server.pem \
--proxy-client-key-file={{ k8s_work_dir }}/ssl/server-key.pem \
--requestheader-allowed-names=kubernetes \
--requestheader-extra-headers-prefix=X-Remote-Extra- \
--requestheader-group-headers=X-Remote-Group \
--requestheader-username-headers=X-Remote-User \
--enable-aggregator-routing=true \
```
2、调整了时区
```
--runtime-config=settings.k8s.io/v1alpha1=true \
```

开启kubelet的webhook认证，不然prometheus的adapter拿不到数据

增加了roles/node/templates/kubelet.conf.j2的配置
```
--authentication-token-webhook=true \
--authorization-mode=Webhook \
```
修改了roles/node/templates/kube-proxy-config.yml.j2的配置,开启了ipvs模式
```
mode: ipvs
ipvs:
  scheduler: "rr"
```
### common初始化时增加
```
- name: 修改内核参数，保证flannel和coredns网络正常，且调整ipvs模块的table full
  copy: src=k8s.conf  dest=/etc/sysctl.d/k8s.conf

- name: 使内核参数修改生效
  shell: sysctl -p /etc/sysctl.d/k8s.conf

- name: 添加内核模块，开启ipvs模式
  copy: src=ipvs.modules  dest=/etc/sysconfig/modules/  mode=0755

- name: 加载内核模块
  shell: bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack

- name: 安装常用软件
  yum: name={{ item }} state=present
  with_items:
    - "ipset"
    - "ipvsadm"
    - "ntpdate"
    - "bash-completion"

- name: 关闭ntpd服务
  service: name=ntpd state=stopped enabled=no

- name: 同步系统时间
  shell: ntpdate ntp.aliyun.com
```