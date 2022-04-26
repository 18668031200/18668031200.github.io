---
title: 研究 Dubbo 网卡地址注册时的一点思考
author: ygdxd
type: 转载
date: 2021-09-14 21:34:50
tags: dubbo
categories: dubbo
---

曾使用k8s部署dubbo服务时使用虚拟ip会导致consumer不能找到对应的provider。

转自https://dubbo.apache.org/zh/blog/2019/10/01/%E7%A0%94%E7%A9%B6-dubbo-%E7%BD%91%E5%8D%A1%E5%9C%B0%E5%9D%80%E6%B3%A8%E5%86%8C%E6%97%B6%E7%9A%84%E4%B8%80%E7%82%B9%E6%80%9D%E8%80%83/

问题：dubbo注册自身ip的问题
--------

原文:可能相当一部分人还不知道我这篇文章到底要讲什么，我说个场景，大家应该就明晰了。在分布式服务调用过程中，以 Dubbo 为例，服务提供者往往需要将自身的 IP 地址上报给注册中心，供消费者去发现。在大多数情况下 Dubbo 都可以正常工作，但如果你留意过 Dubbo 的 github issue，其实有不少人反馈：Dubbo Provider 注册了错误的 IP。如果你能立刻联想到：多网卡、内外网地址共存、VPN、虚拟网卡等关键词，那我建议你一定要继续将本文看下去，因为我也想到了这些，它们都是本文所要探讨的东西！那么“如何选择合适的网卡地址”呢，Dubbo 现有的逻辑到底算不算完备？我们不急着回答它，而是带着这些问题一起进行研究，相信到文末，其中答案，各位看官自有评说。



{% codeblock NetUtils.java lang:java %}
#这里使用的是阿里的2.6.10版本
private static InetAddress getLocalAddress0() {
        InetAddress localAddress = null;
        try {
            localAddress = InetAddress.getLocalHost();
            if (isValidAddress(localAddress)) {
                return localAddress;
            }
        } catch (Throwable e) {
            logger.warn(e);
        }
        try {
            Enumeration<NetworkInterface> interfaces = NetworkInterface.getNetworkInterfaces();
            if (interfaces != null) {
                while (interfaces.hasMoreElements()) {
                    try {
                        NetworkInterface network = interfaces.nextElement();
                        Enumeration<InetAddress> addresses = network.getInetAddresses();
                        if (addresses != null) {
                            while (addresses.hasMoreElements()) {
                                try {
                                    InetAddress address = addresses.nextElement();
                                    if (isValidAddress(address)) {
                                        return address;
                                    }
                                } catch (Throwable e) {
                                    logger.warn(e);
                                }
                            }
                        }
                    } catch (Throwable e) {
                        logger.warn(e);
                    }
                }
            }
        } catch (Throwable e) {
            logger.warn(e);
        }
        return localAddress;
    }
{% endcodeblock %}

{% codeblock NetUtils.java lang:java %}
#这里使用的是apache的3.2.0版本
private static InetAddress getLocalAddress0() {
        InetAddress localAddress = null;

        // @since 2.7.6, choose the {@link NetworkInterface} first
        try {
            NetworkInterface networkInterface = findNetworkInterface();
            Enumeration<InetAddress> addresses = networkInterface.getInetAddresses();
            while (addresses.hasMoreElements()) {
                Optional<InetAddress> addressOp = toValidAddress(addresses.nextElement());
                if (addressOp.isPresent()) {
                    try {
                        if (addressOp.get().isReachable(100)) {
                            return addressOp.get();
                        }
                    } catch (IOException e) {
                        // ignore
                    }
                }
            }
        } catch (Throwable e) {
            logger.warn(e);
        }

        try {
            localAddress = InetAddress.getLocalHost();
            Optional<InetAddress> addressOp = toValidAddress(localAddress);
            if (addressOp.isPresent()) {
                return addressOp.get();
            }
        } catch (Throwable e) {
            logger.warn(e);
        }


        return localAddress;
    }

{% endcodeblock %}
Dubbo 这段选取本地地址的逻辑大致分成了两步

