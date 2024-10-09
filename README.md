
 


# 一、Kubernetes 中 Pod 调度的重要性


在 Kubernetes 的世界里，Pod 调度就像是一个繁忙的交通指挥官，负责把小车（也就是我们的 Pod）送到最合适的停车位（节点）。调度不仅关乎资源的合理利用，还关乎应用的“生死存亡”，下面让我们来看看为什么调度这么重要。


1. **资源优化**: 想象一下，如果每辆小车都随意停放，那简直是停车场的灾难！调度器通过精确的“停车导航”，确保每个 Pod 找到合适的停车位，最大化利用资源，避免“挤得满满当当”。
2. **故障恢复**: 假设某个节点出故障了，就像一辆车突然抛锚。调度器会迅速反应，把出问题的 Pod 送到其他健康的节点，确保你的应用不至于“趴窝”。
3. **负载均衡**: 想象一下，如果所有车都停在同一边，另一边却空荡荡的，那就会造成交通堵塞。调度器会聪明地把 Pod 分散到各个节点，保持负载均匀，就像一个和谐的舞蹈。
4. **策略实施**: Kubernetes 调度器可不止是个简单的指挥官，它还有一套自己的“调度法则”。通过亲和、反亲和、污点和容忍等机制，调度器确保每个 Pod 都能按照自己的“喜好”找到理想的驻地，确保万无一失。
5. **可扩展性**: 当你的应用像气球一样迅速膨胀，调度器的灵活性就显得尤为重要。它可以轻松应对负载的变化，动态扩展和收缩，确保一切运转顺利。


总之，Pod 调度在 Kubernetes 中就像是后台默默工作的英雄，保证了应用的高效、安全和稳定。了解调度机制，能让你在这个容器化的世界里游刃有余，简直是必备技能！


转载请在文章开头注明原文地址：https://github.com/Sunzz/p/18451805


# 二、Node Selector


## 定义与用法


Node Selector，就像是一位细心的“挑剔”朋友，专门帮你选择最合适的聚会场地。在Kubernetes中，Node Selector用来告诉调度器，某个Pod需要在特定的节点上运行。通过这种方式，你可以确保你的应用在最合适的环境中“发光发热”。


想象一下，你的应用是一位超级明星，它希望在拥有高性能显卡的节点上演出，而不是在一个配置较低的机器上“打酱油”。Node Selector正是为了满足这种需求，让你的Pod在适合它们的舞台上展现才华。


## 示例


假设你有一个需要强大磁盘io能力的应用，想把它放在一个“磁盘大咖”的节点上。


### 给节点设置标签


先来给node01设置一个disk\=ssd的label



```
kubectl label nodes k8s-node01.local disktype=ssd
```

查看一下标签



```
kubectl get nodes -l disktype=ssd 
```

`NAME               STATUS   ROLES    AGE    VERSION``k8s-node01.local   Ready       161d   v1.30.0`


然后使用Node Selector来指定节点的标签。例如：



```
# node-selector.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2  # 设置 Pod 副本数为 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
      nodeSelector:
        disktype: ssd
```

### 创建deployment



```
kubectl apply -f node-selector.yaml 
deployment.apps/nginx-deployment created

kubectl get po -o wide 
NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE               NOMINATED NODE   READINESS GATES
nginx-deployment-56f59878c8-cgfsx   1/1     Running   0          8s    10.244.1.32   k8s-node01.local              
nginx-deployment-56f59878c8-z64rz   1/1     Running   0          8s    10.244.1.31   k8s-node01.local              
```

在这个例子中，Pod会被调度到一个标签为 `disktype: ssd` 的节点上。这样，你的超级明星就可以在最佳环境中大放异彩，而不是在普通的节点上苦苦挣扎！


所以，Node Selector就是你的“挑剔朋友”，帮助你的Pod找到最合适的“舞台”，确保它们能够充分发挥潜力。


 


# 三、亲和与反亲和


## 1\. 亲和（Affinity）


### 定义


亲和，听起来像个恋爱中的小年轻，其实它是 Kubernetes 中帮助你选择 Pod 在哪个节点上“约会”的小助手。通过亲和规则，Pod 可以被调度到具有特定标签的节点上，就像在选择一个合适的地方约会一样！


### 类型


**1\. 节点亲和（Node Affinity）**:


就像在大城市里找适合自己的房子一样，节点亲和让你可以将 Pod 调度到具有特定标签的节点上。比如，你可能想把 Pod 安排在 SSD 硬盘的节点上，因为那儿的性能更好。


**2\. Pod 亲和（Pod Affinity）**:


