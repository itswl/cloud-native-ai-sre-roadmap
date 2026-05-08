# 第一阶段学习资料：Linux 与网络基础

生成日期：2026-05-08  
对应路线文档：[工作路线完善版.md](</Users/imwl/Documents/New project/工作路线完善版.md>)

## 目标

第一阶段的目标不是“学会更多命令”，而是把线上故障最终落地的地方看懂：

- 进程为什么卡住。
- 连接为什么断。
- DNS 为什么慢。
- Pod 为什么 OOMKilled。
- Service 为什么不通。
- CPU 明明没满，为什么请求还是慢。
- Nginx 502 / WebSocket 1006 / TLS / Service Mesh 问题最后如何落到 Linux 与网络栈。

阶段结束时，你应该能做到：

- 画出一次请求从客户端到 Pod 再返回的完整路径。
- 用 `ss`、`tcpdump`、`dig`、`journalctl`、`strace`、`/proc` 初步定位问题。
- 解释 cgroup、namespace、OOM、fd、epoll、TCP、DNS、conntrack、NAT 的作用。
- 把 Kubernetes 问题拆回 Linux 进程、网络命名空间、iptables / ipvs、DNS、资源限制这些底层问题。

## 建议环境

不要在生产机器上做实验。建议准备一个可销毁环境：

- 一台 Ubuntu 22.04 / 24.04 虚拟机或云主机。
- Docker 或 containerd。
- kind / minikube / k3d 任意一种本地 Kubernetes 环境。
- 一个普通用户账号和 sudo 权限。

建议安装工具：

```bash
sudo apt update
sudo apt install -y \
  curl wget vim jq tree htop sysstat procps psmisc \
  iproute2 iputils-ping dnsutils net-tools traceroute \
  tcpdump conntrack iptables nftables iperf3 \
  strace lsof
```

如果你使用 macOS 主机，Linux 相关实验尽量放到 Linux VM 里完成。macOS 的网络栈、systemd、cgroup、iptables 行为和 Linux 不同，不能直接类比线上 Kubernetes 节点。

## 学习顺序

推荐顺序：

```text
进程与 /proc
    ↓
fd / socket / epoll
    ↓
systemd 与日志
    ↓
cgroup / namespace / OOM
    ↓
TCP / DNS / MTU / keepalive
    ↓
iptables / nftables / conntrack / NAT
    ↓
映射回 Kubernetes Pod / Service / DNS / Node
```

这个顺序的好处是：先知道 Linux 怎么看进程，再知道连接和 socket 怎么工作，再理解容器隔离，最后再回到 Kubernetes。

## 第 1 周：进程、文件描述符与 /proc

### 要理解什么

进程不是一个抽象概念，它在 Linux 上会暴露出大量可观察状态。你要学会从 `/proc` 里看：

- 进程启动命令。
- 当前工作目录。
- 打开的文件描述符。
- 网络连接。
- 内存状态。
- cgroup 归属。
- namespace 归属。
- OOM 分数。

重点概念：

- PID。
- PPID。
- fd。
- signal。
- zombie process。
- `/proc/<pid>/status`。
- `/proc/<pid>/fd`。
- `/proc/<pid>/limits`。
- `/proc/<pid>/net`。
- `/proc/<pid>/cgroup`。

### 必读资料

