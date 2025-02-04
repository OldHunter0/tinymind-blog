---
title: Argo介绍
date: 2025-02-04T08:15:33.113Z
---

# <font style="color:rgb(23, 43, 77);">一、什么是流水线</font>
在计算机中，**流水线是把一个重复的过程分解为若干个子过程，使每个子过程与其他子过程并行进行的技术，也叫 Pipeline**。由于这种工作方式与工厂中的生产流水线十分相似， 因此也被称为流水线技术。从本质上讲，流水线技术是一种时间并行技术。以“构建镜像”过程为例：

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838190482-0c5f7313-7963-4070-9eee-50c04db34b78.png)

在每一次构建镜像中，我们都需要拉下代码仓库中的代码，进行代码编译，构建镜像，最后推往镜像仓库。在每一次代码更改过后，这一过程都是不变的。使用流水线工具可以极大的提升这一过程的效率，只需要进行简单的配置便可以轻松的完成重复性的工作。这样的过程也被称之为 CI。

上图流程中使用的是 Jenkins。Jenkins 作为老牌流水线框架被大家所熟知。在云原生时代，Jenkins 推出了 Jenkins X 作为基于 Kubernetes 的新一代流水线，另外云原生时代还诞生了两大流水线框架—— Argo 和 Tekton。本文就详细介绍了 Argo 的相关内容。

# <font style="color:rgb(23, 43, 77);">二、Argo是什么</font>
<font style="color:rgb(199, 37, 78);">Argo Workflows</font> 是一个开源的容器原生的工作流引擎，可在 Kubernetes 上编排并行作业。Argo Workflows 通过 Kubernetes CRD方式实现。Argo 基于 Kubernetes，可以直接使用 <font style="color:rgb(199, 37, 78);">kubectl</font> 安装，安装的组件主要包括了一些 CRD 以及对应的 controller 和一个 server。

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838189840-c7c52e4e-229e-4dfd-b387-425c3047925c.png)

注意，上述安装只会执行同 namespace 内的 Workflow，cluster install 详见文档。文档地址：[https://argoproj.github.io/argo-workflows/](https://argoproj.github.io/argo-workflows/)

# <font style="color:rgb(23, 43, 77);">三、三级定义</font>
要了解 Argo 定义的 CRD，先从其中的三级定义入手。概念上的从大到小分别为 WorkflowTemplate、Workflow、Template，这些资源的命名有些相似，要注意分辨。[https://argoproj.github.io/argo-workflows/workflow-concepts/](https://argoproj.github.io/argo-workflows/workflow-concepts/)

## <font style="color:rgb(23, 43, 77);">1、Template</font>
从最简单的 template 说起，一个 template 有多种类型，分别为 <font style="color:rgb(199, 37, 78);">container</font>、<font style="color:rgb(199, 37, 78);">script</font>、<font style="color:rgb(199, 37, 78);">dag</font>、<font style="color:rgb(199, 37, 78);">steps</font>、<font style="color:rgb(199, 37, 78);">resource</font> 以及 <font style="color:rgb(199, 37, 78);">suspend</font>**。**对于 template，我们可以简单的将其理解为一个 Pod ——container/script/resource 类型的 template 都会去实际控制一个 Pod，而 dag/steps 类型的 template 则是由多个基础类型的 template （container/script/resource）组成的。

+ **container**：最常见的模板类型，与 Kubernetes container spec 保持一致。
+ **script**：该类型基于 Container，支持用户在 template 定义一段脚本，另有一个 Source 字段来表示脚本的运行环境。
+ **resource**：该类型支持我们在 template 中对 kubernetes 的资源进行操作，有一个 action 字段可以指定操作类型，如 create, apply, delete 等，并且支持设定相关的成功与失败条件用于判断该 template 的成功与失败。
+ **suspend**：Suspend template 将在一段时间内或在手动恢复执行之前暂停执行。可以从 CLI （使用 argo resume）、API 或 UI 恢复执行。
+ **steps**：Steps Template 允许用户以一系列步骤定义任务。在 Steps 中，[--] 代表顺序执行，[-] 代表并行执行。
+ **dag**：DAG template 允许用户将任务定义为带依赖的有向无环图。在 DAG 中，通过 dependencies设置在特定任务开始之前必须完成的其他任务。没有任何依赖项的任务将立即运行。有关 DAG 的详细逻辑可见源码 [https://github.com/argoproj/a...](https://github.com/argoproj/argo/blob/master/workflow/controller/dag.go#L204)。

## <font style="color:rgb(23, 43, 77);">2、Workflow</font>
在一个 Workflow 中，其 spec 中有一个名为 templates 的字段，在其中至少需要一个 template 作为其组成的任务。

一个最简单的 hello world 例子如下：

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838189791-5d6c88a1-b06d-45ce-b10e-7d219b6c863b.png)

在这个例子中，该 Workflow 的 templates 字段中指定了一个类型为 container 的 template，使用了 whalesay 镜像。

下面是一个稍微复杂的 workflow：

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838190543-054c96d3-1e27-4756-9af1-0838939dc5cc.png)

## <font style="color:rgb(23, 43, 77);">3、WorkflowTemplate</font>
WorkflowTemplate 相当于 Workflow 的模板库，和 Workflow 一样，也由 template 组成。用户在创建完 WorkflowTemplate 后，可以通过直接提交它们来执行 Workflow。

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838190814-6a152d34-0026-473d-8217-9c47e1023e44.png)

