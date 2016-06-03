# CentOS 7 Cobbler安装说明及相关配置

## 准备工作

* **CentOS 7 更换base和epel源**
<pre>
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
</pre>

* **关闭selinux、firewall（防火墙）**

<pre>
systemctl disable selinux
systemctl disable firewalld
</pre>

## 配置相关服务

### 安装需要的相关服务

<pre>
yum install -y httpd dhcp tftp cobbler cobbler-web pykickstart
</pre>

### 开启httpd和cobblerd服务

<pre>
systemctl start httpd
systemctl start cobblerd
</pre>

### 查看cobbler需要修改的配置

<pre>
cobbler chrck
</pre>

### 修改/etc/cobbler/settings相关配置 (注意冒号后面的空格,红色为cobbler服务器地址)

<pre>
 242	manage_dhcp: 1
 258	manage_tftpd: 1
 261	manage_rsync: 1
 272	next_server: <font color=#FF00>192.168.56.11</font>
 292	pxe_just_once： 1
 384	server: <font color=#FF00>192.168.56.11</font>
</pre>

### 修改 /etc/xinetd.d/tftp 配置

<pre>
disable = yes  
=>  
disable = no
</pre>

### 修改系统安装root密码

* **生成密码：**

<pre>
openssl passwd -l salt '任意字符串' '你的密码'
</pre>

* **將生成的密码导入/etc/cobbler/settings文件**

<pre>
 101	default_password_crypted: "生成的密码"
</pre>

### 修改cobbler_web登录密码(添加cobbler用户，并以交互式输入密码)

<pre>
htdigest /etc/cobbler/users.digest "Cobbler" cobbler
</pre>

### 修改dhcp服务模板文件 /etc/cobbler/dhcp.template,只需要修改红色部分为自己的网段地址，其他保持默认。

<pre>
......
subnet <font color=#FF00>192.168.56.0</font> netmask 255.255.255.0 {  
     option routers             <font color=#FF00>192.168.56.2</font>;  
     option domain-name-servers <font color=#FF00>192.168.56.2</font>;  
     option subnet-mask         255.255.255.0;  
     range dynamic-bootp        <font color=#FF00>192.168.56.150 192.168.56.200</font>;
...... 
</pre>

### 联网下载引导文件（pxelinux.0,menu.c32,elilo.efi或yaboot）

<pre>
cobbler get-loaders
</pre>

### 重启服务httpd、cobblerd、xinetd服务

<pre>
systemctl restart httpd
systemctl restart cobblerd
systemctl restart xinetd 
</pre>

## 镜像导入及相关安装参数修改

### 导入系统镜像

* **方法一上传镜像：**

<pre>
mount -t auto -o loop /data/tools/CentOS-7-x86_64-DVD-1511.iso /mnt
cobbler import --path=/mnt --name=CentOS-7-x86_64 --arch=x86_64
</pre>

* **方法二光驱挂载：**

<pre>
mount /dev/cdrom /mnt
cobbler import --path=/mnt --name=CentOS-7-x86_64 --arch=x86_64
</pre>

### 查看当前可用镜像

<pre>
cobbler profile list
</pre>

### 修改默认ks文件为指定文件(需提前上传至目录)

<pre>
cobbler profile edit --name=CentOS-7-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg 
</pre>

### 修改内核参数，使网卡显示为eth*

<pre>
cobbler profile edit --name=CentOS-7-x86_64 --kopts='net.ifnames=0 biosdevname=0'
</pre>

## 指定机器安装相应系统及初始化

### 机器信息及规划信息

<pre>
mac：00:50:56:37:81:63
hostname：linux-node3.example.com
ip：192.168.56.13
subnet：255.255.255.0
gateway：192.168.56.2
dns:223.5.5.5
</pre>

### cobbler配置命令

<pre>
cobbler system add --name=linux-node3.example.com --mac=00:50:56:37:81:63 --profile=CentOS-7-x86_64 \
--ip-address=192.168.56.13 --subnet=255.255.255.0 --gateway=192.168.56.2 --interface=eth0 \
--static=1 --hostname=linux-node3.example.com --name-servers="223.5.5.5" \
--kickstart=/var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg
</pre>

### cobbler+ koan重装系统

* 需要重装系统的机器安装koan

<pre>
yum install -y koan
</pre>

* 查看cobbler上可用镜像(红色为cobbler服务器地址)

<pre>
koan --server=<font color=#FF00>192.168.56.11</font> --list=profiles
</pre>

* 选择系统版本进行重装

<pre>
koan --replace-self --server=<font color=#FF00>192.168.56.11</font> --profile=CentOS-6-x86_64
</pre>