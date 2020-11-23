# 简介

在云计算中，一个重要的概念是黄金镜像（Golden images）可以在中心服务器上管理（其消耗比VDI小），当用户联网的时候，更新和补丁程序可以集中安装到客户端虚拟机上。

在创建Homelab 过程中我想要将系统的安装，变得自动化，智能化，Infrastructure as code 于是我使用了Packer

Packer <https://www.packer.io/>

Packer 是我在德国学习过程中使用的一个工具， 支持镜像的定制和创建。 在德国的项目中我们是使用Packer 在AWS 上创建基础镜像。 然而Packer 在支持创建云镜像的同时， 也支持在VMware VSphere 中创建虚拟机的Template（一种镜像）。

感谢 Guillermo Musumeci 提供的详细指导作为我的入门指南

<https://gmusumeci.medium.com/how-to-use-packer-to-build-a-centos-template-for-vmware-vsphere-dea37d95b7b1>

感谢 Sam Gabrail 提供Github 代码库

<https://github.com/samgabrail/packer-terraform-vmware>



## 知识点

* Packer的安装
* Packer Template （JSON file）with VSphere ISO
* CentOS Kickstart installation
* Password encryption by MD5, SHA256, SHA512

# 步骤

## 1. 安装Packer

### 官方安装文档

<https://learn.hashicorp.com/tutorials/packer/getting-started-install?in=packer/getting-started>

我的操作系统为 Mac OS +CentOS 7 for password hash

### Shell commands

```
# Install and upgrade packer 
brew tap hashicorp/tap
brew install hashicorp/tap/packer
brew upgrade hashicorp/tap/packer
# Verifying the installation 
packer version
```

## 2. 理解Packer template file

在使用Packer 创建黄金镜像中我们需要提供指导 packer 如何创建镜像的配置文件configuration file。在Packer 的术语中称之为Template。

Template 文件是一种用JSON格式的让人和机器都能读懂的配置文件。

CentOS 是我在日常环境中每日使用的操作系统。 这里我比较熟悉， 因此我在学习过程中先使用CentOS 作为实验目标。

在Templare 文件中包含了两个重要的组成部分

* Builders
* Variables.json

Packer builder: 在这里我们使用 vsphere-iso， 这个Packer builder 可以让我们从ISO 文件来创建黄金镜像。

Variables 中 可以存储我们在配置中需要填写的变量。例如VSphere 的用户名密码等等信息。

这些信息是敏感的，并且需要经常编辑的， 并且我也不想将这些敏感的信息上传到代码库中。因此我会将 Template 文件拆分为两个文件。vaiables.json (Vaiables 配置文件)和centos-vsphsre.json（Builder 配置文件）。 在实际使用中这两个文件可以合并成一个JSON 文件。

### centos-vsphere.json

<https://www.packer.io/docs/builders/vmware/vsphere-iso> 在这个文档中列出了所有可配置项。

创建镜像时间无非需要提供 以下几个关键的信息。

在哪里创建？ 使用几个CPU？使用多少内存？使用多少存储？ 网络是什么样子的？

一些自定义的配置项。比如；启动顺序是什么？启动需要等待多久？启动后需要执行什么命令？

具体的配置，以及怎样配置，需要个人耐下性子来去读文档。这个无法帮助每个人。

### Variable.json

 在这个文件中我们需要提供给Packer builder 一些必要的参数。这些参数会传到centos-vsphere.json 文件中

| Variable name        | Value                                               | Note                                                         |
| -------------------- | --------------------------------------------------- | ------------------------------------------------------------ |
| iso_url              | [vm-storage]  os-image/CentOS-7-x86_64-DVD-2009.iso | [] 中支出VSphere 中Datastore 的名称。以及ISO文件在VSphere datastore 中的存放路径 |
| vm-cpu-num           | 1                                                   |                                                              |
| vm-disk-size         | 25600                                               |                                                              |
| vm-mem-size          | 1024                                                |                                                              |
| vm-name              | CentOS7-Template                                    |                                                              |
| vsphere-cluster      | homelab                                             |                                                              |
| vsphere-datacenter   | Home-Lab                                            |                                                              |
| vsphere-datastore    | vm-storage                                          |                                                              |
| vsphere-network      | VM Network                                          |                                                              |
| vsphere-network-card | vmxnet3                                             |                                                              |
| vsphere-password     |                                                     | 用于登陆VSphere 的密码                                       |
| vsphere-server       | md-homelab-vc-home01.m-daily.systems                | Vcenter server 的地址                                        |
| vsphere-user         | administrator@m-daily.systems                       | VSphere 用户名                                               |

