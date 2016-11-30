title: '[转]利用 CocoaPods 建立私有仓库'
date: 2016-11-30 15:05:35
tags: [pod, 私有仓库]
---

在开发过程中，项目会积累很多可以复用的代码包，有些我们不想开源，又想像开源库一样在CocoaPods 中管理、使用它们，那么通过私有仓库来管理就很必要。这种方式也可以用来模块化我们的项目。

<!-- more -->

![](/uploads/2016/11/30/1.png)

## 实践：

例如把项目中 CustomController 的一些公共组件抽取出来进行 Pods 方式的管理

有两种方式：第一种是通过本地的方式管理；

第二种是将代码托管到 git，创建私有 podspec 的方式；

下面会分开介绍具体的实施过程。

### 一、本地方式：

**1.**新建一个工程 ManageLocalCode，新建 Test 类或者直接拖入已经封装好的公共组件，这个 	工程专门存放公共组件。

注意：文件路径一定要填写正确

（1）如果单纯测试，在工程里创建新类 Test，其工程地址在配置文件中填写为：

s.source_files  = “ManageLocalCode/Test.{h,m}”

![img](/uploads/2016/11/30/2.png)

（2）如果使用 subspec 分模块功能，按照不同的模块对文件目录进行整理，比如我们把项目中的 CustomController 的部分公共组件复制进来，其工程地址在配置文件中填写为：

s.source_files = ‘ManageLocalCode/CustomController/Classes/**/*'

（ 我们采用这种分模块的方式，关于source_files是什么后面有介绍）

![img](/uploads/2016/11/30/3.png)

**2.**在工程根目录下，新建 podspec 文件 ManageLocalCode.podspec。

可以通过命令 pod spec create ManageLocalCode，这样生成的 spec 文件各个条目都有,

比较繁多，根据需要填写，重要的代码具体介绍如下：

![img](/uploads/2016/11/30/4.png)

**3.**输入命令 pod lib lint 进行验证，验证通过后进行下一步。验证不通过会有提示，按照提示修改 ManageLocalCode.podspec 配置文件；Xcode 内的警告可以通过命令 pod lib lint --allow-warnings 忽略。

**4.**另外新建一个工程 SouFunDemo，来引入 ManageLocalCode 的代码。并通过命令 pod init新建 podfile 文件，path 是 podspec 的本地路径。配置如下：

![img](/uploads/2016/11/30/5.png)

**5. **pod install。pod 成功之后，在 Development pods 可以看到我们的代码。查看 console 输出 "0"。

![img](/uploads/2016/11/30/6.png)

### 二、创建私有pod，代码用git管理的方式：

假如某个项目中有同事需要使用你的代码，那么本地代码的方式就不太友好了。可以将代码托管到git，创建私有podspec，其他人只需要用git上的podspec就可以了。

**1.**将本地代码托管到git。需要两个 git 仓库：CustomController 和 SouFunSpec；一个放开发代码，一个专门放 podspec 文件。因为 github 的私有仓库是收费的，这里用的是 oschina 的 git 管理，如果公司内部使用的话，需要在自己的服务器搭建 git。

准备工作：

命令行->

pod repo add SouFunSpecs https://git.oschina.net/cooxu/SouFunSpec.git

![img](/uploads/2016/11/30/7.png)

**2.**配置 CustomController.podspec

（1）clone 项目到本地，提交忽略文件；

（2）工程根目录下，新建.podspec文件

pod spec create CustomController

（3）配置 CustomController.podspec ，参考上面本地方式，这里需要改动的：

source 路径为  s.source  = { :git => "https://git.oschina.net/cooxu/						 CustomController.git", :tag => "0.0.1" }

source_files 路径为代码的相对路径

s.source_files="CustomController/CustomController/XXTool.{h,m}"

![img](/uploads/2016/11/30/8.png)

（4）本地验证.podspec文件 pod lib lint

本地验证必须通过，按照提示修改 error，Xcode警告可以用

pod lib lint --allow-warnings 忽略掉。

（5）将配置好的CustomController.podspec 文件 push 到 git 服务器后打 tag，tag值和.podspec配置文件中保持一致

（6）验证远程库 pod spec lint；输出 CustomController.podspec passed validation.		         为成功

![img](/uploads/2016/11/30/9.png)

**3.**私用库 XXSpec 中添加工具库 CustomController

pod repo push XXSpec CustomController.podspec

此时查看 XXSpec，工具库的配置文件已经被 push 过来

![img](/uploads/2016/11/30/10.png)

通过~/.cocoapods/ 前往文件夹，可以看到 cocoaPods 中存在了我们的私有库

注：master 文件里面是其他我们可以Pod的第三方开源库，初次安装cocoaPods时，就是从source 'https://github.com/CocoaPods/Specs.git' 中 clone 到我们本地的。

![img](/uploads/2016/11/30/11.png)

**4.**使用我们的私有库

（1）新建一个 demo 程序

pod search CustomController;搜索到了我们的私有库

![img](/uploads/2016/11/30/12.png)

（2）Podfile 文件配置注意一定要填写 source，否则会导致无法安装

![img](/uploads/2016/11/30/13.png)

（3）pod install 安装完成后

打开我们的 TestPodFrameworkDemo.xcworkspace，可以看到我们的私有库

已经存在Pods中，导入头文件，进行测试。

![img](/uploads/2016/11/30/14.png)

总结：通过上面的方式，我们可以把某些模块独立出来，建立我们自己的私有库，方便项目管理，在以后的新项目中也可以通过 Pods 的方式快速集成。

文／cooxu（简书作者）
原文链接：http://www.jianshu.com/p/7683cc690409
著作权归作者所有，转载请联系作者获得授权，并标注“简书作者”。