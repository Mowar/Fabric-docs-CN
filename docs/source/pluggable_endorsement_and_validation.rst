Pluggable transaction endorsement and validation - 交易背书和验证的插件化
===========================================================================

Motivation - 动机
--------------------

当一个交易提交后，在验证的时候，节点会在应用这些状态变更之前进行交易自带的多种检查：
- 验证签名该交易的身份信息。

- 验证交易背书者的签名。

- 确定交易满足相关链码命名空间的背书规则。

有一些场景需要不同于 Fbaric 默认验证规则的自定义验证规则，比如：

- **UTXO （未消费交易输出）：** 当验证账户中的输入是否被双花的时候。

- **匿名交易：** 当背书中不包含节点身份信息，但是共享的签名和公钥没有和节点的身份信息相关联的时候。

Pluggable endorsement and validation logic - 插件化背书和验证逻辑
--------------------------------------------------------------------

Fabric 允许以插件的方式在节点上实现和部署自定义的相关链码的背书和验证逻辑。这个逻辑可以是编译进节点的内置可选逻辑，也可以是编译后作为 `Golang 插件 <https://golang.org/pkg/plugin/>`_ 和节点部署在一起的。


重申一下，每一个链码都是在初始化的时候关联它的背书和验证逻辑的。如果用户没有选择，就会选择内置的逻辑。
节点管理员可以在节点启动的时候改变背书/验证逻辑，根据扩展节点的本地配置选择自定义的背书/验证逻辑来加载和应用。

Configuration - 配置
-------------------------

每一个节点都有一个本地配置文件（``core.yaml``）声明了一个背书/验证逻辑的名字和启用映射。

默认逻辑被称为 ``ESCC`` （“E”代表 endorsement ）和 ``VSCC`` （validation），
你可以在节点本地配置文件中的 ``handlers`` 部分找到他们:

.. code-block:: YAML

    handlers:
        endorsers:
          escc:
            name: DefaultEndorsement
        validators:
          vscc:
            name: DefaultValidation

当背书和验证被编译到节点中的时候， ``name`` 属性代表要运行的为了包含创建背书/验证逻辑实例的工厂函数的初始化函数。

这个函数是 ``core/handlers/library/library.go`` 中 ``HandlerLibrary`` 结构的一个实例方法，
为了增加自定义背书和验证逻辑，这个结构需要被其他任何方法扩展。

因为这很麻烦，而且对部署构成了挑战，所以通过在 ``name`` 下增加额外的
属性 ``library`` 作为一个 Golang 插件来部署自定义的背书和验证。

例如，如果我们实现了一个自定义背书和验证逻辑的插件，我们可以在配置文件 ``core.yaml`` 中增加如下入口：

.. code-block:: YAML

    handlers:
        endorsers:
          escc:
            name: DefaultEndorsement
          custom:
            name: customEndorsement
            library: /etc/hyperledger/fabric/plugins/customEndorsement.so
        validators:
          vscc:
            name: DefaultValidation
          custom:
            name: customValidation
            library: /etc/hyperledger/fabric/plugins/customValidation.so

而且我们必须把 ``.so`` 插件文件放在节点的本地文件系统中.

.. note:: 从这里往后，实现的自定以背书和验证逻辑将被称为“插件”，包括编译到节点中的。


Endorsement plugin implementation - 背书插件的实现
--------------------------------------------------

为了实现背书插件，必须实现 ``core/handlers/endorsement/api/endorsement.go`` 中
的 ``Plugin`` 接口：

.. code-block:: Go

    // Plugin endorses a proposal response
    type Plugin interface {
    	// Endorse signs the given payload(ProposalResponsePayload bytes), and optionally mutates it.
    	// Returns:
    	// The Endorsement: A signature over the payload, and an identity that is used to verify the signature
    	// The payload that was given as input (could be modified within this function)
    	// Or error on failure
    	Endorse(payload []byte, sp *peer.SignedProposal) (*peer.Endorsement, []byte, error)

    	// Init injects dependencies into the instance of the Plugin
    	Init(dependencies ...Dependency) error
    }

