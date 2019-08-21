Service Discovery - 服务发现
==============================

Why do we need service discovery? - 为什么需要服务发现？
--------------------------------------------------------

应用程序需要连接到通过SDK发布的API来完成诸如在peer上执行链码、将交易提交给
orderer、或接收交易状态变更的信息。

然而，SDK需要很多信息才能够将应用程序接入到相关的网络节点。这些信息不光包括：用来和peer
及orderer建立链接的的证书、peer和orderer的网络IP地址及端口。还要提供相关的背书策略，和
链码安装在哪些peer节点上（便于应用程序知晓要将链码草案发送到哪些peer节点）等信息。


在v1.2之前，这些信息都是静态，且需手动编码提供。并且，无法响应相关的网络变更（例如，链码peer节点的
新增与下线），这些静态配置也不能支持应用程序有效应对背书策略的变化（例如，一个新组织加入channel）。

而且，客户端应用也无法知道哪些peer节点已经更新了账本，哪些peer节点还没有完成更新。
这可能会导致应用程序向还未完成账本更新的peer节点提交交易草案，然后在提交时被拒，造成
资源浪费。

**发现服务** 通过让peer节点来计算所需动态信息，然后提供给SDK去使用。

How service discovery works in Fabric - Fabric中的发现服务如何工作
------------------------------------------------------------------

应用程序在启动时知道可以访问哪些可信的peer节点来执行发现查询。和应用程序在同一个组织
内的peer节点会是很好的选择。

应用程序向发现服务发送一个配置查询，即可获得网络的静态信息。这些信息在需要的时候即可通过
向发现服务发送新的查询获得。

发现服务运行于peer节点上，利用gossip通讯层维护的元数据来得知哪些peer节点在线。发现服务也
从peer节点的状态数据库提取其他诸如背书策略等信息。

有了发现服务，应用程序不必在指定要从哪些peer节点获得背书。SDK只要依据给定的通道
和链码ID，向发现服务发送一个简单的查询。发现服务就可以计算出包含下面两个对象的描述：

1. **布局**: peer节点的分组清单，和每组中选出的对应数量的peer节点

2. **分组和peer节点映射**: 从布局中的分组到通道中的peer节点。实际应用中，节点分组
   通常由统一组织中的peer节点组成。只是服务API是通用的，因而忽略组织而采用分组。

下面的示例描述了两个组织，每个组织中包含两个peer节点，且采用``AND(Org1, Org2)``的
评估策略：

.. code-block:: text

   Layouts: [
        QuantitiesByGroup: {
          “Org1”: 1,
          “Org2”: 1,
        }
   ],
   EndorsersByGroups: {
     “Org1”: [peer0.org1, peer1.org1],
     “Org2”: [peer0.org2, peer1.org2]
   }

换句话，背书策略要求Org1中的一个peer节点和Org2中的一个peer节点共同参与背书。而且，描述还
表明Org1和Org2分别有哪些peer节点可以参与背书（Org1和Org2中的``peer0``和``peer1``）。

SDK则从上述描述中随机选择一个布局。换句话说，上例中背书策略是Org1``AND``Org2。如果
背书策略是``OR``的话，SDK会随机的选择Org1或者Org2。应为一个组织中的一个peer节点的签名
既满足背书策略。

SDK选定布局后，更具客户端定义的条件选出peer节点（SDK因为知道账本高度，因此能够做这件事）。
例如，依据布局分组中peer节点的数量，可以选择账本高度高的peer节点，或者排除已下线peer节点。
如果并没有peer节点满足要求的优先条件，SDK则随机选择次优peer节点。

Capabilities of the discovery service - 发现服务的能力
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

发现服务可以支持一下查询：

* **配置布局**: 返回通道中包含所有组织和orderder节点endpoint的``MSPConfig``信息。

* **Peer membership query**: 返回已加入通道的peer节点。

* **Endorsement query**: 返回给定通道中给定链码的背书策略描述。

* **Local peer membership query**: 返回处理查询请求的peer节点的本地会员信息。缺省情况下，
  peer节点在客户端是管理员的情况下会处理次请求。

Special requirements - 特殊要求
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当peer节点使用TLS是，客户端必须提供TLS证书才能链接peer节点。如果peer节点根据配置没有
验证客户端证书，TLS证书可以自我验签。

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
.. Translated to Simplified Chinese by @[xue35](github.com/xue35)
