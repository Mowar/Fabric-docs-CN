Getting Started - 入门
========================

.. toctree::
   :maxdepth: 1
   :hidden:

   prereqs
   install

在我们开始之前，如果你还没有操作这一步，你不妨检查你是否已在将要开发区块链应用程序和/或运行Hyperledger Fabric的平
台上安装了所有 :doc:`prereqs` 。

安装必备组件后，即可下载并安装HyperLedger Fabric。在我们为Fabric二进制文件开发真正的安装程序时，我们提供了一个
脚本，可以 :doc:`install` 到您的系统中。该脚本还将下载Docker镜像到你的本地注册表。

Hyperledger Fabric SDKs - Hyperledger Fabric 软件开发包
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Hyperledger Fabric提供了许多软件开发包来支持各种编程语言。有两个正式发布的软件开发包Node.js和Java：

- `Hyperledger Fabric Node 软件开发包 <https://github.com/hyperledger/fabric-sdk-node>`__ 和 `Node SDK documentation <https://fabric-sdk-node.github.io/>`__.
- `Hyperledger Fabric Java 软件开发包 <https://github.com/hyperledger/fabric-sdk-java>`__.


此外，还有三个尚未正式发布的软件开发包（适用于Python，Go和REST），但它们仍可供下载和测试：

- `Hyperledger Fabric Python 软件开发包 <https://github.com/hyperledger/fabric-sdk-py>`__.
- `Hyperledger Fabric Go 软件开发包 <https://github.com/hyperledger/fabric-sdk-go>`__.
- `Hyperledger Fabric REST 软件开发包 <https://github.com/hyperledger/fabric-sdk-rest>`__.

Hyperledger Fabric CA
^^^^^^^^^^^^^^^^^^^^^^^^^^

Hyperledger Fabric提供可选的 `证书授权服务 <http://hyperledger-fabric-ca.readthedocs.io/en/latest>`_，您可以选择使用
该服务生成证书和密钥材料，以配置和管理区块链网络中的身份。不过，你也可以使用其它任何可以生成ECDSA证书的CA。

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