一个给定插件类型（通过识别方法名是否为 ``HandlerLibrary`` 的实例方法或者 ``.so`` 插件的路径）的背
书插件实例，是通过让节点执行 ``PluginFactory`` 接口中的 ``New`` 方法了来为每一个通道创建的，这个方法需要插件的开发者来实现。

.. code-block:: Go

    // PluginFactory creates a new instance of a Plugin
    type PluginFactory interface {
    	New() Plugin
    }


``Init`` 方法接收 ``core/handlers/endorsement/api/`` 声明的
所有依赖项，他们被表示为嵌入 ``Dependency`` 接口。

在 ``Plugin`` 实例被创建之后，节点将调用 ``Init`` 方法，并将依赖项作为参数传递。

目前 Fabric 的背书插件有如下依赖项：

- ``SigningIdentityFetcher`` ： 返回一个基于给定签名提案的 ``SigningIdentity`` 实例。

.. code-block:: Go

    // SigningIdentity signs messages and serializes its public identity to bytes
    type SigningIdentity interface {
    	// Serialize returns a byte representation of this identity which is used to verify
    	// messages signed by this SigningIdentity
    	Serialize() ([]byte, error)

    	// Sign signs the given payload and returns a signature
    	Sign([]byte) ([]byte, error)
    }

- ``StateFetcher`` ：获取一个和世界状态交互的 **State** 对象。

.. code-block:: Go

    // State defines interaction with the world state
    type State interface {
    	// GetPrivateDataMultipleKeys gets the values for the multiple private data items in a single call
    	GetPrivateDataMultipleKeys(namespace, collection string, keys []string) ([][]byte, error)

    	// GetStateMultipleKeys gets the values for multiple keys in a single call
    	GetStateMultipleKeys(namespace string, keys []string) ([][]byte, error)

    	// GetTransientByTXID gets the values private data associated with the given txID
    	GetTransientByTXID(txID string) ([]*rwset.TxPvtReadWriteSet, error)

    	// Done releases resources occupied by the State
    	Done()
     }

Validation plugin implementation - 验证插件的实现
-------------------------------------------------------

实现验证插件，必须实现 ``core/handlers/validation/api/validation.go`` 中的 ``Plugin`` 接口：

.. code-block:: Go

    // Plugin validates transactions
    type Plugin interface {
    	// Validate returns nil if the action at the given position inside the transaction
    	// at the given position in the given block is valid, or an error if not.
    	Validate(block *common.Block, namespace string, txPosition int, actionPosition int, contextData ...ContextDatum) error

    	// Init injects dependencies into the instance of the Plugin
    	Init(dependencies ...Dependency) error
    }

每一个 ``ContextDatum`` 都是节点传递给验证插件的附加的运行时导出的元数据。
现在，传递的唯一 ``ContextDatum`` 表示链码的背书规则。

.. code-block:: Go

   // SerializedPolicy defines a serialized policy
  type SerializedPolicy interface {
	validation.ContextDatum

	// Bytes returns the bytes of the SerializedPolicy
	Bytes() []byte
   }


一个给定插件类型（通过识别方法名是否为 ``HandlerLibrary`` 的实例方法或者 ``.so`` 插件的路径）的验证插件实例，
是通过让节点执行 ``PluginFactory`` 接口中的 ``New`` 方法了来为每一个通道创建的，这个方法需要插件的开发者来实现。

.. code-block:: Go

    // PluginFactory creates a new instance of a Plugin
    type PluginFactory interface {
    	New() Plugin
    }

``Init`` 方法接收 ``core/handlers/endorsement/api/`` 声明的所有依赖项，
他们被表示为嵌入 ``Dependency`` 接口。

