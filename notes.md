## introduction
containers are an abstraction over serveral different linux technologies,
think of containers as a particular way of combining those linux primitives together

my philophy will be the following:
1. demonstrating the technology
2. makeing a demo

at the end, we will combine what we've learnt so far by actually building a container!

## control groups
cgroups are a linux system for tracking, grouping an organzing the processes that run

every process is tracked with cgroups, regardless of whether it's in a container or not!

cgroups are typically used to associate processes with resources
with croups you can track how much a grou of processes are using for a given kind of resource
besides giving you the ability to limit or prioritize what resources are avaialbe to a group of processes


<insert images of container and linux kernel>


the way you interact with cgroups, is with subsystems
subsystems are concrete implementations that are actually bound to resourcesThere are bunch of subystems that are abialable on linux
:
- memory
- cpu time
- block I/O
- Number of discrete processes (pids)
- cpu & memory pinning
- freezer (used by docker pause)
- devices
- network priority

those subsytesm are called "resouces controllers", meaning they can track or limit a particular kind of resource for the processes assigned to the group

:note: each of the subsystem are indepedent, they can organize processes seperate from each other
for example: 
you could've a single cpu cgroup with two processes, but those two processes can be assigned to two different memory cgroup


### Hierachical representation
<insert image>
All of the cgroup subsystems arrange processes in a hierarchy 
each subsystem has indepedent hirerachy
every task or process id running on the host is represented in exactly one of the cgroups within a given subsytem hirerachy

what does this indepednability offers us?
it offers an expressive segmentation across resource types, for example,you could hae two processes share the total amount of memory that they consume, but give one process more cpu time than the other


another note:
some resource controllers apply settings from the parent level to the child levels, while others consider each lvl in the hireachy independtaly

when a new process is started it begins in the same cgroup as its parent

### Interacting with cgroups
<insert image>
you can interact with cgroups, via a virtual file system <put link here>
typically mounted at /sys/fs/cgroup
a bunch of fils and folders are here, but they are just interfaces into the kernel's data structures for cgroups

- each directory inside a given "subsystem" root, represents a cgroups,
in each directory, you'll see a file called "task"

this file hold all of the process ids for the processes assigned to that particaular cgroup


### Demo
```bash
root@ef9bd7c7f5b6:/# ls /sys/fs/cgroup/devices
cgroup.clone_children  devices.allow  devices.list       tasks
cgroup.procs           devices.deny   notify_on_release
```
here you can see the files that control how the cgroup subsystems itself are configured which prefixes with cgroup
and controlling devices prefixed with devices
ans task file

:question: what if i want to see which cgroup does any process belongs to?
we can do this by using proc virtual file system

the proc file system contains a directory that corresponds to each process id 


```bash
root@ef9bd7c7f5b6:/# cat /sys/fs/cgroup/devices/tasks
1
16
root@ef9bd7c7f5b6:/# echo $$
1
root@ef9bd7c7f5b6:/# ls /proc
1          crypto       ioports      loadavg       partitions   timer_list
17         devices      irq          locks         sched_debug  tty
acpi       diskstats    kallsyms     mdstat        schedstat    uptime
buddyinfo  dma          kcore        meminfo       self         version
bus        driver       key-users    misc          softirqs     vmallocinfo
cgroups    execdomains  keys         modules       stat         vmstat
cmdline    filesystems  kmsg         mounts        swaps        zoneinfo
config.gz  fs           kpagecgroup  mtrr          sys
consoles   interrupts   kpagecount   net           sysvipc
cpuinfo    iomem        kpageflags   pagetypeinfo  thread-self
root@ef9bd7c7f5b6:/# cat /proc/1/cgroup
27:name=systemd:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
26:rdma:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
25:pids:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
24:hugetlb:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
23:net_prio:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
22:perf_event:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
21:net_cls:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
20:freezer:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
19:devices:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
18:memory:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
17:blkio:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
16:cpuacct:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
15:cpu:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
14:cpuset:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
0::/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
root@ef9bd7c7f5b6:/#
```

now let's see how to move a process to a cgroup and change some settings on that cgroup

we can make a new cgroup by creating a new directory 

