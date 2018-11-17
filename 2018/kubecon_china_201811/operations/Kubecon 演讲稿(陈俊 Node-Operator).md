# Kubecon 演讲稿(陈俊 Node-Operator)

注：

- 节点： Node
- 监听： Watch
- 自定义资源实例： CR, Custom Resource, the instance of CustomResourceDefinition
- Machine: Machine CR,  the instance of Machine CRD
- Cluster: Cluster CR, the instance of Cluster CRD

### 第一页

大家好，我叫陈俊，目前在蚂蚁金服专注 Sigma 核心组件的设计和开发工作（Sigma 是我们蚂蚁金服内部 对 Kubernetes 项目进行改造并扩展一些组件，符合蚂蚁金服需求的 Kubernetes 集群）。

今天我将和大家分享蚂蚁金服使用 Operator 模式管理 Kubernetes 集群和 Worker 节点。但是 Operator 模式不仅局限于去管理 Kubernetes 集群。其实，Kubernetes 组件里面绝大多数的组件都是 Operator 模式写的，比如各个 controller，其实调度器也是符合 Operator 模式的。

希望通过我今天的分享，能带给大家一些灵感，然后大家也可以尝试去使用 Operator 去管理复杂的应用，或者笼统的说，可以管理任何资源。好了，我们言归正传，回到我们的今天要分享的话题。



### 第二页

这个我今天分享内容的大纲。

首先我会简短的介绍一下蚂蚁金服的背景：包括蚂蚁金服的资源调度集群，集群规模，运维需求，以及在使用Operator 之前我们是怎管理和部署集群的。

然后，我会简单的介绍一下 Operator 模式。

介绍完 Operator 模式之后，我将介绍一下 Node-Operator 设计、架构和实现，以及它的一部分功能。

之后，会进行一个简短的介绍我们是如何通过 Operator 管理多个 Kubernetes 集群的。

最后如果时间充裕、条件允许的话，我会回答一下大家的提问。



### 第三页

在使用 kubernetes 作为蚂蚁金服的资源调度集群之前，我们使用了 Swarm 1.24 作为了我们的资源调度集群，我们内部的项目代号是 Sigma 2.0。

从今年的6月开始，我们开始立项 Sigma 3.1 项目，即将社区的 kubernetes 做一些改造、增加一些我们定制化的组件，去满足蚂蚁目前的需求，完成 资源调度集群 架构的升级，从而完成在蚂蚁金服落地 kubernetes 。

趁着这次的 资源调度集群 架构的升级，我学习了一些 kubernetes 的理念，然后做了一些 Sigma 2.0 运维系统的反思，将我们之前的运维集群的系统也做了一次架构升级和技术改造。这也就是今天我要分享给大家的内容。



### 第四页

在进入正题之前，我先给大家介绍一下蚂蚁金服进群的规模。

在蚂蚁的生产环境，大概有十个左右的机房，然后大概有二、三十个集群。然后每个集群的节点(Node)规模大概在 3k ~ 5k 左右，最大的集群规模达到了 1w 个节点。

在线下测试环境，我们几乎每时每刻都在创建一个新的 kubernetes 集群，这些短暂创建、销毁的集群是为集成测试服务的。当我们的git仓库每次提交，都会触发一个测试任务，任务的开始就是用最新的代码创建出一个 kubernetes 集群，然后开始跑我们的测试，跑完测试之后，立刻销毁 kubernetes 集群。

当然，我们线下也有大概 十个左右的稳定集群，每个集群大概在四五百节点(Node)的规模。



### 第五页

运维那么多的集群和节点，要保证稳定性和可靠性，其实有非常多的需求。在这里我大概列一些几个主要的需求：

1. 快速、方便地创建和删除 kubernetes 集群。正如我上面所说，我们组件的仓库的每一次提交，都会创建集群跑测试，然后再删除集群。线上环境也有大促临时去搭建一些云机房的集群，然后再大促结束时销毁集群的需求。
2. 然后，节点也是一样的，也要便捷的能在任何时候往集群里面添加节点和删除节点。这也许是最基本的要求了。
3. 能够可靠地升级 Master 组件 和 Node 组件。我们不希望在升级过程中，出现服务不可用的现象。
4. 金丝雀发布，这是英文直译的。在国内，称为灰度发布，或者蓝绿发布大家可能更明白一些。即挑选一批节点将某个组件的版本发布到新版本，观察一会儿新版本的组件是否稳定，然后再决定是否全面的发布新版本，或者直接回滚到老版本。
5. 最后，是能管理 Master 组件 和 Node 组件 的版本，能够快速找到历史版本进行回滚。



