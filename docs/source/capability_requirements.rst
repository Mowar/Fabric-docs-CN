Capability Requirements 能力需求
---------------------------------

因为Fabric是一个分布式系统，通常涉及多个组织(有时在不同的国家甚至大洲)，
所以在网络中可能存在许多不同版本的Fabric代码。然而，重要的是网络需要以同样的方式处理事务，
以便每个节点对当前网络状态有相同的观点。


这意味着，每个网络 -- 以及网络中的每个通道 -- 必须定义一组我们称为“能力”的东西，以便能够参与处理事务。
例如Fabric v1.1引入了新的MSP角色类型“Peer”和“Client”。但是，如果v1.0的Peer不理解这些新角色类型，
它将无法适当地评估引用它们的背书策略。这意味着在使用新的角色类型之前，网络必须同意启用v1.1 ``channel`` 的能力需求，
确保所有Peer都做出相同的决定。

只有支持所需能力的节点才能够参与到channel中，新的节点在启用相应能力之前不会启用新的验证逻辑。
通过这种方式，能力需求可以确保即使是不同的Fabric版本，网络也不可能产生状态分叉的结果。

Defining Capability Requirements 定义能力需求
==============================================

能力需求是在通道配置中的每个通道中定义的(在通道的最新配置块中可以找到)。通道配置包含三个位置，每个位置都定义了不同类型的能力。

+------------------+-----------------------------------+----------------------------------------------------+
| Capability Type  | Canonical Path                    | JSON Path                                          |
+==================+===================================+====================================================+
| Channel          | /Channel/Capabilities             | .channel_group.values.Capabilities                 |
+------------------+-----------------------------------+----------------------------------------------------+
| Orderer          | /Channel/Orderer/Capabilities     | .channel_group.groups.Orderer.values.Capabilities  |
+------------------+-----------------------------------+----------------------------------------------------+
| Application      | /Channel/Application/Capabilities | .channel_group.groups.Application.values.          |
|                  |                                   | Capabilities                                       |
+------------------+-----------------------------------+----------------------------------------------------+

* **Channel:** 这些能力适用于Peer和orders，并且位于根 ``Channel`` 组中。

* **Orderer:** 仅适用于orders，并且位于 ``Orderer`` 组中。

* **Application:** 仅适用于peers，并且位于 ``Application`` 组中。

为了与现有的管理结构保持一致，这些能力被分解到这些组中。更新orderer能力是ordering组织自己的事，独立与application orgs。
类似的，更新application能力只是application管理员的事情， 与ordering无关。
通过将“Orderer”和“Application”之间的能力分离，假设一个网络可以运行在v1.6的ordering service，
同时又支持运行在v1.3的peer网络。

译者注： 这里的application应该就是除了order service之外的任何一个应用通道的Peer网络，可以简单理解为一个应用通道中所有peers。

然而，有些能力跨越了“Application”和“Orderer”组。正如我们前面看到的，添加新的MSP角色类型是orderer管理员和application管理员都同意并且需要认识到的。
orderer必须理解MSP角色的含义，以便允许事务通过排序，而peers必须理解角色，以便验证事务。
这种能力 -- 它跨越了applitaion和orderer组件 -- 在顶层“Channel”组中定义。

.. note:: channel能力可能被定义为v1.3版本，而orderer和application分别被定义为1.1版本和v1.4版本。
          在“Channel”组级别启用能力并不意味着在特定的“Orderer”和“Application”组级别可以使用相同的能力。

Setting Capabilities-设置能力
================================

能力被设置为通道配置的一部分(或者作为初始配置的一部分(我们将稍后讨论)，或者作为重新配置的一部分)。

.. note::  我们有两个文档讨论了通道重新配置的不同方面。首先，我们有一个教程，为您演示添加一个Org到一个通道的过程。
          :doc:`channel_update_tutorial` 。我们也有另一个文档，讨论了如何更新一个通道配置，
          它给出了各种更新的概述以及对签名过程的更全面的了解 :doc:`config_update` 。

因为在默认情况下，新通道会复制Orderer系统通道的配置，因此在新通道创建时会使用和Orderer系统通道
一样的Orderer和channel能力，以及application能力自动配置新通道。 然而，已经存在的通道必须重新配置。

Capabilities在protobuf中定于的结构如下：

.. code:: bash

  message Capabilities {
        map<string, Capability> capabilities = 1;
  }

  message Capability { }

用JSON格式举例：

.. code:: bash

  {
      "capabilities": {
          "V1_1": {}
      }
  }

Capabilities in an Initial Configuration 初始化配置中的Capabilities
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Fabric源代码config路径下的 ``configtx.yaml`` 文件中， 在 ``Capabilities`` 部分列举了每种能力类型
(Channel, Orderer, and Application)。

启用能力的最简单方法是选择一个v1.1示例概要文件并为您的网络定制它。例如:

.. code:: bash

    SampleSingleMSPSoloV1_1:
        Capabilities:
            <<: *GlobalCapabilities
        Orderer:
            <<: *OrdererDefaults
            Organizations:
                - *SampleOrg
            Capabilities:
                <<: *OrdererCapabilities
        Consortiums:
            SampleConsortium:
                Organizations:
                    - *SampleOrg

注意，在根级别(用于channel capabilities)和在Orderer级别(用于Orderer能力)定
义了一个 ``Capabilities`` 部分。
上面的示例使用YAML引用的方式将定义在文件底部的capabilities部分包含进来。

在定义orderer系统通道时，不存在Application部分，因为这些能力是在创建application通道时定义的。
要在通道创建时定义新通道的application能力，application管理员应该在 ``SampleSingleMSPChannelV1_1`` 中对其通道创建事务建模。

.. code:: bash

   SampleSingleMSPChannelV1_1:
        Consortium: SampleConsortium
        Application:
            Organizations:
                - *SampleOrg
            Capabilities:
                <<: *ApplicationCapabilities

Applicatoin部分的 ``Capabilities`` 元素引用了定义在YAML文件底部的 ``ApplicationCapabilities`` 部分。

.. note:: 应用通道中的Channel和Orderer capabilities继承自ordering系统通道中的定义，在创建通道的时候被自动包含进来

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
