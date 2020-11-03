# 性能优化

## 简介

性能优化是一个很大的系统工程，一般我们需要通过整套的监控系统
从硬件与软件的角度去关注相关的性能指标。

硬件层面需要关注指标:
+ CPU
> + CPU 使用率
> + 饱和度(运行队列长度，或者是线程调度器延时的数值)
+ 磁盘IO
> + IOPS
> + 吞吐量
> + IOWAIT
+ 物理内存
> + 内存使用量
> + 缓存大小
> + swap换页情况

软件层面需要关注指标:
+ 虚拟内存
+ 网络IO
> + IOPS
> + 吞吐量
> + 延时
+ 文件系统
> + IOPS
> + 吞吐量
> + 延时
+ 上下文切换
+ 中断次数

但是往往现实中的问题很有隐蔽性，给排查带来很大的干扰。
## 问题
+ 服务cpu占用很高怎么办
+ 服务IO占用很高怎么办
+ 服务吞吐上不去怎么办
+ 服务内存告警怎么办


## linux 性能工具

常见的工具有:
+ top
+ pidstat
+ mpstat
+ iowait
+ vmstat
+ ifconfig
+ netstat
+ ss
+ lsof
+ perf
+ strace

### top

top - display Linux processes

top工具提供的是系统的summary信息以及按进程分组的进程相关的信息。

top输出需要关注的指标信息:
+ 平均负载
> load average
> + 平均负载表示的是系统处于可运行状态与不可中断状态的平均进程数。
> 可运行状态表示正在运行和等待运行的进程 ps命令的r状态进程。
> 不可中断表示正在内核执行关键流程的进程（比如响应硬中断）
> 一般平均大于CPU数的70%就需要警惕观察

+ task信息 当前系统进程运行情况
> total,  running,  sleeping, stopped, zombie
> + 需要关注运行中的进程数，僵尸进程情况
+ cpu信息
> us,  sy, ni, id,  wa,  hi, si, st
> + us 用户空间占用cpu的比例
> + sy 内核占用cpu的比例
> + wa iowait占用cpu的比例
> + hi 硬中断占用cpu的比例
> + si 软中断占用cpu的比例

> 一旦发生cpu占用过多，这些信息可以帮助我们判断是哪引起的瓶颈（应用层还是内核，io还是中断）
+ 内存情况
> + KiB Mem : total, free, used, buff/cache
> + KiB Swap: total, free, used. avail Mem 

> 一旦出现swap相关信息说明系统发生了换页，内存很紧张，需要进一步排查内存问题

+ 进程情况
> PID USER PR NI VIRT RES SHR S %CPU %MEM COMMAND
> + VIRT 虚拟内存总量
> + RES 物理内存(包含匿名内存+共享内存+mmap文件映射内存)
> + SHR 共享内存
> + S 进程状态(R 运行，s 睡眠，z 僵尸)
> + %CPU 进程使用CPU的比例
> + %MEM 进程使用物理内存比例

### vmstat

vmstat 从命名就知道是虚拟内存统计，对整个系统的虚拟内存、进程、CPU活动进行监控。

监控指标如下:
+ procs
> + r 运行中的进程数
> + b 等待中的进程数

> 运行中进程数+等待中进程数远大于cpu的数量，可能上下文切换次数也会增大
+ memory
> + swpd 切换到内存交换区的内存数量
> + free 空闲内存
> + buff buffer缓存（块设备的缓存）
> + cache cache缓存 (文件系统缓存)

> swapd 不为0同时需要观察swap的信息，swap中的si和so长期大于0说明在发生换页操作，
需要排查内存问题
+ swap
> + si 交换区写入内存（磁盘读）
> + so 内存写入交换区（写磁盘）

> swap 信息长期不为0 说明可能出现内存问题，需要进一步排查进程。

+ io
> + bi 从块设备读入数据的总量（读磁盘）
> + bo 块设备写入数据的总量（写磁盘）
+ system
> + in 设备中断数
> + cs 上下文切换次数

> 中断数与上下文切换可能会引起cpu使用率变高
+ cpu
> + us 用户空间占用cpu的比例
> + sy 内核占用cpu的比例
> + wa iowait占用cpu的比例

### pidstat

pidstat是针对进程统计，统计进程占用CPU，内存，设备IO，任务切换等信息。

pidstat 不同的选项输出不同的数据:
+ -u cpu信息
> PID  %usr %system %guest %CPU CPU Command
+ -r 内存统计
> PID  minflt/s  majflt/s     VSZ    RSS   %MEM  Command
> + minflt/s: 正常的缺页中断 page fault次数
> + majflt/s: 每秒主缺页错误次数(major page faults)，
当虚拟内存地址映射成物理内存地址时，相应的page在swap中，这样的page fault为major page fault，一般在内存使用紧张时产生
> + VSZ:      该进程使用的虚拟内存(以kB为单位)
> + RSS:      该进程使用的物理内存(以kB为单位)
> + %MEM:     该进程使用内存的百分比
+ -d io统计
> PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
> + kB_rd/s：每秒从磁盘读取的KB
> + kB_wr/s：每秒写入磁盘KB
> + kB_ccwr/s：任务取消的写入磁盘的KB。当任务截断脏的pagecache的时候会发生。
+ -w 上下文切换
> PID cswch/s nvcswch/s Command
> + cswch/s 自愿上下文切换(正常的系统调用，io等待之类)
> + nvcswch/s 非自愿上下文切换（时间片到达）

