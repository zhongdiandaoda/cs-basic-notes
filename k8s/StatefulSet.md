“Deployment用于部署无状态服务，StatefulSet用来部署有状态服务”。

具体的，什么场景需要使用StatefulSet呢？官方给出的建议是，如果你部署的应用满足以下一个或多个部署需求，则建议使用StatefulSet。

- 稳定的、唯一的网络标识。
- 稳定的、持久的存储。
- 有序的、优雅的部署和伸缩。
- 有序的、优雅的删除和停止。
- 有序的、自动的滚动更新。

稳定的主要是针对 Pod 发生 re-schedule 后仍然要保持之前的网络标识和持久化存储。

这里所说的网络标识包括 hostname、集群内 DNS 中该Pod对应的 Record，并不能保证 Pod re-schedule 之后IP不变。

借助 StatefulSet 的稳定的唯一的网络标识特性，我们能比较轻松的实现 Pod 的固定 IP 需求，然后如果使用 Deployment，那么将会复杂的多，你需要考虑滚动更新的过程中的参数控制(maxSurge、maxUnavailable)、每个应用的 IP 池预留造成的IP浪费等等问题。

# 需求分析

1. Pod 之间并非相同的副本，每个Pod有一个独立标识
2. Pod的独立标识需要对应到一个固定的网络标识，并在发布升级后能保持
3. 每个Pod有一块独立的存储盘，并在发布升级后还能继续挂载
4. 应用发布时，按照固定顺序升级 Pod

# 与deployment的对比

|   |   |   |
|---|---|---|
|特性|Deployment|StatefulSet|
|是否暴露到外网|可以|一般不|
|请求面向的对象|serviceName|指定pod的域名|
|灵活性|只能通过service/serviceIp访问到k8s自动转发的pod|可以访问任意一个自定义的pod|
|易用性|只需要关心Service的信息即可|需要知道要访问的pod启动的名称、headlessService名称|
|PV/PVC绑定关系的稳定性（多replicas）|（pod挂掉后重启）无法保证初始的绑定关系|可以保证|
|pod名称稳定性|不稳定，因为是通过template创建，每次为了避免重复都会后缀一个随机数|稳定，每次都一样|
|启动顺序（多replicas）|随机启动，如果pod宕掉重启，会自动分配一个node重新启动|pod按 app-0、app-1...app-（n-1），如果pod宕掉重启，还会在之前的node上重新启动|
|停止顺序（多replicas）|随机停止|倒序停止|
|集群内部服务发现|只能通过service访问到随机的pod|可以打通pod之间的通信（主要是被发现）|
|性能开销|无需维护pod与node、pod与PVC 等关系|比deployment类型需要维护额外的关系信息|

- 如果是不需额外数据依赖或者状态维护的部署，或者 replicas 是1，优先考虑使用 Deployment；
- 如果单纯的要做数据持久化，防止 pod 宕掉重启数据丢失，那么使用 pv/pvc 就可以了；
- 如果要打通 app 之间的通信，而又不需要对外暴露，使用 headlessService 即可；
- 如果需要使用 service 的负载均衡，不要使用 StatefulSet，尽量使用 clusterIP 类型，用 serviceName 做转发；
- 如果是有多replicas，且需要挂载多个pv且每个pv的数据是不同的，因为pod和pv之间是 一 一对应的，如果某个pod挂掉再重启，还需要连接之前的pv，不能连到别的pv上，考虑使用StatefulSet，能不用StatefulSet，就不要用。

# 最佳实践

- StatefulSet 对应 Pod 的存储最好通过 StorageClass 来动态创建：每个 Pod 都会根据 StatefulSet 中定义的 VolumeClaimTemplate 来创建一个对应的 PVC，然后 PVC 通过 StorageClass自动创建对应的 PV，并挂载给 Pod。所以这种方式，需要你事先创建好对应的 StorageClass。当然，你也可以通过预先由管理员手动创建好对应的 PV，只要能保证自动创建的 PVC 能和这些 PV 匹配上。
- 为了数据安全，当删除 StatefulSet 中 Pods 或者对 StatefulSet 进行缩容时，Kubernetes 并不会自动删除 StatefulSet 对应的PV，而且这些 PV 默认也不能被其他 PVC Bound。当你确认数据无用之后再手动去删除PV的时候，数据是否删除取决于 PV 的 ReclaimPolicy 配置。Reclaim Policy 支持以下三种:

