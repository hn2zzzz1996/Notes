# ftrace - Linux kernel internal tracer

[Reference](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_for_real_time/7/html/tuning_guide/appe-detailed_description_of_ftrace)

## The Debug File System

用户的接口被挂载在`/sys/kernel/debug/tracing`下，可以手动挂载debug文件系统：

```Shell
mount -t debugfs nodev /sys/kernel/debug
```

## Ftrace files

在tracing目录下有很多的文件，下面一一介绍：

### trace

该文件中保存的是ftrace的输出，当读取该文件的时候会停止tracing，如果停止了trace，那么每次读取该文件的时候都是相同的内容。

如果想要清空trace buffer，仅仅写入该文件就可以，这会清楚该trace buffer中所有的内容：

```shell
># echo > trace
```

### trace_pipe

和`trace`文件类似，but is used to read the trace live. 这是一个生产者/消费者的trace，这次读取了下次读取就没有了。但是这主要是用于查看实时的trace的输出，并且不用暂停trace。

### available_tracers

一系列被编译进内核的ftrace tracers，当前可以使用的tracer.

### current_tracer

enables or disables a ftrace tracer.

### events

a directory that contains events to trace and can be used to enable or disable events as well as set filters for the events.

### tracing_on

Disable and enable recording to the ftrace buffer.

注意，通过该文件去disable trace不会disable内核中正在进行的tracing，它只会disable把trace的信息写到trace buffer中。



## Tracers and Events

### nop tracer

"nop" 是默认的tracer. 它本身不提供任何的trace的能力，所以"nop" tracer被用于仅仅tracing events的场景。

当"nop" tracer被激活并且trace buffer是空的时候，"trace"文件中的内容如下所示：

```shell
># cat trace
# tracer: nop
#
# entries-in-buffer/entries-written: 0/0   #P:8
#
#                              _-------=> irqs-off          
#                            /  _------=> need-resched      
#                            |/  _-----=> need-resched_lazy 
#                            ||/  _----=> hardirq/softirq   
#                            |||/  _---=> preempt-depth     
#                            ||||/  _--=> preempt-lazy-depth
#                            ||||| / _-=> migrate-disable   
#                            |||||| /     delay
#           TASK-PID   CPU#  |||||||    TIMESTAMP  FUNCTION
#              | |       |   |||||||       |         |
```

* TASK-PID 表示进程名和PID号
* CPU表示当前进程在哪个CPU上运行
* 然后是一系列的状态
* TIMESTAMP 是发生该events的时间
* FUNCTION 是在哪个函数中触发了该events

### Events

Events被分解成各个子系统，每一个子系统的events都有一个在`events`目录下的子目录。

```Shell
># ls -F events
block/       header_event  lock/     printk/        skb/       vsyscall/
compaction/  header_page   mce/      random/        sock/      workqueue/
drm/         i915/         migrate/  raw_syscalls/  sunrpc/    writeback/
enable       irq/          module/   rcu/           syscalls/
ext4/        jbd2/         napi/     rpm/           task/
ftrace/      kmem/         net/      sched/         timer/
hda/         kvm/          oom/      scsi/          udp/
hda_intel/   kvmmmu/       power/    signal/        vmscan/
```

上面展示的每一个目录都是一个events子系统，注意到`events`目录下还有三个文件：

* enable
* header_event
* header_page

需要注意的只有`enable`文件，向`enable`中写入1的时候会enable所有的events，写入0的时候会disable所有的events。

`header_event`和`header_page`文件向`trace-cmd`工具提供足够的信息，无需我们关注。

下面来看一看每一个子系统目录下面都有什么？

```Shell
># ls -F events/sched
enable                   sched_process_exit/  sched_stat_sleep/
filter                   sched_process_fork/  sched_stat_wait/
sched_kthread_stop/      sched_process_free/  sched_switch/
sched_kthread_stop_ret/  sched_process_wait/  sched_wait_task/
sched_migrate_task/      sched_stat_blocked/  sched_wakeup/
sched_pi_setprio/        sched_stat_iowait/   sched_wakeup_new/
sched_process_exec/      sched_stat_runtime/
```

可以看到又有许多的子目录，这每一个子目录代表一个单独的event。注意到在子系统目录下有两个文件：

* enable

* filter

The `enable`文件用于控制所有的events。`filter`文件用于过滤events。

然后看看某一个event下面都有些什么：

```Shell
># ls -F events/sched/sched_wakeup/
enable  filter  format  id
```

* format: shows the fields that are written when the event is enabled, as well as the fields that can be used for the filter. 
* id: used for perf tool.

