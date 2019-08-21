Endorsement policies - 背书策略
==================================

每一个链码都有一个背书策略，用于指定一些执行和背书执行结果的节点，以此来验证一个交易的有效性。这些背书策略定义了组织中（其中的节点）哪些必须“背书”（或者说，同意）一个提案的执行。

.. note:: 重申一下 **状态** 的概念，和区块数据不同，它是由一个键值对表示的。详细信息参考文档 :doc:`ledger/ledger`

作为节点验证交易步骤的一部分，每一个验证节点都会进行检查，以确保交易包含适当的背书数量并且都来源于期望的地
方（这些都是在背书策略中定义的）。也会检查背书是否正确（例如，会验证签名是否来源于有效的证书）。

Two ways to require endorsement - 请求背书的两种方法
----------------------------------------------------

一般地，在通道链码初始化或者更新的时候会指定背书策略（也就是说，一个背书策略覆盖一个链码中所有相关的状态）。

然而，有些情况下一些特殊的状态（或者说，一个特殊的键值对）可能会需要不同的背书策略。这里的 **基于状态的背书** 允许
一个指定键的背书策略覆盖默认的背书策略。


为了解释使用这两种类型背书策略的情况，想像一个汽车置换的通道。“创建” ——或者说是“发布”——一辆可以交易的汽车作为资产（换句话说，
就是在世界状态中加入一个键值对），需要满足链码级的背书策略。在后边的段落中你可以找到怎么设置链码级背书策略。

如果一辆车需要指定一个背书策略，在车辆被创建前和创建后都可以设定。有很多种理由解释为什么指定一个特定状态的背书
策略是必要的。这辆车可能具有历史重要性或价值，因此有必要得到持有执照的评估师的认可。车辆的主人也想确认评估师的
节点在交易上签了名（如果他们在同一个通道）。在这两种情况下， **一个特殊资产需要一个和其相关链码上的其他资产的背书策略不同的背书策略。**

我们将在下一节中展示如何定义一个基于状态的背书策略。但是在那之前，我们先看一下如何设置一个链码级背书策略。

Setting chaincode-level endorsement policies - 设置链码级背书策略
-------------------------------------------------------------------

链码级背书策略在链码初始化的时候指定，通过 SDK (有一些例子演示了怎么做，点击
`here <https://github.com/hyperledger/fabric-sdk-node/blob/f8ffa90dc1b61a4a60a6fa25de760c647587b788/test/integration/e2e/e2eUtils.js#L178>`_)或者 peer CLI 命令后边 ``-P`` 加背书策略都可以。

.. note:: 现在不要担心这里的策略语法 （``'Org1.member'`` 等等），我们会在下一部分介绍相关语法。

例如：

::

    peer chaincode instantiate -C <channelid> -n mycc -P "AND('Org1.member', 'Org2.member')"

这条命令部署了一个名为 ``mycc`` ("my chaincode") 的链码，
并设置了一个策略 ``AND('Org1.member', 'Org2.member')`` ,
其含义为交易将请求 Org1 和 Org2 的成员签名。

注意，如果身份类别启用的话 （参考 :doc:`msp`），你就可以使用 ``PEER`` 角色来限制只能 peer 背书。

例如：

::

    peer chaincode instantiate -C <channelid> -n mycc -P "AND('Org1.peer', 'Org2.peer')"

在初始化后新加入的组织可以查询链码（是否有查询权限是链码的通道策略和应用层检查定义的）但是不能执行和背书链码。
背书策略需要更改以允许提交的交易可以被新组织背书。

.. note:: 如果在初始化的时候没有指定，默认背书策略就是“通道中的组织的所有成员”。例如，在一个有 “Org1” 和 “Org2” 的通道中，默认背书策略就是 "OR('Org1.member', 'Org2.member')" 。

Endorsement policy syntax - 背书策略语法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

就像你上边看到的，策略就是一组规则（ “规则” 就是角色匹配的身份）。规则描述为 ``'MSP.ROLE'`` ，
这里 ``MSP`` 代表MSP ID ， ``ROLE`` 代表四个可接受角色
中的一个： ``member`` ， ``admin`` ， ``client`` ，和 ``peer`` 。

这是一些有效规则的例子：

  - ``'Org0.admin'`` ： ``Org0`` 中任意管理员
  - ``'Org1.member'`` ： ``Org1`` 中任意成员
  - ``'Org1.client'`` ： ``Org1`` 中任意客户端
  - ``'Org1.peer`` ： ``Org1`` 中任意节点

语法如下：

``EXPR(E[, E...])``