1.先去 /etc/hosts 文件中找 hostname 对应的 IP 地址，找到则返回；找不到则转 
2.轮询网卡，寻找合适的 IP 地址，找到则返回；找不到返回 null，在 getLocalAddress0 外侧还有一段逻辑，如果返回 null，则注册 127.0.0.1 这个本地回环地址


这里与原文的区别在于都使用了try catch捕获了异常

尝试获取 hostname 映射 IP
-------
Dubbo 首先选取的是 hostname 对应的 IP，在源码中对应的 InetAddress.getLocalHost(); 在 *nix 系统实际部署 Dubbo 应用时，可以首先使用 hostname 命令获取主机名

原文:
```shell
xujingfengdeMacBook-Pro:~ xujingfeng$ hostname
xujingfengdeMacBook-Pro.local
```
紧接着在 /etc/hosts 配置 IP 映射，为了验证 Dubbo 的机制，我们随意为 hostname 配置一个 IP 地址

```shell
127.0.0.1	localhost
1.2.3.4 xujingfengdeMacBook-Pro.local
```
接着调用 NetUtils.getLocalAddress0() 进行验证，控制台打印如下：
```shell
xujingfengdeMacBook-Pro.local/1.2.3.4
```

我在本地使用 阿里的版本
```shell
ygdxd_admin$ hostname
localhost
```
debug时InetAddress.getLocalHost() 会返回localhost/127.0.0.1
因为使用不了localhost所以不会返回

```java
System.out.println(NetUtils.getLocalAddress().toString());
```
返回的是
/192.168.0.102

![demo.java](/images/dubbo_net_debug.jpg)

使用debug 发现使用的是eh0的网卡地址


判定有效的 IP 地址
-----------------
{% codeblock NetUtils.java lang:java %}
private static Optional<InetAddress> toValidAddress(InetAddress address) {
    if (address instanceof Inet6Address) {
        Inet6Address v6Address = (Inet6Address) address;
        if (isValidV6Address(v6Address)) {
            return Optional.ofNullable(normalizeV6Address(v6Address));
        }
    }
    if (isValidV4Address(address)) {
        return Optional.of(address);
    }
    return Optional.empty();
}
{% endcodeblock %}

阿里的版本直接去掉了ipv6的验证

{% codeblock NetUtils.java lang:java %}
private static boolean isValidAddress(InetAddress address) {
        if (address == null || address.isLoopbackAddress())
            return false;
        String name = address.getHostAddress();
        return (name != null
                && !ANYHOST.equals(name)
                && !LOCALHOST.equals(name)
                && IP_PATTERN.matcher(name).matches());
    }
{% endcodeblock %}

原文：
```java
static boolean isValidV6Address(Inet6Address address) {
    boolean preferIpv6 = Boolean.getBoolean("java.net.preferIPv6Addresses");
    if (!preferIpv6) {
        return false;
    }
    try {
        return address.isReachable(100);
    } catch (IOException e) {
        // ignore
    }
    return false;
}
```

最新的apache dubbo 版本
```java
private static Optional<InetAddress> toValidAddress(InetAddress address) {
        if (address instanceof Inet6Address) {
            Inet6Address v6Address = (Inet6Address) address;
            if (isPreferIPV6Address()) {
                return Optional.ofNullable(normalizeV6Address(v6Address));
            }
        }
        if (isValidV4Address(address)) {
            return Optional.of(address);
        }
        return Optional.empty();
    }
```
已经取消掉了通过 isReachable 判断网卡的连通性,而在轮训网卡中使用了
```java
public static NetworkInterface findNetworkInterface() {

        List<NetworkInterface> validNetworkInterfaces = emptyList();
        try {
            validNetworkInterfaces = getValidNetworkInterfaces();
        } catch (Throwable e) {
            logger.warn(e);
        }

        NetworkInterface result = null;

        // Try to find the preferred one
        for (NetworkInterface networkInterface : validNetworkInterfaces) {
            if (isPreferredNetworkInterface(networkInterface)) {
                result = networkInterface;
                break;
            }
        }

        if (result == null) { // If not found, try to get the first one
            for (NetworkInterface networkInterface : validNetworkInterfaces) {
                Enumeration<InetAddress> addresses = networkInterface.getInetAddresses();
                while (addresses.hasMoreElements()) {
                    Optional<InetAddress> addressOp = toValidAddress(addresses.nextElement());
                    if (addressOp.isPresent()) {
                        try {
                            if (addressOp.get().isReachable(100)) {
                                result = networkInterface;
                                break;
                            }
                        } catch (IOException e) {
                            // ignore
                        }
                    }
                }
            }
        }

        if (result == null) {
            result = first(validNetworkInterfaces);
        }

        return result;
    }
```
轮询网卡
---------

