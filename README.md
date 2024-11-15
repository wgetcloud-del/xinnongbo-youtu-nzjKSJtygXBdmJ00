
之前有过一系列的文章来介绍我们的[多云系统](https://github.com)，多云系统可以将不同云厂商不同云账号不同云区域下的资源进行集中管理，通过多云系统，用户可以像使用一朵云一样使用多云，极大的提升了多云使用的便捷性。虽然如今已经步入云时代，多云的使用十分广泛，但IDC机房仍然存在，不少企业也都还是IDC机房托管的模式，尤其是一些传统行业或是金融机构等。鉴于此，我们对多云系统进行了全面的升级，升级成了全新的资产系统，不仅可以管理多云资源，同时还能够接入IDC机房资源，实现云上云下的统一管理


资产系统针对资源的纳管提供了三种方式，分别是：多云自动同步、Agent主动上报和手工添加，这篇文章将详细介绍下这三种方式的设计、用法和区别。大概的图示如下：


![](https://static.ops-coffee.cn/static/images/2024/1105.01.png)


这里边有个**资源池**的概念，在之前[「多云系统之资源管理」](https://github.com):[楚门加速器](https://shexiangshi.org)的文章里介绍过，资源池是个虚拟的池子，用来**暂存**云上同步下来的资源，为什么说是暂存，主要是因为资源池内的资源都是没有项目归属的，属于无人认领或暂未认领的无主资源，而所有的资源最终都应该归属于项目，以便于项目维护和成本核算。由此可知，一个资源要么归属于项目，要么暂存于资源池，正常情况下所有资源都应归属到项目，那资源池也应该是空的。资源池内的资源有两种方式归属到项目，其一是通过动态规则自动处理，其二便是人工认领手动处理，这里我们当然是推荐动态规则自动处理啦，减少人为介入，一切都自动化完成


还有个**资源树节点**的概念，资源树就是我们通常所说的服务树，或者CI树，都是一样的东西，只是不同的地方叫法不一样，这里就统一叫资源树好了，资源树是一个树状结构，会有很多的节点，项目下的资源就隶属于不同的节点，通过树的方式去组织资源，可以很清晰的了解资源关系，使用起来也十分方便，应用特别广泛。就像下方图示的左侧就是一个资源树，资源归属到不同的树节点，可以根据不同的资源节点动态获取节点下的资源数据，使用起来清晰方便


![](https://blog.ops-coffee.cn/static/images/2022/0610.02.png)


有了这个基本的概念我们再来聊聊这三种资源纳管的方式


## 多云同步


首先是多云同步，多云同步是通过云平台的API或SDK，自动同步云资源数据到资源池，这里只需要添加云平台的账号即可，自动同步程序会根据配置的时间周期自动去云上拉取资源数据然后入库，入库时会自动判断是更新资源属性还是新建资源。新增还是更新的主要判断依据是：云账号\+云区域\+资源ID联合唯一，正常情况下同一云账号同一云区域下的资源ID是唯一的，所以遇到这三者一致的资源则更新，否则就新建。整体的逻辑如下图所示


![](https://static.ops-coffee.cn/static/images/2024/1105.02.png)


动态规则自动匹配需要资源具有一定的规律性，最常用的就是通过资源名称来区别，例如一个项目A正式环境使用的Nginx服务器，我们想要给归属到项目A下的资源树节点`ProjectA/PROD/Nginx`下，那我们最好在资源命名时进行同一，例如都符合`ProjectA-PROD-Nginx-{number}`这样的格式，那动态规则就比较好创建，所有名称以`ProjectA-PROD-Nginx-`开头的资源都自动给归属到资源树节点`ProjectA/PROD/Nginx`下


![](https://static.ops-coffee.cn/static/images/2024/1105.03.png)


当然资源命名没有那么规范且数量不是很多的情况下，也可以直接在资源列表页面，直接添加资源池内的资源到资源树节点下，手动添加有一个前提，那就是资源已经在资源池里了，资源到资源池里除了多云自动同步外，还有个方式那就是我们下边会讲到的Agent主动上报


![](https://static.ops-coffee.cn/static/images/2024/1105.04.png)


多云同步比较简单，如果用到了云，那首先推荐的就是这种方式进行资源纳管，这里的云可以是公有云，例如阿里云、腾讯云、华为云等等，或者是私有云，例如VMWare、OpenStack、OpenShift等等


## Agent上报


除了多云同步的方式进行资源纳管之外，我们还提供了Agent，可以部署在服务器上以实现资源主动上报统一纳管，这种方式对于IDC之类没有提供API/SDK获取资源的方式来说非常有效，通过安装客户端Agent，自动上报云资源数据，这里只需要服务器安装Agent即可，Agent只要与服务器连通，则会自动上报资源数据到资源池，然后配合动态规则自动将资源归属到资源树节点，这个过程与多云到自动纳管逻辑一致


![](https://static.ops-coffee.cn/static/images/2024/1105.05.png)


对于安装了Agent的服务器来说，除了资源先到资源池再配合动态规则自动归属到资源树节点的管理方式外，Agent还提供直接归属到资源树节点而不经过资源池的方式。这就涉及Agent的高级配置，Agent的配置文件默认只需要配置一个Server服务器的地址即可启动，像下边这样



```
SERVER-ADDRESS: "wss://agent-server.ops-coffee.cn/api/v1/agent"

```

Agent启动之后会与Server建立连接，Server通过周期任务把Agent上报的资源入库，默认也是先进了资源池，但当Agent配置了参数`BIZ-ID`时，Server入库则首先去匹配`BIZ-ID`所对应的资源树节点是否存在，如果存在则直接给归属到资源树节点下，不存在则进资源池。这样的话，当用户明确知道自己的资源归属到哪个资源树节点时，则可以在Agent的配置文件中添加`BIZ-ID`配置，像下边这样



```
SERVER-ADDRESS: "wss://agent-server.ops-coffee.cn/api/v1/agent"
BIZ-ID: "37"

```

当Server入库数据时会先检查ID为37的资源树节点是否存在，如果存在则直接归属到节点，这样就实现了Agent直接归属到节点，而无需经过资源池的步骤，这样的话也就不需要动态规则自动匹配再归属到资源树节点了，更加方便


## 手动添加


以上两种方式无论是多云自动同步还是Agent主动上报，都可以实现资源的自动入库，自动归属到资源树节点，这两种方式都有一定的前提条件，要么属于云资源可以自动同步，要么安装了Agent可以主动上报，如果不属于云资源，也不想安装Agent，如何管理资源呢？那系统提供了一种最原始的方式，用户自己手工录入，或是通过Excel批量导入资源


![](https://static.ops-coffee.cn/static/images/2024/1105.06.png)


这两种方式都比较原始，资源属性的更新依赖人工操作，不像多云同步或Agent上报，资源属性的更新都可以自动完成，所有不是特殊情况都不建议使用，这里就不多赘述了


至此，资产系统所支持的三种资源纳管理介绍完毕