``EXPR`` 是 ``AND`` ， ``OR`` ， 或 ``OutOf`` 其中
之一， ``E`` 是一个规则（语法如上）或者另外一个嵌套的 ``EXPR`` 。

例如：

  - ``AND('Org1.member', 'Org2.member', 'Org3.member')`` 请求这三个规则中的每一个签名。
  - ``OR('Org1.member', 'Org2.member')`` 请求这两个规则中的任一个。
  - ``OR('Org1.member', AND('Org2.member', 'Org3.member'))`` 请求 ``Org1`` 中
    的一个成员的签名或者请求 ``Org2`` 和 ``Org3`` 中各一个成员的签名。
  - ``OutOf(1, 'Org1.member', 'Org2.member')`` ，
    和 ``OR('Org1.member', 'Org2.member')`` 一样。
  - 类似地， ``OutOf(2, 'Org1.member', 'B.member')``
    和 ``AND('Org1.member', 'Org2.member')`` 一样。


.. _key-level-endorsement:

Setting key-level endorsement policies - 设置键级背书策略
---------------------------------------------------------

设置一个规则的链码级背书规则，是绑定在链码的生命周期内的。他们只能在链码所在通道初始化或者升级的时候更改。

相反，键级的背书策略可以在链码中以更细粒度的方式设置和修改。修改是常规交易的读写集的一部分。


shim API 提供了下边的函数来设置和检索键上的背书策略。

.. note:: 下边的 ``ep`` 代表 "endorsement policy" ，它可以使用上边的语法定义，也可以用下边描述的函数。两种方式都会生成背书策略的二进制版本用来被基础 shim API 使用。

.. code-block:: Go

    SetStateValidationParameter(key string, ep []byte) error
    GetStateValidationParameter(key string) ([]byte, error)

对于集合中属于 :doc:`private-data/private-data` 的键，应用一下函数：

.. code-block:: Go

    SetPrivateDataValidationParameter(collection, key string, ep []byte) error
    GetPrivateDataValidationParameter(collection, key string) ([]byte, error)

为了帮助设置背书策略和将他们封送到验证参数字节数组，shim 提供了方便的函数，让链码开发者根据组织中的MSP表示来处理背书策略。

.. code-block:: Go

    type KeyEndorsementPolicy interface {
        // Policy returns the endorsement policy as bytes
        Policy() ([]byte, error)

        // AddOrgs adds the specified orgs to the list of orgs that are required
        // to endorse
        AddOrgs(roleType RoleType, organizations ...string) error

        // DelOrgs delete the specified channel orgs from the existing key-level endorsement
        // policy for this KVS key. If any org is not present, an error will be returned.
        DelOrgs([]string) error

        // DelAllOrgs removes any key-level endorsement policy from this KVS key.
        DelAllOrgs() error

        // ListOrgs returns an array of channel orgs that are required to endorse changes
        ListOrgs() ([]string, error)
    }

例如，为一个键设置一个需要两个指定组织背书才能变更的背书策略，
传递两个组织的 ``MSPIDs`` 给 ``AddOrgs()`` ，然后调用 ``Policy()`` 来构造
要传递给 ``SetStateValidationParameter()`` 的背书规则字节数组。

Validation - 验证
-------------------

At commit time, setting a value of a key is no different from setting the
endorsement policy of a key --- both update the state of the key and are
validated based on the same rules.

在提交阶段，设置一个键的背书策略和设置键的值没有区别 —— 都是更新键的状态和根据同样的规则验证。

+---------------------+-----------------------------+--------------------------+
| Validation          | no validation parameter set | validation parameter set |
+=====================+=============================+==========================+
| modify value        | check chaincode ep          | check key-level ep       |
+---------------------+-----------------------------+--------------------------+
| modify key-level ep | check chaincode ep          | check key-level ep       |
+---------------------+-----------------------------+--------------------------+

就像我们上边讨论的，如果一个键更改了而且没有存在键级的背书策略，默认使用链码级的背书策略。
第一次设置键级背书策略也是一样 —— 新的键级的背书策略必须先经过已有的链码级背书策略背书。

如果一个键被更改了，而且存在键级背书策略，键级背书策略就覆盖链码级背书策略。实践中，这意味着
键级背书策略可以比链码级背书策略限制更少，也可以更严格。因为首次设置键级背书策略必须经过链码级背书策略认可，因此没有违反信任假设。

如果一个键的背书策略移除了（被设置为 nil），链码级背书策略就有成了默认值。

如果一个交易改变了多个关联不同背书策略的键，所有的策略都满足，这个交易才算有效。

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
