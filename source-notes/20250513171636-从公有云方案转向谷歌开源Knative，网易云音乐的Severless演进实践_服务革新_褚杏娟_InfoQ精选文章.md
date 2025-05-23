---
type: source-note
title: 从公有云方案转向谷歌开源Knative，网易云音乐的Severless演进实践_服务革新_褚杏娟_InfoQ精选文章
id: 20250513170536
created: 2025-05-13T17:16:36
source:
  - web
url: https://www.infoq.cn/article/ma6mqquhalklxysan9do
tags:
  - source-note
  - cloud-native/serverless/knative
processed: false
archived: false
---
**

**

**

**

![从公有云方案转向谷歌开源Knative，网易云音乐的Severless演进实践](https://static001.infoq.cn/resource/image/83/ec/836b8a3fccc3188e7c2593944e6fd5ec.jpg)

云主机时代，资源焦虑几乎普遍存在。突增的巨大任务量、短时间突然调集使用大量的计算资源等类型的业务需求越来越多，企业不愿为了应对短暂的流量高峰买本地资源，对服务和扩缩容进行解耦，并接管过自动扩缩容任务的 Serverless 进入大众视野。

Serverless 自带“降本增效”基因，特点之一就是可以缩容到零之后再按资源使用情况收费，这自然吸引了大量企业使用， [网易云音乐](https://www.infoq.cn/article/mBgvbc3bQWAbavwmBhXR "xxx") 便是其中之一。

最早，网易云音乐主要使用云厂商的 FaaS 产品。随着 Serverless 社区的发展，2020 年，网易云音乐开始关注到 Google 开源的 Knative 项目，到了 2021 年 5 月，团队决定优先利用内部的私有云资源，在满足业务的异步处理事件以及弹性扩缩容需求的前提下，通过在线和离线服务的混合部署，提升系统资源利用率，同时完成降本增效目的。

于是，在做了简单的 POC 测试并与业务沟通后，网易云音乐便协同网易数帆云原生团队面向音视频处理，打造了基于 [Knative](https://www.infoq.cn/article/rDL06CdUNEPXtPLzT-3O "xxx") 的 Serverless 解决方案。

## 如何做技术选型

网易云音乐每天都有数十万的歌曲入库，曲库团队则需要对这部分歌曲做准实时的加工处理、理解分析（包括歌曲转码、副歌识别、特征分析等），相关处理结果用于歌曲播放、VIP 歌曲试听等业务场景。这类业务的特点就是弹性特别大，任务时多时少，多的时候甚至要对大量存量歌曲数据进行重新计算。这就对资源交付方式提出了新的要求。

按照网易云音乐在云主机时代的使用经验，传统的资源交付方式主要存在以下几个问题：

• 弹性效率低下：大型活动业务扩容时，各个角色如应用运维、机房等深度耦合，进行一次大型活动需要非常长的准备时间。

• 计算焦虑：由于规模问题，机房计算资源没办法实现在活动期间的快速资源弹性需求，因此常常需要准备很多闲置资源。

• 运维繁琐：扩容变更时，很多是以工单、人工化为主的低效过程，无论效率还是质量都不尽如人意。

• 成本问题：总体 CPU 等资源利用率不高，小于 20%，缺乏自动化的管理和调度能力，资源无法得到充分利用。

• 稳定性：应用发生故障后，无法自动重新拉起或重新调度，核心业务的服务质量很难得到保障。

虽然基于 Kubernetes 以及生态里的很多创新云原生解决方案，上述棘手问题得到了一定程度的解决，但 Serverless 的解决方案相对来说更加高效易用。 [Serverless](https://www.infoq.cn/article/vHCG1pJpsLapBBMvbvZM "xxx") 向业务提供了语言无关、框架无关的研发模式，通过自动化 Metric、自动扩缩容等手段让业务聚焦业务逻辑，无需关注周边与资源扩缩、没有服务器管理，降低了程序生命周期中的大量运维成本。

当前，Serverless 大概有三个技术方向：Serverless 容器服务、Serverless 应用托管和函数计算（FaaS）。

使用 Serverless 容器服务的用户不需要维护 Kubernetes 集群的计算节点，系统根据服务使用的 pod 数量进行计费，但 Serverless 容器服务并不能提供完备的周边配套设施；Serverless 应用托管则会包含应用生命周期的管理、CI/CD、发布策略，蓝绿或者灰度发布功能等，用户只需将服务部署后就能坐享应用托管所提供的基础能力；函数计算的抽象程度更高。对于 Python 等解释类语言，开发者使用 FaaS 将代码片段上传后，函数计算的底层便可快速将服务对外部署，从而实现对外服务。

在云原生团队看来，Serverless 应用托管或 FaaS 平台相对来说是更好的选择，因为业务不只需要弹性伸缩功能，还要解决 CI/CD、发布策略、消息引擎等问题，做更好的开发封装。只有涵盖了这些周边配套服务，才能将开发的心智负担降至最低。方向确认后，具体怎么做呢？

容器镜像需要基于编程语言的定制化编译和构建，从而生成二进制的文件，最后再经过 Dockerfile 将其构建成容器。但是，曲库团队内部有多种语言所开发的算法，很难用通用的流水线进行容器镜像构建。因此，曲库团队内部只要求底层 Serverless 平台或 FaaS 平台能够接受 Dockerfile 即可，具体 Dockerfile 怎么写则由其内部自行消化。

因此，云原生团队的任务就是将曲库团队上传的 Dockerfile 进行镜像构造。云原生团队为此进行了一次全面调研，内容包括消息引擎对上下游解耦及弹性扩容的需求、相关开源软件（Knative、OpenFaas、Fission、Nuclio 等）与需求的匹配度对比，最后确定基于 Knative Serving 做动态扩缩容、基于 Knative Eventing 构建事件处理框架。

  

![](https://static001.geekbang.org/infoq/83/83e9a3022bab0e0a0d307a6da4f6f18e.png)

  

网易云音乐 Serverless 事件处理框架图

在如何进行技术选型上，网易数帆云原生架构师闫东晓表示主要有三点需要考虑。第一，产品一定要能够满足业务场景和应用场景需求；第二，关注产品背后的支持情况，比如 Knative 有谷歌、IBM 等大企业加持，OpenShift 背后有 RedHat 支持；第三，产品具有易用性，通常易用性是落地时团队要帮助业务解决的问题，但如果项目足够稳定，就不需要改变底层框架。

在众多开源软件中，Knative 的扩展性较好、可以选择消息引擎，并且生产和消费的客户端可以以插件的形式嵌入到 Serverless 系统中。因此，云原生团队最后选择基于 Knative 对每个实例或创建的 Knative Service（类似 Kubernetes 的 Deployment）进行动态扩缩容。

当时的 Knative 本身还处于快速迭代阶段，没有稳定的版本，网易云音乐使用的还是 0.20、0.21 版本。

2021 年上云之后，网易云音乐开始使用事件驱动架构。这次迁移期间，云原生团队还在 Knative Eventing 事件框架中内嵌了一个插件（Knative 之中包含 Knative Serving 和 Knative Eventing 两个项目），将消息引擎 Kafka 也集成到了 Serverless 平台之中。业务只需要在 8080 端口接收通过 Knative Eventing 事件框架转发的请求，并通过 Kafka 触发消息即可实现事件驱动。

具体来讲，业务将支持 Knative Eventing 格式的事件请求通过暴露的 URL 发送到接口，再由接口将消息转发到消息引擎，系统层面在监听到事件触发后会消费 Kafka 的消息，最后再将其转发给后端算法进行处理。

自此，网易云音乐拥有了一个异步事件处理框架，在偏向离线的场景中可以慢慢地消费消息，从而确保私有云底层的有限资源能得到合理、充分地使用。

这是一种通用技术，要求启动的服务不依赖私有云节点，不能在宿主机上的某些路径下存在文件等依赖形式，否则会无法弹出导致启动失败。但如果所有依赖均在容器镜像内部，或者可以通过运行时动态地请求依赖方获取信息，那么就可以应用这种弹性能力。

## 迁移后，需要解决哪些问题？

冷启动是 Serverless 使用时被重点考量的点。影响启动速度的因素有很多，比如，容器镜像大小不同，pod 的启动速度也不同。部分厂商通过预先启动部分 pod 的方式来解决冷启动问题，但网易云音乐没有这么做。云原生团队使用了更通用的解决方案，比如 Dockerfile 采用多阶段构建、P2P 加速容器镜像拉取速度等。

网易云音乐的应用场景偏离线、非实时，因此对负载均衡和并发控制的需求比较高。音视频算法每个 pod 可处理的并发度很低，理想情况是上游在下发请求时控制并发数量，确保每个 pod 都在处理自己能处理的并发请求。但是，数据链路上会有数据不均衡的情况，经过队列的请求会超过 pod 可处理的并发数量上限，从而导致队列阻塞和其他 pod 空闲。

为此，云原生团队调整了 Knative 内部的负载均衡算法策略，从默认的 Round Robin 改为 Least Request，将请求发给并发处理数最少的 pod，让每个 pod 都有任务。

另外业务对稳定性要求也很高，而业务稳定性主要体现在对上游并发的控制上。业务将服务请求全部发送到消息队列后，如果将消息全部分发给底层服务处理，那么将扩容出非常多 pod；如果 pod 与在线应用在同一个 node 上，则势必会影响在线应用的稳定性。因此，除了 Knative 本身所提供的服务外，云原生团队还收集业务指标并提供监控告警功能，来给业务信心。

通过与业务的需求沟通，云原生团队利用 Serverless 暴露出的数据链路指标信息形成定制的可视化看板，其中包括监控告警、扩缩容频率、每个 pod 的负载情况、推送消息的消费情况等业务基础信息，此外也有 Serverless 内部运维的巡检监控，如 CPU、内存的利用率，消费队列消费延时情况、业务化扩缩容实现等。

当监控效果不达预期时，云原生团队则需要调整或借鉴其他优化手段做提升。值得注意的是，这些监控指标收集都是使用的基础 Kubernetes 系列开源产品，并不是 Serverless 独有的。Serverless 是作为整个架构部分的存在，需要与其它产品配合使用。

在调优方面，业务研发可以自行登录容器查看进程信息，也可以通过日志收集的方式查看。调试方面则使用了云主机时代的远程调试方法，这种方式在容器化时代依旧可用。

为了完成“最后一公里”的交付，云原生团队在网易开源的云原生应用交付平台 Horizon 上交付了一个部署模板，曲库团队基于 Horizon 平台填写数据表单，云原生团队负责模板化实例生成。Horizon 平台（开源地址： [https://github.com/horizoncd](https://github.com/horizoncd) ）通过引入模板系统解决了各种应用负载标准化的问题，支持 Deployment、Argo Rollout、Knative 等负载，Serverless 平台则复用了 Horizon 的部分基础能力，进而为业务提供动态扩缩容和事件处理框架能力。

通过结合业务进行探索和迭代，网易云音乐用了一年多的时间基于 Knaitve 构建了相对完善的 Serverless 平台：

- 多语言的构建方式：包括 Dockerfile 、JAVA、Golang、Node、Python 等。
- 多场景：支持弹性在线应用和弹性数据处理，支持同步调用模式和异步调用模式。
- 丰富的发布策略：支持蓝绿发布和基于流量的灰度发布，确保业务的无损发布。
- 自动扩缩容：根据业务并发以及 QPS、任务量等实现秒级自动扩缩容。
- 全链路监控：全链路的采集指标、采集日志，自动将数据整合到应用监控。
- 丰富的触发器：除了支持 HTTP、还支持网易内部的 Kafka、Nydus 队列作为 Serverless 触发器进行数据处理。
- 无限容量：通过混合云、混合部署等方式，快速、自动地通过 ECI 等方式弹到阿里云、AWS 等公有云厂商。

## 落地效益如何？

“对于企业来说，如果一开始使用的是私有云，那么在既有 IT 成本的前提下，Serverless 只是提升内部资源的利用率。但如果前提是公有云，那么只要能保证容器不依赖于主机环境，那么在解决信息安全、日志、指标监控等问题的前提下，Serverless 是一定可行的。”闫东晓表示。

目前，网易云音乐内部大量使用 Serverless 平台的场景包括音视频分析、AI 推理分析、前端 SSR、弹性日志 ETL 等。Serverlesss 通过与在线业务混合部署的方式，大大提升了机房资源的利用率，峰值时超过了 50%，资源整体占比达到 20%左右。

网易云音乐的 Node 负载有波峰、波谷之分，云原生团队希望在波峰时段减少 Serverless 的使用，并在凌晨 2-8 点左右提升资源利用率，运行 Serverless 的非实时任务。其中，波峰时段主要是内部私有云在线服务，这也是整个 Kubernetes 资源利用率的波峰。

如今，网易云音乐的私有云上已经部署了超过 500 个 Serverless 应用，高峰期会使用 1 万多虚拟核心。从内部 Node 级别的资源利用率来看，有 20%的 CPU 核心供给了 Serverless 应用使用，通过在线离线混合部署，在不扩容机器增加成本的情况下，基本满足了业务对底层计算资源的诉求。

网易云音乐还可以做到优先使用自有机房计算资源，直到饱和时再使用公有云上的计算资源，比如将服务弹出到阿里云的 ECI（弹性容器实例）上进行临时的计算辅助，并在执行完成后将其缩容，从而完全解决资源焦虑，大大提高资源交付效率。需要注意的是，这是一种临时调用，而非将服务固定在私有云和公有云上混合使用。

在接入 Serverless 平台两年以来，曲库音视频平均使用资源的 CPU 核数日平均峰值 5000 核，日平均谷值 3000 核。同时，一部分算法服务还借助公有云的 Spot 弹性资源和包月资源，利用竞价模式，持有弹性的 GPU，快速申请、快速释放。云原生团队的调研显示，即使是简单的每天修改副本数，业务对这些弹性扩缩容手段的好感度也非常高。

另外在运维方面，底层运维的成本并没有因为使用 Serverless 而增加，运维人员的实际操作量减少，将精力更多放在了 Kubernetes 的资源是否能满足业务需求上。

不过，闫东晓提醒道，对于业务研发而言，云原生团队可以将同一类的工具链封装得更稳定、使用更简单，这时 Serverless 使用效率较高，但是对于非同一类工具链，如算法等无法抽象出 CI 流水线的，收益就比较有限。

## 要不要用 Serverless

从云厂商产品为主到基于开源产品二次开发，网易云音乐的 Serverless 架构虽然更加贴合内部应用场景，但也需要花精力紧跟社区迭代。闫东晓表示，Serverless 也非“银弹”，本身自带如冷启动方面启动慢、销毁时造成客户端异常、对在线类服务不太能友好等问题。另外，在既有成本的情况下，固定副本数要比弹性扩缩容要好。

对于想要接入 Serverless 的企业，闫东晓建议可以从降本增效的角度，或者自有机房或私有云的系统资源利用率角度，看是否有偏离线的计算密集型业务。“一些离线应用往往会在短时间内需要大量的资源，这种需求往往也是一次性的。此时，可以考虑使用 Serverless 提升系统利用率。”

对于使用公有云的企业，如果直接将所有服务全部迁移到 Serverless 架构上，则更需要考虑各种风险，比如扩缩容过程中的冷启动问题、服务启停是否会影响业务、缩容时 pod 的销毁是否会同时关闭未处理完成的用户请求、扩容时 pod 创建是否够快、是否会导致扩容时间内的请求高延迟等。

企业如果考虑使用云厂商产品，闫东晓表示需要了解云厂商的技术是否封闭、是否跟随社区前进，否则之后做厂商切换、产品切换时都会比较麻烦。尤其如果云厂商的 Serverless 产品在底层没有统一标准，那么平滑迁移必然会带来成本问题。

如果内部只是将固定副本数的普通云主机迁移至 Kubernetes，那么对于封装流水线和接口的方式，业务层感知不到底层上云前后的差别，也不需要太多知识。但如果是使用微服务、选择自身技术栈的情况，那么使用方需要能提供 Dockerfile、自行将容器封装运行，这就需要具备容器、Kubernetes 方面的知识，否则用起来会感到困惑。

## 结束语

网易云音乐的 Serverless 应用还在继续，比如网易云音乐考虑在事件框架中引入 RocketMQ、调度方面会引入定时并发控制，以及充分利用硬件在波谷时段的资源等。总的来说，网易云音乐 Serverless 的落地还是围绕“降本增效”进行更细化的工作。

当然，对于整个 Serverless 行业来说，未来也还有很多路要走。 Serverless 能否借助当下企业对降本增效需求的契机得到进一步发展，我们也将拭目以待。

  

**延伸阅读：**

  

[15 年了，我们到底怎样才能用好 Serverless](https://www.infoq.cn/article/6SUNgEE6BX1xtDkRXHME "xxx")  

[备受云厂商们推崇的 Serverless，现在究竟发展到什么水平了？](https://www.infoq.cn/article/vHCG1pJpsLapBBMvbvZM "xxx")  

[落地 4 年，工商银行如何进行 Serverless 架构迭代](https://www.infoq.cn/article/uhgZm4TRELfXgizNFtGM "xxx")  

4228

** [服务革新](https://www.infoq.cn/topic/1200) [开源](https://www.infoq.cn/topic/opensource) [云原生](https://www.infoq.cn/topic/CloudNative) [最佳实践](https://www.infoq.cn/topic/best-practices) [技术选型](https://www.infoq.cn/topic/technology-selection) [企业动态](https://www.infoq.cn/topic/%20industrynews) [性能优化](https://www.infoq.cn/topic/1213) [音视频（前端）](https://www.infoq.cn/topic/1214) [音视频（后端）](https://www.infoq.cn/topic/1178) [编程语言](https://www.infoq.cn/topic/programing-languages) [框架](https://www.infoq.cn/topic/1180) [微服务](https://www.infoq.cn/topic/microservice) [多云/混合云](https://www.infoq.cn/topic/1182) [在离线混部](https://www.infoq.cn/topic/1193) [银行](https://www.infoq.cn/topic/1169) [汽车](https://www.infoq.cn/topic/1172) [行业深度](https://www.infoq.cn/topic/1168) [云计算](https://www.infoq.cn/topic/cloud-computing) [管理/文化](https://www.infoq.cn/topic/1204) [架构](https://www.infoq.cn/topic/architecture) [大数据](https://www.infoq.cn/topic/bigdata) [大前端](https://www.infoq.cn/topic/1208) [后端](https://www.infoq.cn/topic/1174)

![](https://static001.infoq.cn/resource/image/e2/d9/e2bd4c587716b3bc7efbea500e38b3d9.jpg)

## 评论

暂无评论