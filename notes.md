## Table of contents

1.  History of containerization
    
2.  What Docker does && its components architecture
    
3.  tracing syscall from the docker client to `unshare` command
    
4.  Leveraging Linux kernel features by building a container
    
5.  Using Docker images without Docker
    
6.  Where to go?
    

## Prerequisites

*   Linux command line familiarity
    
*   Basic knowledge of networking and operating systems
    

## Introduction

Most people probably give little thought to how containers work under the covers. Still, I think it’s important to understand the underlying technologies – it helps to inform our decision‑making processes.

This writeup is divided into two main sections, in the first section I will explain the basic idea behind Linux containers using its kernel features like Namespaces, Cgroups, and chroot.

In the second section, I will walk you through a practical example of all that you learned so far by building a container from scratch.

Aren't you excited like me? let's do it

%[https://media.giphy.com/media/BpGWitbFZflfSUYuZ9/giphy.gif] 

* * *

## History of containerization

Before we start talking about containers, let's see first what is Docker and what does it for you.

later we'll see what it does on behalf of you

![High level Docker Architecture ](https://cdn.educba.com/academy/wp-content/uploads/2019/10/Docker-Architecture-2.png align="center")

### Tracing System Calls Starting From Docker Client

### **What is a container?**

In 4 bullet points:

*   Containers share the host kernel
    
*   Containers use the kernel's ability to group processes for resource control
    
*   Containers ensure isolation through namespaces
    
*   Containers feel like lightweight VMs (lower footprint, faster), but are **not Virtual Machines!**
    

Components of a container **ecosystem** include:

*   Runtime
    
*   Image distribution
    
*   Tooling
    

## Essential Technologies

### **1️⃣ Kernel Namespaces**

Allow you to create isolation of:

*   Process trees (PID Namespace)
    
*   Mounts (MNT namespace) `wc -l /proc/mounts`
    
*   Network (Net namespace) `ip addr`
    
*   Users / UIDs (User Namespace)
    
*   Hostnames (UTS Namespace) `hostname`
    
*   Inter-Process Communication (IPC Namespace) `ipcs`
    

### 2️⃣ Cgroups

Kernel control groups (cgroups) allow you to do accounting on resources used by processes, a little bit of access control on device nodes and other things such as freezing groups of processes.

cgroups consist of one hierarchy (tree) per resource (CPU, memory, …) . for example:

```bash
cpu                      memory
├── batch                ├── 109
│   ├── hadoop           ├── 88 <
│   │   ├── 88 <         ├── 25
│   │   └── 109          ├── 26
└── realtime             └── databases
    ├── nginx                ├── 1008
    │   ├── 25               └── 524
    │   └── 26          
    ├── postgres 
    │   ├── 524  
    └── redis    
        └── 1008
```

* * *

## chroots

With `chroot` you can set up and [run programs or interactive shells](https://man7.org/linux/man-pages/man1/chroot.1.html) such as Bash in an encapsulated filesystem that is prevented from interacting with your regular filesystem. Everything within the `chroot` environment is penned in and contained. Nothing in the `chroot` environment can see out past its own, special, root directory without escalating to root privileges. That has earned this type of environment the nickname of a `chroot` jail

In some senses, `chroot` environments are closer to containers such as [LXC](https://linuxcontainers.org/) than to virtual machines. They’re lightweight, quick to deploy, and creating and firing one up can be automated

We need to create directories to hold the portions of the operating system our `chroot` environment will require. We’re going to set up a minimalist Linux environment that uses Bash as the interactive shell.

follow along with me

```sh
docker run -it --name docker-host --rm --privileged ubuntu:bionic
```

Then inside this ubuntu container, we're going to make our `chroot` environment

`ldd` is used to list the command's dependencies

```bash
root@6639a75f6ca3:/# mkdir my-new-root
root@6639a75f6ca3:/# cp bin/bash my-new-root/bin
root@6639a75f6ca3:/# ldd bin/bash
root@6639a75f6ca3:/# mkdir my-new-root/lib{,64}
root@6639a75f6ca3:/# chroot my-new-root bash
```

To see which libs bash depends on

```sh
$ ldd bin/bash

linux-vdso.so.1 (0x00007ffd55bfe000)
	libtinfo.so.5 => /lib/x86_64-linux-gnu/libtinfo.so.5 (0x00007fdb3e708000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fdb3e504000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fdb3e113000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fdb3ec4c000)
```

so we need to move those libs to their dedicated folders

```go
1. using package manager
```

```bash
root@6639a75f6ca3:/# cp /lib/x86_64-linux-gnu/libtinfo.so.5 /lib/x86_64-linux-gnu/libdl.so.2 lib/x86_64-linux-gnu/libc.so.6 my-new-root/lib
root@6639a75f6ca3:/# cp /lib64/ld-linux-x86-64.so.2 my-new-root/lib64/
```

after moving we can now change the root to our new root directory and start a bash session

```bash
root@6639a75f6ca3:/# chroot my-new-root bash
```

🔴 if you try to use any command you'll notice there are no commands to be executed, that's because all we've got inside this new root is the bash only

you will notice two important things:

*   now our root directory is my-new-root and we can't reach outside this environment now
    
*   if we want to add a new command, for example, we've to explicitly do this by moving the command from whatever it resides in our host and moving it's libs to our `chroot` environment
    

So what's the problem with chroot?

> what's the problem?

## Namespaces

### What are Namespaces? 🤔

> ***Namespaces are a feature of the Linux kernel that partitions kernel resources such that one set of processes sees one set of resources while another set of processes sees a different set of resources***

The idea of namespaces is to let each process take its own process IDs, network layer, resources, etc to avoid conflict with other processes.

There are more than one type of namespace, for example:

A [**process ID (PID) namespace**](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html) assigns a set of PIDs to processes that are independent of the set of PIDs in other namespaces. The first process created in a new namespace has PID 1 and child processes are assigned subsequent PIDs. If a child process is created with its own PID namespace, it has PID 1 in that namespace as well as its PID in the parent process’ namespace.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1670159658259/-9brMk2EK.png align="left")

From the image above, we can see that:

*   the `PID namespace 1(parent)` can see child namespaces PIDs, but the child can't see outside themselves
    
*   in each namespace, the first process is `PID 1`
    

To demonstrate this idea in practice, let's get our hands dirty and start a long-running process inside your main container

```bash
root@6639a75f6ca3:/ echo "top secret" >> /my-new-root/secret.txt
root@6639a75f6ca3:/ tail -f my-new-root/secret.txt
```

after that connect to the same main container from another shell

```bash
docker exec -it docker-host bash
```

From the `Docker host 2` you've just connected to, we're going to list all the processes

```bash
# Docker-host 2
root@6639a75f6ca3:/# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0  18520  3348 pts/0    Ss   21:21   0:00 bash
root          46  0.0  0.0   4580   832 pts/0    S+   21:38   0:00 tail -f secret.txt
root          47  0.1  0.0  18520  3376 pts/1    Ss   21:41   0:00 bash
root          58  0.0  0.0  34416  2904 pts/1    R+   21:42   0:00 ps aux
```

🔵😃 notice that we see the `tail -f` and `bash [47]` from our chroot environment.

let's kill that process

```bash
// Docker-host 2
kill 46 # pkill <processName>
```

you'll notice that the process got terminated from `Docker host 1`

```sh
// Docker-host 1
root@6639a75f6ca3:/# tail -f secret.txt
secret
Terminated
```

so with namespaces, we can solve this problem by isolating each process from seeing what other processes are doing! we'll use a command called `unshare`

### 🧪 A practical example

1.  we'll create a child process (jailed process)
    
2.  then we'll unshare \[processes, file systems, network layer\] from that child process
    
3.  from docker host 2 we'll see how each host can control the child process but the child cannot
    

**🟢 using** `debootstrap` **which is used to bootstrap a new environment and you can chroot the directory into it.**

```sh
// Docker host 1
$ apt-get update
$ apt-get install debootstrap -y
// creating a new bare minimal set of a file system we need to run a debian based ubuntu
$ debootstrap --variant=minbase bionic /better-root
```

then we can unshare this newly created environment

```sh
// Docker host 1
$ unshare --mount --uts --ipc --net --pid --fork --user --map-root-user chroot /better-root bash
root@6639a75f6ca3:/# ps aux
Error, do this: mount -t proc proc /proc
# we need to mount a process first
$ mount -t proc none /proc mount -t sysfs none /sys mount -t tmpfs none /tmp
```

> **From the host we can see inside the child, from a child we can't see outside of itself**

```bash
# Docker-host 1
$ echo "sample" >> sample.txt
$ tail -f sample.txt

# Docker-host 2
root@6639a75f6ca3:/# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0  18520  3356 pts/0    Ss   21:21   0:00 bash
root          47  0.0  0.0  18520  3396 pts/1    Ss   21:41   0:00 bash
root        7406  0.0  0.0   4532   772 pts/0    S    21:53   0:00 unshare --mount --uts --ipc --net --pid --fork --user --map-root-user chroot /better-root bash
root        7407  0.0  0.0  18512  3424 pts/0    S    21:53   0:00 bash
root        7427  0.0  0.0   4572   768 pts/0    S+   22:05   0:00 tail -f sample.txt
root        7428  0.0  0.0  34416  2928 pts/1    R+   22:05   0:00 ps aux

$ kill 7427

# Docker-host 1
root@6639a75f6ca3:/# tail -f sample.txt
welcome
Terminated
```

🟢 **the host can control everything that's happening inside the child process, but the child process can't see outside itself**

## Control groups (Cgroups)

A control group (cgroup) is a Linux kernel feature that limits, accounts for, and isolates the resource usage (CPU, memory, disk I/O, network, and so on) of a collection of processes.

Cgroups provide the following features:

*   **Resource limits** – You can configure a cgroup to limit how much of a particular resource (memory or CPU, for example) a process can use.
    
*   **Prioritization** – You can control how much of a resource (CPU, disk, or network) a process can use compared to processes in another cgroup when there is resource contention.
    
*   **Accounting** – Resource limits are monitored and reported at the cgroup level.
    
*   **Control** – You can change the status (frozen, stopped, or restarted) of all processes in a cgroup with a single command.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1670160480614/wr1G6-1Sz.png align="left")

### Creating a Cgroup

The following command creates a v1 cgroup (you can tell by pathname format) called `foo` and sets the memory limit for it to 50,000,000 bytes (50 MB).

by creating a directory and

```bash
root # mkdir -p /sys/fs/cgroup/memory/foo
root # echo 50000000 > /sys/fs/cgroup/memory/foo/memory.limit_in_bytes
```

```bash
Making a long running process u
root # ./test.sh & 
[1] 2428
root # cgroup  tool

root # echo 2428 > /sys/fs/cgroup/memory/foo/cgroup.procs
```

```bash
root # ps -o cgroup 2428
CGROUP
12:pids:/user.slice/user-0.slice/\
session-13.scope,10:devices:/user.slice,6:memory:/foo,...
```

## Final words

In the next writeup, I will be showing you how to build a container using Golang, it will be a sequel to this writeup, see you there :D

follow up

## Resources

You can dig deeper by reading about those features in detail

*   [How to use chroot - howtogeek](https://www.howtogeek.com/441534/how-to-use-the-chroot-command-on-linux) \[done\]
    
*   [Bocker - A simple docker implemented in 100 lines of code](https://github.com/p8952/bocker/blob/master/bocker) \[will not\]
    
*   [Resource Management using cgroups - Red Hat](https://access.redhat.com/documentation/enus/red_hat_enterprise_linux/6/html/resource_management_guide/ch01) \[will not\]
    
*   [What are namespace, cgroups in linux kernerl - Nginx blog](https://www.nginx.com/blog/what-are-namespaces-cgroups-how-do-they-work/) \[Done\]
    
*   [How Docker works - LiveOverflow](https://www.youtube.com/watch?v=-YnMr1lj4Z8) \[Not done\]
    
*   [What are containers made of - Docke conference](https://www.youtube.com/watch?v=sK5i-N34im8)
    
*   [http://docker-saigon.github.io/post/Docker-Internals](http://docker-saigon.github.io/post/Docker-Internals) \[not done\]
    
*   [Linux Container Primitives: cgroups, namespaces, and more!](https://www.youtube.com/watch?v=x1npPrzyKfs&list=LL&index=2&t=1586s) \[not done\]
    
*   [write Docker container using Golang](https://www.youtube.com/watch?v=-NzfOhSAZpA&list=LL&index=4)
    

## Introduction

containers are an abstraction over serveral different linux technologies, think of containers as a particular way of combining those linux primitives together

my philosophy will be the following:

1.  demonstrating the technology
    
2.  making a demo
    

in the end, we will combine what we've learned so far by actually building a container!

## control groups

cgroups are a linux system for tracking, grouping an organzing the processes that run

every process is tracked with cgroups, regardless of whether it's in a container or not!

cgroups are typically used to associate processes with resources with croups you can track how much a grou of processes are using for a given kind of resource besides giving you the ability to limit or prioritize what resources are avaialbe to a group of processes

the way you interact with cgroups, is with subsystems subsystems are concrete implementations that are actually bound to resourcesThere are bunch of subystems that are abialable on linux :

*   memory
    
*   CPU time
    
*   block I/O
    
*   Number of discrete processes (pids)
    
*   cpu & memory pinning
    
*   freezer (used by docker pause)
    
*   devices
    
*   network priority
    

that subsystems are called "resources controllers", meaning they can track or limit a particular kind of resource for the processes assigned to the group

:note: each of the subsystems are indepedent, they can organize processes seperate from each other for example you could have a single cpu cgroup with two processes, but those two processes can be assigned to two different memory cgroup

### Hierarchical representation

All of the cgroup subsystems arrange processes in a hierarchy each subsystem has indepedent hirerachy every task or process id running on the host is represented in exactly one of the cgroups within a given subsytem hirerachy

what does this indepednability offers us? it offers an expressive segmentation across resource types, for example,you could hae two processes share the total amount of memory that they consume, but give one process more cpu time than the other

another note: some resource controllers apply settings from the parent level to the child levels, while others consider each lvl in the hireachy independtaly

when a new process is started it begins in the same cgroup as its parent

### Interacting with cgroups

you can interact with cgroups, via a virtual file system typically mounted at /sys/fs/cgroup a bunch of fils and folders are here, but they are just interfaces into the kernel's data structures for cgroups

*   each directory inside a given "subsystem" root, represents a cgroups, in each directory, you'll see a file called "task"
    

this file hold all of the process ids for the processes assigned to that particaular cgroup

### Demo

```bash
root@ef9bd7c7f5b6:/# ls /sys/fs/cgroup/devices
cgroup.clone_children  devices.allow  devices.list       tasks
cgroup.procs           devices.deny   notify_on_release
```

here you can see the files that control how the cgroup subsystems itself are configured which prefixes with cgroup and controlling devices prefixed with devices ans task file

:question: what if i want to see which cgroup does any process belongs to? we can do this by using proc virtual file system

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

the limit is controlled by pids.max file let's see an example

```bash
echo 2 | sudo tee /sys/fs/cgroup/pids/lfnw/pids.max
```

now try to cat the file again `bash $(cat /sys/fs/cgroup/pids/lfnw/tasks 1>&2)`

### How Docker uses Cgroup?

we're going to run a container with a cpu resource control \`\`\`bash

````bash


## Namespaces

### Mount Namespaces
```bash
mount

output:
````

### Procfs virtual filesystem

```bash
readlink /proc/$$/ns/*


output:
```