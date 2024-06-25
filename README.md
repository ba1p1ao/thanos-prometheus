# 基于 Thanos 架构的高可用 Prometheus 监控系统


## 一、调研
  由于公司目前使用的 influxdb 社区版不支持高可用，所以希望使用 prometheus 去代替，需要找一个 prometheus 高可用的方案，来替换现在的 influxdb
  找到了目前使用较多的 thanos 架构来实现 prometheus 监控

## 二、Thanos 架构
Thanos 是一个基于 Prometheus 实现的监控方案，其主要设计目的是解决原生 Prometheus 上的痛点，并且做进一步的提升，主要的特性有：全局查询，高可用，动态拓展，长期存储。
这是官方给出的架构图：

![alt text](images/thanos架构图.png)

Thanos 主要由如下几个特定功能的组件组成：

Thanos Query: 实现了 Prometheus API，将来自下游组件提供的数据进行聚合最终返回给查询数据的 client (如 grafana)，类似数据库中间件。
Thanos Sidecar: 连接 Prometheus，将其数据提供给 Thanos Query 查询，并且/或者将其上传到对象存储，以供长期存储。
Thanos Store Gateway: 将对象存储的数据暴露给 Thanos Query 去查询。
Thanos Ruler: 对监控数据进行评估和告警，还可以计算出新的监控数据，将这些新数据提供给 Thanos Query 查询并且/或者上传到对象存储，以供长期存储。
Thanos Compact: 将对象存储中的数据进行压缩和降低采样率，加速大时间区间监控数据查询的速度。
  从使用角度有两种方式去使用 Thanos，sidecar 架构和 receiver 架构：

- sidecar 架构图：

![alt text](images/sidecar架构图.png)

Thanos Sidecar 组件需要和 Pormetheus 实例一起部署，用于代理 Thanos Query 组件对本地 Prometheus 的数据读取，允许 Query 使用通用、高效的 StoreAPI 查询 Prometheus 数据。第二是将 Prometheus 本地监控数据通过对象存储接口上传到对象存储中，这个上传是实时的，只要发现有新的监控数据保存到磁盘，会将这些监控数据上传至对象存储。

Thanos Sidecar 组件在 Prometheus 的远程读 API 之上实现了 Thanos 的 Store API，这使得 Query 可以将 Prometheus 服务器视为时间序列数据的另一个来源，而无需直接与它的 API 进行交互。

因为prometheus 每 2 小时生成一个时序数据块，Thanos Sidecar 会每隔 2 小时将这个块上传到一个对象存储桶中。这样 Prometheus 服务器就可以以相对较低的存储空间运行，同时通过对象 存储提供历史数据，使得监控数据具有持久性和可查询性。但是这样并不意味着 Prometheus 可以完全无状态，因为如果 Prometheus 崩溃并重新启动，我们将失去大约 2 个小时的指标数据，所以 Prometheus 在实际运行中还是需要持久性磁盘的。
- receiver 架构图：

![alt text](images/reveiver架构图.png)

Thanos Receiver 实现了 Prometheus 远程写 API，它构建在现有的 Prometheus TSDB 之上，并保持其实用性，同时通过长期存储、水平可伸缩性和下采样扩展其功能。Prometheus 实例被配置为连续地向它写入指标，然后 Thanos Receiver 默认每 2 小时将时间序列格式的监控数据块上传到一个对象存储的桶中。Thanos Receiver 同样暴露了 Store API，以便 Thanos Query 可以实时查询接收到的指标。

最后使用的是 sidecar 架构
## 三、部署
  通过上面架构的对比，最终使用了 sidecar 架构

1. 准备对象存储配置（thanos-objectstorage-secret.yaml）
    - 比如使用 aws 的 S3 存储
