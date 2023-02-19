# 使用方法

## 配置auditd

在实际生成我们需要的审计日志之前,需要先配置 aditd 本身.`/etc/audit/auditd.conf`配置文件确定了 auditd 启动后的运行方式.

```
log_file = /var/log/audit/audit.log
log_format = RAW
log_group = root
priority_boost = 4
flush = INCREMENTAL
freq = 20
num_logs = 5
disp_qos = lossy
dispatcher = /sbin/audispd
name_format = NONE
##name = mydomain
max_log_file = 6
max_log_file_action = ROTATE
space_left = 75
space_left_action = SYSLOG
action_mail_acct = root
admin_space_left = 50
admin_space_left_action = SUSPEND
disk_full_action = SUSPEND
disk_error_action = SUSPEND
##tcp_listen_port =
tcp_listen_queue = 5
tcp_max_per_addr = 1
##tcp_client_ports = 1024-65535
tcp_client_max_idle = 0
cp_client_max_idle = 0
```

- log_file,**log_format**,log_group
  - log_file写入磁盘路径,log_format写入磁盘的方式(方式包括 raw(完全按照内核发送消息时的格式存储消息)或者 nolog(丢弃消息,不将其写入磁盘,但是发往审计调度程序的数据将不受影响)),log_group 定义拥有审计信息的组.
  - priority_bost:确定 audit 因获得优先级提升量.取值 0-20 
  - flush,freq:指定是否以及以多少频率写入磁盘.flush 有效值为 none,incremental,data,sync.none 表示守护进程将审计报告写入磁盘时无需进行特殊处理,incremental 显式的将数据刷写到磁盘.如果使用 incremental 必须指定频率.freq 值 20 表示每生成 20 条报告写一次磁盘.data 始终将磁盘文件的数据部分保持同步,sync 将同时处理元数据和数据
  - num_logs:指定当 rotate 作为 max_log_file_action时要保留的日志文件数.取值范围是 0-99.小于 2 表示不轮换日志文件.**如果增大轮换的文件数,会增大 auditd 的工作量,执行这种轮换时,auditd 无法始终快速地处理内核的新数据,这可能造成积压状况.**这种情况可以更改`/etc/audit/audit.rules`文件中`-b`参数的值
  - disp_qos,dispatcher:auditd 启动期间会启动调度程序.守护进程会将审计消息中继到 dispatcher 指定的应用程序.**此程序必须以 root 身份运行**disp_qos会确定是否允许 auditd 与audispd 之间进行 lossy(有损)或者lossless(无损)通信,选择 lossy 时,auditd 会在消息队列已满时丢弃一些审计信息,如果 log_format为 raw,这些事件仍会写入磁盘,如果时 lossless将阻止审计消息写入磁盘,知道消息队列中出现空位.
  - name_format,name:name_format控制如何解析计算机名称.其中 user 选项需要使用 name 参数定义
  - max_log_file,max_log_file_action:max_log_file 指定触发可配置的 max_log_file_action 之前最大审计日志文件大小(以 Mb 为单位),max_log_file_action 可选地动作为:ignore,syslog,suspend,rotate,keeplogs.ignore 表示不执行任何操作,syslog 表示 auditd 向 syslog 日志系统发送警告日志,suspend 导致 auditd 停止向磁盘写入日志,但是 auditd 仍处于活跃状态,rotate 使用 num_logs 设置出发日志轮换.keep_logs 也会触发日志轮换,但不使用 num_logs 设置,因此始终保留所有日志.
  - space_left和 space_left_action:space_left接受表示磁盘剩余空间的数字值(以 Mb 为单位),用于触发space_left_action 操作.操作包括 ignore,syslog,email,execute,suspend,single,halt.其中 email 表示向 action_mail_acct 设置定的账户发送邮件.exec 加上脚本路径会执行指定脚本.(**无法向脚本传递参数**)single 会触发系统降级到单用户模式,halt 触发系统完全关闭.
  - action_mail_acct:email 地址只要在`/usr/lib/sendmail`中存在,可以向任何地址发送邮件(系统必须正确配置 email 和网络)
  - admin_space_left,admin_space_left_action:和 space_left 类似
  - disk_full_action:指定当系统耗尽了用于保存审计日志的磁盘时需要执行的操作.有效值和 space_left_action 相同
  - disk_error_action:指定当 auditd 写入期间遇到的任何类型的磁盘错误时需要执行的操作
  - tcp_listen_prot,tcp_listen_queue,tcp_client_ports,tcp_client_max_idle,tcp_max_per_addr:
    - TODO:

