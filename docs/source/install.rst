Install Samples, Binaries and Docker Images - 安装示例，二进制文件和Docker镜像
=================================================================================

在我们为Hyperledger Fabric二进制文件开发真正的安装程序的同时，我们提供了一个脚本，可以下载并安
装样本和二进制文件到你的系统。我们认为你会发现安装的示例应用程序对了解有关Hyperledger Fabric的功能和操作的更多信息非常有用。

.. note:: 如果您在 **Windows** 上运行，则需要使用Docker快速启动终端来执行即将发布的终端命令。 如果您之前没有安装，请访问 :doc:`prereqs` 。

      如果你在Windows 7或macOS系统下使用Docker Toolbox，则在安装和运行示例时，需要使用 ``C:\Users`` （Windows 7）或 ``/Users`` （macOS）下的位置。

      如果你是在Mac系统下使用Docker，则需要在 ``/Users``， ``/Volumes``，``/private`` 或 ``/tmp`` 下使用一个位置。要使用其他位置，请参阅Docker文档以获取 `文件共享 <https://docs.docker.com/docker-for-mac/#file-sharing>`__。
      如果你是在Windows系统下使用Docker，请参阅Docker文档中的 `共享驱动器 <https://docs.docker.com/docker-for-windows/#shared-drives>`__ 并使用其中一个共享驱动器的位置。

确定计算机上要放置 `fabric-samples` 存储库的位置，并在终端窗口中输入该目录。 后面的命令将执行以下步骤：

#. 如果需要，克隆 `hyperledger/fabric-samples` 存储库
#. 查看适当的版本标签
#. 将指定版本的Hyperledger Fabric平台特定二进制文件和配置文件安装到fabric-samples存储库的根目录中
#. 下载指定版本的Hyperledger Fabric docker镜像

准备好后，在要安装Fabric Samples和二进制文件的目录中，继续执行以下命令：

.. code:: bash

  curl -sSL http://bit.ly/2ysbOFE | bash -s 1.3.0

.. note:: 如果要下载Fabric，Fabric-ca和第三方Docker镜像，则必须将版本标识符传递给脚本。

.. code:: bash

  curl -sSL http://bit.ly/2ysbOFE | bash -s <fabric> <fabric-ca> <thirdparty>
  curl -sSL http://bit.ly/2ysbOFE | bash -s 1.3.0 1.3.0 0.4.13

.. note:: 如果运行上述curl命令时出错，则可能是旧版本的curl不能处理重定向或不受支持的环境。请访问 :doc:`prereqs` 页面，了解有关在何处查找最新版本curl并获取正确环境的其他信息。或者，你可以替换未缩写的 URL：https://github.com/hyperledger/fabric/blob/master/scripts/bootstrap.sh

.. note:: 你可以在任何已发布的Hyperledger Fabric版本使用上述命令。 只需将 `1.3.0` 替换为你要安装的版本的版本标识符即可。

上面的命令下载并执行一个bash脚本，该脚本将下载并提取设置网络所需的所有特定于平台的二进制文件，并将它们放入您在上面创建的克隆仓库中。它检索以下特定平台的二进制文件：

  * ``cryptogen``,
  * ``configtxgen``,
  * ``configtxlator``,
  * ``peer``,
  * ``orderer``,
  * ``idemixgen``, 和
  * ``fabric-ca-client``

并将它们放在当前工作目录的 ``bin`` 子目录中。

你可能希望将其添加到PATH环境变量中，以便在不完全限定每个二进制文件的路径的情况下拾取这些变量。例如：

.. code:: bash

  export PATH=<path to download location>/bin:$PATH

最后，该脚本会将 `Docker Hub <https://hub.docker.com/u/hyperledger/>`__ 中的Hyperledger Fabric docker映像下载到本地Docker注册表中，并将其标记为“最新”。

该脚本列出了结束时安装的Docker镜像。

查看每个镜像的名称；这些组件最终将构成我们的Hyperledger Fabric网络。你还会注意到，你有两个具有相同镜像
ID的实例——一个标记为“amd64-1.x.x”，另一个标记为“最新”。在1.2.0之前，下载的图像由 ``uname -m`` 确
定，并显示为“x86_64-1.x.x”。

.. note:: 在不同的体系结构中，x86_64/amd64将替换为标识你的体系结构的字符串。

.. note:: 如果你有本文档未解决的问题，或遇到任何有关教程的问题，请访问 :doc:`questions` 页面，获取有关在何处寻求其他帮助的一些提示。


.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