在 ``Plugin`` 实例被创建之后，节点将调用 **Init** 方法，并将依赖项作为参数传递。


目前 Fabric 的验证插件有如下依赖项：

- ``IdentityDeserializer`` ：将表示身份的字节码转换为 ``Identity`` 对象，
  以便通过和他们相关的 MSP 验证他们的签名，
  和判断是否满足 **MSP 规则** 。
  完整的定义在 ``core/handlers/validation/api/identities/identities.go`` 。

- ``PolicyEvaluator`` ：判断给定的策略是否合适：

.. code-block:: Go

    // PolicyEvaluator evaluates policies
    type PolicyEvaluator interface {
    	validation.Dependency

    	// Evaluate takes a set of SignedData and evaluates whether this set of signatures satisfies
    	// the policy with the given bytes
    	Evaluate(policyBytes []byte, signatureSet []*common.SignedData) error
    }

- ``StateFetcher`` ：获取一个和世界状态交互的 **State** 对象。

.. code-block:: Go

    // State defines interaction with the world state
    type State interface {
        // GetStateMultipleKeys gets the values for multiple keys in a single call
        GetStateMultipleKeys(namespace string, keys []string) ([][]byte, error)

        // GetStateRangeScanIterator returns an iterator that contains all the key-values between given key ranges.
        // startKey is included in the results and endKey is excluded. An empty startKey refers to the first available key
        // and an empty endKey refers to the last available key. For scanning all the keys, both the startKey and the endKey
        // can be supplied as empty strings. However, a full scan should be used judiciously for performance reasons.
        // The returned ResultsIterator contains results of type *KV which is defined in protos/ledger/queryresult.
        GetStateRangeScanIterator(namespace string, startKey string, endKey string) (ResultsIterator, error)

        // GetStateMetadata returns the metadata for given namespace and key
        GetStateMetadata(namespace, key string) (map[string][]byte, error)

        // GetPrivateDataMetadata gets the metadata of a private data item identified by a tuple <namespace, collection, key>
        GetPrivateDataMetadata(namespace, collection, key string) (map[string][]byte, error)

        // Done releases resources occupied by the State
        Done()
    }

Important notes - 重要提醒
--------------------------------

- **验证插件的跨节点一致性：** 未来的发布版本中，Fabric 通道基础设施将确保通道中所有节点的给定链码使用同样的验证逻辑，
  以消除错误配置的可能性，它可能导致意外运行不同实现的节点的状态差异。但是，现在系统操作员和管理员唯一的职责就是确保它不会发生。

- **验证插件错误处理：** 任何时候验证插件不能判定一个交易是否合法，由于某些临时执行问题，比如无数据库权限，
  它应该返回一个 **ExecutionFailureError** 类型的错误，
  该错误定义在 ``core/handlers/validation/api/validation.go`` 。
  其他返回的错误，被当做背书策略错误并且验证逻辑把交易标记为无效。另外，如果返回一个 ``ExecutionFailureError`` ，
  链处理将停止而不是标记交易为无效。这是防止不同节点之间的状态差异。

- **私有元数据检索的错误处理：** 如果插件使用 ``StateFetcher`` 接口来检索私有数据的元数据，
  必须按一下方式处理错误： ``CollConfigNotDefinedError``
  和 ``InvalidCollNameError`` 表示指定集合不存在，
  应该按确定性错误处理而不应该让插件返回 ``ExecutionFailureError`` 。

- **将 Fabric 代码导入插件：** 不鼓励将属于 Fabric 而不是 protobufs 的代码作为插件的一部分，
  当不同发布版本的 Fabric 代码不同时会导致问题，或者在运行不同节点版本时导致不可操作的问题。
  理想情况下，插件代码应该值使用给定的依赖项，最小化导入 protobufs 以外的值。

  .. Licensed under Creative Commons Attribution 4.0 International License
     https://creativecommons.org/licenses/by/4.0/