```Shell
># cat events/sched/sched_wakeup/format
name: sched_wakeup
ID: 249
format:
		# Common fileds for all events
        field:unsigned short common_type;        offset:0;        size:2;        signed:0;
        field:unsigned char common_flags;        offset:2;        size:1;        signed:0;
        field:unsigned char common_preempt_count;        offset:3;        size:1;        signed:0;
        field:int common_pid;        offset:4;        size:4;        signed:1;
        field:unsigned short common_migrate_disable;        offset:8;        size:2;        signed:0;
        field:unsigned short common_padding;        offset:10;        size:2;        signed:0;

		# specific fields for the event
        field:char comm[16];        offset:16;        size:16;        signed:1;
        fieldid_t pid;        offset:32;        size:4;        signed:1;
        field:int prio;        offset:36;        size:4;        signed:1;
        field:int success;        offset:40;        size:4;        signed:1;
        field:int target_cpu;        offset:44;        size:4;        signed:1;

print fmt: "comm=%s pid=%d prio=%d success=%d target_cpu=%03d", REC->comm, REC->pid, REC->prio, REC->success, REC->target_cpu
```

通过format的格式我们可以知道这个event输出了什么，从而能够更高效的过滤信息。

### Filtering events

下面介绍一下如何高效的过滤event的输出。

可以使用的过滤判断条件有如下几种：

对于数字的field：
==, !=, <, <=, >, >=

对于字符串的field:

==, !=, ~

逻辑运算符 && 和 || 也可以被使用。

如果用语法来描述的话：

```Shell
<filter> = FIELD <pred-num> | FIELD <pred-string> |
    '(' <filter> ')' | <filter> '&&' <filter> | <filter> '||' <filter>

 <pred-num> = <num-op> <number>

 <pred-string> = <string-op> <string>

 <num-op> = '==' | '!=' | '<' | '<=' | '>' | '>='

 <string-op> = '==' | '!=' | '~'

 <number> = <digits> | '0x'<hex-number> 

 <digits> = [0-9] | <digits><digits>

 <hex-number> = [0-9] | [a-f] | [A-F] | <hex-number><hex-number>

 <string> = '"' VALUE '"'

The glob expression '~' is a very simple glob. it can only be:

 <glob> = VALUE | '*' VALUE | VALUE '*' | '*' VALUE '*'

That is, anything more complex will not be valid. Such as:

  VALUE '*' VALUE
```

需要注意的是`~`这个字符串运算符代表通配符匹配，但是这个通配符匹配所实现的规则非常的简单，只能在单个字符串的前后进行通配符匹配，像`VALUE * VALUE`这样的通配符匹配是不符合规则的。

通配符匹配可以写成这样：

```
comm ~ "kwork*"
```

## Function tracer



## Function_graph tracer

相对于Function的tracer，function_graph tracer更直观，更适合阅读。

要使用function_graph tracer也是非常的简单：

```Shell
# echo function_graph > current_tracer
# cat trace

# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 5)   0.125 us    |            source_load();
 5)   0.137 us    |            idle_cpu();
 5)   0.105 us    |            source_load();
 5)   0.110 us    |            idle_cpu();
 5)   0.132 us    |            source_load();
 5)   0.134 us    |            idle_cpu();
 5)   0.127 us    |            source_load();
 5)   0.144 us    |            idle_cpu();
 5)   0.132 us    |            source_load();
 5)   0.112 us    |            idle_cpu();
 5)   0.120 us    |            source_load();
 5)   0.130 us    |            idle_cpu();
 5) + 20.812 us   |          } /* find_busiest_group */
 5) + 21.905 us   |        } /* load_balance */
 5)   0.099 us    |        msecs_to_jiffies();
 5)   0.120 us    |        __rcu_read_unlock();
 5)               |        _raw_spin_lock() {
 5)   0.115 us    |          add_preempt_count();
 5)   1.115 us    |        }
 5) + 46.645 us   |      } /* idle_balance */
 5)               |      put_prev_task_rt() {
 5)               |        update_curr_rt() {
 5)               |          cpuacct_charge() {
 5)   0.110 us    |            __rcu_read_lock();
 5)   0.110 us    |            __rcu_read_unlock();
 5)   2.111 us    |          }
 5)   0.100 us    |          sched_avg_update();
 5)               |          _raw_spin_lock() {
 5)   0.116 us    |            add_preempt_count();
 5)   1.151 us    |          }
 5)   0.122 us    |          balance_runtime();
 5)   0.110 us    |          sub_preempt_count();
 5)   8.165 us    |        }
 5)   9.152 us    |      }
 5)   0.148 us    |      pick_next_task_fair();
 5)   0.112 us    |      pick_next_task_stop();
 5)   0.117 us    |      pick_next_task_rt();
 5)   0.123 us    |      pick_next_task_fair();
 5)   0.138 us    |      pick_next_task_idle();
 # 进程切换
 ------------------------------------------
 5)   ksoftir-39   =>    <idle>-0   
 ------------------------------------------

 5)               |      finish_task_switch() {
 5)               |        _raw_spin_unlock_irq() {
 5)   0.260 us    |          sub_preempt_count();
 5)   1.289 us    |        }
 5)   2.309 us    |      }
 5)   0.132 us    |      sub_preempt_count();
 5) ! 151.784 us  |    } /* __schedule */
 5)   0.272 us    |  } /* sub_preempt_count */
```

要过滤trace的函数的话，往"set_graph_function"文件中写入想要追踪的函数名就好。