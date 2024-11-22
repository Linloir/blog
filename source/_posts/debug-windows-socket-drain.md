---
title: 问题定位回顾 - Windows 上发起 tcp 连接时提示 Only one usage of each socket address (protocol/network address/port) is normally permitted
date: 2024-11-22 11:52:37
tags:
  - 工作
  - 问题定位
categories:
  - 技术
---

## 问题背景

某天在 Windows 宿主机上执行任务时，发现 wda 指令请求一直失败，查看日志发现唯一有效的错误日志是 `Only one usage of each socket address (protocol/network address/port) is normally permitted`，回顾宿主机环境在过去一段时间没有进行过变更，并且该问题是第一次出现，此前相同环境并没有出现过这个问题

尝试在其他宿主机以及本地开发机上执行相同命令均不能稳定复现该问题，其中 Linux 开发机无法复现该问题

![error info](/img/debug-windows-socket-drain/error_info.png)

## 定位过程

### 报错分析

由于报错信息只有一行，也只能从这个信息来入手。搜索引擎检索得到如下内容：

- StackOverflow: [How do I fix the error "Only one usage of each socket address (protocol/network address/port) is normally permitted"?](https://stackoverflow.com/questions/14654998/how-do-i-fix-the-error-only-one-usage-of-each-socket-address-protocol-network)：文中指出，这个问题可能由多次 `listen` 同一端口引起，阅读提问者提供的代码确实有此问题
- StackOverflow: [HttpClient: Only one usage of each socket address (protocol/network address/port) is normally permitted](https://stackoverflow.com/questions/26428827/httpclient-only-one-usage-of-each-socket-address-protocol-network-address-port)：*More likely, you are probably posting HTTP requests too often, and maybe not fully consuming the responses, which would prevent ASP from pooling and reusing connections and thus encountering port exhaustion over time.*，该回答指出如果连接请求过于频繁、没有正确关闭，可能会导致端口耗尽

由于此前宿主机并没有暴露出该问题，优先考虑并不是发起请求及接收请求的 client 及 wda 侧的问题，即第一篇文章中提到的可能，而是从宿主机上的 tcp 连接状况入手来进一步分析这个问题，于是主要的关注点转向如下方面：

1. 问题出现时，宿主机上的 tcp 连接情况如何，是否存在异常，异常现象怎样？
2. 如果是 tcp 连接问题，为什么 Linux 机器无法复现该问题

### 链路分析

在开始检查 tcp 连接情况前，首先需要确认大致的请求链路，从而确定需要关注的 tcp 连接范围。问题发生在 Windows 上 docker 容器内服务向宿主机发送 wda 请求时，因此从这一条请求链路入手：

请求的末端是 usbmuxd 服务，该服务负责和连接的 iPhone 进行通信，所有请求最后都会发送到 usbmuxd 来进行多路复用 (multiplexing)，从而达到在一个 usb 链路上同时执行多个请求。

usbmuxd 服务在 Linux 和 Windows 上的表现并不相同，在 Linux 上，其监听一个 UNIX socket 套接字来提供服务，而在 Windows 上，其监听 `127.0.0.1:27015` 端口提供服务，这里应该是处于安全考虑，其只监听了 `127.0.0.1:27015` 而不是 `0.0.0.0:27015`。

由于 Windows 上的 usbmuxd 服务仅监听 `127.0.0.1:27015` 端口，对于容器内打出来的请求，考虑到其 ip 可能不是 `127.0.0.1`，遂在宿主机有起一个反代服务，从 `0.0.0.0:23333` 端口反向代理到 `127.0.0.1:27015`，这样不是以 `127.0.0.1` 发起请求的服务可以通过连接到宿主机的 `23333` 端口来和 usbmuxd 通信。在 Linux 上，同样有这个反代服务，不过它的作用是将请求代理到对应的 UNIX socket 上。

之后就是 docker 内的服务了，其通过 `host.docker.internal` 域名向宿主机的 `23333` 端口建立连接，并和 usbmuxd 通信来发送 wda 的请求。

这样，在宿主机上需要关注的连接就很明了了：

1. 所有直接向 `127.0.0.1:27015` 服务发起的通信，包括宿主机对 usbmuxd 服务的请求以及反代服务发起的请求
2. 向宿主机 `23333` 端口发起的通信

### tcp 链接分析

在 Windows 再次复现问题时，使用 `netstat -ano | findstr 27015` 以及 `netstat -ano | findstr 23333`，并辅助 `| Mesaure-Object` 方法来查看连接情况发现：

1. ESTABLISHED 状态连接数正常，差不多在十到百量级，符合当前宿主机负载情况
2. CLOSED 状态几乎没有
3. TIME_WAIT 状态非常多，平均在 8000 左右

上述现象佐证：

1. 宿主机中应该没有发生连接泄露的情况，连接泄露一般表现为 tcp 打开连接后并不正常关闭导致 ESTABLISHED 数量异常增多，此时 ESTABLISHED 状态数正常，故暂时排除这个原因
2. 存在高并发的短连接请求负载，TIME_WAIT 为请求发起方正确通告服务端关闭 tcp 连接后进入的 2MSL 的等待窗口，该标记意味请求已经正常关闭，但是由于短时间内发起量太大导致大量 TIME_WAIT 状态连接堆积

联系 [报错分析](#报错分析) 中的第二篇参考文章，初步猜测可能由于存在进程大量发起请求，请求结束后 tcp 连接进入 TIME_WAIT 状态，由于没有设置 SO_REUSEADDR 此时端口号仍然处于被占用的状态，如果继续建立连接确实可能会出现端口号用尽导致报错。

这也就引导关注点到如下问题：

1. 20000 ~ 65535 这么多端口，怎么会这么快就用尽？`netstat -ano | findstr 27015 | findstr | findstr TIME_WAIT | Measure-Object` 或是 `netstat -ano | findstr 23333 | findstr TIME_WAIT | Measure-Object` 反映出来的 TIME_WAIT 连接也不到一万，问题是如何触发的？
2. 是哪个进程在大量发起这样的请求？是否符合预期？

### Windows 侧动态端口范围分析

先关注第一个问题，通过检索搜索引擎 *windows dynamic port range* 可以检索到如下文档：

[The default dynamic port range for TCP/IP has changed since Windows Vista and in Windows Server 2008](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/default-dynamic-port-range-tcpip-chang)：*To comply with Internet Assigned Numbers Authority (IANA) recommendations, Microsoft has increased the dynamic client port range for outgoing connections in Windows Vista and Windows Server 2008. The new default start port is 49152, and the new default end port is 65535.*，也即 Windows 上的默认动态端口范围自 Windows Server 2008 开始，默认从 49152 开始到 65535，一共 16384 个，同时使用 `netsh int ipv4 show dynamicport tcp` 可以查看实际配置的值

Powershell 执行 `netsh int ipv4 show dynamicport tcp`，确实得到 `16384` 这个结果：

```text
协议 tcp 动态端口范围
---------------------------------
启动端口        : 49152
端口数          : 16384
```

联系 [tcp 链接分析](#tcp-链接分析) 中的数量，如果与 `27015` 和 `23333` 端口建立的连接数量均达到 8000，确实会有端口耗尽的可能。所以下一步的关键就是：究竟是哪个进程在建立这些连接？是否是自己工程逻辑有问题？

### TIME_WAIT 源头分析

由于 `netstat -ano | findstr 27015 | findstr TIME_WAIT` 无法展示实际创建连接的 PID，为此我专门写了一个 python 小脚本，这个脚本不断获取非 TIME_WAIT 状态的连接列表来获取连接的 PID，再获取 TIME_WAIT 连接并找到它们原先对应的 PID 输出：

```python
import subprocess
import time
import signal
import sys

# Dictionary to keep track of dual tcp ip:port pair to pid
tcp_map = {}

def update_tcp_map():
    global tcp_map
    # Execute the command to get established connections
    command = "powershell -Command \"netstat -ano | findstr /V TIME_WAIT\""
    result = subprocess.run(command, capture_output=True, text=True, shell=True)
    
    if result.returncode == 0:
        lines = result.stdout.strip().split('\n')
        for line in lines:
            parts = line.split()
            if len(parts) >= 5:
                local_address = parts[1]
                remote_address = parts[2]
                pid = parts[4]
                tcp_map[(local_address,remote_address)] = pid

def check_time_wait_sockets():
    global tcp_map
    # Execute the command to get TIME_WAIT sockets
    command = "powershell -Command \"netstat -ano | findstr TIME_WAIT | findstr /V :443 | findstr /V :80\""
    result = subprocess.run(command, capture_output=True, text=True, shell=True)
    
    if result.returncode == 0:
        lines = result.stdout.strip().split('\n')
        for line in lines:
            parts = line.split()
            if len(parts) >= 5:
                local_address = parts[1]
                remote_address = parts[2]
                pid = parts[4]
                # Check if the TIME_WAIT socket was previously tracked
                if (local_address,remote_address) in tcp_map:
                    print(f"TIME_WAIT socket: {local_address} -> {remote_address}, previously tracked PID: {tcp_map[(local_address,remote_address)]}")
                    # Remove the tracked record
                    del tcp_map[(local_address,remote_address)]

def signal_handler(sig, frame):
    print("\nExiting...")
    sys.exit(0)

def main():
    signal.signal(signal.SIGINT, signal_handler)
    
    print("Monitoring TCP connections. Press Ctrl+C to exit.")
    while True:
        update_tcp_map()
        check_time_wait_sockets()

if __name__ == "__main__":
    main()
```

脚本写的比较简单，考虑到没有太多干扰连接，直接抓取的所有的 TIME_WAIT 连接的原始 PID，同时过滤了下 `443` 和 `80` 这种明显干扰的结果。

同时为了分析 [报错分析](#报错分析) 中的第二个问题，同样也写了一份针对 Linux 机器的分析脚本：

```python
import subprocess
import time
import signal
import sys

# Dictionary to keep track of dual tcp ip:port pair to pid
tcp_map = {}

def update_tcp_map():
    global tcp_map
    # Execute the command to get established connections
    command = "netstat -anolp | grep -v TIME_WAIT"
    result = subprocess.run(command, capture_output=True, text=True, shell=True)
    
    if result.returncode == 0:
        lines = result.stdout.strip().split('\n')
        for line in lines:
            parts = line.split()
            if len(parts) >= 7:
                local_address = parts[3]
                remote_address = parts[4]
                pid = parts[6]
                tcp_map[(local_address,remote_address)] = pid

def check_time_wait_sockets():
    global tcp_map
    # Execute the command to get TIME_WAIT sockets
    command = "netstat -anolp | grep TIME_WAIT"
    result = subprocess.run(command, capture_output=True, text=True, shell=True)
    
    if result.returncode == 0:
        lines = result.stdout.strip().split('\n')
        for line in lines:
            parts = line.split()
            if len(parts) >= 7:
                local_address = parts[3]
                remote_address = parts[4]
                # Check if the TIME_WAIT socket was previously tracked
                if (local_address,remote_address) in tcp_map:
                    print(f"TIME_WAIT socket: {local_address} -> {remote_address}, previously tracked PID: {tcp_map[(local_address,remote_address)]}")
                    # Remove the tracked record
                    del tcp_map[(local_address,remote_address)]

def signal_handler(sig, frame):
    print("\nExiting...")
    sys.exit(0)

def main():
    signal.signal(signal.SIGINT, signal_handler)
    
    print("Monitoring TCP connections. Press Ctrl+C to exit.")
    while True:
        update_tcp_map()
        check_time_wait_sockets()

if __name__ == "__main__":
    main()
```

分析脚本的输出，发现：

1. 在 Linux 上，几乎所有的 TIMEWAIT 请求从容器内发出，并与宿主机的反代建立连接。
2. 在 Linux 上单容器任一时刻均会存在约 1000 TIMEWAIT 请求，这一数量与 Windows 上基本保持一致。
3. 在 Windows 上，TIMEWAIT 分为几乎数量完全相同的两个部分：
   1. 由 `127.0.0.1` 发起的，与反代监听端口建立的连接，PID 查看发起进程为 `com.docker.backend.exe`
   2. 由反代进程发起的，与 usbmuxd 监听端口建立的连接

其中第一点符合预期，因为在 Linux 上反代通过 UNIX socket 与 usbmuxd 通信，并不经过 tcp 连接。

第二点则提示工程确实存在冗余逻辑，1000 请求量在 120s TIMEWAIT 窗口下代表约 8qps 请求负载，这明显存在逻辑错误，自动化再怎么跑也不应该出现如此高的请求负载。

第三点令人感到疑惑：为什么从 docker 容器内发出的请求实际发起 ip 为 `127.0.0.1`，并且是由 docker 进程发起的？

由于 Linux 下，请求的发起 ip 均为 docker bridge 网卡下容器的 ip，可以推断 Linux 下容器内发起的请求不会占用宿主机的动态端口。而如果 Windows 会由 docker 代为发起请求，从现象上看起来确实是会占用宿主机的端口范围，这很有可能是 Linux 上无法复现这个问题的，除了 Linux 上与 usbmuxd 通信不使用 tcp 连接之外的另一个关键原因。

至此，可以产生一些阶段性的结论：

1. 工程逻辑一定是存在问题需要优化，8qps 请求压力不合理。
2. 问题大概率是由于端口耗尽问题引起，Linux 由于 docker 容器网络与 Windows 原理不同不占用宿主机端口，同时与 usbmuxd 也不占用端口，因此不存在这个问题。

接下来，更多的是验证上述结论的正确性，目光转向下列方向：

1. 如果确定是端口耗尽导致的问题，手动构造场景是否可以复现这个问题？
2. 工程是否真的有问题？因为也有可能是发起请求的组件存在问题。
3. bridge 模式在 Windows 上发起请求时的行为是否真的与上述一致？是否可以复现？是否有文档支撑？

### 问题复现

有了猜想，想要复现问题其实比较容易，[Windows 侧动态端口范围分析](#windows-侧动态端口范围分析) 中提到，Windows 默认动态端口范围就是 16384 个，为了减少额外操作干扰，就不改动这个范围，直接手动打 16000 个连接看看问题能不能复现即可。

遂随便手搓了一对 server-client 脚本，server 甚至还有没办法退出的 bug（不过管他呢，反正也是复现问题用的，能跑就行）

client 可以手动通过 `open <number>` 或 `close <number>` 指定打开或关闭多少个连接，从而达到精确控制，免得开得不够多或者开得太多，主打一个灵活。

```python
"""server"""
import socket
import threading

def handle_client(client_socket):
    """Handles a single client connection."""
    try:
        while True:
            # Keep the connection alive
            data = client_socket.recv(1024)
            if not data:
                break
    finally:
        client_socket.close()

def start_server():
    """Starts the TCP server."""
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.bind(('0.0.0.0', 23335))
    server.listen(5)
    print("Server listening on port 23335...")
    while True:
        client_socket, addr = server.accept()
        print(f"Accepted connection from {addr}")
        client_handler = threading.Thread(target=handle_client, args=(client_socket,))
        client_handler.start()

if __name__ == "__main__":
    start_server()
```

```python
"""client"""
import socket
import sys

class ConnectionManager:
    def __init__(self):
        self.connections = []
    def open_connections(self, number):
        """Open a specified number of TCP connections."""
        for _ in range(number):
            try:
                client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                # client_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)  # Allow reuse of the address
                client_socket.connect(('127.0.0.1', 23335))
                self.connections.append(client_socket)
                print(f"Opened connection {len(self.connections)}")
            except Exception as e:
                print(f"Failed to open connection: {e}")
    def close_connections(self, number):
        """Close a specified number of TCP connections."""
        to_close = min(number, len(self.connections))
        for _ in range(to_close):
            client_socket = self.connections.pop()
            client_socket.close()
            print(f"Closed connection, {len(self.connections)} remaining")

def listen_for_input(connection_manager):
    """Listens for user input to open/close connections."""
    while True:
        user_input = input("Enter command (open <number> / close <number>): ")
        parts = user_input.split()
        if len(parts) != 2:
            print("Invalid command. Use 'open <number>' or 'close <number>'.")
            continue
        command, number_str = parts
        try:
            number = int(number_str)
        except ValueError:
            print("Please enter a valid number.")
            continue
        if command == "open":
            connection_manager.open_connections(number)
        elif command == "close":
            connection_manager.close_connections(number)
        else:
            print("Unknown command. Use 'open' or 'close'.")

if __name__ == "__main__":
    connection_manager = ConnectionManager()
    try:
        listen_for_input(connection_manager)
    except KeyboardInterrupt:
        print("\nExiting client.")
        for conn in connection_manager.connections:
            conn.close()
        sys.exit(0)
```

测试发现，当打开了 16000 个端口左右再全部 close 后，问题确实可以复现。

为了交叉验证，在等待一段时间 （大概相当于第一个 open 的端口关闭 2 分钟后）问题自行消失并且在不另开新连接时不再复现。

同时，还通过设置 `SO_REUSEADDR` 标记进一步验证是因为 TIME_WAIT 端口无法被复用导致的问题，在设置该标记后，TIME_WAIT 阶段的端口可以被新的连接复用，实际测试反复开关连接超过 16000 次仍让不会导致问题复现。

至此，基本可以肯定问题出现的原因就是短连接过多导致 TIME_WAIT 堆积，最后动态端口耗尽无法创建新的连接使得 wda 请求失败。

### 工程问题排查

由于此处与工程相关逻辑强相关，不详细展开。经过代码 review，发现部分环节存在多次反复获取同一信息的问题，即有一些连续执行的逻辑链路 `A-B-C-D-E`，其中每一步都会请求 `get_necessary_data` 方法，由于五个方法分开实现，串联五个方法的代码作者与方法编写人又各不相同，没有注意到这里方法的反复多次冗余调用，最终导致了短时间高频调用的逻辑。暂时通过请求缓存优化后得以解除。

此处也佐证了高频的 wda 请求确实与工程的逻辑相关，进一步排除了底层工具链存在问题的可能性。

### Docker 在 Windows 下的网络驱动行为

在搜索引擎检索相关内容，发现并没有太多相关的内容，个人也没有找到官方有关上述发现的行为相关的文档。通过将 [问题复现](#问题复现) 中的 client 代码搬到 docker 容器内执行并请求 `host.docker.internal` 地址，可以复现这种行为，也即容器内与 docker 网卡网关地址建立连接，宿主机上由 docker 发起另一段连接，并不直通，而在 Linux 上执行则可以看到不会有这样的行为。

在 Windows 上执行 `ipconfig` 同样也可以看到不会有 Linux 下执行 `ip a` 所看到的 `docker0` 或是 `vethX` 网卡，进一步推测两个平台上 docker 对于 bridge 网络驱动的实现方案不相同。

同时，我也对 gpt 进行了相关问题的询问，下面是 gpt 给出的解答，大家参考着看，不具有权威性：

```text
[Prompt]

I'm using docker on Windows, and I'm aware of a weird behavior:

When I use host.docker.internal to access the host port, say, I try to connect to a tcp server listening on 0.0.0.0:23333 on host machine, I open a socket inside the container with the container ip:port as one end and host.docker.internal:23333 as the other.

However, when I check the open connections on host, it shows that the just opened socket has 127.0.0.1:another port as on end and 127.0.0.1:23333 as the other. It seems that docker act as an intermediate, accept the connection to host.docker.internal and reopen one on host. Is this real and expected? Why is this behavior different than that on linux?

Also, I've checked ipconfig on Windows and ifconfig on WSL, I can't see any additional interfaces like `veth` or `docker0` or `br-xxx` as these interfaces exists on Linux

------
[Answer]

Yes, your observation about Docker's behavior on Windows is correct and expected due to the differences in how Docker is implemented on Windows compared to Linux.

### Docker Networking on Windows vs. Linux

On Linux, Docker uses the host's kernel features directly, such as namespaces and iptables, to manage containers. It creates virtual network interfaces (like `veth` pairs), bridges (`docker0`), and uses the native Linux networking stack. This setup allows containers to communicate with the host and each other in a manner that closely resembles how separate physical machines would interact on a network.

On Windows, Docker operates differently because Windows does not natively support the same kernel-level features as Linux. Docker on Windows has two main modes: using Hyper-V (with Docker Desktop) or using the Windows Subsystem for Linux (WSL2). Each of these modes handles networking in its own way:

1. **Hyper-V**: When using Docker Desktop with Hyper-V, Docker runs containers inside a lightweight virtual machine. All Docker networking goes through this VM. The `host.docker.internal` DNS name resolves to an internal gateway IP that Docker sets up to route traffic from the container to the host. This is why you see connections appearing to come from `127.0.0.1` on different ports; Docker is essentially NATing (Network Address Translation) the traffic between the container and the host.

2. **WSL2**: In this mode, Docker utilizes a utility VM that runs a real Linux kernel, and Docker containers are managed much like they are on a native Linux system. However, the networking is still handled through a translation layer between Windows and the Linux kernel in WSL2, leading to similar behaviors as seen with Hyper-V.

### Why the Difference?

The primary reason for this difference is the underlying OS architecture and the lack of native container support in Windows. Docker must rely on virtualization and network translation to enable similar functionalities, which inherently changes how networking behaves.

### Lack of `veth`, `docker0`, etc., on Windows

As you've noticed, network interfaces like `veth`, `docker0`, or `br-xxx` are not visible on Windows. This is because these interfaces are part of the Linux networking architecture, which Windows does not natively support. Docker on Windows uses other methods (like creating a virtual network inside Hyper-V or leveraging the network stack in WSL2) to handle container networking.

### Conclusion

The behavior you're seeing is expected given the architectural differences in how Docker operates on Windows versus Linux. Docker must adapt to the underlying operating system's capabilities, leading to different networking setups and behaviors. If you need more consistent behavior across different OSes, you might consider using Linux VMs on Windows or deploying your Docker containers on native Linux systems.
```

## 总结

问题的根因是工程逻辑变更后请求次数异常增加导致，直接原因是由于请求次数大量增加后，Windows 上动态端口耗尽触发了该报错，同时，docker 在 Windows 上的网络行为也进一步加剧了请求数量增长的影响（导致请求数量翻倍）。

由于 usbmuxd 和 docker 在 Windows 和 Linux 上行为的差异性导致了这个问题在 Linux 机器上无法复现，同时由于同一时间宿主机上任务负载量不同，导致了在同一个宿主机上也不能够稳定复现。通过手动构造极端场景可以做到稳定复现。

## 解决方案

找到了问题的原因，解决方案也就很明了了：

1. 优先级最高的就是直接干掉工程内部的逻辑了，通过优化掉工程的错误调用可以快速解决目前问题
2. 考虑到后面宿主机仍有负载增长空间，代码编写也仍然不一定完美，可以通过调整 Windows 动态端口范围和直接从容器内访问 usbmuxd 服务来将允许的请求并发量提高大约 4 倍。

此处额外解释一下第二点中为什么又可以直接从容器内访问 usbmuxd 了：前面发现容器内请求 `host.docker.internal` 会让 docker 以 `com.docker.backend.exe` 的身份使用 `127.0.0.1` 发起一个代理的请求，这还正好满足了 usbmuxd 只监听 `127.0.0.1:27015` 的行为了，这样宿主机的反代服务也就不需要了。

对于 Linux 服务，如果需要保持架构上的统一，一个可能的优化方向是将 usbmuxd 监听的 socket 映射到容器内？考虑到 docker daemon 的 socket 可以做到，大概这个方向也可行，但目前还没有做尝试，就留作一些可行的方案放在此处吧~