## 使用 auditctl 控制审计系统
auditd 配置完成之后,下一步重点控制 auditd 执行的审计计量,并为其指派足够其顺利运行的资源和限制

### 简介
auditctl 负责控制审计守护程序的状态和某些基本系统参数.它控制对系统执行的审计量
auditctl 使用审计规则来控制系统的那些组件需要接受审计及对其审计的范围.
此命令主要用于临时进行审计工作

### 审计参数

- auditctl -e [0|1|2]   用于启用和禁用审计,2表示启用并锁定配置
- auditctl -f [0|1|2]   控制故障标志,0静默,1printk,2panic
- auditctl -r [rate]    控制审计消息的速率上限
- auditctl -b [number]  控制积压上限
- auditctl -s 查询 auditd 当前状态
- auditctl -w [file_path] 文件系统审计

    ```
        -w /etc/shadow #请求此文件访问权限的所有系统调用都会被分析
        -w /etc -p rx  #-p表示针对此文件的某些特权做审计
        -w /etc/passwd -k fk_passwd -p rwxa #-k 表示添加一个键值,以便以后用来过滤此特定事件
    ```
- auditctl -a 系统调用审计(详情见过滤系统调用)
- auditctl -[D|d|W] 删除审计规则或者事件,其中-D 表示删除之前添加的所有规则,通常在 rules 文件的第一行,-d 表示删除某个系统调用审计规则,-W 表示删除某个文件系统的审计规则
- auditctl -l 列出所有规则

### 将参数传入审计系统
除了上述的命令行传参外,也可以使用`auditctl -R`命令从文件中批量执行配置.一般会在 auditd 启动后由 init 脚本从`/etc/audit/audit.rules`文件中加载规则.**命令行执行的操作不会保留到下次启动.每次修改 rules 文件需要重新启动 auditd.service 服务**

### aureport 生成自定义审计摘要报告

`/var/log/audit`目录中存储的原始审计报告会逐渐变得庞大而且难以理解.想要轻松的查找相关的消息可以使用 aureport 工具并创建自定义报告

aureport 选项:
- 不带任何参数创建粗略摘要报告
- -i:将数字项转换为文本
- --failed:创建失败事件的摘要报告
- --success:创建成功事件的摘要报告
- --summary:与其他选项结合创建特定关注的方面的摘要
- -e:生成所有事件的带编号列表摘要报告
- -p:生成所有进程事件的带编号列表摘要报告
- -s:生成所有系统调用时间创建摘要报告
- -x:生成所有可执行文件创建摘要报告
- -f:生成所有文件相关事件的摘要报告
- -u:生成有关用户执行应用程序的摘要报告
- -l:生成有关登录的报告
- -t:根据时间范围创建报告,-ts 表示开始时间,-te 表示结束时间

### ausearch 查询auditd 日志

ausearch 参数:
- -i:将数字转换成文本
- -a:根据事件 ID 搜索
- -m:根据事件类型查询
- -ul:根据登录 ID 查询
- -ua:根据用户 ID 查询
- -ga:根据组 ID 查询
- -c:根据命令名称查询
- -x:根据可执行文件查询
- -sc:根据系统调用查询
- -p:根据进程 ID 查询
- -sv:查看特定系统调用成功值的记录
- -f:搜索的包含特定文件名
- -tm:与特定终端相关的记录
- -hn:按主机名查询
- -k:按照特定键查询
- -w:按照特定字符串搜索
- -t:指定时间段,-ts,-te

### autrace 分析进程

autrace 对各个进程进行专门的审计,需要注意的是在使用 autrace 时需要从队列中清楚所有审计规则

### 直观呈现审计数据

### 中继审计事件通知

