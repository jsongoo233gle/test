# ssh-agent.sh

## ssh-agent.sh
启用ssh-agent:
```
$ eval `ssh-agent`
Agent pid 3240
```
添加私钥:
```
$ ssh-add
Identity added: /root/.ssh/id_rsa (/root/.ssh/id_rsa)
```
编缉/etc/ssh/ssh_config文件: ForwardAgent yes
让ssh-agent能转发,这样就可以这样登陆了:node—->host1—->host2,到此请注意,如果host1上没有设定转发的话就登不了host2了,设定了转发后可以进一步跳到host2上了.
```
[root@node ~]#ssh host1
Last login: Thu Oct 18 16:21:29 2007 from node
[root@host1 root]# vi /etc/ssh/ssh_config 
[root@host1 root]# ssh host2
Last login: Thu Oct 18 16:20:28 2007 from node
[root@host2 root]# 
```
到这里基本上已经大功告成了,还有一个小问题那就是总不能每次都手动运行ssh-agent吧!最省事的方法就是将它写到profile中去. 
可以在/etc/profile.d下建一个ssh-agent.sh文件.
这样就不会生成太多的ssh-agent程序了,而且支持GUI环境.当打开一个终端时:
```
Stale agent file found. Spawning new agent...
Agent pid 4096
Identity added: /root/.ssh/id_rsa (/root/.ssh/id_rsa)
[root@node ~]#
```
添加了新的密钥.

## ssh-find-agent.sh
很多时候我们要使用我们的私钥的时候,我们都需要输入我们的密码,然后才能正常的使用私钥去登陆远程主机,去git pull我们的repo,去干一些事,但是,我们并不想每一次都输入(安全原因咱是不考虑),有没有办法能让我们输入一次,在一段时间内(或者重启),都免于再次如入我们密钥的密码?答案是肯定的,那就是我们常用的ssh-agent和ssh-add.

**启动ssh代理并添加密钥**
### ssh-agent
ssh-agent是一种控制用来保存公钥身份验证所使用的私钥的程序. ssh-agent在X会话或登录会话之初启动,所有其他窗口或程序则以客户端程序的身份启动并加入到ssh-agent程序中.通过使用环境变量,可定位代理并在登录到其他使用ssh机器上时使用代理自动进行身份验证.不同于ssh,ssh-agent是个长时间持续运行的守护进程(daemon),设计它的唯一目的就是对解密的专用密钥进行高速缓存.
ssh包含的内建支持允许它同ssh-agent通信,允许ssh不必每次新连接时都提示您要密码才能获取解密的专用密钥.对于ssh-agent,您只要使用ssh-add把专用密钥添加到ssh-agent的高速缓存中.这是个一次性过程;用过ssh-add之后,ssh将从ssh-agent获取您的专用密钥,而不会提示要密码短语来烦您了.
其实ssh-agent就是一个密钥管理器,运行ssh-agent以后,使用ssh-add将私钥交给ssh-agent保管,其他程序需要身份验证的时候可以将验证申请交给ssh-agent来完成整个认证过程.通过使用ssh-agent就可以很方便的在不的主机间进行漫游了.假如我们手头有三台server: host1、host2、host3且每台server上到保存了本机(node)的公钥,因此我可以通过公钥认证登录到每台主机:
```
[root@node ~]#ssh host1
Last login: Thu Oct 18 13:56:08 2007 from node
[root@host1 root]# exit
exit
[root@node ~]#ssh host2
Last login: Fri Oct 12 11:14:44 2007 from node
[root@host2 root]#exit 
exit
[root@node ~]#ssh host3
Last login: Sat Sep 29 10:21:32 2007 from node
[root@host3 root]# 

```
但是这三台server之间并没有并没有保存彼此的公钥,而且我也不可能将自己的私钥存放到server上(不安全),因此彼此之间没有公钥进 行认证(可以密码认证,但是这样慢,经常输密码,烦且密码太多容易忘).如果我们启用ssh-agent,问题就可以迎刃而解了.
从这个agent我们就知道这是一个代理,代理什么呢?代理我们的私钥,这个ssh-agent可以在我们的后台运行,然后我们可以通过ssh-add将我们的key加入到后台运行的ssh-agent中,这样,我们就可以使用我们的key并且不需要每次都输入密码了,如何启动呢?
首先,如果想要使用ssh代理,我们则需要先启动ssh代理,也就是启动ssh-agent程序,如下两条命令都可以启动代理,但是略有不同.
- ssh-agent $SHELL  
  创建子shell,在子shell中运行ssh-agent进程,退出子shell自动结束代理.
