<h4 id="P2R0u">背景信息：</h4>
       某AI大模型六小龙客户近期在做模型的性能对比，陡然发现，新加坡的集群业务启动竟然比北京集群启动足足慢了15秒（北京5s，新加坡20s），按照客户的说法，启动的时候业务是什么也不干的，凭啥慢这么多？TAM同学带着这个疑问，找到了我们。。。

**pod内的进程启动时间：**

![](https://intranetproxy.alipay.com/skylark/lark/0/2025/png/26092/1745379230455-e0f18e1a-b903-4f8b-81f3-4327d8ad0946.png)

**节点上观测到的亦是如此：**

![](https://intranetproxy.alipay.com/skylark/lark/0/2025/png/26092/1745219081866-779abd29-f189-4c1b-bcbc-7804be0fd82a.png)



<h5 id="DVvea">查前思考：</h5>
      了解了这个背景后，我的第一反应是客户在新加坡发布的镜像代码里面，大概率是写了往国内注册/连接的服务，比如说 ：

+ 微服务注入（nacos，mse）
+ Redis的连接
+ 数据库的连接等

但是事与愿违，这么简单可能也不会找到我们，与客户交流及抓包看到业务启动过程中确实没有网络报文的发生



<h4 id="rwJ5G">既然如此，那就开始整活：</h4>
   客户允许我们干掉s6进程做测试，S6是容器的根（1号）进程，干掉后会被kubelet重拉container，所以我们只需要在拉起来之前触发strace跟踪即可，下面的脚本是while循环监控s6-svscan的进程名称然后获取pid，并触发跟踪， 

<h6 id="rGTnX">while循环strace脚本：</h6>
```plain
#!/bin/bash

process_name="s6-svscan"

pid=""

while [ -z "$pid" ]; do
    pid=$(pidof $process_name)
    if [ -n "$pid" ]; then
        strace -F -ff -T -tt -s 4096 -r -o kx.out -p $pid
        break
    else
        echo `date`
        sleep 0.1
    fi
done
```

不幸的是发现触发式跟踪会有遗漏，比如说 1号进程拉起s6-supervise这里没抓到，就没法进一步分析，且s6-svscan出现的时候已经延迟完了，，，

![](https://intranetproxy.alipay.com/skylark/lark/0/2025/png/26092/1745386374351-09842701-1d1c-4577-8ebe-1ff4f1a9ca70.png)

注意：s6-svscan出现的时候，延迟已经发生过了！大家思考一下为什么是这样的？

<h5 id="gxCUT">跟踪containerd-shim试试：</h5>
这种情况我们可以从进程树上看一下，既然可以直接kill s6-svscan来复现，因此呢我们实际跟踪这个container的主id（203232）即可

<h6 id="DbNY9">kill前的进程树：</h6>
![](https://intranetproxy.alipay.com/skylark/lark/0/2025/png/26092/1745377678416-2d5c23db-84a7-4423-ac57-bec4d95f7548.png)

```plain
strace -F -ff -T -tt -r -s 4096 -o s6.out -p 203232
```

如下图所示，被kill的1号进程 拉起业务进程的时候已经间隔了8秒 

![](https://intranetproxy.alipay.com/skylark/lark/0/2025/png/26092/1745377445615-50ca31e5-12f9-4f71-b934-6a106d031d4a.png)

<h6 id="eEsdc">kill后的进程树：</h6>
此时的进程树是这样的，不过我们先不管他![](https://intranetproxy.alipay.com/skylark/lark/0/2025/png/26092/1745377784437-885fd6ea-a145-404e-a0af-0375fa23f2dd.png)

<h5 id="O8GdI">分析strace信息：</h5>
根据进程树的拉起关系，我们先看主进程s6-svscan 以及他的第一个子进程s6-supervise s6-linux-init-shutdownd的跟踪信息

```plain
 226855  203232 Wed Apr 23 11:01:34 2025 /package/admin/s6/command/s6-svscan -d4 -- /run/service
 226981  226855 Wed Apr 23 11:01:42 2025 s6-supervise s6-linux-init-shutdownd
```

<h6 id="wpYQE">strace的跟踪日志显示：</h6>
从s6-supervise s6-linux-init-shutdownd的日志来看，这个进程fork出来的时候 已经是8秒后了，

```plain
# more s6.out.226981
11:01:42.297482 (+     0.000021) execve("/package/admin/s6-2.13.1.0/command/s6-supervise", ["s6-supervise", "s6-linux-init-shutdownd"], 0x7ffe17c54320 /* 1 var */) = 0 <0.000820>
```

所以我们要把视线放到主进程身上，**敲黑板了**，有没有课代表说下 为什么看到的进程id comm名称明明是s6-svscan，但是strace跟踪的时候 先看到的是runc:init呢？（container创建的过程，进程的替换）

```plain
# more s6.out.226855
11:01:35.040609 (+     0.000009) prctl(PR_SET_NAME, "runc:[2:INIT]") = 0 <0.000006
这里是runc init在工作
```

<h6 id="m0JL2">读取环境变量：</h6>
```plain
11:01:35.050212 (+     0.000035) read(3, "{\"args\":[\"/init\"],\"env\":[\"PATH=/lsiopy/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\",\"HOSTNAME=s6-5cd86f495d-ndwwl\",\
"HOME=/config\",\"LANGUAGE=en_US.UTF-8\",\"LANG=en_US.UTF-8\",\"TERM=xterm\",\"S6_CMD_WAIT_FOR_SERVICES_MAXTIME=0\",\"S6_VERBOSITY=1\",\"S6_STAGE2_HOOK=/docker-mods\",\"VIRTUAL_ENV=/lsiopy\
",\"DISPLAY=:1\",\"PERL5LIB=/usr/local/bin\",\"OMP_WAIT_POLICY=PASSIVE\",\"GOMP_SPINCOUNT=0\",\"START_DOCKER=true\",\"PULSE_RUNTIME_PATH=/defaults\",\"NVIDIA_DRIVER_CAPABILITIES=all\",\"LSI
O_FIRST_PARTY=true\",\"TITLE=C", 512) = 512 <0.000010>

11:01:35.050301 (+     0.000022) read(3, "hromium\",\"MY_SERVICE_52_SERVICE_HOST=192.168.139.117\",\"MY_SERVICE_440_PORT_80_TCP_PORT=80\",\"MY_SERVICE_336_SERVICE_PORT_80_80=80\",\"MY_SERVI
CE_280_PORT_80_TCP_ADDR=192.168.162.77\",\"MY_SERVICE_454_SERVICE_PORT=80\",\"MY_SERVICE_365_SERVICE_PORT=80\",\"MY_SERVICE_194_SERVICE_PORT_80_80=80\",\"MY_SERVICE_100_PORT_80_TCP_PORT=80\
",\"MY_SERVICE_379_SERVICE_PORT_80_80=80\",\"MY_SERVICE_49_PORT_80_TCP_PORT=80\",\"MY_SERVICE_150_SERVICE_PORT=80\",\"MY_SERVICE_307_PORT_80_TCP=tcp://192.168.101.68:80\",\"MY_SERVICE_153_P
ORT_80_TCP_PROTO=tcp\",\"MY_SERVICE_321_SERVICE_PORT_80_80=80\",\"MY_SERVICE_344_PORT_80_TCP=tcp://192.168.52.15:80\",\"MY_SERVICE_484_PORT_80_TCP=tcp://192.168.25.142:80\",\"MY_SERVICE_457
_SERVICE_PORT_80_80=80\",\"MY_SERVICE_318_SERVICE_HOST=192.168.251.22\",\"MY_SERVICE_234_SERVICE_PORT_80_80=80\",\"MY_SERVICE_208_SERVICE_PORT=80\",\"MY_SERVICE_367_PORT_80_TCP_PORT=80\",\"
MY_SERVICE_429_PORT_80_TCP_PORT=80\",\"MY_SERVICE_265_PORT_80_TCP_PORT=80\",\"MY_SERVICE_122_PORT=tcp://192.168.54.41:80\",\"MY_SERVICE_342_SERVICE_PORT_80_80=80\",\"MY_SERV", 1024) = 1024
<0.000007>

```

<h6 id="ThDIT">读取系统的信息：</h6>
```plain
11:01:35.098571 (+     0.000022) mount("", "/", 0xc0003191ee, MS_REC|MS_SLAVE, NULL) = 0 <0.000074>
11:01:35.098664 (+     0.000093) openat(AT_FDCWD, "/proc/self/mountinfo", O_RDONLY|O_CLOEXEC) = 6 <0.000015>
11:01:35.098697 (+     0.000032) fcntl(6, F_GETFL) = 0x8000 (flags O_RDONLY|O_LARGEFILE) <0.000006>
11:01:35.098718 (+     0.000021) fcntl(6, F_SETFL, O_RDONLY|O_NONBLOCK|O_LARGEFILE) = 0 <0.000006>
11:01:35.098740 (+     0.000021) epoll_ctl(7, EPOLL_CTL_ADD, 6, {events=EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET, data={u32=507510788, u64=9187844879638593540}}) = 0 <0.000007>
11:01:35.098766 (+     0.000026) read(6, "558 557 253:3 / / rw,relatime master:1 - ext4 /dev/vda3 rw\n559 558 0:21 / /sys rw,nosuid,nodev,noexec,relatime master:2 - sysfs sysfs rw\n560 559
```

<h6 id="hf8xR">配置环境变量：</h6>
```plain
11:01:35.383653 (+     0.000023) openat(3, "MY_SERVICE_343_SERVICE_PORT", O_WRONLY|O_CREAT|O_TRUNC|O_NONBLOCK|O_LARGEFILE|O_CLOEXEC, 0666) = 4 <0.000066>
11:01:35.383744 (+     0.000091) fcntl(4, F_GETFL) = 0x8801 (flags O_WRONLY|O_NONBLOCK|O_LARGEFILE) <0.000006>
11:01:35.383766 (+     0.000022) fcntl(4, F_SETFL, O_WRONLY|O_LARGEFILE) = 0 <0.000005>
11:01:35.383788 (+     0.000021) write(4, "80", 2) = 2 <0.000012>
```

环境变量配置完毕，看起来是s6自己要用到这些环境变量，并将环境变量写到/run/s6/container_environment这个路径下

```plain
11:01:42.292525 (+     0.000026) openat(3, "MY_SERVICE_342_SERVICE_PORT", O_WRONLY|O_CREAT|O_TRUNC|O_NONBLOCK|O_LARGEFILE|O_CLOEXEC, 0666) = 4 <0.000087>
11:01:42.292712 (+     0.000028) write(4, "80", 2) = 2 <0.000028>
11:01:42.294449 (+     0.000022) chmod("/run/s6/container_environment:envdump:kKJJPk", 0700) = 0 <0.000015>
11:01:42.294484 (+     0.000035) rename("/run/s6/container_environment:envdump:kKJJPk", "/run/s6/container_environment") = 0 <0.000026>
```

而正八经的业务进程s6-svscan则是在环境变量初始化完成后才开始介入得

```plain
11:01:42.295069 (+     0.000020) execve("/package/admin/s6/command/s6-svscan", ["/package/admin/s6/command/s6-svscan", "-d4", "--", "/run/service"], 0x7ffc2b662c40 /* 1 var */) = 0 <0.00084
0>
11
```

一顿操作猛如虎之后开始拉服务了，可以跟上面的shutdownd的服务启动对比下

```plain
主进程拉shutdown
11:01:42.296879 (+     0.000031) stat("s6-linux-init-shutdownd", {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0 <0.000007>

shutdown进程的pid记录
11:01:42.297482 (+     0.000021) execve("/package/admin/s6-2.13.1.0/command/s6-supervise", ["s6-supervise", "s6-linux-init-shutdownd"], 0x7ffe17c54320 /* 1 var */) = 0 <0.000820>
```

看下上面的环境变量写入的目录（/run/s6/container_environment），果然，，，变量都被他存储起来了

![](https://intranetproxy.alipay.com/skylark/lark/0/2025/png/26092/1745378742955-f5549224-c9fd-49bd-ad01-68386b508b91.png)

由于环境变量较多，s6的变量写入是串行的，因此写入的整体时间较长，网上搜了下S6-svscan似乎是做进程、服务管理的，就是需要加载env，不能从S6的配置上关闭env的读取，这里不做赘述了，感兴趣的同学自行网络搜索一下



<h5 id="r2vbD">那么使用ack的场景中，如何优化这种问题？</h5>
<h6 id="fVW9x"><font style="color:rgb(51, 51, 51);">解决方案：</font></h6>
<font style="color:rgb(51, 51, 51);">   给pod增加 enableServiceLinks: false 禁用自动注入 service 信息到环境变量的方式来规避这个方式即可(后续svc走coredns解析做服务发现)</font>

![](https://intranetproxy.alipay.com/skylark/lark/0/2025/png/26092/1745387104466-f0f9896b-aa22-4bd2-a53c-d1939d267111.png)

<font style="color:rgb(51, 51, 51);"> 参考链接：</font>[<font style="color:rgb(49, 126, 208);">https://kubernetes.io/zh-cn/docs/tutorials/services/connect-applications-service/</font>](https://kubernetes.io/zh-cn/docs/tutorials/services/connect-applications-service/?spm=a2c9r.14518950.0.0.738d5e3bsJBOkT)

<h6 id="Mdhbr">为什么北京集群快呢？</h6>
    与客户聊了一下确认，北京的生产集群客户测试完毕之后都会把对应的资源如deployment，svc ，ing删除，而异常集群在测试期间起了大量的svc没有删除导致海外集群的pod启动慢

<h6 id="pm3Zo"><font style="color:rgb(51, 51, 51);">随时可复现，欢迎大家尝试，给出不同见解</font></h6>
<font style="color:rgb(51, 51, 51);">复现yaml参考，请注意走海外集群拉取镜像</font>

```plain
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: s6
  name: s6
  namespace: my
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: s6
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: s6
    spec:
      containers:
        - image: 'linuxserver/chromium:version-d3764286'
          imagePullPolicy: IfNotPresent
          name: s6
          resources:
            requests:
              cpu: 100m
              memory: 512Mi
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30

```

<font style="color:rgb(51, 51, 51);"></font>