audit 子系统还允许外部应用程序实时访问和使用 auditd,这是由 audispd 程序提供.auditd启动时会拉起 audispd,并分发一份审计信息给 audispd,然后 audispd 会根据其配置将审计信息分发给用户自定义的插件程序.其配置信息存储在`/etc/audisp/audispd.conf`中.
- q_depth:指定时间调度程序内部队列大小.
- overflow_action:内部队列溢出时的反应.ignore(不执行任何操作),syslog(向系统日志发送警告),suspend(停止处理事件),single(将计算机设置为单用户模式),halt(关闭系统)
- priority_boost:指定审计事件的优先级
- name_format:指定在审计事件中插入计算机节点的名称的方式.none(不插入),hostname(gethostname 返回的名称),fqd(计算机的完全限定域名),numeric(计算机 ip 地址),user(name 选项中自定义字符串)
- name:用于标识计算机的用户自定义字符串
- max_restarts:尝试重启崩溃插件的次数
- plugin_dir:插件配置文件 

插件:
插件程序将其配置文件安装在专用于 audispd 插件的`/etc/audisp/plugins.d`中
- active:制定程序是否使用 audispd,yes or no
- direction:指定插件预期会采用什么方式与audit 通讯.in or out
- path:插件可执行文件的绝对地址
- type:指定运行插件的方式,builtin 或者 always,对内部插件(af_unix 和 syslog)使用 buildin,其他插件大部分使用 always
- args:传递给插件程序的参数
- format:指定传递给插件程序的数据格式.binary 或者 string.binary 以audispd 从 auditd 接收数据时的原始格式传递,string 指示 audispd将事件更改为可由审计分析库分析的字符串.


插件示例:
配置文件`/etc/audisp/plugins.d/test.conf`:
```
active = yes
direction = out
path=/home/yushun/test.sh
type=always
args=hello
format=string
```
插件脚本示例：
```
#! /usr/bin/bash
while(( 1 ))
do
    read name
    echo "${name}" >> /home/log.txt
done
```
脚本不停的读取标准输入，然后写入日志文件中。奇怪的地方是这里是从标准输入读取，和audispd有什么关系呢？

**解释**：
首先我们的插件是由audispd拉起而不是我们手动拉起的，在audispd创建了一个socket对之后，会将pair[0]重定向到标准输入中，然后创建子进程拉起插件，显然插件的标准输入就变成了socket对中的pair[0]，所以这里脚本从标准输入读取就是从sockt对的pair[0]中读取。，在审计信息从auditd传入audispd之后，audispd又将审计信息写入了socket对的pair[1]中，然后脚本就可以通过read从pair[0]中读取数据了。其实这里就是使用socket创建了一个父子进程间的通信管道。(这个可以在audispd.c中的`safe_exec()`函数中找到,先用`soketpair()`函数创建一对socket，然后使用`dup2()`函数重定向pair[0]到标准输入)

这是由audispd拉起的插件程序，如果我们需要自己启动进程，然后audispd有数据我们接收处理即可。可以利用af_unix.conf来让我们自己启动插件然后接收审计信息：
修改`/etc/auditsp/plugins.d/af_unix.conf`:
```
active = yes # 启用af_unix功能
direction = out
path = builtin_af_unix
type = builtin
args = 0640 /var/run/audispd_events
format = string
```
然后重启auditd

自定义插件示例：
```C
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <sys/un.h>
#include <unistd.h>

int main(int argc,char** argv){
    struct sockaddr_un server_addr;
    char buff[1024]={0};
    server_addr.sun_family = AF_UNIX;
    //这里的路径就是af_unix.conf中的路径
    strcpy(server_addr.sun_path,"/var/run/audispd_events");

    int sock = socket(AF_UNIX,SOCK_STREAM,0);
    if (sockfd < 0)
        return -1;
    
    //本地通信连接到audisp启动的服务端
    int result = connect(sockfd,(struct sockaddr*)&server_addr,sizeof(server_addr));
    if(result < 0 ){
        close(sockfd);
        return -1;
    }

    while(1){
        memset(buff,0,1024);
        if(read(sockfd,buff,1024)>0){
            printf("%s\n",buff);
        }
    }
    return 0;
}
```
这里的方式是建立一个本的af_unix通信，audispd会根据`af_unix.conf`建立一个af_unix本地通信服务端。我们只需要在建立一个客户端然后连接到服务端，等待消息即可。