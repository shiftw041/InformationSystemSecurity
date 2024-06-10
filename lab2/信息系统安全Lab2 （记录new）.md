### 信息系统安全lab2

##### 任务1.1 针对ping程序使用apparmor进行访问控制并修改

1.环境配置

```
sudo apt install vim
sudo systemctl start apparmor
sudo systemctl enable apparmor
sudo apt install apparmor-profiles
sudo apt install apparmor-utils
```

2.创建空白文件，同时打开另外一个窗口ping

~~~bash
//创建空白配置文件（如果存在可以直接修改配置）
//sudo aa-genprof /bin/ping
~~~

~~~bash
ping www.baidu.com
~~~

![image-20230612215759265](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230612215759265.png)

首先先按scan截获到log，随后deny ，finish

![image-20230612215825560](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230612215825560.png)

3.查看配置

~~~bash
sudo cat /etc/apparmor.d/usr.bin.ping
~~~

![image-20230612220412962](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230612220412962.png)

4.再次ping发现无法执行

![image-20230612220530702](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230612220530702.png)

##### 任务2.1 chroot

###### (1)

1.配环境

~~~bash
sudo apt-get update
sudo apt install gcc
sudo apt install gcc-multilib
sudo apt install make
sudo apt install net-tools//使用ifconfig命令需要
sudo apt install wireshark
(之后找到wireshark应用并收藏到收藏夹)
(启动wireshark——sudo wireshark)//不使用root权限直接点开wireshark的话可能看不到想看的接口（打开的接口是loopback：lo）
sudo apt install apache2
sudo /etc/init.d/apache2 stop		//关闭apache2
~~~

2.生成文件并为程序添加setuid root权限，启动执行

~~~bash
make

sudo chown root touchstone
sudo chmod +s touchstone
./touchstone
~~~

![image-20230613003302508](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613003302508.png)

3.此时打开网络浏览器会发现有一个登录界面

![image-20230613003224789](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613003224789.png)

注册一个用户名和密码都是lm的账户。

![image-20230613003403458](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613003403458.png)

退出后打开wireshark抓包，重新登录，截获信息如下：

![image-20230613003624071](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613003624071.png)

4.关闭ASLR

~~~bash
sudo sysctl -w kernel.randomize_va_space=0
~~~

![image-20230613003915542](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613003915542.png)

5.查看libc加载基址0xf7da1000

~~~bas
ldd ./banksv
~~~

![image-20230613003925816](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613003925816.png)

6.查看其他所需信息偏移

~~~bash
readelf -s /lib32/libc.so.6 | grep system
strings -tx /lib32/libc.so.6 | grep "/bin/sh"
~~~

![image-20230613004011392](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613004011392.png)

41370；18b363

7.继续查看其他所需信息偏移

~~~bash
readelf -s /lib32/libc.so.6 | grep exit
readelf -s /lib32/libc.so.6 | grep unlink
~~~

![image-20230613004114974](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613004114974.png)

33ed0;f2760

8.ebp就填写参考运行touchstone后出现的Frame Pointer的地址0xffa9a698或者是**0xffa2c1b8**

![image-20230613004315449](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613004315449.png)

![image-20230613004331344](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613004331344.png)

9.打开之前wireshark抓获的包进行TCP Stream跟踪确定Content-length为54，并修改相应的用户口令

![image-20230613004455485](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613004455485.png)

10.再次运行攻击脚本

![image-20230613010015634](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613010015634.png)

11.根据报错安装pwn库

~~~bash
sudo apt install pip
pip install pwn
~~~

![image-20230610203727235](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230610203727235.png)

12.测试准备

~~~path
touch /tmp/test.txt
sudo chown root /tmp/test.txt
ls -al /tmp/test.txt
~~~

![image-20230613010854316](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613010854316.png)

13.运行攻击脚本

~~~bash
python3 ./exploit-template.py 127.0.0.1 80
~~~

这里提示一下，最好先按ctrl+c终止掉touchstone程序，之后重启touchstone程序，之后再修改脚本里面的ebp的值。之后再运行即可完成攻击。

![image-20230613192200915](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613192200915.png)

###### （2）

1.修改server.c代码，添加chroot函数重新编译。

~~~c
int err=chroot("/jail");
if(!err) printf("==============chroot to /jail succeed===============");
~~~

![image-20230613192823405](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613192823405.png)

2.重新编译

~~~bash
make
~~~

3.运行chroot setup脚本设置环境

~~~bash
chmod +x chroot-setup.sh chroot-copy.sh
sudo su
./chroot-setup.sh
~~~

![image-20230613193441517](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613193441517.png)

4.退出root账户，并进入jail文件夹中

~~~bash
exit
cd /jail
~~~

![image-20230613193631386](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613193631386.png)

5.创建两个test文件

~~~bash
echo "test" >/tmp/test.txt ;sudo chown root /tmp/test.txt
echo "jailtest" >/jail/tmp/test.txt;sudo chown root /jail/tmp/test.txt
~~~

![image-20230613193920184](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613193920184.png)

6.链接库由于chroot发生变化，需要重新获取libc base

~~~bash
sudo ./touchstone(之后执行login logout初始化一下下)
ps -ef|grep banksv
sudo gdb attach pid
info proc mappings
~~~

![image-20230613213029104](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613213029104.png)

![image-20230613212955171](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613212955171.png)

7.再次查看文件夹中的文件

~~~bash
cat /tmp/test.txt; cat jail/tmp/test.txt
~~~

![image-20230613213406807](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230613213406807.png)

##### 任务2.2 改变进程EUID

0.环境配置（关闭ASLR）

~~~bash
sudo sysctl -w kernel.randomize_va_space=0
~~~

1.修改server.c（一共有三处）

~~~c
setresuid(1000,1000,1000);
~~~