如果你想让某些 Pod 在一起生活，互相照应，Pod 亲和就派上用场了。它允许你把新的 Pod 调度到已经存在某个 Pod 的节点上，形成一个“温暖的大家庭”。


### 示例


下面是一个部署 Nginx 的 YAML 文件，使用节点亲和来确保 Pod 在带有 `disktype=ssd` 标签的节点上运行：



```
# affinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
```

创建deployment



```
kubectl apply -f affinity.yaml 
```

`deployment.apps/nginx-deployment created`



```
kubectl get pods -l app=nginx -o wide
```

![](https://img2024.cnblogs.com/blog/1157397/202410/1157397-20241008155605647-1271743284.png)


可以看到都运行在node01的节点上。


## 反亲和（Anti\-affinity）


### 定义


反亲和是 Kubernetes 中的另一种调度规则，它的目标是避免将 Pod 调度到与某些特定 Pod 相同的节点上。这就像在选择朋友时，某些人你就是不想和他们一起住，即使他们的房子很漂亮。


### 用途


反亲和通常用于提高应用程序的可用性和容错性。比如，如果你有多个副本的 Pod，而它们都运行在同一个节点上，那么这个节点出问题时，所有副本都会受到影响。反亲和可以确保它们分布在不同的节点上，就像把鸡蛋放在不同的篮子里，以免一篮子摔了，鸡蛋全没了。


### 示例


下面是一个使用反亲和规则的 YAML 文件，确保 Nginx Pod 不会与已经存在的 Nginx Pod 一起运行：



```
# anti-affinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
```

创建deployment



```
kubectl apply -f anti-affinity.yaml
```

创建后结果如下图所示



```
kubectl get po -l app=nginx -o wide
```

![](https://img2024.cnblogs.com/blog/1157397/202410/1157397-20241008160345277-755643768.png)


可以看到已经有两个pod在运行，分别以node01和node02上，另一个处于peding中。


查看一下为何处于pending中



```
kubectl describe pod nginx-deployment-5675f7647f-vgkrl  
```


```
kubectl describe pod nginx-deployment-5675f7647f-vgkrl  

Name:             nginx-deployment-5675f7647f-vgkrl
Namespace:        default
Priority:         0
Service Account:  default
Node:             
Labels:           app=nginx
                  pod-template-hash=5675f7647f
Annotations:      
Status:           Pending
IP:               
IPs:              
Controlled By:    ReplicaSet/nginx-deployment-5675f7647f
Containers:
  nginx-container:
    Image:        nginx:latest
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-qwjc2 (ro)
Conditions:
  Type           Status
  PodScheduled   False 
Volumes:
  kube-api-access-qwjc2:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  2m7s  default-scheduler  
  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 
  2 node(s) didn't match pod anti-affinity rules. 
  preemption: 0/3 nodes are available: 1 Preemption is not helpful for scheduling, 
  2 No preemption victims found for incoming pod.
```

![](https://img2024.cnblogs.com/blog/1157397/202410/1157397-20241008160754124-614585304.png)


可以看到最后边的信息，说明没有可用的节点。


转载请在文章开头注明原文地址：https://github.com/Sunzz/p/18451805


## 总结


通过亲和与反亲和，Kubernetes 可以根据你的需求，精确调度 Pod，就像一位优秀的派对策划者，确保每个 Pod 在合适的节点上“社交”。这不仅提高了资源利用率，还增强了应用的稳定性。下次部署时，别忘了这些小秘密哦！


 


# 四、污点与容忍：如何让你的 Pod 不那么“娇气”


## 污点（Taints）


### **定义与用途**


污点，就像一块“禁止进入”的牌子，告诉某些 Pod：“嘿，别过来，我不想跟你玩！” 它的作用就是让 Kubernetes 中的节点标记出特殊要求，只有“合适”的 Pod 才能在那里运行。


比如，你有一个超强的节点，需要做一些高强度的计算任务，那你就可以给这个节点加个“污点”，这样普通的 Pod 就不会误闯进来占用资源了。


### **示例**



```
kubectl taint nodes k8s-node01.local key=value:NoSchedule
```

在这个例子中，我们给节点 `k8s-node01.local` 添加了一个叫做 `key=value` 的污点，并设置为 `NoSchedule`。这意味着，除非 Pod 具备容忍这个污点的“超级能力”，否则 Kubernetes 不会调度任何 Pod 到这个节点上。


来创建deployment测试一下看还会不会调度到node01上去



```
# deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3 
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

创建deployment



```
kubectl apply -f deployment.yaml 
```

查看pod信息



```
kubectl get pods -l app=nginx -o wide
```

![](https://img2024.cnblogs.com/blog/1157397/202410/1157397-20241008161649732-930008842.png)


可以看到pod都运行在node02上，符合预期，说明污点生效了，默认是不会往node01去调度的。


## 容忍（Tolerations）


### **定义与用途**


容忍就是 Pod 的“通行证”，让它可以无视节点上的“禁止进入”标志（污点）。Pod 如果想进驻被“污点”标记的节点，就必须带上这个“通行证”才行。这就像是：有些节点很挑剔，但有的 Pod 很“宽容”，它说：“没关系，我能忍。”


### **如何配合污点使用**


容忍的作用就是让 Pod 在被打了污点的节点上仍然能够正常调度。这两者就像是门卫和 VIP 卡的关系——门卫不让随便人进，但有 VIP 卡的 Pod 可以说：“我有容忍力，我能过！”


示例：



```
# tolerations-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      tolerations:
      - key: "key"
        operator: "Equal"
        value: "value"
        effect: "NoSchedule"
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

### 解释：


* **apiVersion**: `apps/v1` 表示我们创建的是 `Deployment` 资源，Kubernetes 中的 `Deployment` 用于管理应用的副本集（即多个 Pod）。
* **replicas**: 定义了我们希望创建的 `nginx` Pod 的副本数量，这里我们设置为 `2`，因此会有两个 `nginx` Pod。
* **selector**: `matchLabels` 通过 `app: nginx` 标签选择需要管理的 Pod。
* **template**: 定义了要部署的 Pod 模板，包括容忍设置和容器规范。
* **tolerations**: 配置了这个 `nginx` Deployment 的 Pod 可以容忍有特定 `key=value:NoSchedule` 污点的节点，允许它们被调度到带有该污点的节点上。
* **containers**: Pod 内的容器，这里使用 `nginx:latest` 镜像，并暴露端口 80。


这个配置部署了两个 `nginx` Pod，并且允许它们被调度到有特定污点的节点上（根据配置的 `tolerations`）。


创建deployment



```
kubectl get po -l app=nginx -o wide
```

![](https://img2024.cnblogs.com/blog/1157397/202410/1157397-20241008162335087-1494146905.png)


这个示例 Pod 中的 `tolerations` 配置，允许它无视 `key=value:NoSchedule` 这样的污点，顺利地跑到有“污点”的节点上。


## 污点与容忍，谁才是调度的老大？


总结一下，污点是节点上的“门禁”，防止无关 Pod 来捣乱，而容忍就是 Pod 的“VIP 卡”，让它无视门禁，照样可以顺利进驻。这两者相辅相成，是 Kubernetes 调度机制中不可或缺的一部分。


有了这些设置，你就可以有效地控制哪些 Pod 能够运行在哪些节点上。记住：带上“通行证”，你就不怕节点的“臭脾气”了！


转载请在文章开头注明原文地址：https://github.com/Sunzz/p/18451805


# 五、优先级与抢占：Kubernetes 的「大佬」调度策略


在 Kubernetes 世界中，Pod 不再是平等的。是的，Pod 也有「大佬」和「小透明」之分！这一切全靠优先级（Priority）和抢占（Preemption）来决定。让我们一起看看这两位神秘力量如何影响 Pod 的命运吧！


## 优先级（Priority）: 谁是大佬？


**定义**：


优先级就是给 Pod 排座次的机制。Kubernetes 允许你给每个 Pod 设定一个「重要程度」，这个重要程度就决定了它在资源紧张时，是被优先照顾，还是默默无闻地排队。


**如何设置**：


我们需要先定义一个 `PriorityClass`，然后在 Nginx 的 `Deployment` 里引用它。就像是给 Nginx 发了一张“贵宾卡”。



```
# priority.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "High priority for important Nginx pods."

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      priorityClassName: high-priority
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

解释： 我们创建了一个名为 `high-priority` 的优先级类，给它赋值 1000，然后用这个优先级类部署了两个 Nginx Pod。这意味着这些 Nginx Pod 在资源调度上会得到特别的优待，资源紧张时它们会优先被分配。


创建deployment



```
kubectl apply -f priority.yaml 
```

查看pod信息



```
kubectl get po -l app=nginx -o wide

kubectl describe pod nginx-deployment-646cb6c499-6k864 
```

**![](https://img2024.cnblogs.com/blog/1157397/202410/1157397-20241008164526355-8070408.png)**


## 抢占（Preemption）：Nginx 也能“赶人”？


**定义**： 抢占就像给 Nginx 一个特权，告诉 Kubernetes 如果资源紧张，允许这个高优先级的 Pod 抢占低优先级 Pod 的位置。这样，Nginx 能在关键时刻优雅地“赶走”别人，自己稳稳上场。


**示例**：



```
# preemption.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: vip-priority
value: 2000
preemptionPolicy: PreemptLowerPriority
globalDefault: false
description: "VIP Nginx pods with preemption power."

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment-vip
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-vip
  template:
    metadata:
      labels:
        app: nginx-vip
    spec:
      priorityClassName: vip-priority
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

**解释**： 这里，我们为 `vip-priority` 类的 Nginx Pod 赋予了“抢占”特权。如果资源不足，系统会强制“请走”低优先级的 Pod，让 Nginx VIP 顺利登场。


创建



```
 kubectl apply -f preemption.yaml
```

查看pod信息


![](https://img2024.cnblogs.com/blog/1157397/202410/1157397-20241008165043580-775275499.png)


## 总结：


通过设置优先级，我们可以让某些重要的 Nginx Pod 享有调度特权，而通过抢占，它们甚至能赶走低优先级的 Pod，占据宝贵的资源位。用这种方式，Nginx 不仅能在服务器上跑得快，还能在调度策略上“先发制人”！


 


# 六、调度策略：谁来决定 Nginx Pod 住哪里？


Kubernetes 的调度器就是负责给 Pod 找房子的“房屋中介”。它得看哪儿房源充足、住得舒服，同时还得考虑全局平衡。Pod 会被分配到哪台节点上运行，背后其实有一套策略。下面我们来深挖几种常见的调度策略，看看 Nginx Pod 是怎么找到“新家”的。


## **轮询调度（Round Robin）：大家轮着来，公平第一！**


**通俗解释**：就像大家吃火锅的时候，食材一轮轮下锅，谁都不会落下。轮询调度按顺序遍历节点，把 Pod 均匀地分配到每个节点上。节点A、B、C都分到活儿，不会让某个节点忙死，而其他节点闲得发霉。


**举个例子**：Nginx Pod 1 给节点A，Pod 2 给节点B，Pod 3 给节点C，下一个再轮到A。


**优点**：**简单公平**，特别适合资源相对均衡的情况，能避免某个节点过载。**缺点**：轮着来不等于“聪明地来”，有可能会给那些快要爆满的节点安排更多任务，徒增压力。




---





 







## **随机调度（Random）：碰运气，Pod 和节点看缘分！**


**通俗解释**：调度器甩骰子，随便挑个节点来安排 Nginx Pod。这种策略就是“看心情”，随机分配，不按套路出牌，Pod 住哪里全靠缘分。


**举个例子**：有三个节点 A、B、C，Nginx Pod 1 可能给 A，Pod 2 随机分配给 C，Pod 3 再给 B。每次选择都是随机的。


**优点**：简单直接，偶尔会带来意想不到的“运气”。**缺点**：太随机！有时候会导致部分节点被过度使用，而其他节点却没啥事做。




---


### **基于资源调度（Resource\-based）：资源至上，优先住“大房子”！**


**通俗解释**：调度器在分房子之前会先看房源——CPU、内存等资源。基于资源调度就像挑个有豪华设施的房子住进去，Nginx Pod 会优先安排在资源最多、最宽裕的节点上。


**举个例子**：如果节点A资源还剩 60%，节点B只剩 10%，而节点C还有 80%，调度器会选择把 Pod 安排在资源最丰富的节点C，这样住得最舒服。


**优点**：聪明！能合理利用资源，避免浪费。**缺点**：稍微复杂点，可能导致调度时间增加，尤其是节点众多时。




---


## 其他调度策略，个性化服务一应俱全！


除了轮询、随机和资源优先这几种常见的调度策略，Kubernetes 还有一些更加“个性化”的方式来满足不同需求：


* **分散调度（Spread）**：尽量把 Pod 分散到不同节点上，避免所有 Nginx Pod 都集中到一个节点。如果这个节点挂了，Pod 全军覆没可就尴尬了。
* **紧凑调度（Binpack）**：与分散调度相反，这个策略会尽量把 Pod 塞到一个节点里，把资源用满，最大化利用空间，像打包行李一样让“房子”住得更紧凑。




---


## 总结一下：


Kubernetes 的调度器不只是个“搬运工”，它更像是一个智能中介，依据不同的调度策略为 Pod 找到最合适的家。无论是公平分配的轮询调度、看运气的随机调度，还是讲究资源的“豪宅优先”，调度器总能帮 Nginx Pod 安排一个舒适的居所。而且如果有特殊需求，分散调度和紧凑调度等个性化策略也随时待命，让你的集群更高效稳定。


选择什么策略，全看你怎么安排这个“搬家”计划！









# 七、Volume Affinity：Pod 和存储卷的“宿命连结”！


**定义与用途**：你知道 Pod 和 Volume 之间的关系吗？它们的默契就像“灵魂伴侣”。在 Kubernetes 世界里，**Volume Affinity** 就是确保 Pod 能和它最需要的存储卷待在一起，不分离、不断电。这个机制会让存储卷和 Pod 彼此靠得更近，就像给你的 Pod 安装了“GPS”，确保它们的存储数据触手可及。


**举个通俗例子**：想象一下，你的 Pod 就像一个咖啡师，而存储卷就是它的咖啡豆仓库。Volume Affinity 确保咖啡师不会跑到一个没有咖啡豆的地方开店，简直是开店之前先定好仓库的节奏！这让 Pod 不用担心去取远在天边的存储数据，一切就近服务，方便又高效。




### 例子：Nginx 和它的专属“存储豆仓”


我们再用熟悉的 Nginx 举个例子。假设我们想要 Nginx Pod 和它的存储卷靠得近一些，Volume Affinity 会帮忙安排这个“亲密关系”。



```
# volume-affinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        volumeMounts:
        - name: nginx-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: nginx-storage
        persistentVolumeClaim:
          claimName: nginx-pvc
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx
            topologyKey: "kubernetes.io/hostname"
```





**解释一下**：


* **volumeMounts** 和 **volumes** 部分定义了 Pod 和它的存储卷（这就是那袋咖啡豆）的关系。
* **podAffinity** 确保 Pod 跟存储卷靠得更近，通过设置 `topologyKey`，Pod 会优先安排到跟存储卷位于相同节点的地方。


**小总结**：Volume Affinity 就像是给 Pod 分配了一个存储保镖，Pod 再也不用担心去异地存储提取数据了。这样不仅提升了性能，还减少了存储之间的“异地恋”，毕竟存储和计算还是要多点联系才甜蜜嘛！








转载请在文章开头注明原文地址：https://github.com/Sunzz/p/18451805
# 总结：调度机制，Kubernetes 的“资源调度大师”！


让我们再来好好回味一下 Kubernetes 的各种调度机制吧！就像是在一场完美的大合唱中，每一个音符都得精准地落在该落的位置，Kubernetes 的调度机制就是确保你的 Pod 被安排得明明白白，让每一份计算资源都物尽其用。
 





**Node Selector**这是 Kubernetes 的“房屋中介”，确保你的 Pod 住进对口的节点小区。比如，你说“我要 SSD 存储的节点”，Kubernetes 立马帮你安排。简单粗暴，直击要点。


**亲和与反亲和（Affinity \& Anti\-affinity）**就像是朋友之间的“朋友圈”，你可以给你的 Pod 配个好邻居，也可以让它远离不合适的“前任”。亲和性确保它跟喜欢的节点待在一起，而反亲和性则避免与那些“看不顺眼”的 Pod 混在一起。


**污点与容忍（Taints \& Tolerations）**这是 Kubernetes 的“入门防护”，一些节点可能不适合普通 Pod“打扰”，但有些 Pod“训练有素”，它们带着容忍度可以自由通行，这让整个集群更加秩序井然。


**优先级与抢占（Priority \& Preemption）**当资源紧张时，这就像在机场登机口前的 VIP 通道。有些 Pod 拥有更高的优先级，它们会被优先安排到最好的“座位”上。而抢占机制则是把一些普通乘客挤到后面去，给紧急任务让路。


**调度策略**调度策略就像是你集群的“活动策划师”，它可以按资源需求、轮询、随机、或者任何你设计的策略来安排 Pod，确保集群里的每个资源点都忙得不亦乐乎。


**Volume Affinity**这就像为你的 Pod 找个近在咫尺的“咖啡豆仓库”，确保它的存储卷触手可及，减少“异地传输”，提高效率！




---


### 重申：调度让一切变得井井有条


Kubernetes 这些调度机制不仅仅是为了“好看”，它们是资源优化的大杀器！无论是提高节点的利用率，还是确保关键任务优先执行，或者让存储和计算更加亲密无间，所有机制都有一个共同的目标——让你的应用跑得更快，资源用得更好，集群管理更轻松。


Kubernetes 的调度机制就是这样，既聪明又懂分配，让你的 Pod 像是有了“神队友”一样，被安排到最适合的地方。而你呢，作为集群的大总管，当然可以舒舒服服地看着这些机制把一切打理得妥妥当当！








 





 



 


 本博客参考[豆荚加速器](https://yirou.org)。转载请注明出处！
