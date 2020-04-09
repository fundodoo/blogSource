---
title: Linux 基础
categories:
  - Doc
top: false
date: 2020-04-08 14:05:29
tags:
- Linux
keywords: Linux,Linux常用命令
description: 
---

## Linux文件系统

### 文件类型

### Linux目录结构

### 常见目录说你
- **/bin**: 存放二进制可执行文件（`ls、cat、mkdir`等），常用命令一般都在这里
- **/etc**: 存放系统管理和配置文件
- **/home**: 存放所有用户文件的根目录，是用户目录的基点；eg：用户drunk的祝目录就是/home/drunk，可以用～drunk表示
- **/usr**: 用于存放系统应用程序
- **/opt**: 额外安装的可选应用程序包所放置的位置。一般情况下，我们可以将nginx安装到这里
- **/proc**: 虚拟文件系统目录，是系统内存的映射。可以直接访问这个目录来获取系统信息
- **/root**: 超级用户（系统管理员）的主目录，**特权**阶级
- **sbin**: 存放二进制可执行文件，只有`root`才能访问
- **/dev**: 存放设备文件
- **/mnt**: 系统管理员安装临时文件系统的安装点，系统提供这个目录是让用户临时挂载其他的文件系统
- **/boot**: 存放用于系统引导使用的各种文件
- **/lib**: 存放着和系统运行相关的库文件
- **/tmp**: 用于存放各种**临时**文件，是公用的临时文件存储点
- **/var**: 存放运行时需要改变数据的文件，eg：各种服务的日志文件、系统启动日志等
- **/lost+found**: 系统非正常关机而留下‘**无家可归**’的文件


## Linux常用命令

### 目录切换命令
- `cd usr`: 切换到<span style="color:green;">当前</span>目录下的usr目录
- `cd .. (或cd ../)`: 切换到上一层目录
- `cd /`: 切换到<span style="color:red;">**系统**</span>根目录
- `cd ~`: 切换到<span style="color:blue;">**用户**</span>主目录
- `cd -`: 切换到上一个所在目录

### 目录的增删改查命令
- `mkdir 目录名称`: 新建目录
- `ls 或者 ll`: 查看目录信息（`ll` 是 `ls -l` 的缩写，`ll`查看该目录下的所有目录和文件的详细信息）
- `find 目录 参数`: 查找目录
- `mv 目录名称 新目录名称`: 修改目录的名称 （注意：`mv`不仅可以对目录进行重命名，还可以对**各种文件**重命名）
- `mv 目录名称 目录的新位置`: 移动目录的位置--剪切
- `cp -r 目录名称 目录拷贝的目标位置`: 拷贝目录，`-r` 代表递归拷贝
- `rm [-rf] 目录`: 删除目录

### 文件的增删改查命令
- `touch 文件名称`: 创建文件
- `cat/more/less/tail 文件名称`： 文件的查看
  - `cat`: 只能显示**最后一屏**内容
  - `more`: 可以显示百分比，`回车`可以向下一行，`空格`可以向下一页，`q`可以退出查看
  - `less`: 可以使用键盘上的`PageUp`和`PageDown`向上和向下翻页，`q`结束查看
  - `tail -10`: 查看文件的后`10`行，`Ctrl + C` 结束
- `vim 文件`: 修改文件内容。<span style="color:red;">注意：</span>`vim`编辑器是linux中的强大组建。 vim 文件-->进入文件-->命令模式-->输入`i`进入编辑模式-->输入`Esc`进入底行模式-->输入`:wq/q!`(`wq`:写入内容并退出，即保存；`q!`:强制退出不保存。)
- `rm -rf 文件`: 删除文件 （同目录删除）

### 压缩文件的操作命令

#### 打包并压缩文件
> Linux中的打包文件一般是以`.tar`结尾的，压缩命令一般是以`.gz`结尾的。
> 一般情况下打包和压缩是一起进行的，打包并压缩后的文件的后缀一般是.tar.gz。命令：`tar -zcvf 打包压缩后的文件名 要打包压缩的文件`。
> z: 调用gzip压缩命令进行压缩
> c: 打包文件
> v: 显示运行过程
> f: 指定文件名

- eg：
  test目录下有a.txt,b.txt,c.txt三个文件，将test目录打包并命名为test.tar.gz的命令为：`tar -zcvf test.tar.gz a.txt b.txt c.txt 或 tar -zcvf test.tar.gz /test/`

