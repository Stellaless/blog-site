---
date = '2026-08-14T10:09:53+08:00'
draft = false
title = 'K8s中kubelet模式配置为Webhook的相关总结'
categories: ["学习笔记"]
tags: ["k8s","CKS"]
---

 最近在学习CKS时，修改kubelet认证模式这道题，ai说mode改为Webhook会直接造成死锁，后对这个参数进行学习，总结如下：

#### k8s的三种认证模式

1、AlwaysAllow

行为：全部请求直接放行，不做任何授权校验，kubelet 收到请求，不去调用 apiserver，本地直接允许访问kubelet API (10250)

2、Webhook（一般为云厂商托管场景，且cacheAuthorizedTTL不为0）

行为：kubelet 自己不做权限判断，发 SubjectAccessReview 请求给 apiserver，由 RBAC 做授权判决

- 请求会 HTTP 回调 apiserver 做鉴权；可以配置鉴权缓存 `cacheAuthorizedTTL`
- ⚠️ 风险点：**control‑plane 节点 + cacheAuthorizedTTL:0s + exec/logs/top‑node，高并发存在 goroutine 耗尽竞争死锁**
- worker 节点使用 Webhook 完全无死锁风险。

3、AlwaysDeny

行为：全部访问 kubelet 的请求全部拒绝，一律 403.

一般为测试调试，几乎不会生产使用。 开启后：exec/logs/top node 全部失效，kubelet 拒绝所有外部调用。

#### 配置文件

```
配置文件：/var/lib/kubernetes/config.conf
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
```

#### 命令是否调用apiserver

| 命令                                                | 是否 apiserver 代理 kubelet  | 是否会触发 master 上 Webhook+TTL=0 死锁风险                 |
| --------------------------------------------------- | ---------------------------- | ----------------------------------------------------------- |
| get pod / get node / apply/delete                   | ❌不访问 kubelet              | ❌无风险                                                     |
| exec / logs / attach / cp / port‑forward / top node | ✅apiserver 代理到 kubelet    | ✅操作 master 本机 pod 时存在竞争死锁风险；worker pod 无风险 |
| top pod                                             | ❌metrics‑server 直连 kubelet | ❌无风险                                                     |

#### **触发死锁完整流程（master，Webhook，TTL=0）**

1. `kubectl exec` → 发给 apiserver。
2. apiserver 需要执行容器 exec，发起网络请求访问**本机 kubelet:10250**。
3. apiserver 的**一个 goroutine（工人 A）** 发起这个调用，然后**阻塞等待 kubelet 返回 exec 结果**。👉 工人 A 被占住，不干别的，就等 kubelet。
4. kubelet 收到 apiserver 过来的请求；因为 `mode:Webhook` + `TTL=0无缓存`。 kubelet 必须向外发 `SubjectAccessReview` 请求，问 apiserver：这个调用者有权访问我 kubelet 吗？
5. kubelet 把鉴权请求又打回**同一台机器的 apiserver**。
6. apiserver 收到这个鉴权请求，**需要拿另一个空闲 goroutine（工人 B）来处理鉴权、查 RBAC、返回 yes/no**。

> ✅ 如果此时还有空闲工人 B：鉴权完成，kubelet 放行 exec，整个流程全部成功，就是你刚才测试正常的现象。

> 🚨 如果**所有 apiserver goroutine 全部用光了**：包括工人 A 在内，全部都在等待各类外部 IO（等待 kubelet、等待 etcd 等）。 新来的鉴权请求**没有空闲 goroutine 去执行处理**。
>
> - kubelet 就在那里傻傻等待鉴权响应，收不到回复。
> - apiserver 的工人 A 还在傻傻等待 kubelet 返回 exec。 **两边都在等对方，没有工人干活，就卡死。**
