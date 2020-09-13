
ps -aux | grep xxx

### 帮助命令

manual

man [num]  \[-a] command

type command

内部   help  command

外部   command --help

info ls



### vi command

Esc 退回正常模式

#### 插入模式

正常模式进入： i  I  a  A  o O



光标移动   h j k l

[num]yy 复制几行    y$ 光标到行尾  

剪切  dd   d$

粘贴 p （paste）

撤销  u（undo）

ctrl r   前进

x 删除字符 

r 替换 



:set nu





shift g



#### 命令模式  :

: 末行模式

:w [/path/filename] 保存

:q  退出

:!   临时执行其他命令

:/x  查找x    n 下一个    shift n 上一个

:s/old/new  替换光标所在行

:%s/old/new 全替换

:[num,num]s   替换指定行区间 

:set   单次修改     nu 显示行号

持久修改 改配置

vim /etc/vimrc



#### 可视模式

v  字符

V 行 

ctrl+v 块



### 用户管理

root

useradd username  添加用户

userdel username 删用户    -r 删数据（/home/username）

usermod  chage

 

passwd username   设置密码



id  username



groupadd   groupdel



su 切换用户

su - username  使用 login shell 方式切换用户

sudo 以其他用户身份执行命令

visudo   设置需要使用sudo 的用户（组）  



shutdown -h

shutdown -c



查找 which  xxx

grep



用户配置文件

/etc/passwd

username:x(需要密码验证)\:userid\:groupid:注释:用户目录:/bin/bash(命令解析器)



/etc/shadow

username:$xidfw$(加密后密码)::::

用户组配置文件

/etc/group



### 文件管理

more，cat，find，cut，cmp

mv，cp，mkdir，rm

ln，touch，split

* 文件权限修改

chmod  u+x  修改文件，目录权限    (u  g  o  a)  (+-=)  (rwx)

chmod  777     4   2   1  0

chown  username/:groupname  file/directory 更改属主，属组

chgrp  单独更改属组

特殊权限 

suid   用于二进制执行文件，执行命令时取得文件属主权限

sgid   用于目录，在该目录下新建目录和文件，权限自动更改为该目录属组

sbit   用于目录，在该目录下新建目录和文件，仅 root 和自己可以删除



root用户 拥有最高权限 不受限制

文件权限

> \-        (r w -) (- - -) (- - -)  1 username usergroup  xxx  mtime  filename
>
> 类型  权限(文件属主权限/文件属组权限/其他用户权限) 
>
> 类型  ： - 普通 d 目录 b 块特殊 c 字符特殊 l 符号链接 f 命名通道 s 套接字
>
> 权限 ： r=4 读/ w=2 写 / x=1 可执行   chmod 764
>
> 目录文件 x 进入目录  rx 显示目录内文件名 wx 修改目录内文件名

### 文档编辑

sort，let，grep

### 文件传输

ftp，ncftp

### 磁盘管理

cd，df，du，mkdir，pwd，rmdir，ls

## 系统管理

sleep，kill，top，

ps -ef //显示所有命令，连带命令行 process

ps -aux 显示所有包含其他使用者的行程

## 系统设置

clear，enable，passwd，logout
su，sudo，whois

```
export [-fnp][变量名称]=[变量设置值]  命令用于设置或显示环境变量
free //显示内存使用信息
```



### 备份压缩





## 其他