#### 解压压缩包
> 命令： `tar [-xvf] 压缩文件`
> x: 代表解压

- eg：
  1. 将/test下的test.tar.gz解压到**当前**目录下：`tar -xvf test.tar.gz`
  2. 将/test下的test.tar.gz解压到指定的根目录/usr下: `tar -xvf text.tar.gz -C /usr` (<span style="color:red;">-C</span> : 指定解压后文件存放位置)


### 其他常用命令
- `pwd`: 显示当前所在的位置
- `grep 要搜索的字符串 要搜索的文件 --color`: 搜索命令，--color 代表高亮显示 
- `ps -ef/ps aux`: 这两个命令都是查看系统正在运行的进程，两者的区别是展示格式不同。如果要查看特定进程可以使用：`ps aux|grep redis` (查看包括redis字符串的进程)
- `kill -9 进程的pid`: 杀死进程 （<span style="color:red;">-9</span> 表示**强制**终止）
- **网络通信命令**
  - `ifconfig`: 查看当前系统的**网卡**信息
  - `ping`: 查看与某台机器的**连接情况**
  - `netstat -an`: 查看当前系统的**端口**使用
- `shutdown`: 
  - `shutdown -h now`: **立即**关机
  - `shutdown +5 "System will shutdown after 5 minutes"`: 指定5分钟后关机，同时送出警告给登陆的用户
- `reboot`: 重启机器
  - `reboot -w`: 模拟重启（只有记录并不会真的重启机器）

## scp命令详解
> scp是secure copy的缩写，scp是linux系统下基于ssh登陆进行**安全**的远程文件拷贝命令。linux的scp命令可以在linux服务器之间复制*文件和目录*。

### 语法（命令格式）
- scp [参数] [原路径] [目标路径]

### 参数说明
- `-1`: 强制scp命令使用**ssh1**协议
- `-2`: 强制scp命令使用**ssh2**协议
- `-4`: 强制scp命令只使用**IPV4**寻址
- `-6`: 强制scp命令只使用**IPV6**寻址
- `-B`: 使用**批处理**模式（传输过程中不询问传输口令或短语）
- `-C`: 允许**压缩**（将-C标志传递给ssh，从而打开压缩功能）
- `-p`: **保留**原文件的*修改时间*、*访问时间*和*访问权限*
- `-q`: 不显示传输进度条
- `-r`: **递归**复制整个目录
- `-v`: 详细方式显示输出。scp和ssh(1)会显示整个过程的调试信息，这些信息用于调试连接、验证和配置问题
- `-c cipher`: 以**cipher**将数据传输进行**加密**，这个选项将直接传递给ssh
- `-F ssh_config`: 指定一个替代ssh配置文件，此参数直接传递给ssh
- `-i identity_file`: 从指定文件中读取传输时使用的密钥文件，此参数直接传递给ssh
- `-l limit`: **限定**用户所能使用的**带宽**，以**Kbit/s**为单位
- `-o ssh_option`: 
- `-P port`: 注意P是**大写**，port是指定数据传输用到的端口号
- `-S program`: 指定加密传输时所使用的程序，此程序必须能够理解ssh(1)的选项

### 使用实例

- 实例1: 从远程服务器复制文件到本地目录
  ```sh
  // 从192.168.10.18机器上的/opt/soft/的目录中下载nginx-0.5.38.tar.gz文件到本地/opt/soft/目录中
  scp root@192.168.10.18:/opt/soft/nginx-0.5.38.tar.gz /opt/soft/
  ```

- 实例2: 从远程服务器复制文件夹及其里面的文件到本地目录
  ```sh
  // 从192.168.10.18机器上的/opt/soft/的目录中下载mongodb目录到本地/opt/soft/目录中
  scp -r root@192.168.10.18:/opt/soft/mongodb /opt/soft/
  ```

- 实例3: 上传本地文件到远程机器的指定目录
  ```sh
  // 复制本地/opt/soft/目录下的nginx-0.5.38.tar.gz到远程服务器192.168.10.18的/opt/soft/scptest目录
  scp /opt/soft/nginx-0.5.38.tar.gz root@192.168.10.18:/opt/soft/scptest
  ```

- 实例4: 上传本地目录到远程机器的指定目录
  ```sh
  // 复制本地/opt/soft/mongodb目录到远程服务器192.168.10.18的/opt/soft/scptest目录
  scp -r /opt/soft/mongodb root@192.168.10.18:/opt/soft/scptest
  ```