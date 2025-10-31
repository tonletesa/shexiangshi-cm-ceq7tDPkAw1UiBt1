**目录**

* [一、Session Affinity工作机制剖析](#_label0)
  + [什么是Session Affinity？](#_label0_0)
  + [默认行为 vs 启用会话保持](#_label0_1)
* [二、问题根源：为什么Session Affinity会导致流量失衡？](#_label1)
  + [1. 客户端IP集中性](#_label1_0)
  + [2. 长连接的"雪上加霜"](#_label1_1)
* [三、问题诊断与排查指南](#_label2)
  + [1. 检查Service配置](#_label2_0)
  + [2. 验证Endpoints状态](#_label2_1)
  + [3. 监控流量分布](#_label2_2)
* [四、解决方案与实践](#_label3)
  + [方案一：彻底禁用会话保持（适用于无状态服务）](#_label3_0)
  + [方案二：精细化流量控制（适用于需要会话保持的场景）](#_label3_1)
* [五、预防措施与最佳实践](#_label4)
  + [1. 明确应用需求](#_label4_0)
  + [2. 代码与配置审查](#_label4_1)
  + [3. 监控与告警](#_label4_2)
  + [4. 文档化配置标准](#_label4_3)
* [六、总结](#_label5)

---

在K8s集群中，一个配置项就可能导致服务流量严重倾斜，让部分Pod"压力山大"，而其他Pod却"无所事事"。最近，我们就遭遇了这样一次故障：某个服务的多个Pod实例流量分配严重不均，部分Pod负载极高，而其他Pod几乎处于空闲状态。

通过执行一条命令，我们迅速恢复了流量的均衡分布：

```
kubectl patch svc my-service -n  -p '{"spec": {"sessionAffinity": "None"}}'
```

故障得以解决，但探究其背后原理至关重要。本文将复盘整个排查过程，深入解析Session Affinity的工作机制、问题根源，并提供全面的解决方案与最佳实践。

[回到顶部](#_labelTop)

## 一、Session Affinity工作机制剖析

:[nuts坚果](https://nutsvqn.com)

### 什么是Session Affinity？

Session Affinity，又称会话保持或会话粘滞，是Kubernetes Service的一种负载均衡机制。当配置为`sessionAffinity: ClientIP`时，Kubernetes会基于客户端的IP地址进行会话保持，来自同一IP的请求在会话保持的超时时间窗口内会被持续转发到同一个后端Pod。

### 默认行为 vs 启用会话保持

**默认配置**：

```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  # 默认 sessionAffinity: "None" - 使用轮询负载均衡
```

**启用会话保持的配置**：

```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  sessionAffinity: ClientIP  # 启用客户端IP会话亲和性
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 会话保持时间（默认3小时，仅在iptables模式下有效）
```

> **重要说明**：默认超时时间（10800秒/3小时）仅在`kube-proxy`以`iptables`模式运行时生效。如果使用IPVS模式，超时配置可能有所不同。

[回到顶部](#_labelTop)

## 二、问题根源：为什么Session Affinity会导致流量失衡？

对于无状态服务，Session Affinity配置不当是流量不均的"头号杀手"。其核心原因在于：

### 1. 客户端IP集中性

生产环境中，用户请求通常经过**有限的网关、负载均衡器或NAT设备**。这导致海量用户请求在到达后端Service时，源IP被收敛为少数几个。

当启用`ClientIP`模式的会话保持时，Kubernetes会将这些"浓缩"的客户端IP固定到特定的后端Pod。结果是：

* **少数Pod**需要承载绝大部分流量，不堪重负
* **其他Pod**几乎闲置，造成资源浪费

### 2. 长连接的"雪上加霜"

现代应用普遍使用**HTTP/Keep-Alive**等长连接技术。在同一TCP连接上发起的所有请求，由于源IP和端口不变，在Session Affinity作用下会始终命中同一个Pod。

这意味着，即使客户端快速发起多个请求，只要连接未断开，流量就无法被分摊到其他Pod上，进一步加剧了负载不均。

[回到顶部](#_labelTop)

## 三、问题诊断与排查指南

当发现Pod流量不均时，建议按照以下流程系统性排查：

sessionAffinity=ClientIP

sessionAffinity=None

发现Pod流量不均

检查Service SessionAffinity配置

禁用或调整会话保持

排查其他原因：1. Endpoints状态2. kube-proxy模式3. Pod资源分配

问题解决

### 1. 检查Service配置

使用以下命令查看Service的会话保持设置：

```
kubectl describe service  -n
```

重点关注输出中的`Session Affinity`字段。若显示`ClientIP`，则说明已启用会话保持。

### 2. 验证Endpoints状态

确保所有健康的Pod都已注册到Service的Endpoints中：

```
kubectl get endpoints  -n
```

如果某个健康的Pod未出现在Endpoints列表中，请求自然不会转发到该Pod。

### 3. 监控流量分布

通过**Prometheus + Grafana**等监控工具观察各Pod的流量接收情况，这是确认流量不均现象最直观的方法。

[回到顶部](#_labelTop)

## 四、解决方案与实践

### 方案一：彻底禁用会话保持（适用于无状态服务）

对于绝大多数无状态服务，**最简单的解决方案就是直接禁用Session Affinity**。

#### 1. 使用kubectl patch命令（立即生效）

针对特定Service的修复：

```
kubectl patch svc  -n  -p '{"spec": {"sessionAffinity": "None"}}'
```

批量修复命名空间内所有Service：

```
# 注意：此命令将修改指定命名空间下 ALL Services，请确认是否符合预期
kubectl get svc -n  -o name | xargs -I {} kubectl patch {} -n  -p '{"spec": {"sessionAffinity": "None"}}'
```

#### 2. 通过编辑YAML文件（持久化修改）

如果需要持久化修改，可以编辑Service的YAML配置：

```
kubectl edit service  -n
```

然后将`sessionAffinity`字段的值从`ClientIP`改为`None`，或直接删除该字段（默认为`None`）。

#### 3. 验证修改结果

执行命令后，立即验证配置是否生效：

```
kubectl get svc  -n  -o jsonpath='{.spec.sessionAffinity}'
```

输出应为 `None`。同时，观察监控系统，可以看到各Pod的流量曲线会逐渐趋于平衡。

### 方案二：精细化流量控制（适用于需要会话保持的场景）

如果应用确实需要会话保持，但不希望流量严重不均，可以考虑以下替代方案：

#### 1. 调整会话保持时间

缩短会话保持的超时时间，平衡"保持会话"与"负载均衡"的需求：

```
sessionAffinity: ClientIP
sessionAffinityConfig:
  clientIP:
    timeoutSeconds: 3600  # 从默认3小时缩短为1小时
```

#### 2. 使用Ingress控制器实现更智能的会话保持

Nginx Ingress支持基于Cookie的会话保持，比IP-based方式更精细：

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
    nginx.ingress.kubernetes.io/session-cookie-expires: "3600"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "3600"
```

#### 3. 使用服务网格（如Istio）实现智能负载均衡

Istio可以基于实际负载情况（如延迟、错误率）动态调整流量分发：

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-service
spec:
  host: my-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN  # 最少连接数算法
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 1m
```

[回到顶部](#_labelTop)

## 五、预防措施与最佳实践

为了避免类似问题再次发生，建议采取以下预防措施：

### 1. 明确应用需求

在部署服务前，架构师和开发团队应该明确回答：

* 这是有状态服务还是无状态服务？
* 是否真正需要会话保持？
* 如果需要，什么级别的会话保持（IP-based、Cookie-based）是合适的？

### 2. 代码与配置审查

将Service配置审查纳入CI/CD流程：

* 在Pull Request中自动检查Session Affinity配置
* 对于无状态服务，如果发现`sessionAffinity: ClientIP`配置，应该提出警告
* 建立配置模板和规范

### 3. 监控与告警

建立完善的监控体系：

* 监控各Pod的请求量、CPU使用率、内存使用率
* 设置Pod间负载差异告警阈值（如：最大负载Pod的请求量是最小负载Pod的3倍以上）
* 定期生成负载分布报告

### 4. 文档化配置标准

在团队内部分享Session Affinity的影响，形成统一的配置标准：

```
# Kubernetes Service配置标准

## Session Affinity配置准则

1. **无状态Web服务**: sessionAffinity: None
2. **需要会话状态的服务**: 
   - 首选: 通过Ingress Controller的Cookie-based会话保持
   - 次选: 缩短超时时间的ClientIP会话保持
3. **有状态服务**: 使用StatefulSet而非Session Affinity

## 审查清单
- [ ] 确认服务是否真正需要会话保持
- [ ] 如果启用会话保持，是否设置了合理的超时时间
- [ ] 是否有监控机制检测流量分布
```

[回到顶部](#_labelTop)

## 六、总结

这次故障排查经历揭示了Kubernetes Session Affinity配置的重要性。通过本文的分析和解决方案，希望大家能够：

1. **深入理解Session Affinity的工作机制**及其对流量分发的深远影响
2. **掌握快速诊断和修复的方法**，特别是使用kubectl patch命令进行批量操作的能力
3. **根据应用特性合理选择**是否启用会话保持，以及如何精细化配置

记住，Session Affinity是一把双刃剑：对于需要保持会话状态的应用，它能确保用户体验的一致性；但对于无状态应用，它可能成为流量不均的罪魁祸首。

正确处理Session Affinity配置，结合适当的监控和告警机制，能够让你的Kubernetes集群运行更加稳定高效，真正发挥云原生架构的优势。

---

**经验教训**：在K8s中，看似简单的配置项背后可能隐藏着复杂的影响。始终保持对配置的敬畏之心，建立完善的审查和监控机制，才能在云原生之旅中行稳致远。