第一部分：


	3：Linux的命令行与图形界面切换
			1：命令行界面
				1：在图形化界面中打开命令行窗口
					应用程序-》附件-》终端			
				2：切换到纯命令行是使用Ctrl加Alt加F1~F6：
					注意： 使用ctrl+alt 和Vmware软件有冲突，需要修改VMware的快捷键
					（在VMware左上角edit->preferences->hot keys，将ctrl+alt 修改为ctrl+shift+alt）
					1.Ctrl+Alt+F1  切换到控制台1（tty1）
					2.Ctrl+Alt+F2  切换到控制台2（tty2）
					3.Ctrl+Alt+F3  切换到控制台3（tty3）
					4.Ctrl+Alt+F4  切换到控制台4（tty4）
					5.Ctrl+Alt+F5  切换到控制台5（tty5）
					6.Ctrl+Alt+F6  切换到控制台6（tty6）
	            3: 使用who命令查看当前已经登录的用户
		    2：图形用户界面(Graphical User Interface，简称 GUI)
			    2：ctrl+alt+f7切换到图形用户界面
	4: 普通用户与超级管理员
		1:显示“$”标识表示是普通用户
			例如： 进入linux系统，控制台出现：itcast@ubuntu:~$ 
			 ~用户的根目录
		2:显示“#”标识表示是超级管理员：
		3：切换用户
			切换用户（使用su命令切换用户）：
			当从普通用户切换到root用户（超级管理员）或其他用户时，需要输入目标用户的密码。
			当从root用户切换到普通用户时，不需要输入密码。
	5：Linux目录结构
		1：Linux所有内容是以文件形式进行管理
		2：/ 根目录
			1：bin  存放二进制可执行文件(ls,cat,mkdir等)
			2：boot 存放用于系统引导时使用的各种文件
			3：dev 用于存放设备文件
				dev是device的简写，就是“设备”的意思。Linux把每个硬件也看作是一个文件
			4：etc  存放系统配置文件
				1：例如安装jdk配置环境变量
			5：home 存放所有用户文件的根目录
				1：用户登录系统后默认所在的目录
			6：mnt  系统管理员安装临时文件系统的安装点
				例如：挂载光驱。
			7：opt  额外安装的可选应用程序包所放置的位置
				例如：我们可以安装自定义程序1：安装eclipse，安装tomcat
			8：root  超级用户目录
				1：管理员
			9：sbin  存放二进制可执行文件，只有root才能访问
			10：usr  用于存放系统应用程序，有些类似windows的Program Files
				 例如：软件中心下载的软件默认安装在usr/bin中。
				 我们也可以将jdk安装在此目录中。
			图形化界面查看
				Places -Documents-File System (/ 根目录)
