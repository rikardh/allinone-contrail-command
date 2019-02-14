# **All In One Contrail Networking and Openstack Kolla with Contrail Command**
#### 2019-02-14 by Rikard
###### Version 5.0.2-0.360 based

>*Before you read on, please understand that you need login credentials for Juniper Networks docker registry below.*

I'll cover 2 versions ment for testing/lab use only. For production installations, please refer to installation documents on juniper.net
These installation methods do not use the GUI of Contrail Command but just an instances.yml file from the command line. This is faster but less intuitive perhaps.
## **AIO**
An all-in-one install means that all roles will be installed on the same machine. Here we are using the following roles/functions:
- Contrail Command (New Juniper Networks GUI that includes the Fabric mangement utilities)
- Contrail Controller
- Compute node (which has the "vRouter" installed)
- Openstack (Openstack Kolla is used)

The instructions below are based on using a virtual machine runing Centos linux. Of course you can use a physical server however a physical bare metal server makes less sense for a lab, IMHO. Choice is yours.

#### **2 versions of AIO below**
1. Single NIC for fabric/data and management
2. Dual NIC. NIC 1 for fabric/data access and NIC 2 for management access. 
___
## **1. Single NIC version**
Create a virtual machine. Configure the VM with:
- 2x vCPUs (with hw virtulaization turned on if you want to spawn any small VM) 
- 32G RAM (for VM's add more. This should fit 2x Cirros though)
- 50G disk
- Single NIC
Install Centos 7.5 (1804) minimal (file CentOS-7-x86_64-Minimal-1804.iso)
- Configure it as per below if you dont have a DHCP server doing it all for you:
>I always choose the root password of "c0ntrail123" for my personal labs because this is the default in the config files and I'm really lazy. The second character is a zero.
```
[root@localhost ~]# cat /etc/centos-release
CentOS Linux release 7.5.1804 (Core)
```

### **Configure Centos basics**  
---
Set your hostname to "aio" using. 

   ```
[root@localhost ~]# hostnamectl set-hostname aio
```   
   >(logout and login in again for name to change on the prompt.)  

   Now check which interface we need to configure for IP details:
```
[root@aio ~]# ip ad
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 00:50:56:a8:ec:84 brd ff:ff:ff:ff:ff:ff
    inet 172.30.104.192/23 brd 172.30.105.255 scope global dynamic ens160
       valid_lft 3370sec preferred_lft 3370sec
    inet6 fe80::68fa:fea7:a609:9428/64 scope link
       valid_lft forever preferred_lft forever
[root@aio ~]#
```
As you can see I'm using “ens160”. You need to edit your interface appropriately unless that config via DHC is what you want.  
Lets edit with the "vi" editor.
```
[root@aio ~]# vi /etc/sysconfig/network-scripts/ifcfg-ens160
```
>Replace "ens160" in the line above with the name of your interface.  
Inside "vi" press “i” to insert. Edit. When done press “esc”, then “:” and type “x” to exit saving changes.

My "ifcfg-ens160" looks like this:
```
TYPE="Ethernet"
BOOTPROTO="none"
NAME="ens160"
DEVICE="ens160"
ONBOOT="yes"
IPADDR=172.30.104.53
PREFIX=23
GATEWAY=172.30.104.1
DNS1=172.30.104.10
```
...and restart the network service:
```
[root@aio ~]# systemctl restart network
```

---
### **Continue with some packages**...
Make sure you have internet access and now run:
```
  yum install -y yum-utils device-mapper-persistent-data lvm2
  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  yum install -y docker-ce
  systemctl start docker
  systemctl enable docker
```
You need 2 files to move forward. Put these in the /root directory of your AIO server. Otherwise you'll need to change the path in the docker command further down where you see "/root...."  
The files are included in the repo and called:  
```
command_servers.yml
instances.yml
```
You need to change some stuff in them. I have marked changes required with "<>". This is where things usually go wrong like missing information, wrong info or just plain typos. I mean, we are all in a hurry ;)
  
Now you're gonna need those **login credentials** for the docker registry. Make sure you have this before moving forward.

Login to registry:
```
[root@aio ~]# docker login hub.juniper.net
```
With a successful login, run the command below to install Contral Command and thereafter Contrail Networking with Openstack Kolla.
```
[root@aio ~]# docker run -t --net host -e action=provision_cluster -v /root/command_servers.yml:/command_servers.yml -v /root/instances.yml:/instances.yml -d --privileged --name contrail_command_deployer hub.juniper.net/contrail/contrail-command-deployer:5.0.2-0.360
```
To see Contrail Command installing:
```
[root@aio ~]# docker logs -f contrail_command_deployer
```
Contrail Command is successfully installed when you see the below text. "failed=0" is what we want.
```
TASK [import_cluster : Import/Provision cluster] *******************************
ok: [172.30.104.53]

PLAY RECAP *********************************************************************
172.30.104.53              : ok=35   changed=7    unreachable=0    failed=0
localhost                  : ok=4    changed=2    unreachable=0    failed=0

[root@aio ~]# 
```
If Command is installed successfully Contrail/Openstack will install. Watch this with:
```
[root@aio ~]# docker exec contrail_command tail -f --retry /var/log/ansible.log
```
All is successfully installed when you see the below text. Again, "failed=0" is what we want.
Like this:
```
2019-02-14 03:28:34,057 p=8600 u=root |  TASK [install_contrail : start contrail vcenter-manager] ***********************
2019-02-14 03:28:34,216 p=8600 u=root |  skipping: [172.30.104.53]
2019-02-14 03:28:34,226 p=8600 u=root |  PLAY RECAP *********************************************************************
2019-02-14 03:28:34,227 p=8600 u=root |  172.30.104.53              : ok=72   changed=47   unreachable=0    failed=0
2019-02-14 03:28:34,227 p=8600 u=root |  localhost                  : ok=36   changed=1    unreachable=0    failed=0
^@^C
[root@aio ~]# 
```
If you get "failed=0" on all lines above, you are done!
Also status check with:
```
[root@aio ~]# contrail-status
```
All should show "active".
>You might want to keep an eye on RAM and disk usage:
```
[root@aio ~]# free -h
              total        used        free      shared  buff/cache   available
Mem:            31G         23G        6.3G         15M        1.9G        7.6G
Swap:            0B          0B          0B

[root@aio ~]# df -h | grep -1 root
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   44G   32G   13G  72% /
devtmpfs                  16G     0   16G   0% /dev
[root@aio ~]#
```
### **Accessing your setup**
When install has succeeded, browse to:

|What |Where|
|---:|---|
Contrail Command GUI: | https://\<your ip>:9091
Contrail "old" GUI: | https://\<your ip>:8143
Openstack Horizon GUI: | http://\<your ip>


  > Default username/pass is root/contrail123
___
## **2. Dual NIC version**
Will start soon.