- Retain，意味着需要你手动清理；
- Recycle，等同于rm -rf /thevolume/*
- Delete，默认值，依赖于后端的存储系统自己实现。注意：

- 目前只有NFS和HostPath支持Recycle;
- EBS,GCE PD, Azure Disk，Openstack Cinder支持Delete。

- 请小心删除 StatefulSet 对应的PVC，首先确保 Pods 已经完全 Terminate，然后确定不需要 Volume 中的数据后，再考虑删除 PV。因为删除 PVC 可能触发对应 PV 的自动删除，并根据StorageClass 中的 recalimPolicy 配置可能造成 volume 中的数据丢失。
- 通过配置 pvc template，可以使每个 Pod 拥有一块独享的pv存储。
- 因为部署的是有状态应用，我们需要自己创建对应的 Headless Service，注意 Label 要和 StatefulSet 中 Pods 的 Label 匹配。Kubernetes 会为该 Headless Service 创建对应 SRV Records，包含所有的后端 Pods，KubeDNS 会通过 Round Robin 算法进行选择。

# 如何理解稳定的网络标识

StatefulSet中反复强调的“稳定的网络标识”，主要指Pods的hostname以及对应的DNS Records。

- HostName：StatefulSet的Pods的hostname按照这种格式生成：$(statefulset name)-$(ordinal)， ordinal从0 ~ N-1(N为期望副本数)。

- StatefulSet Controller在创建pods时，会给pod加上一个pod name label：statefulset.kubernetes.io/pod-name, 然后设置到Pod的pod name和hostname中。
- pod name label有啥用呢？我们可以创建独立的Service匹配到这个指定的pod，然后方便我们单独对这个pod进行debug等处理。

- DNS Records：

- Headless Service的DNS解析：$(service name).$(namespace).svc.cluster.local 通过DNS RR解析到后端其中一个Pod。SRV Records只包含对应的Running and Ready的Pods，不Ready的Pods不会在对应的SRV Records中。
- Pod的DNS解析：$(hostname).$(service name).$(namespace).svc.cluster.local解析到对应hostname的Pod。

# 如何理解稳定的持久化存储

- 每个Pod对应一个PVC，PVC的名称是这样组成的:$(volumeClaimTemplates.name)-$(pod's hostname),跟对应的Pod是一一对应的。
- 当 Pod 发生 re-schedule（其实是recreate）后，它所对应的 PVC 所 Bound 的 PV 仍然会自动的挂载到新的Pod中。
- Kubernetes 会按照 VolumeClaimTemplate 创建 N(N为期望副本数)个 PVC，由 PVCs 根据指定的 StorageClass 自动创建 PVs。
- 当通过级联删除 StatefulSet 时并不会自动删除对应的 PVCs，所以 PVC 需要手动删除。
- 当通过级联删除 StatefulSet 或者直接删除对应 Pods 时，对应的 PVs 并不会自动删除。需要你手动的去删除 PV。

# 部署和伸缩时与Deployment的区别

- 当部署有N个副本的StatefulSet应用时，严格按照index从0到N-1的递增顺序创建，下一个Pod创建必须是前一个Pod Ready为前提。
- 当删除有N个副本的StatefulSet应用时，严格按照index从N-1到0的递减顺序删除，下一个Pod删除必须是前一个Pod shutdown并完全删除为前提。
- 当扩容StatefulSet应用时，每新增一个Pod必须是前一个Pod Ready为前提。
- 当缩容StatefulSet应用时，没删除一个Pod必须是前一个Pod shutdown并成功删除为前提。
- 注意StatefulSet的pod.Spec.TerminationGracePeriodSeconds不要设置为0。

# Node网络异常等情况下该如何处理

- 正常情况下，StatefulSet Controller会保证集群内同一namespace下不会出现多个相同network identity的StatefulSet Pods。
- 如果集群内出现以上情况，那么有可能导致该有状态应用不能正常工作、甚至出现数据丢失等致命问题。

那么什么情况下会导致出现同一namespace下会出现多个相同network identity的StatefulSet Pods呢？我们考虑下Node出现网络Unreachable的情况：

- 如果你使用Kubernetes 1.5之前的版本，当Node Condition是NetworkUnavailable时，node controller会强制从apiserver中删除这个Node上的这些pods对象，这时StatefulSet Controller就会自动在其他Ready Nodes上recreate同identity的Pods。这样做其实风险是很大的，可能会导致有一段时间有多个相同network identity的StatefulSet Pods，可能会导致该有状态应用不能正常工作。所以尽量不要在Kubernetes 1.5之前的版本中使用StatefulSet，或者你明确知道这个风险并且无视它。
- 如果你使用Kubernetes 1.5+的版本，当Node Condition是NetworkUnavailable时，node controller不会强制从apiserver中删除这个Node上的这些pods对象，这些pods的state在apiserver中被标记为Terminating或者Unknown，因此StatefulSet Controller并不会在其他Node上再recreate同identity的Pods。当你确定了这个Node上的StatefulSet Pods shutdown或者无法和该StatefulSet的其他Pods网络不同时，接下来就需要强制删除apiserver中这些unreachable pods object，然后StatefulSet Controller就能在其他Ready Nodes上recreate同identity的Pods，使得StatefulSet继续健康工作。

那么在Kubernetes 1.5+中，如何强制从apiserver中删除该StatefulSet pods呢？有如下三种方法：

- 如果Node永久的无法连接网络或者关机了，意味着能确定这个Node上的Pods无法与其他Pods通信了，不会对StatefulSet应用的可用性造成影响，那么建议手动从apiserver中删除该NetworkUnavailable的Node，Kubernetes会自动从apiserver中删除它上面的Pods object。
- 如果Node是因为集群网络脑裂导致的，则建议去检查网络问题并成功恢复，因为Pods state已经是Terminating或者Unkown,所以kubelet从apiserver中获取到这个信息后就会自动删除这些Pods。
- 其他情况才考虑直接手动从apiserver中删除这些Pods，因为这时你无法确定对应的Pods是否已经shutdown或者对StatefulSet应用无影响，强制删除后就可能导致出现同一namespace下有多个相同network identity的StatefulSet Pods，所以尽量不要使用这种方法。

- kubectl delete pods <pod> --grace-period=0 --force

小知识：当前Node Condition有以下6种：

| Node Condition     | Description                                                                                                                                                                                             |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| OutOfDisk          | True if there is insufficient free space on the node for adding new pods, otherwise False                                                                                                               |
| Ready              | True if the node is healthy and ready to accept pods, False if the node is not healthy and is not accepting pods, and Unknown if the node controller has not heard from the node in the last 40 seconds |
| MemoryPressure     | True if pressure exists on the node memory – that is, if the node memory is low; otherwise False                                                                                                        |
| DiskPressure       | True if pressure exists on the disk size – that is, if the disk capacity is low; otherwise False                                                                                                        |
| NetworkUnavailable | True if the network for the node is not correctly configured, otherwise False                                                                                                                           |
| ConfigOK           | True if the kubelet is correctly configured, otherwise False                                                                                                                                            |
|                    |                                                                                                                                                                                                         |

# StatefulSet的Pod管理策略

Kubernetes 1.7+，StatefulSet开始支持Pod Management Policy配置，提供以下两种配置：

- OrderedReady，StatefulSet的Pod默认管理策略，就是逐个的、顺序的进行部署、删除、伸缩，也是默认的策略。
- Parallel，支持并行创建或者删除同一个StatefulSet下面的所有Pods，并不会逐个的、顺序的等待前一个操作确保成功后才进行下一个Pod的处理。其实用这种管理策略的场景非常少。

# StatefulSet的更新策略

StatefulSet的更新策略（由.spec.updateStrategy.type指定）支持以下两种：

- OnDelete, 含义同Deployment的OnDelete策略，大家应该很熟悉了，不多介绍。
- RollingUpdate，滚动更新过程也跟Deployment大致相同，区别在于：

- 相当于Deployment的maxSurge=0，maxUnavailable=1（其实StatefulSet是不存在这两个配置的）
- 滚动更新的过程是有序的（逆序），index从N-1到0逐个依次进行，并且下一个Pod创建必须是前一个Pod Ready为前提，下一个Pod删除必须是前一个Pod shutdown并完全删除为前提。
- 支持部分实例滚动更新，部分不更新，通过.spec.updateStrategy.rollingUpdate.partition来指定一个index分界点。

- 所有ordinal大于等于partition指定的值的Pods将会进行滚动更新。
- 所有ordinal小于partition指定的值得Pods将保持不变。即使这些Pods被recreate，也会按照原来的pod template创建，并不会更新到最新的版本。
- 特殊地，如果partition的值大于StatefulSet的期望副本数N，那么将不会触发任何Pods的滚动更新。