第二部分：
Linux的常用命令：
​	1.注销、关机、重启命令
​		注销：logout或exit
​		关机：halt或shutdown -h now（要是root用户或是有授权才可以）
​		重启：reboot或shutdown -r now（要是root用户或是有授权才可以）
​	2：Linux的基本命令
​		1：ls 显示文件和目录列表 　
​			1： -l 列出文件的详细信息
​			2： -a 列出当前目录所有文件，包含隐藏文件
​		2：mkdir 创建目录 　（ 删除？rmdir  非空） 
​			1：-p 父目录不存在情况下先生成父目录
​		3：cd 切换目录
​		4：touch 生成一个空文件	
​		5：echo 生成一个带内容文件     
​			1：echo abcd>a.txt
​		6：cat、tac 显示文本文件内容
​		7：cp 复制文件或目录
​			1：cp a.txt /home/itcast/abc/ddd
​		8：rm 删除文件
​			1：rm a.txt
​			2：rm -rf abc
​		9：mv 移动文件或目录、文件
​			1：mv  aaa bbb 将aaa改名为bbb
​			2：mv bbb /home/itcast/abc/ccc
​		10：find 在文件系统中查找指定的文件
​			1：find  -name  文件名
​		11：wc 统计文本文档的行数，字数，字符数 
​			1：wc a.txt
​		12：grep 在指定的文本文件中查找指定的字符串
​			1：grep aa a.txt
​		13：pwd 显示当前工作目录 
​		命令练习：
​			创建一个目录 家庭A（目录）
​			进入familyA
​			家庭A中有一个父亲，母亲，女儿，儿子（4个空文件）
​			家庭有房子（目录）
​			房子有厨房，卫生间，3卧室（目录）
​			男孩房有床（空文件），有书（带内容的文件）
​			女孩房同样有床和书，女孩房有娃娃（空文件）。
​			男孩房也要有娃娃（空文件），男孩把娃娃删掉。将房间的沙发移动到男孩房。
​			删除厨房			
​		14：ln 建立链接文件（\*\*\*）
​			1：ln -s /home/itcast/familyA/house/roomB   /home/roomB 
​				1：当访问一个目录较深的文件，可以建立链接文件。
​				2： 遇到 Permission denied（权限拒绝）说明itcast用户没有权利做这件事
​					1：使用sudo 可以借用root的权限，输入itcast的密码
​				3：在home下就可以直接访问roomB的文件
​				4：例如安装jdk路径需要配置环境变量，如果路径较长书写麻烦可以配置连接文件
​			第一，ln命令会保持每一处链接文件的同步性，也就是说，不论你改动了哪一处，其它的文件都会发生相同的变化；
​			第二，ln的链接又 软链接和硬链接两种，软链接就是ln –s \*，它只会在你选定的位置上生成一个文件的镜像，不会占用磁盘空间
​			硬链接ln \*，没有参数-s， 它会在你选定的位置上生成一个和源文件大小相同的文件，无论是软链接还是硬链接，文件都保持同步变化。
　			如果你用ls察看一个目录时，发现有的文件后面有一个@的符号，那就是一个用ln命令生成的文件，用ls –l命令去察看，就可以看到显示的link的路径了。	
​		15：more、less 分页显示文本文件内容 
​			1：查看配置文件时，很长需要分页处理
​			2：more（一页一页翻）
​				1：空格键向下翻页
​				2：Enter键向下滚动一行
​				3：:f 显示出文件名及当前的行数
​				4：	q 离开more
​				5： b 往回翻
​			3：less（一页一页翻）
​				1：空格 向下翻一页
​				2：PageDown 向下翻一页
​				3：PageUp 向上翻一页
​				4：q 离开
​		16：head,tail分别显示文件开头和结尾内容
​		17：man 命令帮助信息查询
​			1：man ls
​		18：管道（***）
​			1：  cat /etc/passwd | wc -l
​				使用cat命令显示passwd文件中的内容，但是并没有显示在屏幕上，而是通过管道“|” 接受，wc命令从管道中取出内容进行统计，然后显示结果
​				这个输出时该文件有多少行（多少个用户）
​		19：重定向
​			1：>
​				cat /etc/passwd>/home/itcast/a.txt
​				echo "hello java">a.txt  （覆盖上一个a.txt）
​			2：>>
​			  1：追加，不会覆盖
​				cat /etc/passwd>>/home/itcast/a.txt 
​				echo "---------">>a.txt   			
​	3：Linux系统命令
​		1：stat 显示指定文件的相关信息 
​			1：stat familyA
​				access 进入
​				Modify 修改
​				Change 改变
​				access time是文档最后一次被读取的时间。因此阅读一个文档会更新它的access时间，
​                                但它的modify时间和change时间并没有变化。cat、more 、less、grep、tail、head这些命令都会修改文件的access时间。
​                                change time是文档的索引节点(inode)发生了改变(比如位置、用户属性、组属性等)；
​                                modify time是文本本身的内容发生了变化。[文档的modify时间也叫时间戳(timestamp)
​		2：who、显示在线登录用户 
​			1：想要知道当前有多少用户登录系统。
​			2：who
​				1：显示2个一个是命令行，一个是图形界面的只有一个itcast
​		3：whoami 显示用户自己的身份 
​		4：hostname 显示主机名称 
​			1：hostname
​			2：hostname -i 显示主机IP
​		5：uname 显示系统信息 
​			1：uname -a 显示全部信息 
​			Linux ubuntu 2.6.35-22-generic #33-Ubuntu SMP Sun Sep 19 20:34:50 UTC 2010 i686 GNU/Linux
​		6：top 显示当前系统中耗费资源最多的进程 动态显示过程，实时监控
​			1：类似于windows的任务管理器
​			2：主要看 cpu mem command
​			3；ctrl+c 退出，或者q
​		7：ps 显示瞬间进程状态
​			1：ps -aux  显示所有瞬间进程状态
​		8：du 显示指定的文件（目录）已使用的磁盘空间的总量 
​			1：du
​			2：du familyA  （以K为单位）
​			3：du -h familyA 
​		9：df 显示文件系统磁盘空间的使用情况 
​			1：df -h 
​		10：free 显示当前内存和交换空间的使用情况 	
​		11：ifconfig 显示网络接口信息 
​			1：windows 是ipconfig
​		12：ping 测试网络的连通性 
​		13：clear 清屏
​		14：kill 杀死一个进程
​		15：关机/重启命令 
​			1：shutdown 命令可以安全的关闭Linux系统，shutdown命令必须有超级用户才能执行。shutdown命令执行后会以广播的形式通知正在系统中工作的所有用户，
​				1：shutdown  -h now  （关机不重启）
​				2：shutdown  -r now  （关机重启）
​				3：shutdown  now （关机）
​				4：shutdown  15:22
​			2：halt 关机后关闭电源 
​			3：reboot 重新启动
​	4：备份压缩命令
​		1：tar
​			1:打包
​				1：tar -cvf familyA.tar familyA （tar -cvf 保存路径/包名 打包目录）
​			2：拆包
​				1：tar -xvf /home/itcast/familyA.tar 
​		2：gzip 命令
​			gzip 压缩（解压）文件，压缩文件后缀为gz 
​			1：压缩
​				1：把/home/itcast目录下的familyA目录下所有文件压缩成.gz文件
​					1：gzip只能压缩文件，目录（文件夹不能处理），需要使用tar对文件夹打包
​					1：gzip familyA.tar 进行压缩

			2：查看压缩文件
				1：gzip -l familyA.tar.gz 查看压缩包详细信息
					1：compressed 压缩后大小
					2：uncompressed 原始大小
					3：ratio  压缩比
					4：uncompressed_name  原始文件名
			3：解压
				1：gzip -d familyA.tar.gz   显示文件名和压缩比
			4：压缩比
				1：高压缩（速度稍慢）
					gzip -9 familyA.tar 高压缩比
					gzip -l familyA.tar.gz 
				2：低压缩比（速度快）
					gzip -d familyA.tar.gz （解压）
					gzip -1 familyA.tar 低压缩比
					gzip -l familyA.tar.gz
				3：默认是6
		3：bzip2 命令
			bzip2 压缩（解压）文件或目录，压缩文件后缀为bz2 
			1：压缩
				1：把/home/itcast目录下的familyA目录下所有文件压缩成.bz2文件
					1：bzip2 -z familyA.tar 压缩需加上参数-z
			2：解压缩
				1：bzip2 -d familyA.tar.bz2 
		4：tar命令压缩和解压
			1：将整个/home/itcast/familyA目录下的文件全部打包成为/home/itcast/familyA.tar
				1：仅打包，不压缩
					1：tar -cvf familyA.tar familyA
				2：打包后，以gzip压缩
					1：tar -zcvf familyA.tar.gz familyA
					拆包
					sudo tar -zxvf familyA.tar.gz
				3：打包后，以bzip2压缩
					1：tar -jcvf familyA.tar.bz2 familyA
					拆包
					sudo tar -jxvf familyA.tar.bz2 
第三部分：
软件管理
​	1：直接安装.deb包 dpkg软件包
​		1：安装以.deb结尾的软件包，需要使用root的权限
​			1：sudo dpkg -i 软件包名
​		2：卸载
​			1：sudo dpkg -r 软件包名
​		3：安装tree
​			1：进入file文件 cd /home/itcast/Desktop/file
​			2：sudo dpkg -i tree_1.5.3-1_i386.deb
​		4：看Ubuntu系统已安装所有软件包列表
​			1：sudo dpkg -l
​		很多情况软件安装:/usr/local之下
​	2：使用apt-get管理软件（推荐使用）
​		1：前提连接互联网，自动搜素，自动下载自动安装
​		2：安装
​			1：sudo apt-get install eclipse
​			2：sudo apt-get install openjdk-6-jdk
​		3：卸载
​			1：sudo apt-get remove packagename   
​		3：使用广泛，默认下载路径
​			1：/var/cache/apt
​		同时也可以可以更新和搜索软件
​		1.执行“apt-get update”更源列表
​		2.执行“apt-cache search 名称”搜索软件	
​		ubuntu系统如何查看软件安装的位置？装到哪里去了，可以这样看：
​		代码:	
​			dpkg -L  软件名
​		软件中心下载的软件默认保存路径：/var/cache/apt/archives.
​	9：vim编辑器
​		编辑模式
​		插入模式
​		命令模式
​		VIM的运行模式 <C-F12>
​		编辑模式：等待编辑命令输入
​		插入模式：编辑模式下，输入 i 进入插入模式，插入文本信息
​		命令模式：在编辑模式下，输入 “：” 进行命令模式	
​		1：安装
​			1：安装进入入file文件 cd /home/itcast/Desktop/file
​			2：sudo dpkg -i vim-runtime_7.2.330-1ubuntu4_all.deb
​			3：sudo dpkg -i vim_7.2.330-1ubuntu4_i386.deb  
​			4：检查安装成功 敲入vim即可 退出 :q。
​		2：在/home/itcast/目录下建立一个bank.txt文件
​			1：cd /home/itcast/familyA/
​			2：touch bank.txt
​		3：使用vim编辑
​			1：	vim bank.txt
​			2：数据命令i 进入插入模式
​			3：输入内容
​				ICBC
​				RMB:10000000000
​				USD:100000000000
​				user:familyA.father
​			4：ctrl+C 退出插入模式或者敲ESC切换至命令模式
​			5：:wq 回车 保存
​		4：编辑bank.txt 内容不保存 退出
​			1：vim bank.txt		
​			2：数据命令i 进入插入模式
​			3：随便输入内容
​			4：ctrl+C 退出插入模式或者敲ESC
​			5：:q! 回车 强制退出
​			6：cat bank.txt  使用cat查看内容
​		5：显示行号
​			1：vim bank.txt		
​			2：:set number 回车
​			3：:q 回车 正常退出
​		6：取消行号
​			1：：set nonumber
第四部分：
​	搭建Java服务器并进行远程管理
​	查看linux 是否开始SSH 服务(是否安装服务端)
​	sudo apt-cache policy openssh-client openssh-server
​	1：搭建SSH服务器			
​		1：在Ubuntu Linux服务器上安装SSH软件
​			执行“apt-get install openssh-server”			
​		2：在Windows客户机上访问SSH服务器
​			使用“putty.exe”登录服务器（命令行），可用于在服务器端执行命令。
​			使用“winscp.exe”连接服务器，可用于上传下载文件。
​	2：搭建Java环境（JDK6）
​		方法一：执行“apt-get install openjdk-6-jdk”	
​		注意: 需要联网.				
​		方法二：
​		1.进入opt目录
​			cd /opt
​		2.在opt目录下建立java安装目录
​			sudo mkdir java
​		3.将jdk-6u24-linux-i586.bin拷贝到java目录下
​			sudo cp /home/itcast/Desktop/file/jdk-6u39-linux-i586.bin /opt/java
​		4.安装jdk
​			cd /opt/java
​			sudo ./jdk-6u39-linux-i586.bin	
​		配置JAVA_HOME			
​		修改/ect/profile文件
​		编辑配置文件
​			sudo vim /etc/profile
​		添加如下内容：	
​			export JAVA_HOME="/opt/java/jdk1.6.0_39"
​			export PATH="$JAVA_HOME/bin:$PATH"
​		重启机器。或者(source /etc/profile)				
​		3：验证环境
​		查看环境变量
​			env
​		验证JAVA_HOME是否配好：
​		echo $JAVA_HOME
​		echo $PATH
​		注意： 	
​			$是取环境变量的值
​			JAVA_HOME是环境变量			
​		验证PATH是否配好：
​		java -version
​	3: 搭建Tomcat服务器
​		1，从tomcat官网上下载.tar.gz包，并放到服务器上。
​		2，解压tomcat.tar.gz包：“tar -zxvf tomcat6.tar.gz”。
​			sudo tar -zxvf 该命令字节进.gz的文件包解压。
​		3，执行“/opt/tomcat6/bin/startup.sh”便可启动Tomcat。
​		如果启动不了，请尝试
​		sudo -i 切换到root用户再重新启动
​		sudo ./startup.sh
​		测试http:127.0.0.1:8080/
 		sudo firefox (如果无法正常打开浏览器使用该命令)	

zip，tar，gzip