## <font style="color:rgb(23, 43, 77);">4、Workflow Overview</font>
![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838191153-a91f8023-f503-4fc0-b872-e3eca076746f.png)

在了解了 Argo 的三级定义后，我们首先来深入一下 Argo 中最为关键的定义，Workflow。**Workflow 是 Argo 中最重要的资源，有两个重要的功能：**

+ **定义了要执行的工作流。**
+ **存储了工作流的状态。**

由于这些双重职责，Workflow 应该被视为一个 Active 的对象。它不仅是一个静态定义，也是上述定义的一个“实例”。

**Workflow Template 的定义与 Workflow 几乎一致，除了类型不同。**正因为 Workflow 既可以是一个定义也可以是一个实例，所以才需要 WorkflowTemplate 作为 Workflow 的模板，WorkflowTemplate 在定义后可以通过提交（Submit）来创建一个 Workflow。

而 **Workflow** 由一个 entrypoint 及一系列 template 组成，entrypoint 定义了这个 workflow 执行的入口，而 template 会实际去执行一个 Pod，其中，用户定义的内容会在 Pod 中以 Main Container 体现。此外，还有两个 **Sidecar** 来辅助运行。

# <font style="color:rgb(23, 43, 77);">四、Sidecar</font>
在 Argo 中，这些 Sidecar 的镜像都是 **<font style="color:rgb(199, 37, 78);">argoexec</font>**。Argo 通过这个 executor 来完成一些流程控制。

## <font style="color:rgb(23, 43, 77);">1、Init</font>
当用户的 template 中需要使用到 inputs 中的 <font style="color:rgb(199, 37, 78);">artifact</font> 或者是 script 类型时（script 类型需要注入脚本），Argo 都会为这个 pod 加上一个 **InitContainer** —— 其镜像为 argoexec，命令是 argoexec init。在这个 <font style="color:rgb(199, 37, 78);">Init Container</font> 中，主要工作就是加载 <font style="color:rgb(199, 37, 78);">artifact</font>：

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838190641-3262c5e4-a46a-436e-b091-58c0d55f93dd.png)

## <font style="color:rgb(23, 43, 77);">2、Wait</font>
除了 Resource 类型外的 template，Argo 都会注入一个 Wait Container，用于等待 Main Container 的完成并结束所有 Sidecar。这个 Wait Container 的镜像同样为 <font style="color:rgb(199, 37, 78);">argoexec</font>，命令是 <font style="color:rgb(199, 37, 78);">argoexec wait</font>。（Resource 类型的不需要是因为 Resource 类型的 template 直接使用 <font style="color:rgb(199, 37, 78);">argoexec</font> 作为 Main Container 运行）

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838191323-2d057b12-e706-4632-818f-80c4f3ae7b24.png)

# <font style="color:rgb(23, 43, 77);">五、Inputs and Outputs</font>
在运行 Workflow 时，一个常见的场景是输出产物的传递。通常，一个 Step 的输出产物可以用作后续步骤的输入产物。**在 Argo 中，产物可以通过 Artifact 或是 Parameter 传递。**

## <font style="color:rgb(23, 43, 77);">1、Artifact</font>
**要使用 Argo 的 Artifact，首先必须配置和使用 Artifact 存储仓库。**具体的配置方式可以通过修改存有 Artifact Repository 信息的默认 Config Map 或者在 Workflow 中显示指定，详见 配置文档，在此不做赘述。下表为 Argo 支持的仓库类型。

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838191413-98ab862a-6c67-443b-aa8f-d025fdd3000a.png)

一个简单的使用了 Artifact 的例子如下：

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838192034-8d2633e2-3f83-4c1d-a861-009af4a5cf0d.png)

**默认情况下，Artifact 被打包为 tgz(tar +gzip)包，我们也可以使用 archive 字段指定存档策略。**

在上面的例子里，名为 whalesay 的 template 使用 cowsay 命令生成一个名为 /tmp/hello-world.txt 的文件，然后将该文件作为一个名为 hello-art 的 Artifact 输出。名为 print-message 的 template 接受一个名为 message 的输入 Artifact，在 /tmp/message 的路径上解包它，然后使用 cat 命令打印 /tmp/message 的内容。

在前面 Sidecar 介绍中提到过，**Init Container 主要用于拉取 Artifact 产物**。这些 Sidecar 正是产物传递的关键。下面，我们通过介绍另一种产物传递的方式来体验 Argo 中传递产物的关键。