### 第六页

在没有采用 Operator 改造之前，我们的运维管控系统，是采用类似工单、工作流的思想去设计的。

当 SRE(或者说 PE) 需要发布升级、或者新建集群之类的，他会到我们运维管控系统的页面上面提交工单，或者说是任务。然后运维管控系统会将他提的工单类型，转化成相应需要做的运维操作。比如 SRE 提交了 Worker 节点的升级，我们的系统会将它变成由一个个任务组成的工作流。如右图所示，工作流由并行升级 节点1、节点2、节点N 的任务组成。每个节点升级的任务，又由串行的升级各个组件的子任务组成。

这里有个问题，就是在 SRE 提交升级的那一刻，我们那么大量的机器上面，总有几台因为各种原因是处于故障状态的。那么他的这个发布工单的结果很可能是失败的。



### 第七页

这种单纯的基于工单的运维管控系统大概有这么一些缺陷：

1. 很容易导致不一致性。可能某个 SRE 提交了一次发布，另一个 SRE 也提交了另一个有冲突的发布。理论上，我们应该以最后提交的为准，但是在一个并发量那么大的并发系统里面，我们很难做到这样。我们不想通过各种加锁来增加系统复杂度。
2. 故障不能感知。如我上文所讲，在发布的那一刻，总有几台机器是故障的。我们的运维管控系统需要订阅其它系统的消息，当节点故障恢复时，去触发提交一个新的工单，去做这个机器的该有的机器的恢复。然后，当一个健康节点发生故障时，我们的系统也无法自动地去恢复它，也需要其它系统的发现并触发任务。
3. 复杂的架构。完成我上述的工单发布系统需要各种中间件和数据库。这导致我们无法便捷的输出我们的技术和系统，然后自身运维和学习开发的成本也非常高。

综上，单纯的工单系统很难满足我们的需求。我们在基于工单的运维管控系统之上，需要开发一些其它的功能配合，才能将我们所希望的资源保持在我们想要的状态。



### 第八页

其实上面我提到了，工单系统(或者说任务执行系统)只能作为一个具体执行任务的系统，并不能一直将资源维持在我们想要的状态。如果我们需要维持资源在我们想要的状态，那么我们需要另一个监视系统，一直监听资源的变化，然后再次发起工单。

其实，工单系统和监视系统结合在一起，就差不多是我今天要讲的这种模式--Operator。

Operator 模式其实非常的简单，它一直不停的只做一件事情，这件事情由三个动作组成：

1. 监听资源的 期望状态 和 当前状态 ，一旦两者状态的其一发变化，就开始做第二个动作。
2. 分析 期望状态 和 当前状态 哪里不一样了，得出应该采取的行动
3. 最后，执行第二步的分析出来应该执行的行动，做完行动之后，资源就维持到了一致的状态。

三个动作做完之后，重复回到了第一个动作，一直循环。

Operator 模式的系统非常适合管理资源。用户只需要声明你期望的资源状态，比如 Pod，Deployment，我们只要开发一个对应的 Operator 去守护各自的资源即可。

### 第九页

Operator 模式的系统非常适合管理资源。比如 Kubernetes 的原生资源 Pod, Deployment, Daemonset 等，都是使用 Operator 模式去管理的。

Kubernetes 的组件，绝大多数的组件都是采用 Operator 模式。比如 controller-manager 里的各个 controller，scheduler。

Kubernetes 不但自身的核心组件使用 Operator，更是为开发者提供了基于 Kubernetes 开发 Operator 系统的便利。Operator，或者说基于 Kubernetes 的 Operator 具有如下一些非常突出的优点：

1. 声明式的系统。Operator 会一直尝试将资源保持到期望的状态。正如我上面所说。
2. 面向 kube-apiserver 编程。它带来的好处就更多了：
   1. Kubernetes 为我们带来了 CRD。Pod，ConfigMap 等是 kubernetes 原生的资源， kubernetes 同时提供给我们了非常便捷的扩展自定义资源方式。我们可以方便的扩展一种我们自定义资源。我们可以使用 kubectl 操作 Pod 一样，来操作我们的自定义资源。
   2. 然后，Operator 系统构建在 kubernetes 之上。Operator 能享受  kube-apiserver 提供的通用的接口访问自定义资源实例(CR)。比如用 kubectl 提交 自定义资源实例(CR) 到 kube-apiserver，我们还能用 informer 去监听我们的 自定义资源实例(CR)  的变化。
   3. 上文所说的功能，我们都能通过 Kubernetes 项目的 git 仓库快速实现。
