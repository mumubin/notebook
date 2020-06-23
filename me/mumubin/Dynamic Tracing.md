经常看今日说法，特羡慕警察，破案什么的特有成就感，作为一个程序员，怎么才能有这种成就感呢？当然就是生产问题的troubleshooting，生产问题又分为两种
，一种是业务问题，一种是性能问题，作为通用技术，咱要学就学性能问题Troubleshooting！

博客:
1. [动态追踪技术漫谈](https://blog.openresty.com.cn/cn/dynamic-tracing/)
2. [《性能之巅》学习笔记之Dtrace](https://zhuanlan.zhihu.com/p/71437161)
3. [用高魔的姿势调试python程序](https://github.com/jin10086/pycon_2016/blob/master/sh/4%20%E3%80%8A%E7%94%A8%E9%AB%98%E9%AD%94%E7%9A%84%E5%A7%BF%E5%8A%BF%E8%B0%83%20python%20%E7%A8%8B%E5%BA%8F%E3%80%8B%E9%83%AD%E6%B5%A9%E5%B7%9D.pdf)
4. [如何读懂火焰图](http://www.ruanyifeng.com/blog/2017/09/flame-graph.html)
5. https://github.com/emfree/systemtap-python-tools TODO
6. https://myaut.github.io/dtrace-stap-book/
7. https://speakerdeck.com/emfree/python-tracing-superpowers-with-systems-tools
8. http://www.brendangregg.com/overview.html
9. DTrace: Dynamic Tracing in Oracle Solaris, Mac OS X, and FreeBSD,
- iostat
- perf
10. https://www.slideshare.net/brendangregg/java-performance-analysis-on-linux-with-flame-graphs


## Learn - BPF Performance Tools: Linux System And Application Observability
### Chapter 1: Technologies


    - execsnoop
    - biolatency

Reference:
    - https://github.com/iovisor/bcc
    - https://github.com/iovisor/bpftrace
    - http://www.brendangregg.com/bpf-performance-tools-book.html.
实践:
1.安装systemtap
```shell script
yum group install "Development Tools"
yum distro-sync
```