## <font style="color:rgb(23, 43, 77);">2、Script</font>
先来看一个简单的例子：

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838192117-674fdcc9-c96a-401f-9535-5a2b6da93c1e.png)

在上面的例子中，有两个类型为 script 的 template，script 允许使用 source 规范脚本主体。这将创建一个包含脚本主体的临时文件，然后将临时文件的名称作为最后一个参数传递给 command（执行脚本主体的解释器），这样便可以方便的执行不同类型的脚本（bash、python、js etc）。

Script template 会将脚本的标准输出分配给一个名为 result 的特殊输出参数从而被其他 template 调用。在这里，通过 **{{steps.generate.outputs.result}}** 即可获取到名为 generate 的 template 的脚本输出。

{{xxx}} 是 Argo 固定的变量替换格式：

+ 关于变量的格式详见文档，文档地址：[https://github.com/argoproj/a...](https://github.com/argoproj/argo/blob/master/docs/variables.md)
+ 关于变量替换的逻辑详见源码，源码地址：[https://github.com/argoproj/a...](https://github.com/argoproj/argo/blob/master/workflow/common/util.go#L305)

那么，容器内部应该如何获取这个脚本输出呢？我们回到 Sidecar，在 Wait Container 中，有这样一段逻辑：

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838192798-26e1b785-2006-4acf-833b-3699349b4b29.png)  
![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838192229-9f4c379c-ed8f-48dc-ad0c-131b2df6af35.png)

再来看看这个 Wait Container 的 Volume Mount 情况：

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838193222-87232852-dfb8-4215-b058-6ae5e03d2a96.png)

现在就十分明确了，**Wait Container 通过挂载 docker.sock 以及 service account，获取到 Main Container 中的输出结果，并保存到 Workflow 中**。当然，因为 Workflow 中保存了大量的信息，当一个 Workflow 的 Step 过多时，整个 Workflow 的结构会过于庞大。

## <font style="color:rgb(23, 43, 77);">3、Parameter</font>
**Parameter 提供了一种通用机制，可以将步骤的结果用作参数。**Parameter 的工作原理与脚本结果类似，除了输出参数的值会被设置为生成文件的内容，而不是 stdout 的内容。如：  
![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838193784-85674502-3be5-45fb-85a2-59247ef46783.png)

## <font style="color:rgb(23, 43, 77);">4、Volume</font>
这并不是 Argo 处理产物传递的一种标准方式，但是通过共享存储，我们显然也能达到共通产物的结果。当然，如果使用 Volume，我们则无需借助 Inputs 和 Outputs。在 Workflow 的 Spec 中，我们定义一个 Volume 模板：

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838193712-6d971fa2-2dd9-47a4-91af-e7f08bfe3c71.png)

并在其他的 template 中 mount 该 volume：

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838193802-199965dd-f1f1-4efd-9e45-6d06aaa2f7d4.png)

# <font style="color:rgb(23, 43, 77);">六、流程控制功能</font>
## <font style="color:rgb(23, 43, 77);">1、循环</font>
在编写 Workflow 时，能够循环迭代一组输入通常是非常有用的，如下例所示:

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838194298-97cb500d-20e4-4892-b969-335322792140.png)

在源码实现中，将会去判断 withItems，如果存在，则对其中的每个元素进行一次 step 的扩展。

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838194329-2aa83877-1d75-49ab-9529-73f42bc3dcb7.png)

## <font style="color:rgb(23, 43, 77);">2、条件判断</font>
通过 <font style="color:rgb(199, 37, 78);">when</font> 关键字指定：

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838195481-4ba78952-224c-4c03-89f9-5963091f901c.png)

## <font style="color:rgb(23, 43, 77);">3、错误重试</font>
![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838194850-298e69cb-57f3-4e17-8d59-bcf07a2ef1d2.png)

## <font style="color:rgb(23, 43, 77);">4、递归</font>
**Template 可以递归地相互调用**，这是一个非常实用的功能。例如在机器学习场景中：可以设定准确率必须满足一个值，否则就持续进行训练。在下面这个抛硬币例子中，我们可以持续抛硬币，直到出现正面才结束整个工作流。

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838195591-57f6c6d6-8b33-46ff-83bd-02092cec9fd8.png)

以下是两次执行的结果，第一次执行直接抛到正面，结束流程；第二次重复三次后才抛到正面，结束流程。

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838195767-9e1b3c95-c85a-437f-aa55-ef318ab0b73e.png)

## <font style="color:rgb(23, 43, 77);">5、退出处理</font>
**退出处理是一个指定在 workflow 结束时执行的 template，无论成功或失败。**

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1713838195136-b86c12ee-20d7-4e09-825a-9f22090ec5a8.png)

# <font style="color:rgb(23, 43, 77);">七、参考资料</font>
+ [https://segmentfault.com/a/1190000038979821](https://segmentfault.com/a/1190000038979821)
+ [https://github.com/argoproj/argo-workflows](https://github.com/argoproj/argo-workflows)

  


