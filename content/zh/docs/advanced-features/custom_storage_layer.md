Clusterpedia 通过存储层来接入不同的存储组件（例如 MYSQL/PostgreSQL, Memroy, ES）

Clusterpedia 内置了两个存储层：
* 用于接入关系型数据库的 [internalstorage]() 存储层
* 基于内存的 [memory]() 存储层

尽管已经有了对关系型数据库和内存的支持，但是用户的需求往往是多变和复杂的，统一的存储层可能无法满足不同用户对于存储组件和性能的要求，于是 Clusterepdia 支持了一种通过插件的方式来接入用户自己实现的存储层，我们称这种方式为 `自定义存储层插件`，简称为 `存储插件`。

通过存储插件，用户可以做到以下事情：
* 使用任意的存储组件，例如 Elasticsearch，RedisGraph，ETCD，包括 MessageQueue 也没有问题
* 针对用户自己的业务优化资源的存储，和查询性能
* 针对存储组件来实现一些更高级的检索功能

Clusterpedia 也维护了以下存储层插件，用户可以根据需求选用：
* [Elasticsearch Storage](https://github.com/clusterpedia-io/elasticsearch-storage)

存储插件是通过 `Go Plugin` 的方式来接入到 Clusterpedia 组件中，相比 RPC 或者其他方式，这种方式可以在没有任何损耗的前提下提供非常灵活的插件接入。
> 关于 Go Plugin 对性能的影响可以参考 https://github.com/uberswe/goplugins

众所周知，Go Plugin 在开发和使用上都比较繁琐，不过 Clusterpedia 通过一些机制来巧妙的优化了存储插件的使用和开发，并且提供了 [clusterpedia-io/sample-storage](https://github.com/clusterpedia-io/sample-storage) 作为参考。

下面我们介绍如何开发和使用 `自定义存储层插件`

## 使用
我们先介绍如何在 Clusterpedia 组件二进制中设置存储插件，这样也可以更好的理解存储插件和 Charts 是如何工作的

存储插件实际是一个 *.so* 后缀的动态链接库，Clusterpedia 在启动时加载存储插件，并根据指定的存储插件名称来使用具体的存储插件。

我们以 [clusterpedia-io/sample-storage](https://github.com/clusterpedia-io/sample-storage) 为例，构建出一个存储插件二进制
```bash
$ git clone --recursive https://github.com/clusterpedia-io/sample-storage.git && cd sample-storage
$ make build-plugin
```

使用 `file` 命令查看存储插件信息
```
$ file ./plugins/sample-storage-layer.so
./plugins/sample-storage-layer.so: Mach-O 64-bit dynamically linked shared library x86_64
```

`ClusterSynchro Manager` 和 `APIServer` 两个组件需要设置存储插件，需要通过以下环境变量和命令参数需要设置：
* `STORAGE_PLUGINS` 环境变量设置插件所在目录，Clusterpedia 会将该目录下所有插件都加载到组件中
* `--storage-name` 插件命令参数，设置存储层名称
* `--storage-config` 插件命令参数，设置存储层配置

同样以 [clusterpedia-io/sample-storage](https://github.com/clusterpedia-io/sample-storage) 为例，本地构建 Clusterpedia 组件
```bash
$ # 在 sample-storage 目录
$ make build-components
$ ls -al ./bin
drwxr-xr-x   6 icebergu  staff       192 11  7 11:17 .
drwxr-xr-x  16 icebergu  staff       512 11  7 11:15 ..
-rwxr-xr-x   1 icebergu  staff  90707488 11  7 11:15 apiserver
-rwxr-xr-x   1 icebergu  staff  91896016 11  7 11:16 binding-apiserver
-rwxr-xr-x   1 icebergu  staff  82769728 11  7 11:16 clustersynchro-manager
-rwxr-xr-x   1 icebergu  staff  45682000 11  7 11:17 controller-manager
```
> 关于存储插件与 Clusterpedia 组件的构建可以查看 [开发自定义存储层插件]()

本地运行 APIServer 和 ClusterSynchro Manager 组件
```bash
$ STORAGE_PLUGINS=./plugins ./bin/apiserver --storage-name=sample-storage-layer --storage-config=./config.yaml <... other flags>
$ STORAGE_PLUGINS=./plugins ./bin/clustersynchro-manager --storage-name=sample-storage-layer --storage-config=./config.yaml <... other flags>
```

### 存储插件镜像 + Helm Charts
我们在真正使用`存储插件`时，并不需要重新构建和设置环境变量，只需要配合 Charts 设置相应的存储插件镜像即可。

Clusterpeida 已经提供了四个 Charts:
* [clusterpedia](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia) 使用 internalstorage 存储层的 Chart
* [clusterpedia-core](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-core) 支持配置任意存储层的 Chart，通常作为子 Chart 来使用
* [clusterpedia-mysql](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-mysql) 使用 MySQL 作为存储组件的 Chart，基于 clusterpedia-core 实现
* [clusterpedia-postgresql](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-postgresql) 使用 PostgreSQL 作为存储组件的 Chart，基于 clusterpedia-core 实现

用户可以直接使用 clusterpedia-mysql 或者 clusterpedia-postgresql，这时并不需要关心存储插件的配置，在 Charts 内会为用户自动设置。

我们介绍如何单独使用 clusterpedia-core，在 [开发自定义存储层插件]() 会介绍如何基于 clusterpedia-core 实现具体存储组件的 Chart。

## 开发自定义存储层插件