- eval \`ssh-agent\`  
  单独启动一个代理进程,退出当前shell时最好使用`ssh-agent -k`关闭对应代理.
  
当我们使用"ssh-agent $SHELL"命令时,会在当前shell中启动一个默认shell,作为当前shell的子shell,ssh-agent程序会在子shell中运行,当执行"ssh-agent $SHELL"命令后,我们也会自动进入到新创建的子shell中,centos中默认shell通常为bash,所以,在centos中上述命令通常可以直接写为ssh-agent bash,当然,如果你的默认shell已经指定为其他shell,比如csh,那么你也可以直接使用ssh-agent csh,效果都是相同的,我们来实验一下.
当前使用的centos系统的默认shell为bash,在未启动ssh-agent程序时,我们在当前bash中执行pstree命令,查看sshd的进程树,如下:
```
$ pstree
...
├─sshd─┬─sshd───bash───pstree
│      └─sshd───sftp-server
```
然后,执行`ssh-agent $SHELL`命令(注意:SHELL为大写),启动ssh代理,执行此命令后,再次使用pstree命令,查看sshd的进程树:
```
ssh-agent $SHELL
```
启动ssh代理后,查看sshd进程树,如下所示:
```
$ pstree
...
├─sshd─┬─sshd───bash───bash─┬─pstree
│      │                    └─ssh-agent
│      └─sshd───sftp-server
```
在原来的bash中新生成了一个子bash,ssh-agnet运行在子bash中,我们的命令也同样在子bash中执行.  
此时,在当前会话中,我们已经可以使用ssh-agent了,ssh-agent会随着当前ssh会话的消失而消失,这也是一种安全机制,比如,退出当前子shell,再次查看进程树,
找到sshd的进程树,如下,如果你操作的非常快,可能会在与sshd进程平级的区域看到一个还没有来得及关闭的ssh-agent进程,但是最终sshd的进程树如下图所示:
```
$ exit
exit
$ pstree
...
├─sshd─┬─sshd───bash───pstree
│      └─sshd───sftp-server
```
可以看到ssh-agent已经不再存在了,如果你的服务器中开启了图形化环境,使用"ps -ef | grep ssh-agent"命令查找ssh-agent进程,仍然能够看到一个ssh-agent进程,这个ssh-agent进程是跟随图形化界面开机启动的,但是通常,服务器中很少会开启图形化界面,所以我们不用在意它.  
虽然我们已经知道了怎样启动ssh-agent,但是,如果想让ssh代理帮助我们管理密钥,还需要将密钥添加到ssh代理中,这个话题一会儿再聊.  
刚才执行了ssh-agent $SHELL命令,现在试试eval \`ssh-agent\` 命令.
```
$ eval `ssh-agent`
Agent pid 4096
$ pstree
...找到进程树的如下部分
├─ssh-agent
├─sshd─┬─sshd───bash───pstree
│      └─sshd───sftp-server
```
可以看到,ssh-agent进程已经启动了,此刻,如果我们退出当前bash,此ssh-agent进程并不会自动关闭,所以,我们应该在退出当前bash之前,手动的关闭这个进程,在当前bash中,使用`ssh-agent -k`命令可以关闭对应的ssh-agent进程,但是,如果在退出了当前bash以后再使用`ssh-agent -k`命令,是无法关闭对应的ssh-agent进程的,除非使用kill命令,当然,其实在使用`ssh-agent $SHELL`命令时,也可以使用`ssh-agent -k`命令关闭ssh代理.  
事实上ssh-agent的输出是一系列bash命令,如果这些命令被执行,则将设置两个环境变量:SSH_AUTH_SOCK和SSH_AGENT_PID.内含的export命令使这些环境变量对之后运行的任何附加命令都可用.如果shell真对这些行进行计算,这一切才会发生,但是此时它们只是被打印到标准输出(stdout)而已.要使之确定,才像eval \`ssh-agent\`这样来调用.  
启动ssh-agent的最佳方式就是把上面这行添加到您的~/.bash_profile中;这样,在您的登录shell中启动的所有程序都将看到环境变量,而且能够定位ssh-agent，并在需要的时候向其查询密钥.尤其重要的环境变量是SSH_AUTH_SOCK;SSH_AUTH_SOCK包含有ssh和scp可以用来同ssh-agent建立对话的UNIX域套接字的路径.
### ssh-add
好了,我们已经了解了怎样启动ssh代理,以及怎样关闭ssh代理,但是,我们还没有真正的使用过代理,如果想要真正的使用ssh代理帮助我们管理密钥,还需要将密钥添加到代理中,添加密钥需要使用到ssh-add命令.  
ssh-agent启动时高速缓存是空的,里面不会有解密的专用密钥.在我们真能使用ssh-agent之前,首先还需要使用ssh-add命令把我们的专用密钥添加到ssh-agent的高速缓存中以备使用.  
ssh-add命令的使用方法非常简单,示例如下:  
`ssh-add  ~/.ssh/id_rsa_custom`  
上述命令表示将私钥id_rsa_custom加入到ssh代理中,如果你没有正确的启动ssh-agent,那么你在执行ssh-add命令时,可能会出现如下错误提示:  
`Could not open a connection to your authentication agent.`  
完成上述步骤后,ssh代理即可帮助我们管理id_rsa_custom密钥了,那么有哪些具体的使用场景呢,我们继续聊.  
一旦您已经用ssh-add把专用密钥(或多个密钥)添加到ssh-agent的高速缓存中,并在当前的shell中(如果您在~/.bash_profile中启动ssh-agent,情况应当是这样)定义SSH_AUTH_SOCK,那么您可以使用scp和ssh同远程系统建立连接而不必提供密码短语.

**ssh代理帮助我们选择对应的私钥进行认证**


我们可以在我们的.bashrc中添加如下代码:
```
eval $(ssh-agent) > /dev/null
```
但是这有一个问题,问题就是每次打开我们的shell,我们都会创建一个新的ssh-agent,久而久之我们就有好多个ssh-agent在跑了,并且每次启动一个新的,我们都需要通过ssh-add来添加我们的key,这并不是我们想要的.  
让我们先来看一下ssh-agent是如何工作的：
```
$ ssh-agent 
SSH_AUTH_SOCK=/tmp/ssh-sp4EAi4iNWF5/agent.26520; export SSH_AUTH_SOCK;
SSH_AGENT_PID=26521; export SSH_AGENT_PID;
echo Agent pid 26521;

$ ps x | grep ssh-agent
21983 ?        Ss     0:00 ssh-agent
26521 ?        Ss     0:00 ssh-agent
26540 pts/5    S+     0:00 grep --color=auto ssh-agent
```
我们可以从第一个命令中得到$SSH_AUTH_SOCK and $SSH_AGENT_PID.  
其中,$SSH_AGENT_PID,这个就是ssh-agent的id,我们可以通过ssh-agent -k $SSH_AGENT_PID来干掉它;  
$SSH_AUTH_SOCK,这个环境变量非常重要,如果这个变量有问题,你无法使用ssh-add命令来添加私钥.  
我们也可以在多个shell中使用同一个ssh-agent,但是我们需要手动设定环境变量$SSH_AUTH_SOCK.通过调用ssh-find-agent.sh脚本来进行设置.  
编辑`.bashrc`文件,加入如下配置,这样就不会每次都创建新的ssh-agent了.
```
source ~/ssh-find-agent.sh
set_ssh_agent_socket
```