轮询网卡对应的源码是 NetworkInterface.getNetworkInterfaces().

这里记录下干扰因素

1.Docker 网桥
2.TUN/TAP 虚拟网络设备
3.干扰因素三：多网卡


记录下网卡工作原理
------

![eh0](/images/dubbo_net_eh0.jpg)

上图中的 eth0 表示我们主机已有的真实的网卡接口 (interface)。

网卡接口 eth0 所代表的真实网卡通过网线(wire)和外部网络相连，该物理网卡收到的数据包会经由接口 eth0 传递给内核的网络协议栈(Network Stack)。然后协议栈对这些数据包进行进一步的处理。

对于一些错误的数据包,协议栈可以选择丢弃；对于不属于本机的数据包，协议栈可以选择转发；而对于确实是传递给本机的数据包,而且该数据包确实被上层的应用所需要，协议栈会通过 Socket API 告知上层正在等待的应用程序。

TUN 工作原理
------

![TUN](/images/dubbo_net_tun0.jpg)

我们知道，普通的网卡是通过网线来收发数据包的话，而 TUN 设备比较特殊，它通过一个文件收发数据包。

如上图所示，tunX 和上面的 eth0 在逻辑上面是等价的， tunX 也代表了一个网络接口,虽然这个接口是系统通过软件所模拟出来的.

网卡接口 tunX 所代表的虚拟网卡通过文件 /dev/tunX 与我们的应用程序(App)相连，应用程序每次使用 write 之类的系统调用将数据写入该文件，这些数据会以网络层数据包的形式，通过该虚拟网卡，经由网络接口 tunX 传递给网络协议栈，同时该应用程序也可以通过 read 之类的系统调用，经由文件 /dev/tunX 读取到协议栈向 tunX 传递的所有数据包。

此外，协议栈可以像操纵普通网卡一样来操纵 tunX 所代表的虚拟网卡。比如说，给 tunX 设定 IP 地址，设置路由，总之，在协议栈看来，tunX 所代表的网卡和其他普通的网卡区别不大，当然，硬要说区别，那还是有的,那就是 tunX 设备不存在 MAC 地址，这个很好理解，tunX 只模拟到了网络层，要 MAC地址没有任何意义。当然，如果是 tapX 的话，在协议栈的眼中，tapX 和真实网卡没有任何区别。

是不是有些懵了？我是谁，为什么我要在这篇文章里面学习 TUN！因为我们常用的 VPN 基本就是基于 TUN/TAP 搭建的，如果我们使用 TUN 设备搭建一个基于 UDP 的 VPN ，那么整个处理过程可能是这幅样子：
![TUN](/images/dubbo_net_tun1.jpg)

TAP 工作原理
------

TAP 设备与 TUN 设备工作方式完全相同，区别在于：

TUN 设备是一个三层设备，它只模拟到了 IP 层，即网络层 我们可以通过 /dev/tunX 文件收发 IP 层数据包，它无法与物理网卡做 bridge，但是可以通过三层交换（如 ip_forward）与物理网卡连通。可以使用ifconfig之类的命令给该设备设定 IP 地址。
TAP 设备是一个二层设备，它比 TUN 更加深入，通过 /dev/tapX 文件可以收发 MAC 层数据包，即数据链路层，拥有 MAC 层功能，可以与物理网卡做 bridge，支持 MAC 层广播。同样的，我们也可以通过ifconfig之类的命令给该设备设定 IP 地址，你如果愿意，我们可以给它设定 MAC 地址。
关于文章中出现的二层，三层，我这里说明一下，第一层是物理层，第二层是数据链路层，第三层是网络层，第四层是传输层。