```bash
~ via 🐹 v1.19.2 via  v10.19.0 
❯ echo $$
3147

~ via 🐹 v1.19.2 via  v10.19.0 
❯ cat /proc/3147/cgroup 
13:rdma:/
12:blkio:/user.slice
11:perf_event:/
10:pids:/user.slice/user-1000.slice/user@1000.service
9:freezer:/
8:memory:/user.slice/user-1000.slice/user@1000.service
7:cpu,cpuacct:/user.slice
6:hugetlb:/
5:devices:/user.slice
4:cpuset:/
3:net_cls,net_prio:/
2:misc:/
1:name=systemd:/user.slice/user-1000.slice/user@1000.service/apps.slice/apps-org.gnome.Terminal.slice/vte-spawn-8a2ac531-aa46-4222-aedf-2d9ef54b37af.scope
0::/user.slice/user-1000.slice/user@1000.service/apps.slice/apps-org.gnome.Terminal.slice/vte-spawn-8a2ac531-aa46-4222-aedf-2d9ef54b37af.scope

~ via 🐹 v1.19.2 via  v10.19.0 
❯ sudo mkdir /sysfs/cgroup/pids/lfnw
[sudo] password for meska: 
Sorry, try again.
[sudo] password for meska: 
mkdir: cannot create directory ‘/sysfs/cgroup/pids/lfnw’: No such file or directory

~ via 🐹 v1.19.2 via  v10.19.0 took 6s 
❯ sudo mkdir /sys/fs/cgroup/pids/lfnw

~ via 🐹 v1.19.2 via  v10.19.0 
❯ ls /sys/fs/cgroup/pids/lfnw
cgroup.clone_children  notify_on_release  pids.events  tasks
cgroup.procs	       pids.current	  pids.max

~ via 🐹 v1.19.2 via  v10.19.0 
❯ echo 3147 | sudo tee /sys/fs/cgroup/pids/lfnw/tasks
3147

~ via 🐹 v1.19.2 via  v10.19.0 
❯ cat /proc/3147/cgroup                             
13:rdma:/
12:blkio:/user.slice
11:perf_event:/
10:pids:/lfnw
9:freezer:/
8:memory:/user.slice/user-1000.slice/user@1000.service
7:cpu,cpuacct:/user.slice
6:hugetlb:/
5:devices:/user.slice
4:cpuset:/
3:net_cls,net_prio:/
2:misc:/
1:name=systemd:/user.slice/user-1000.slice/user@1000.service/apps.slice/apps-org.gnome.Terminal.slice/vte-spawn-8a2ac531-aa46-4222-aedf-2d9ef54b37af.scope
0::/user.slice/user-1000.slice/user@1000.service/apps.slice/apps-org.gnome.Terminal.slice/vte-spawn-8a2ac531-aa46-4222-aedf-2d9ef54b37af.scope

~ via 🐹 v1.19.2 via  v10.19.0 
❯ cat /sys/fs/cgroup/pids/lfnw/tasks      
3147
3602

~ via 🐹 v1.19.2 via  v10.19.0 
❯ cat /sys/fs/cgroup/pids/lfnw/tasks
3147
3629

~ via 🐹 v1.19.2 via  v10.19.0 
❯ cat /sys/fs/cgroup/pids/lfnw/tasks
3147
3653

~ via 🐹 v1.19.2 via  v10.19.0 
❯ cat /sys/fs/cgroup/pids/lfnw/tasks
3147
3677

~ via 🐹 v1.19.2 via  v10.19.0 
❯ cat /sys/fs/cgroup/pids/lfnw/tasks
3147
3701

## notice that with every time we cat the file, a new process id is found?
## from where comes the new one?
because as we've mentioned before, with the child process here (cat) inherit from parent process (bash) the cgroup
~ via 🐹 v1.19.2 via  v10.19.0 
❯ n 

~ via 🐹 v1.19.2 via  v10.19.0 
❯
```

the pid cgroup controls the number of processes that can run, and limit the impact of mistakes or attacks

the limit is controlled by pids.max file
let's see an example


```bash
echo 2 | sudo tee /sys/fs/cgroup/pids/lfnw/pids.max
```

now try to cat the file again
```bash
$(cat /sys/fs/cgroup/pids/lfnw/tasks 1>&2)
```





### How Docker uses Cgroup?
we're going to run a container with a cpu resource control
```bash
 


```


## Namespaces

### Mount Namespaces
```bash
mount

output:

```
### Procfs virtual filesytesm
```bash
readlink /proc/$$/ns/*


output:
```