2. 部署 Prometheus + sidecar （thanos-prometheus.yaml）
    - Prometheus 使用 StatefulSet 方式部署，挂载数据盘以便存储最新监控数据。
    - 由于 Prometheus 副本之间没有启动顺序的依赖，所以 podManagementPolicy 指定为 Parallel，加快启动速度。
    - 为 Prometheus 绑定足够的 RBAC 权限，以便后续配置使用 k8s 的服务发现 (kubernetes_sd_configs) 时能够正常工作。
    - 为 Prometheus 创建 headless 类型 service，为后续 Thanos Query 通过 DNS SRV 记录来动态发现 Sidecar 的 gRPC 端点做准备 (使用 headless service 才能让 DNS SRV 正确返回所有端点)。
    - 使用两个 Prometheus 副本，用于实现高可用。
    - 使用硬反亲和，避免 Prometheus 部署在同一节点，既可以分散压力也可以避免单点故障。
    - Prometheus 使用 --storage.tsdb.retention.time 指定数据保留时长，默认15天，可以根据数据增长速度和数据盘大小做适当调整(数据增长取决于采集的指标和目标端点的数量和采集频率)。
    - Sidecar 使用 --objstore.config-file 引用我们刚刚创建并挂载的对象存储配置文件，用于上传数据到对象存储。
    - 通常会给 Prometheus 附带一个 quay.io/coreos/prometheus-config-reloader 来监听配置变更并动态加载，但 thanos sidecar 也为我们提供了这个功能，所以可以直接用 thanos sidecar 来实现此功能，也支持配置文件根据模板动态生成：--reloader.config-file 指定 Prometheus 配置文件模板，--reloader.config-envsubst-file 指定生成配置文件的存放路径，假设是 /etc/prometheus/config_out/prometheus.yaml ，那么 /etc/prometheus/config_out 这个路径使用 emptyDir 让 Prometheus 与 Sidecar 实现配置文件共享挂载，Prometheus 再通过 --config.file 指定生成出来的配置文件，当配置有更新时，挂载的配置文件也会同步更新，Sidecar 也会通知 Prometheus 重新加载配置。另外，Sidecar 与 Prometheus 也挂载同一份 rules 配置文件，配置更新后 Sidecar 仅通知 Prometheus 加载配置，不支持模板，因为 rules 配置不需要模板来动态生成。

3. 给 Prometheus 准备配置 (thanos-prometheus-config.yaml)

    - 配置文件，主要配置了 cadvisor 容器指标和自动发现 telegraf 的指标
    - Prometheus 实例采集的所有指标数据里都会额外加上external_labels 里指定的 label，通常用 cluster 区分当前 Prometheus 所在集群的名称，我们再加了个 prometheus_replica，用于区分相同 Prometheus 副本（这些副本所采集的数据除了 prometheus_replica 的值不一样，其它几乎一致，这个值会被 Thanos Sidecar 替换成 Pod 副本的名称，用于 Thanos 实现 Prometheus 高可用）

4. 安装 Query （thanos-query.yaml）

    - 因为 Query 是无状态的，使用 Deployment 部署，也不需要 headless service，直接创建普通的 service。
    - 使用软反亲和，尽量不让 Query 调度到同一节点。
    - 部署多个副本，实现 Query 的高可用。
    - --query.partial-response 启用 Partial Response，这样可以在部分后端 Store API 返回错误或超时的情况下也能看到正确的监控数据(如果后端 Store API 做了高可用，挂掉一个副本，Query 访问挂掉的副本超时，但由于还有没挂掉的副本，还是能正确返回结果；如果挂掉的某个后端本身就不存在我们需要的数据，挂掉也不影响结果的正确性；总之如果各个组件都做了高可用，想获得错误的结果都难，所以我们有信心启用 Partial Response 这个功能)。
    - --query.auto-downsampling 查询时自动降采样，提升查询效率。
    - --query.replica-label 指定我们刚刚给 Prometheus 配置的 prometheus_replica 这个 external label，Query 向 Sidecar 拉取 Prometheus 数据时会识别这个 label 并自动去重，这样即使挂掉一个副本，只要至少有一个副本正常也不会影响查询结果，也就是可以实现 Prometheus 的高可用。同理，再指定一个 rule_replica 用于给 Ruler 做高可用。
    - --store 指定实现了 Store API 的地址(Sidecar, Ruler, Store Gateway, Receiver)，通常不建议写静态地址，而是使用服务发现机制自动发现 Store API 地址，如果是部署在同一个集群，可以用 DNS SRV 记录来做服务发现，比如 dnssrv+_grpc._tcp.prometheus-headless.thanos.svc.cluster.local，也就是我们刚刚为包含 Sidecar 的 Prometheus 创建的 headless service (使用 headless service 才能正确实现服务发现)，并且指定了名为 grpc 的 tcp 端口，同理，其它组件也可以按照这样加到 --store 参数里；如果是其它有些组件部署在集群外，无法通过集群 dns 解析 DNS SRV 记录，可以使用配置文件来做服务发现，也就是指定 --store.sd-files 参数，将其它 Store API 地址写在配置文件里 (挂载 ConfigMap)，需要增加地址时直接更新 ConfigMap (不需要重启 Query)

