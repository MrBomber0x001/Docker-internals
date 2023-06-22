
<h1 align="center">Docker Internals (🏗 under construction....)</h1>

![Docker Int](https://user-images.githubusercontent.com/42917814/210157620-b58e91be-ca3c-4797-85c1-fff863152720.png)


## Why ? 

This repository is a collection of articles, books, and blog posts related to the internals of Docker, as well as personal notes and insights on the topic. 
Whether you're new to Docker or a seasoned user, this repository aims to provide you with valuable resources to help you understand and use Docker more effectively and most importantly writing your own lite version of Docker 😀.

## resources

The resources I've gathered are divided into main categories [Books, articles and videos]

### Books
- Docker in Action, Manning[^1]
- Container Security, Liz Rice, o'reilly[^2]
- Learning modern Linux, o'reilly[^3]

Articles and Demos
---

### cgroups

There's a series of 4 main articles cover **Cgroups** in details.
1.  <a href="https://www.schutzwerk.com/en/blog/linux-container-cgroups-01-intro">Part1: Intro</a>
2. <a href="https://www.schutzwerk.com/en/blog/linux-container-cgroups-02-network-block-io">Part 2: Network Block I/O</a>
3. <a href="https://www.schutzwerk.com/en/blog/linux-container-cgroups-03-memory-cpu-freezer-dev">Part 3: Memory and CPU</a>
4. <a href="https://www.schutzwerk.com/en/blog/linux-container-cgroups-04-groups-kernel">Part 4: Kernel Cgroups</a>

### namespaces

From the same resource as above, a series covering `namespaces` in details

1. <a href="https://www.schutzwerk.com/en/blog/linux-container-namespaces01-intro">Part 1: Intro</a>
3. <a href="https://www.schutzwerk.com/en/blog/linux-container-namespaces02-mnt">Part 2: Mount Namespaces</a>
4. <a href="https://www.schutzwerk.com/en/blog/linux-container-namespaces03-pid-net">Part 3: PID namespaces</a>
5. <a href="https://www.schutzwerk.com/en/blog/linux-container-namespaces04-user">Part 4: User Namespaces</a>
6. <a href="https://www.schutzwerk.com/en/blog/linux-container-capabilities">Capabilites</a>

Resources to Dig Further
------

- <a href="https://www.youtube.com/watch?v=-YnMr1lj4Z8">Intro to Namespaces</a>
- [How to use chroot - howtogeek](https://www.howtogeek.com/441534/how-to-use-the-chroot-command-on-linux) \[done\]
- [Bocker - A simple docker implemented in 100 lines of code](https://github.com/p8952/bocker/blob/master/bocker) \[will not\]
- [Resource Management using cgroups - Red Hat](https://access.redhat.com/documentation/enus/red_hat_enterprise_linux/6/html/resource_management_guide/ch01) \[will not\]
- [What are namespace, cgroups in linux kernerl - Nginx blog](https://www.nginx.com/blog/what-are-namespaces-cgroups-how-do-they-work/) \[Done\]
- [How Docker works - LiveOverflow](https://www.youtube.com/watch?v=-YnMr1lj4Z8) \[Not done\]
- [What are containers made of - Docke conference](https://www.youtube.com/watch?v=sK5i-N34im8)
- [http://docker-saigon.github.io/post/Docker-Internals](http://docker-saigon.github.io/post/Docker-Internals) \[not done\]
- [Linux Container Primitives: cgroups, namespaces, and more!](https://www.youtube.com/watch?v=x1npPrzyKfs&list=LL&index=2&t=1586s) \[not done\]
- [write Docker container using Golang](https://www.youtube.com/watch?v=-NzfOhSAZpA&list=LL&index=4)
- [Docker Internals](https://blog.because-security.com/t/docker-the-universal-build-system-for-system-and-security-development-wiki)

[^1]: https://www.manning.com/books/docker-in-action-second-edition covers advanced topics of Docker
[^2]: https://www.oreilly.com/library/view/container-security/9781492056690/ which covers most of the internals in details.
[^3]: https://www.oreilly.com/library/view/learning-modern-linux/9781098108939/