3. 最后，Operator 本身架构非常简单。非常的敏捷和高效，可扩展性非常高。它部署在 kubernetes 集群里，运行时只依赖和访问 kube-apiserver 接口，所以，可以便捷的输出到任何地方。

### 第十页

介绍完 Operator 模式之后，我给大家介绍一下 Node-Operator 的架构。

Node-Operator 是一个简单的 pod，使用 deployment 部署到 kubernetes 集群里面。它的职责是负责 kubernetes 集群内所有 Node 所依赖的软件的管理，将节点的软件包一直维护到应有的版本和健康状态，软件包括 docker， CNI， kubelet 等，还包括一些它们的配置文件。

它会通过 kube-apiserver 一直监听 Node 和 Machine 资源。Node 是 kubernetes 原生的节点资源定义，Machine 就是我们刚才提到的 Machine CRD 的一个个实例。当  Node 和 Machine 资源发生变化时，Node-Operator 就会尝试通过节点的SSH通道去安装、升级、修复软件。

用户可以通过 kubectl 提交 或者 更新 Machine 到  kube-apiserver，表示需要扩容一个节点 或者 更新一个节点的操作。Node-Operator 都能每次获取到 用户 的操作，然后做出应有的动作。

Node 是用过 kubelet 自身上报的。但是，这些信息不能完全覆盖 Node 当前的状态信息。我们引入了NPD， NPD 是 Node Problem Detector 的简称，它会将 Node 上面的更丰富的信息上报到 kube-apiserver，这些信息最终会以 patch 的方式打在 Node 的 annotation或者condition里面。node-operator 也会根据这些非常丰富的信息，尝试去修复节点。举个例子，比如节点的磁盘到了预警水位，node-operator 会尝试去一些清理临时文件。

所以，node-operator 其实更像一个框架，我们不断的补充一些 diff/action 逻辑。而所有的信息获取，都是通过 kube-apiserver，kube-apiserver 像是一个所有信息的中转枢纽。各个组件的都可以把它所负责的信息都汇总到 kube-apiserver，然后其它组件去订阅这些信息并作出决策。所以我们非常喜欢面向 kube-apiserver 编程，之前的一些中间件好像没有存在的必要了。

### 第十一页

接下来，我给大家详细的将一部分 Node-Operator 的功能，和具体的步骤。

首先，我来介绍一下扩容的场景：

1. 在开始之前，我们有一个 kubelet 的版本，这是一个用于描述组件版本的CRD的实例。比如这里的例子是 kubelet-1.0.1，spec 里描述了如何去获取这个版本的binary。

2. 然后用户需要扩容一台 Node 到集群，那么他可以 kubectl 或者其它方式 将 Machine 提交 kube-apiserver。Machine 里面描述了 Node 的 SSH通道方式，kubelet 版本。

3. 提交到 kube-apiserver 之后，Node-Operator 会监听到这个变更，然后会根据现状得出应该去初始化这个 Node。

4. 之后 Node-Operator 就会通过SSH通道去安装 docker，CNI， kubelet 等。

5. 安装完成之后，会将结果写回到 Machine 的 Status 字段。

6. 然后这个我们的集群就有这个 Node 了



### 第十二页

这一页，会描述一下升级 kubelet 的过程。我们需要将 kubelet 从 1.0.1 升级到 1.0.2 版本：

1. 首先，我们新建一个 kubelet 1.0.2 版本的描述的版本，并提交到 kube-apiserver。
2. 然后，我们通过 kubectl 或者 其它方式，修改需要升级的 Machine。将 Machine 里代表 kubelet 版本的字段设置成 1.0.2 版本。
3. Node-Operator 会监听到这次 Machine 的修改，然后对比 Machine Status 和 Spec 得出，需要进行 kubelet 的升级操作。
4. 然后 Node-Operator 会将节点的 kubelet 做升级。
5. 升级完成之后，将结果写回到 Machine CR Status
6. 我们完成了这个节点的 kubelet 升级。



### 第十三页

这一页讲述了我们如果将一批 Node 的 kubelet 进行一次灰度发布（金丝雀发布）。这里稍微复杂一些，需要 operator 里的两个 controller 一起联合工作。