![image-20230614114423913](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614114423913.png)

2.重新编译

~~~bash
make
~~~

3.为touchstone程序添加setuid root权限，并启动执行

~~~bash
sudo chown root touchstone
sudo chmod +s touchstone
./touchstone
~~~

![image-20230614115607448](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614115607448.png)

4.之后再网站重新注册账户

![image-20230614115640124](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614115640124.png)

密码和账户名字都是lm；

5.在另外端口查看touchstone和附属进程权限如下：

~~~bash
ps -elf
~~~

![image-20230614115931587](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614115931587.png)

6.修改任务2.1中的py脚本里的ebp参数并运行（其他参数在关闭了ASLR情况下其实是一样的），

~~~bash
echo "test" >/tmp/test.txt; sudo chown root /tmp/test.txt
cat /tmp/test.txt
--修改py文件
python3 ./exploit-template.py 127.0.0.1 80
~~~

![image-20230614121724012](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614121724012.png)

发现没有攻击成功！

7.但是用普通用户身份创建的文件可以删除

![image-20230614122239173](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614122239173.png)

##### 任务2.3 seccomp约束

1.配环境

~~~bash
sudo sysctl -w kernel.randomize_va_space=0
sudo apt install libseccomp-dev libseccomp2 seccomp
sudo apt-get update
sudo apt-get install libseccomp-dev:i386 autoconf automake libtool
~~~

2.修改makefile，在编译参数中添加动态链接库 -l seccomp

![image-20230614130136151](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614130136151.png)

3.编译

~~~bash
make
~~~

4.编译后链接库要变成/lib/i386-linux-gnu/libc.so.6，需要重新获取相关地址。

~~~bash
ldd banksv;readelf -s /lib/i386-linux-gnu/libc.so.6|grep -E "system|exit|unlink";strings -tx /lib/i386-linux-gnu/libc.so.6|grep "bin/sh"
~~~

![image-20230614164938103](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614164938103.png)

###### (1)默认允许

1.修改banksv,seccomp规则如下所示：

~~~c
#include <seccomp.h>

void setSeccomp(){
  //init seccomp
  scmp_filter_ctx ctx;
  //默认运行
  ctx =seccomp_init(SCMP_ACT_ALLOW);

  if(ctx==NULL)
	  exit(-1);
  //unlink
  seccomp_rule_add(ctx,SCMP_ACT_KILL,SCMP_SYS(unlink),0);
  seccomp_rule_add(ctx,SCMP_ACT_KILL,SCMP_SYS(mkdir),0);

  seccomp_load(ctx);
  seccomp_release(ctx);
  /*--------------------------*/

}

 setSeccomp();
~~~



![image-20230614131803235](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614131803235.png)

![image-20230614131602856](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614131602856.png)

2.为touchstone程序添加权限，并启动执行

~~~bash
sudo chown root touchstone
sudo chmod +s touchstone
./touchstone

~~~

3.之后再网站中重新注册账户

4.

~~~bash
echo "test" >/tmp/test.txt; sudo chown root /tmp/test.txt
cat /tmp/test.txt
--修改py文件
python3 ./exploit-template.py 127.0.0.1 80
~~~

![image-20230614133654396](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614133654396.png)

###### （2）

1.

~~~bash
void setSeccomp(){
  //init seccomp
  scmp_filter_ctx ctx;
  //默认拒绝
  ctx =seccomp_init(SCMP_ACT_KILL);

  if(ctx==NULL)
	  exit(-1);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(read),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(write),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(openat),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(rt_sigaction),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(socketcall),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(clone),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(set_robust_list),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(getresuid32),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(getcwd),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(getpid),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(statx),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(clone),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(_llseek),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(fcntl64),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(access),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(brk),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(exit_group),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(fstat64),0);
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(stat64),0);  
  
	  
  //unlink
  seccomp_rule_add(ctx,SCMP_ACT_ALLOW,SCMP_SYS(unlink),0);
  seccomp_rule_add(ctx,SCMP_ACT_KILL,SCMP_SYS(mkdir),0);

  seccomp_load(ctx);
  seccomp_release(ctx);
  /*--------------------------*/

}
~~~

![image-20230614134358591](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614134358591.png)

##### 2.4

1.环境配置

~~~bash
service apparmor start
sudo sysctl -w kernel.randomize_va_space=0
~~~

2.给touchstone程序添加setuid root权限

~~~bash
sudo chown root touchstone
sudo chmod +s touchstone
./touchstone
~~~

3.允许aa-geprog对banksv记录

~~~bash
sudo aa-genprof ./banksv
~~~

直接按F完成配置文件的产生

4.编辑策略并重新加载策略

~~~bash
sudo cat /etc/apparmor.d/home.lm.Desktop.lab2.code.banksv
~~~

~~~bash
/home/lm/Desktop/lab2/code/banksv {
  #include <abstractions/base>

  /home/lm/Desktop/lab2/code/users.db rwk,
  /home/lm/Desktop/lab2/code/index.html r,
  owner /home/lm/Desktop/lab2/code/users.db-journal w,
  /home/lm/Desktop/lab2/code/banksv mr,

}
~~~

~~~bash
sudo service apparmor reload
~~~

![image-20230614192524957](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614192524957.png)

![image-20230614192808112](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614192808112.png)

5.创建测试文件

~~~bash
echo "test" >/tmp/test.txt; sudo chown root /tmp/test.txt
cat /tmp/test.txt
--修改py文件
python3 ./exploit-template.py 127.0.0.1 80
~~~

![image-20230614170114815](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614170114815.png)

![image-20230614170518565](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614170518565.png)

![image-20230614171010246](C:\Users\劳明\AppData\Roaming\Typora\typora-user-images\image-20230614171010246.png)