- [proc(5) - Linux manual page](https://www.man7.org/linux/man-pages/man5/proc.5.html)
- [strace(1) - Linux manual page](https://man7.org/linux/man-pages/man1/strace.1.html)
- [socket(7) - Linux manual page](https://man7.org/linux/man-pages/man7/socket.7.html)

### 实验一：观察一个进程

启动一个简单 HTTP 服务：

```bash
python3 -m http.server 8080
```

另开一个终端：

```bash
pid=$(pgrep -f "http.server 8080" | head -1)
echo $pid
ls -l /proc/$pid
cat /proc/$pid/status
ls -l /proc/$pid/fd
cat /proc/$pid/limits
readlink /proc/$pid/cwd
```

再请求一次：

```bash
curl -v http://127.0.0.1:8080/
ss -ntp | grep 8080
lsof -p $pid | head
```

你要回答：

- 这个进程打开了哪些 fd？
- 哪个 fd 是监听 socket？
- 请求进来后，`ss` 里能看到什么连接状态？
- `/proc/$pid/status` 里哪些字段和排障相关？

### 实验二：用 strace 看系统调用

```bash
strace -p $pid -f -tt -T
```

然后再次请求：

```bash
curl http://127.0.0.1:8080/
```

你要观察：

- `accept` / `recvfrom` / `sendto` / `write` 是否出现。
- 哪个系统调用耗时更长。
- 如果服务没响应，是卡在网络读写、文件读写，还是应用逻辑。

## 第 2 周：systemd、日志与服务生命周期

### 要理解什么

线上服务不是直接在 shell 里跑的。大多数 Linux 节点上，服务生命周期由 systemd 管理。你要理解：

- unit 是什么。
- service 如何启动、停止、重启。
- exit code 如何影响服务状态。
- journal 日志如何查询。
- restart policy 如何避免服务退出后无人发现。
- resource limit 如何影响进程。

### 必读资料

- [systemd.service 官方手册](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)
- [systemd.unit 官方手册](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html)
- [journalctl 官方手册](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)

### 实验三：写一个会失败的 systemd 服务

创建一个测试脚本：

```bash
sudo tee /usr/local/bin/demo-fail.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
echo "demo service start"
sleep 2
echo "demo service fail"
exit 1
EOF
sudo chmod +x /usr/local/bin/demo-fail.sh
```

创建 unit：

```bash
sudo tee /etc/systemd/system/demo-fail.service >/dev/null <<'EOF'
[Unit]
Description=Demo failing service

[Service]
ExecStart=/usr/local/bin/demo-fail.sh
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF
```

运行：

```bash
sudo systemctl daemon-reload
sudo systemctl start demo-fail
systemctl status demo-fail
journalctl -u demo-fail -f
```

你要回答：

- systemd 如何显示服务失败？
- 日志在哪里？
- `Restart=on-failure` 带来了什么行为？
- 如果服务频繁重启，你如何判断是应用失败还是 systemd 配置失败？

完成后清理：

```bash
sudo systemctl stop demo-fail
sudo systemctl disable demo-fail
sudo rm -f /etc/systemd/system/demo-fail.service /usr/local/bin/demo-fail.sh
sudo systemctl daemon-reload
```

## 第 3 周：cgroup、namespace 与 OOM

### 要理解什么

容器不是轻量虚拟机。容器的核心基础是 Linux namespace 和 cgroup：

- namespace 负责隔离看到的资源。
- cgroup 负责限制和统计资源。
- Kubernetes 的 requests / limits 最终会落到 cgroup。
- Pod OOMKilled 背后是内核内存回收与 OOM killer。

重点概念：

- PID namespace。
- network namespace。
- mount namespace。
- cgroup v2。
- memory.max。
- cpu.max。
- oom_score。
- oom_score_adj。
- Kubernetes QoS。

### 必读资料

- [namespaces(7) - Linux manual page](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [cgroups(7) - Linux manual page](https://www.man7.org/linux/man-pages/man7/cgroups.7.html)
- [Control Group v2 - Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
- [Kubernetes：About cgroup v2](https://kubernetes.io/docs/concepts/architecture/cgroups/)
- [Kubernetes：Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes：Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)

### 实验四：观察 cgroup v2

```bash
mount | grep cgroup
ls /sys/fs/cgroup
cat /proc/self/cgroup
cat /sys/fs/cgroup/cgroup.controllers
```

如果使用 systemd，可以用 `systemd-run` 创建一个临时受限服务：

```bash
sudo systemd-run --unit=demo-memory --property=MemoryMax=100M --pty bash
```

在新 shell 中执行：

```bash
cat /proc/self/cgroup
python3 - <<'PY'
a=[]
while True:
    a.append("x" * 1024 * 1024)
PY
```

观察另一个终端：

```bash
journalctl -u demo-memory
systemctl status demo-memory
dmesg -T | tail -50
```

你要回答：

- 进程为什么被杀？
- 日志里能看到什么证据？
- cgroup 限制和系统整体内存不足有什么区别？

### 实验五：观察 namespace

```bash
readlink /proc/self/ns/pid
readlink /proc/self/ns/net
readlink /proc/self/ns/mnt
```

如果有 Docker：

```bash
docker run --rm -it --name ns-demo busybox sh
```

宿主机查看：

```bash
pid=$(docker inspect -f '{{.State.Pid}}' ns-demo)
readlink /proc/$pid/ns/pid
readlink /proc/$pid/ns/net
ls -l /proc/$pid/ns
```

你要回答：

- 容器进程和宿主机进程是不是同一个内核？
- 容器里的 PID 1 和宿主机上的 PID 是否一样？
- network namespace 不同意味着什么？

## 第 4 周：TCP、DNS 与基础网络排障

### 要理解什么

很多线上问题表面是“服务不可用”，底层可能是：

- DNS 慢或失败。
- TCP 建连失败。
- 连接被 reset。
- 服务端 backlog 满。
- 客户端 ephemeral port 耗尽。
- keepalive 与 idle timeout 不匹配。
- MTU 问题。
- 中间层 NAT 或 LB 超时。

重点概念：

- TCP 三次握手。
- SYN / SYN-ACK / ACK。
- FIN / RST。
- TIME_WAIT。
- listen backlog。
- keepalive。
- DNS A / AAAA / CNAME。
- `/etc/resolv.conf`。
- search domain。
- UDP / TCP 53。

### 必读资料

- [tcp(7) - Linux manual page](https://man7.org/linux/man-pages/man7/tcp.7.html)
- [ip-sysctl - Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)
- [CoreDNS kubernetes plugin](https://coredns.io/plugins/kubernetes)

### 实验六：抓一次 HTTP 请求

终端一启动服务：

```bash
python3 -m http.server 8080
```

终端二抓包：

```bash
sudo tcpdump -i lo -nn 'tcp port 8080'
```

终端三请求：

```bash
curl -v http://127.0.0.1:8080/
```

你要回答：

- 三次握手在哪里？
- 请求和响应分别对应哪些包？
- 连接关闭是 FIN 还是 RST？

### 实验七：观察 TCP 状态

```bash
ss -s
ss -nt state established
ss -nt state time-wait
ss -lntp
```

你要回答：

- LISTEN、ESTAB、TIME-WAIT 分别表示什么？
- 大量 TIME_WAIT 一定是问题吗？
- 服务没监听时，客户端会看到什么错误？

### 实验八：DNS 排障

```bash
cat /etc/resolv.conf
dig kubernetes.io
dig +trace kubernetes.io
nslookup kubernetes.io
```

如果在 Kubernetes Pod 里：

```bash
kubectl run dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 --restart=Never -- sleep 3600
kubectl exec -it dnsutils -- nslookup kubernetes.default
kubectl exec -it dnsutils -- cat /etc/resolv.conf
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

你要回答：

- Pod 里的 DNS server 是谁？
- `kubernetes.default` 会如何补全？
- 如果 DNS 不通，问题可能在 Pod、CoreDNS、Service、NetworkPolicy、上游 DNS 哪一层？

## 第 5 周：iptables、nftables、conntrack 与 NAT

### 要理解什么

Kubernetes Service、NodePort、Ingress、出网 NAT 都可能涉及连接跟踪和包转发。你要理解：

- iptables / nftables 是规则系统。
- conntrack 是连接跟踪状态。
- NAT 会改写源地址或目标地址。
- kube-proxy 的 iptables / ipvs 模式会影响 Service 转发。
- conntrack 表满会导致看似随机的连接失败。

重点概念：

- PREROUTING。
- INPUT。
- FORWARD。
- OUTPUT。
- POSTROUTING。
- DNAT。
- SNAT / MASQUERADE。
- ESTABLISHED / RELATED。
- conntrack table。

### 必读资料

- [iptables-extensions(8) - Linux manual page](https://man7.org/linux/man-pages/man8/iptables-extensions.8.html)
- [conntrack-tools 官方站点](https://conntrack-tools.netfilter.org/)
- [Kubernetes：Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Kubernetes：Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)

### 实验九：观察规则与连接跟踪

```bash
sudo iptables-save | head -80
sudo nft list ruleset | head -120
sudo conntrack -L 2>/dev/null | head
sudo conntrack -S
cat /proc/sys/net/netfilter/nf_conntrack_max
```

如果你在 kind / minikube 节点上：

```bash
kubectl get svc -A
kubectl get endpoints -A
kubectl get endpointslice -A
kubectl get pods -A -o wide
```

你要回答：

- 一个 ClusterIP Service 背后有哪些 endpoint？
- Service 没有 endpoint 时会发生什么？
- kube-proxy 是否在节点上写了规则？
- conntrack 表接近上限时会有什么风险？

## 第 6 周：映射回 Kubernetes 排障

### 要理解什么

这一周把前面所有知识串回 K8s：

- Pod 是进程。
- Container 是 namespace + cgroup + rootfs。
- Service 是虚拟访问入口。
- DNS 是服务发现入口。
- kube-proxy / CNI / CoreDNS / Node 网络共同决定请求能不能到达。
- request / limit / QoS / probe 决定稳定性和故障表现。

### 必读资料

- [Kubernetes：Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod)
- [Kubernetes：Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [Kubernetes：Debugging Kubernetes Nodes With Kubectl](https://kubernetes.io/docs/tasks/debug/debug-cluster/kubectl-node-debug/)
- [kubectl debug reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_debug/)
- [Kubernetes：Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/)

### 实验十：Pod 到 Service 的完整排障

创建一个 nginx：

```bash
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80 --target-port=80
kubectl run curl --image=curlimages/curl --restart=Never -- sleep 3600
```

验证访问：

```bash
kubectl exec curl -- curl -v http://web.default.svc.cluster.local
kubectl get svc web
kubectl get endpoints web
kubectl get endpointslice -l kubernetes.io/service-name=web
kubectl get pods -o wide
```

制造故障一：让 Service selector 不匹配。

```bash
kubectl patch service web -p '{"spec":{"selector":{"app":"not-exist"}}}'
kubectl get endpoints web
kubectl exec curl -- curl -v --max-time 3 http://web.default.svc.cluster.local
```

恢复：

```bash
kubectl patch service web -p '{"spec":{"selector":{"app":"web"}}}'
```

你要回答：

- DNS 能解析是否代表服务可用？
- Service 有 ClusterIP 是否代表一定有后端？
- endpoint 为空时，访问表现是什么？

### 实验十一：Pod OOMKilled

创建一个低内存 Pod：

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  restartPolicy: Never
  containers:
  - name: app
    image: python:3.12-slim
    command: ["python3", "-c"]
    args:
    - |
      a=[]
      while True:
          a.append("x" * 1024 * 1024)
    resources:
      limits:
        memory: "64Mi"
      requests:
        memory: "64Mi"
EOF
```

观察：

```bash
kubectl get pod oom-demo -w
kubectl describe pod oom-demo
kubectl logs oom-demo --previous
```

你要回答：

- `Reason: OOMKilled` 出现在哪里？
- limit 太低和节点内存不足有什么区别？
- request / limit 如何影响 QoS？

清理：

```bash
kubectl delete pod oom-demo
kubectl delete deployment web
kubectl delete service web
kubectl delete pod curl dnsutils --ignore-not-found
```

## 第一阶段命令清单

进程与资源：

```bash
ps aux
top
htop
pidstat 1
vmstat 1
iostat -xz 1
free -h
uptime
dmesg -T
cat /proc/meminfo
cat /proc/loadavg
ls -l /proc/<pid>/fd
cat /proc/<pid>/status
cat /proc/<pid>/limits
```

服务与日志：

```bash
systemctl status <service>
journalctl -u <service> -f
journalctl -xe
systemctl cat <service>
systemctl show <service>
```

网络连接：

```bash
ss -s
ss -lntp
ss -ntp
ss -nt state established
ss -nt state time-wait
lsof -i
```

DNS：

```bash
cat /etc/resolv.conf
dig example.com
dig +trace example.com
nslookup example.com
```

抓包：

```bash
sudo tcpdump -i any -nn host <ip>
sudo tcpdump -i any -nn 'tcp port 80'
sudo tcpdump -i any -nn 'udp port 53 or tcp port 53'
```

路由与网卡：

```bash
ip addr
ip route
ip neigh
traceroute <host>
ping <host>
iperf3
```

iptables / nftables / conntrack：

```bash
sudo iptables-save
sudo nft list ruleset
sudo conntrack -L
sudo conntrack -S
cat /proc/sys/net/netfilter/nf_conntrack_max
```

Kubernetes 排障：

```bash
kubectl get pod -o wide
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl get svc
kubectl get endpoints
kubectl get endpointslice
kubectl exec -it <pod> -- sh
kubectl debug -it <pod> --image=busybox --target=<container>
kubectl debug node/<node> -it --image=ubuntu
```

## 每周产出

第 1 周产出：

- 一份《Linux 进程观察笔记》。
- 内容包括 `/proc`、fd、socket、strace 的截图或命令输出摘要。

第 2 周产出：

- 一份《systemd 服务失败排查笔记》。
- 内容包括 unit 文件、失败日志、restart 行为解释。

第 3 周产出：

- 一份《cgroup 与 namespace 笔记》。
- 内容包括 cgroup v2 文件、OOM 实验、Docker namespace 观察。

第 4 周产出：

- 一份《TCP 与 DNS 抓包笔记》。
- 内容包括一次 HTTP 请求抓包、DNS 查询路径、TCP 状态解释。

第 5 周产出：

- 一份《conntrack 与 NAT 笔记》。
- 内容包括 iptables / nft 规则观察、conntrack 状态解释。

第 6 周产出：

- 一份《Kubernetes Service 与 OOM 排障笔记》。
- 内容包括 Service selector 故障、endpoint 为空、Pod OOMKilled 的证据链。

## 自测题

你应该能不用查资料回答：

- `/proc/<pid>/fd` 里看到的 socket 是什么？
- `ss -lntp` 和 `ss -ntp` 的区别是什么？
- 进程卡住时，`strace` 能帮你判断什么？
- systemd 的 `Restart=on-failure` 会在什么情况下生效？
- namespace 和 cgroup 分别解决什么问题？
- 容器是不是有自己的内核？
- Pod OOMKilled 一定是节点内存不足吗？
- TCP RST 和 FIN 的区别是什么？
- 大量 TIME_WAIT 一定是故障吗？
- DNS 查询在 Pod 里会先看哪里？
- Kubernetes Service 有 ClusterIP 但 endpoint 为空时会怎样？
- conntrack 表满可能导致什么现象？
- Nginx 502 可能由哪些底层原因导致？
- WebSocket 1006 可能和哪些超时或网络层问题有关？

## 推荐阅读顺序

先读这些：

- [proc(5)](https://www.man7.org/linux/man-pages/man5/proc.5.html)
- [socket(7)](https://man7.org/linux/man-pages/man7/socket.7.html)
- [tcp(7)](https://man7.org/linux/man-pages/man7/tcp.7.html)
- [namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [cgroups(7)](https://www.man7.org/linux/man-pages/man7/cgroups.7.html)
- [Control Group v2](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)

再读这些：

- [systemd.service](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)
- [journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [ip-sysctl](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)
- [CoreDNS kubernetes plugin](https://coredns.io/plugins/kubernetes)

最后回到 Kubernetes：

- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Kubernetes Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod)
- [Kubernetes Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [Kubernetes Debug Nodes](https://kubernetes.io/docs/tasks/debug/debug-cluster/kubectl-node-debug/)

补充性能排障方法：

- [Brendan Gregg: The USE Method](https://www.brendangregg.com/usemethod.html)
- [Linux Performance Analysis in 60s](https://www.brendangregg.com/blog/2015-12-03/linux-perf-60s-video.html)

## 第一阶段完成标准

你可以认为第一阶段过关的标准是：

- 看到 Pod OOMKilled，能从 K8s event、container status、cgroup、日志四个角度解释。
- 看到 Service 不通，能按 DNS、Service、Endpoint、Pod、Node、NetworkPolicy、kube-proxy / CNI 顺序排查。
- 看到连接慢，能区分 DNS 慢、建连慢、应用处理慢、下游慢、网络丢包。
- 看到 502 / 504 / 1006，不会只盯应用日志，而会去看连接、超时、LB、Nginx、Pod readiness、上游 endpoint。
- 能写出一份清晰 RCA，包含现象、影响、证据、根因、修复、预防。

这一阶段真正练出来的是“故障下沉能力”：任何云原生问题，最后都能被你拆到 Linux 进程、资源、网络、连接和内核状态上。