## 3. CentOS Kickstart

CentOS Kickstart 又是一个很深的只是点， 简单来说， 在CentOS操作安装过程中，你鼠标点点点的所有过程都可以通过kickstart 文件来配置。

这对我们 自动化安装CentOS 起到了非常关键的作用。

文档如下， 再次强调，需要你耐下性子静心去读文档。

<https://docs.centos.org/en-US/centos/install-guide/Kickstart2/>

以下 还是取出其中关键的部分进行讲解

### ks.cfg

* 选择系统安装语言

  lang en_US.UTF-8 英语

* 选择日期和时间

  timezone --utc Asia/Shanghai

* 选择键盘

  keyboard es

* 选择磁盘分区

  clearpart --linux --initlabel

* 设置root 用户密码

  authconfig --enableshadow --passalgo=sha512

  rootpw --iscrypted $6$rhel6usgcb$aS6oPGXcPKp3OtFArSrhRwu6sN8q2.yEGY7AIwDOQd23YCtiz9c5mXbid1BzX9bmXTEZi.hCzTEXFosVBI5ng0

补充知识

语言的list：

* 怎么获取： cat /usr/share/system-config-language/locale-list

时区的选择

* 怎么获取：<https://www.tecmint.com/set-time-timezone-and-synchronize-time-using-timedatectl-command/>
* 完整列表：<https://gist.github.com/adamgen/3f2c30361296bbb45ada43d83c1ac4e5>

键盘选择： 

* 怎么获取： /usr/lib/kbd/keymaps/

设置root 用户密码：

* 设置加密方式： 可以使用的有 MD5， SHA265， SHA512

  authconfig --enableshadow --passalgo=sha512

* 设置密码： rootpw --iscrypted $6$rhel6usgcb$aS6oPGXcPKp3OtFArSrhRwu6sN8q2.yEGY7AIwDOQd23YCtiz9c5mXbid1BzX9bmXTEZi.hCzTEXFosVBI5ng0

  怎么得到密码的HASH 值： <https://thornelabs.net/posts/hash-roots-password-in-rhel-and-centos-kickstart-profiles.html> 

  以下的命名只能在CentOS 上执行

  ```shell
  python -c 'import crypt,getpass;pw=getpass.getpass();print(crypt.crypt(pw) if (pw==getpass.getpass("Confirm: ")) else exit())'
  Password:
  Confirm:
  $6$UyijhqrI6JO3tDMk$Gikp7zf/ZBIU0pOCwZYNc6jC6ytUjFQhlulChlKYzocdf48jA2WELQmlWZ.km3****************************0frC/
  ```

  

设置bootloader password protect： 

* 怎么获取： grub2-mkpasswd-pbkdf2

* 以下的命名只能在CentOS 上执行

  ```shell
  [root@CentOS7Template ~]# grub2-mkpasswd-pbkdf2
  Enter password:
  Reenter password:
  PBKDF2 hash of your password is grub.pbkdf2.sha512.10000.391823E82ED367F55306D565D5CD5FEA4BD7A4A358C43183EB5E6FDFD6C1AB357ED7F6852DE8F7D3B7C2C416F57CBAD27DD5933FF93FB8F58D966D7A0BDD1D40.8B650AA75C60D6CFA14960DC1A63FB916809E449FBD37EBF33661************************************E9175E377F04469504210FCE
  ```

  

# Shell commands

```shell

packer build -var-file=variables.json centos-vsphere.json
```

```shell
packer build -var-file=md-variables.json md-centos-vsphere.json
```



# Reference

* Packer official documents: <https://www.packer.io/docs>
* Packer install guide: <https://learn.hashicorp.com/packer/getting-started/install>
* Packer VMware VSphere ISO : <https://www.packer.io/docs/builders/vmware/vsphere-iso>
* CentOS Kickstart installations: <https://docs.centos.org/en-US/centos/install-guide/Kickstart2/>
* CentOS 7 Kickstart Syntax reference: <https://docs.centos.org/en-US/centos/install-guide/Kickstart2/#sect-kickstart-syntax>
* how to using encrypt to protect centos password: <https://thornelabs.net/posts/hash-roots-password-in-rhel-and-centos-kickstart-profiles.html>
* How to use packer to build a CentOS tempalte for VMware VSphere: <https://gmusumeci.medium.com/how-to-use-packer-to-build-a-centos-template-for-vmware-vsphere-dea37d95b7b1>



And that’s all folks. If you liked this story, please show your support by 👏 this story. Thank you for reading!

