---
title: k8s_svc笔记
author: ygdxd
type: 笔记
date: 2022-04-26 23:21:10
tags: k8s
categories: k8s
---

# K8S service 对象笔记(1)

## 1.loadbalance 作用
    loadbalance 对象主要作用是将pod服务暴露出来，并提供负载均衡的作用。loadbalance不像NodePort那样在每个集群的节点上打开一个端口那样访问，他拥有自己独一无二的可公开访问的IP地址，在使用kubectl get svc 的EXTERNAL-IP那一列可以看到，可以通过这些ip访问服务。
    如果k8s在不支持loadbalance环境中运行，则不会调配负责平衡器，但该服务仍将表现得像一个NodePort服务，loadbalance是NodePort服务的扩展。

## 浏览器会话亲和性
    由于浏览器使用keep-alive连接，当浏览器首次与服务连接时，第一次会随机选择一个集群，然后会将属于该连接的所有请求都会发往第一次建立连接的集群，即使会话亲和性设置成None，用户也会使用相同的pod直到连接关闭。

## 不必要的网络跳数
    当外部浏览器通过端口连接服务时，随机选择的pod并不一定在接收连接的同一节点上，这就可能需要额外的网路跳。同时这会可能导致对数据包执行源网络地址转换（SNAP）, 数据包的源ip就会发生变化，会影响后端服务获取客户端的ip地址。
    这时候可以在服务的spec中设置externalTrafficPolicy为Local来避免这种情况。
    ```
    spec:
      externalTrafficPolicy: Local
    ```

    如果服务使用了该设置，并且通过服务的节点端口打开外部连接时，服务代理会选择本地运行的pod，如果没有本地pod，就将连接挂起(并不会像不使用那样转发到全局随机pod),所以需要确保负载均衡到一个有pod服务的节点。
    同时该设置会导致如果节点上pod的数量不相等，使用Local外部流量策略会导致pod的负载分布不均衡。

## ingress工作原理

    客户端先对要访问的域名做DNS查找，DNS服务器返回ingress控制器的ip地址（可以通过kubectl get ingresses 命令的ADDRESS一列中找到对应的ip）。然后客户端向控制器发送请求并在Host头中指定要访问的域名，控制器从该头部中确定访问具体的哪个服务，通过与该服务相关联的Endpoint对象查看ip，然后向其中的一个pod转发。


参考 《Kubernetes in action》