5. 安装 Store Gateway （thanos-store.yaml）

    - Store Gateway 实际也可以做到一定程度的无状态，它会需要一点磁盘空间来对对象存储做索引以加速查询，但数据不那么重要，是可以删除的，删除后会自动去拉对象存储查数据重新建立索引。这里我们避免每次重启都重新建立索引，所以用 StatefulSet 部署 Store Gateway，挂载一块小容量的磁盘(索引占用不到多大空间)。
    - 同样创建 headless service，用于 Query 对 Store Gateway 进行服务发现。
    - 部署两个副本，实现 Store Gateway 的高可用。
    - Store Gateway 也需要对象存储的配置，用于读取对象存储的数据，所以要挂载对象存储的配置文件。

6. 安装 Ruler （thanos-rule.yaml）

    - Ruler 是有状态服务，使用 Statefulset 部署，挂载磁盘以便存储根据 rule 配置计算出的新数据。
    - 同样创建 headless service，用于 Query 对 Ruler 进行服务发现。
    - 部署两个副本，且使用 --label=rule_replica= 给所有数据添加 rule_replica 的 label (与 Query 配置的 replica_label 相呼应)，用于实现 Ruler 高可用。同时指定 --alert.label-drop 为 rule_replica，在触发告警发送通知给 AlertManager 时，去掉这个 label，以便让 AlertManager 自动去重 (避免重复告警)。
    - 使用 --query 指定 Query 地址，这里还是用 DNS SRV 来做服务发现，但效果跟配 dns+thanos-query.thanos.svc.cluster.local:9090 是一样的，最终都是通过 Query 的 ClusterIP (VIP) 访问，因为它是无状态的，可以直接由 K8S 来给我们做负载均衡。
    - Ruler 也需要对象存储的配置，用于上传计算出的数据到对象存储，所以要挂载对象存储的配置文件。
    - --rule-file 指定挂载的 rule 配置，Ruler 根据配置来生成数据和触发告警。

7. 准备 Ruler 配置文件 (thanos-rule-config.yaml)

    - 只是一个实例，后续按需求修改，和prometheus的rule格式一样

8. 安装 Compact （thanos-compact.yaml）

    - Compact 只能部署单个副本，因为如果多个副本都去对对象存储的数据做压缩和降采样的话，会造成冲突。
    - 使用 StatefulSet 部署，方便自动创建和挂载磁盘。磁盘用于存放临时数据，因为 Compact 需要一些磁盘空间来存放数据处理过程中产生的中间数据。
    - --wait 让 Compact 一直运行，轮询新数据来做压缩和降采样。
    - Compact 也需要对象存储的配置，用于读取对象存储数据以及上传压缩和降采样后的数据到对象存储。
    - 创建一个普通 service，主要用于被 Prometheus 使用 kubernetes 的 endpoints 服务发现来采集指标(其它组件的 service 也一样有这个用途)。
    - --retention.resolution-raw 指定原始数据存放时长，--retention.resolution-5m 指定降采样到数据点 5 分钟间隔的数据存放时长，--retention.resolution-1h 指定降采样到数据点 1 小时间隔的数据存放时长，它们的数据精细程度递减，占用的存储空间也是递减，通常建议它们的存放时间递增配置 (一般只有比较新的数据才会放大看，久远的数据通常只会使用大时间范围查询来看个大致，所以建议将精细程度低的数据存放更长时间)

9. 指定 Query 为数据源
    - 查询监控数据时需要指定 Prometheus 数据源地址，由于我们使用了 Thanos 来做分布式，而 Thanos 关键查询入口就是 Query，所以我们需要将数据源地址指定为 Query 的地址


