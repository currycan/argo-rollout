# argocd rollout 蓝绿/金丝雀发布

> 导语：熟悉k8s的同学知道Deployment目前只支持RollingUpgrade和ReCreate两种策略。而对于运维的同学而言，实际生产环境中更多应该使用灰度发布和蓝绿部署，Argo-Rollout 完全可以在不造轮子的前提下实现灰度功能。

## 1. 简介

[Argo-Rollout](https://argoproj.github.io/argo-rollouts/)是一个Kubernetes Controller和对应一系列的CRD，提供更强大的Deployment能力。包括灰度发布、蓝绿部署、更新测试(experimentation)、渐进式交付(progressive delivery)等特性。

**支持特性：**

- 蓝绿部署
- 灰度发布
- 细粒的，带权重的流量调度（traffic shifting）
- 自动rollback和promotion
- 手动管理
- 可定制的metric查询和kpi分析
- Ingress controller集成：nginx，alb
- Service Mesh集成：Istio，Linkerd，SMI
- Metric provider集成：Prometheus, Wavefront, Kayenta, Web, Kubernetes Jobs

**原理：**

Argo原理和Deployment差不多，只是加强rollout的策略和流量控制。当spec.template发送变化时，Argo-Rollout就会根据spec.strategy进行rollout，通常会产生一个新的ReplicaSet，逐步scale down之前的ReplicaSet的pod数量。

## 2. 安装

[官方安装文档](https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation)

### 2.1. 安装argo-rollouts的controller和crd

``` bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://raw.githubusercontent.com/argoproj/argo-rollouts/stable/manifests/install.yaml
```

### 2.2 安装argo-rollouts的kubectl plugin

```bash
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-linux-amd64
mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
kubectl argo rollouts version
```

## 2. 使用

灰度发布包含Replica Shifting和Traffic Shifting两个过程。直接使用官方例子：

```bash
kubectl apply -f basic/argo-rollouts.yaml
```

使用 Argo-Rollout 提供的plugin查看状态:

```bash
$ kubectl argo rollouts get rollout rollouts-demo
Name:            rollouts-demo
Namespace:       default
Status:          ✔ Healthy
Strategy:        Canary
  Step:          8/8
  SetWeight:     100
  ActualWeight:  100
Images:          argoproj/rollouts-demo:blue (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS     AGE  INFO
⟳ rollouts-demo                            Rollout     ✔ Healthy  78s
└──# revision:1
   └──⧉ rollouts-demo-7bf84f9696           ReplicaSet  ✔ Healthy  78s  stable
      ├──□ rollouts-demo-7bf84f9696-2bj7p  Pod         ✔ Running  78s  ready:1/1
      ├──□ rollouts-demo-7bf84f9696-5fc9j  Pod         ✔ Running  78s  ready:1/1
      ├──□ rollouts-demo-7bf84f9696-k5z7c  Pod         ✔ Running  78s  ready:1/1
      ├──□ rollouts-demo-7bf84f9696-vh2fc  Pod         ✔ Running  78s  ready:1/1
      └──□ rollouts-demo-7bf84f9696-zq2g2  Pod         ✔ Running  78s  ready:1/1
```

更新镜像版本，触发 rollout

```bash
$ kubectl argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow
$ kubectl argo rollouts get rollout rollouts-demo -w
Name:            rollouts-demo
Namespace:       default
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/8
  SetWeight:     20
  ActualWeight:  20
Images:          argoproj/rollouts-demo:blue (stable)
                 argoproj/rollouts-demo:yellow (canary)
Replicas:
  Desired:       5
  Current:       5
  Updated:       1
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS     AGE    INFO
⟳ rollouts-demo                            Rollout     ॥ Paused   10m
├──# revision:2
│  └──⧉ rollouts-demo-789746c88d           ReplicaSet  ✔ Healthy  2m22s  canary
│     └──□ rollouts-demo-789746c88d-td2vx  Pod         ✔ Running  2m22s  ready:1/1
└──# revision:1
   └──⧉ rollouts-demo-7bf84f9696           ReplicaSet  ✔ Healthy  10m    stable
      ├──□ rollouts-demo-7bf84f9696-2bj7p  Pod         ✔ Running  10m    ready:1/1
      ├──□ rollouts-demo-7bf84f9696-5fc9j  Pod         ✔ Running  10m    ready:1/1
      ├──□ rollouts-demo-7bf84f9696-k5z7c  Pod         ✔ Running  10m    ready:1/1
      └──□ rollouts-demo-7bf84f9696-zq2g2  Pod         ✔ Running  10m    ready:1/1
```

预期Rollout会创建一个新的ReplicaSet，并且逐步扩容新的ReplicaSet和缩容旧的ReplicaSet。上面的状态中显示的是`pause`,主要原因就在这个spec.strategy，通过这个strategy我们可以看到其为升级设定了steps，由于是个列表，因此其会按照顺序执行。这里第一步就是setWeight：20，意味着需要将20%的pod更新为新版本；第二步动作为pause: {}，意味着将永久暂停，需要人为通过plugin使其继续:

```bash
kubectl argo rollouts promote rollouts-demo
```

之后，所有pod都为新的ReplicaSet的pod

```bash
$ kubectl argo rollouts get rollout rollouts-demo -w
Name:            rollouts-demo
Namespace:       default
Status:          ✔ Healthy
Strategy:        Canary
  Step:          8/8
  SetWeight:     100
  ActualWeight:  100
Images:          argoproj/rollouts-demo:yellow (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS        AGE    INFO
⟳ rollouts-demo                            Rollout     ✔ Healthy     17m
├──# revision:2
│  └──⧉ rollouts-demo-789746c88d           ReplicaSet  ✔ Healthy     9m58s  stable
│     ├──□ rollouts-demo-789746c88d-td2vx  Pod         ✔ Running     9m58s  ready:1/1
│     ├──□ rollouts-demo-789746c88d-nrmpv  Pod         ✔ Running     2m32s  ready:1/1
│     ├──□ rollouts-demo-789746c88d-7fh4l  Pod         ✔ Running     2m21s  ready:1/1
│     ├──□ rollouts-demo-789746c88d-94tgx  Pod         ✔ Running     2m10s  ready:1/1
│     └──□ rollouts-demo-789746c88d-ljhvp  Pod         ✔ Running     119s   ready:1/1
└──# revision:1
   └──⧉ rollouts-demo-7bf84f9696           ReplicaSet  • ScaledDown  17m
```

此例子演示了Argo-Rollout如何控制Replica Shifting，而正常的灰度过程，应该包含Replica Shifting和Traffic Shifting两部分。

目前Argo-Rollout主要集成了Ingress和ServiceMesh两种流量控制方法，使用Ingress做演示, 依旧使用官方 demo, 如下：

```bash
kubectl apply -f nginx/argo-rollouts.yaml
```

总共会部署1个rollout，两个service和一个ingress，Service rollouts-demo-canary 和 rollouts-demo-stable，二者内容一样。selector中暂时没有填上pod-template-hash，Argo-Rollout Controller会根据实际的ReplicaSet hash来修改该值

```bash
$ kubectl get ing
NAME                                        CLASS    HOSTS                 ADDRESS     PORTS   AGE
rollouts-demo-rollouts-demo-stable-canary   <none>   rollouts-demo.local   localhost   80      47s
rollouts-demo-stable                        <none>   rollouts-demo.local   localhost   80      47s
```

Rollout Controller会根据ingress rollouts-demo-stable内容，**自动创建一个ingress用了灰度的流量**，名字为`<ROLLOUT-NAME>-<INGRESS-NAME>-canary`，所以这里多了一个ingress rollouts-demo-rollouts-demo-stable-canary，将流量导向Canary Service(rollouts-demo-canary).

打开 `rollouts-demo.local`，web 界面中只有蓝色圆点

进行发布更新：

```bash
kubectl argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow
kubectl argo rollouts get rollout rollouts-demo
```

此时 web 界面中会有黄色和蓝色会根据配置权重出现。

灰度策略在执行到5%会暂停，之后验证通过后继续执行：

![灰度过程](https://tva1.sinaimg.cn/large/008i3skNgy1gr2t8rccp4j31r50u04qp.jpg)

```bash
$ kubectl argo rollouts get rollout rollouts-demo
Name:            rollouts-demo
Namespace:       default
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/2
  SetWeight:     5
  ActualWeight:  5
Images:          argoproj/rollouts-demo:yellow (canary)
                 argoproj/rollouts-demo:blue (stable)
```

全部迁移新版本：

```bash
kubectl argo rollouts promote rollouts-demo
```

此时流量 100%迁移到新版本，web 界面中会有黄色 圆点

![终态](https://tva1.sinaimg.cn/large/008i3skNgy1gr2t6cztdvj31q60u0qr5.jpg)

此外，argo rollout 提供了一套简易的面板：

```bash
kubectl argo rollouts dashboard
```

`localhost:3100` 即可打开。可以执行回滚和放行操作，省去命令行操作，但是没有用户管理权限，可以在测试环境使用。
![dashboard](https://tva1.sinaimg.cn/large/008i3skNgy1gr2t444h26j31vr0u0qbt.jpg)

## 3. 总结

Argo-Rollout提供更加强大的Deployment，包含比较适合运维的灰度发布和蓝绿发布功能。