1. 首先，我们可以提交一个描述灰度发布的 CR，包含了 Node 筛选规则，灰度发布涉及的软件包的版本。

2. Node-Operator 负责灰度发布的controller 监听到了这此灰度发布 CR，然后将符合筛选规则的 Machine 标记上灰度发布的标志。

3. Node-Operator 负责 Node 软件包的 controller 监听到了 Machine CR 的变化，然后将 kubelet 升级到对应版本。然后回写 Machine CR Status 字段

4. Node-Operator 负责灰度发布controller 监听到了 Machine 的变化，然后将灰度发布的结果写回到 灰度发布 CR Status 字段。如上文所示，名为 my-node-1 的 Node已经灰度发布成功了。


 Node-Operator 的功能我们先介绍到这里，正如我上面所讲的。Node-Operator 更像一个框架，我们可以不停的扩展可以负责的软件包，以及各种故障恢复策略。



### 第十四页

下面我简单的介绍一下我们用于管理多个 kubernetes 集群的 Operator，它叫 kube-on-kube-operator。我们将 kubernetes 的 Master 组件，如 etcd，kube-apiserver 部署到另一个 kubernetes 集群中。

首先我介绍一下称呼这两种 kubernetes 集群的名词。一个我们叫 业务集群，业务集群 是我们提供给业务方的 Kubernetes 集群，用于真正运行业务应用的集群。另一个集群，我们称为 元集群，元集群 只用于运行 业务集群的 Master 组件。

Kube-on-Kube-Operator  的架构和 Node-Operator 如出一辙。只是它们所关心的 CRD 和 真正管理的资源不一样。

Kube-on-Kube-Operator 负责 监听 Cluster 的创建，Cluster 是用于描述 业务集群 的 CRD 的实例，然后将 Cluster 所代表的 业务集群 初始化完成，即完成 业务集群的 Master 组件的部署，业务集群 的 Master 组件都以 Deployment，Pod等形式运行于 元集群 内。之后 Kube-on-Kube-Operator  也会监听 Cluster 的更新，来完成 业务集群 的 Master 组件 的升级。

Kube-on-Kube-Operator 具有非常高的可扩展性，除了部署 Kubernetes 集群的 Master 组件，还能部署它所需的扩展组件，如 kubernetes-dashboard，core-dns 等，还有我们的 Node-Operator 当然也是扩展组件。



### 第十五页

最后，我们来看一下 Kube-on-Kube-Operator  和 Node-Operator 在一起协同工作的场景。在这个图中，我们的 Kube-on-Kube-Operator 只管理了两个 业务集群，实际的真实情况它会管理非常多的集群，正如我在最开头所说的我们需要的集群规模。

我来描述一下 新建一个 业务集群 的场景和会发生的事情：

首先， 元集群 已经部署上了 Kube-on-Kube-Operator 和 Node-Operator。当业务方需要一个新的 业务集群 时，我们可能使用 Node-Operator 扩容 3个 节点 到元集群，这3个 节点  是充当 业务集群 Master 节点 的角色。当然，这一步骤可能不存在，已有的  业务集群 Master 节点 完全可以跑多个 业务集群的 Master 组件。

然后我们提交一个 Cluster，Kube-on-Kube-Operator 就会将 业务集群 Master 组件部署好，那么 业务集群  就初始化好了。然后，同时，Kube-on-Kube-Operator 会在 业务集群 里面部署上 Node-Operator，那么我们就可以开始给 业务集群 扩容 Worker 节点了。



### 第十六页

最后，我来讲一下我们获取的成果。

1. 任何人都能快速的新建、升级一个 kubernetes 集群，用户只需关心 Cluster 怎么写。其实就是一个个 yaml 文件，这就是我们的管理那么多 Kubernetes 集群的统一接口。
2. 我们新建一个 kubernetes 集群只需要 一两分钟，当然这个只是说初始化好了 kubernetes 的 Master 组件，并不包含扩容Worker节点的时间。销毁 kubernetes 集群更是控制在秒级的。
3. 完全自动化的升级和回滚，我们只需要修改 yaml 并提交。
4. 最后，非常自动的集群和节点故障恢复。它是在是非常的酷。



### 第十七页

今天我要分享的基本都结束了，右边是我的微信二维码，大家有兴趣我们可以做进一步的交流。谢谢大家。



--EOF--