> 一旦nvcswch很高说明很可能cpu耗时在进程争抢上
+ -t 显示选择任务的线程的统计信息外的额外信息

### iostat

iostat 是针对磁盘io的性能工具，提供了每个磁盘的使用率，IOPS，吞吐量等指标。

iostat关注指标:
+ r/s 每秒发生给磁盘的读请求（合并后的请求数）
+ w/s 每秒发送给磁盘的写请求（合并后写请求）
+ rkB/s 每秒读取数据量
+ wkB/s 每秒写数据量
+ rrqm/s 每秒合并读的请求数
+ wrqm/s 每秒合并写请求数
+ r_await 读请求处理完成等待时间(等待时间+处理时间)
+ w_await 写请求处理完成等待时间
+ aqu-sz 平均请求长度
+ rareq-sz 平均读请求大小
+ wareq-sz 平均写请求大小
+ svctm 处理请求所需的时间（不包含等待时间）
+ util 磁盘处理io的时间百分比

重点关注指标
+ 吞吐量（rkB，wkB）
+ 等待长度 （aqu-sz）
+ iowait（r_await，w_await）
+ util 磁盘繁忙比例
### mpstat

mpstat是CPU维度的统计，重点关注CPU的统计情况

性能指标如下：
CPU %usr %nice %sys %iowait %irq %soft %steal %guest %idle
+ CPU 具体的CPU
+ usr 应用层占用比例
+ sys 内核占用比例
+ iowait io等待时间比例
+ irq 硬中断比例
+ soft 软中断比例

### netstat

netstat 是针对网络io的性能工具


## 性能查询


### 服务cpu占用很高

cpu占用很高可能是很多原因导致，我们需要先理解cpu占用运行可能由哪些因素组成。

CPU占用:
+ 用户程序
+ 内核程序
+ cpu上下文切换
+ iowait

我们一旦发现cpu占用很高也是从这些因素开始考虑，通常的排查步骤如下：
+ mpstat 查看CPU的统计信息
> + irq, soft 高说明是中断次数高，需要排查中断
> + iowait 高需要排查io问题，文件系统与磁盘
> + usr 高需要排查应用程序
> + sy 高需要排查上下文切换情况

+ top 查看系统平均负载，进程占用CPU使用情况
> + usr 高 应用程序占用高，排查应用程序问题，进一步采用perf之类工具定位耗CPU的函数
> + sy 高 内核占用高，需要排查的方向， 上下文切换

iowait 排查
+ iostat 吞吐量（rkB，wkB），等待长度 （aqu-sz），
iowait（r_await，w_await），util 磁盘繁忙比例
+ 通过iostat确定io问题，进一步通过pidstat确定哪个进程出现的io问题

上下文切换排查，CPU上下文切换有以下情况:
+ 进程上下文切换（非自愿）
+ 线程上下文切换（非自愿）
+ 中断上下文切换（非自愿）
+ 系统调用 （自愿）

> + pidstat -w 分析上下文切换次数，非自愿多说明可能是进程多导致的调度问题，
自愿可能是资源不足
> + vmstat 查看r b，r就绪队列远大于CPU说明是系统的进程数或者线程数太多
#### 常见的CPU优化方法

应用程序优化:
+ 多线程代替多进程
+ 善用缓存
+ 异步处理

系统优化:
+ CPU绑定
+ CPU独占

### 服务IO占用很高

服务IO占用高，需要考虑以下情况:
+ 文件系统IO
+ 磁盘IO
+ 系统调用与上下文切换

一般的排查步骤如下：
+ top, iostat 查看系统整体的io情况
+ vmstat 排查是否因内存不足导致的swap引起io问题（是需要排查内存占用问题）
+ pidstat确定哪个进程出现的io问题
+ 分析进程（strace lsof）

#### 常见的IO优化
+ 批量操作（减少IO次数）
+ mmap绕过系统调用与上下文切换
+ 非阻塞IO
+ 随机读写转顺序读写

### 服务内存告警

服务内存占用高，需要关注的情况:
+ 缺页中断（假如是主缺页中断说明这时swap已经介入，意味需要更多的磁盘IO，内存访问更慢）
+ swap 数据

一般的排查步骤如下：
+ free， top 查看整体的内存使用情况
+ vmstat， pidstat查看内存问题类型（内存不足，主缺页中断，缓存太大，swap）
+ 查看内存占用最大进程进一步分析

### 服务吞吐上不去

服务吞吐上不去需要从以下方面排查:
+ 网络原因
+ 请求没有到服务
+ CPU 忙于其他的事，没有处理服务

排查步骤如下:
+ ping 服务查看网络延迟，丢包（延迟大说明很可能是网络原因，可以进一步查看服务网络收发情况）
+ netstat -s 查看网络收发情况（观察收发包数，丢包数，错误包数，重传数）
> + 发现丢包严重，需要进一步排查丢包（可能是iptables规则，可能是连接队列满）
> + 重传比较多说明可能是网络问题或者有丢包导致的重传
> + ss -ltnp 查看队列大小
+ 应用程序排查，CPU，IO方式排查
## 参考资料

https://man7.org/linux/man-pages/man1/top.1.html

https://man7.org/linux/man-pages/man1/pcp-vmstat.1.html