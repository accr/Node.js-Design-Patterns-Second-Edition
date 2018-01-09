# Writing Modules
`Node.js`模块系统弥补了原生`JavaScript`缺乏把代码组织到不同独立单元的这一缺陷。模块系统最大的优点就是能够使用`require()`函数将模块链接在一起，这是一种简单而强大的方法。但是，对于许多新的`Node.js`的开发人员可能会对模块系统的使用产生疑问。实际上，最常见的问题之一是：将组件X的实例传递到模块Y的最佳方式是什么？

有时候，这种疑问可能导致我们滥用单例模式，因为希望找到一种更熟悉的方式来将我们的模块链接在一起。另一方面，我们可能滥用依赖注入模式，利用它来处理任何类型的依赖（甚至无状态）。如果说如何组织模块是`Node.js`中最具争议性和观点性的话题之一应该不足为奇了。主流的组织模块方式很多，但没有任意一个观点处于主导地位。但实际上，每种方法都有其优点和缺点。

在本章中，我们将分析组织模块的各种方法，并强调它们的优缺点，以便我们能够在简单性，可重用性和可扩展性之间平衡，合理地选择和混用这些模块组织方式。具体来说，我们将介绍一些模式，如下所示：

* 硬编码依赖
* 依赖注入
* 服务定位器
* 依赖注入容器

然后，我们将探讨一个与书写模块密切相关的问题，即如何组织`Node.js`插件模块。对于这个问题，大多数书写插件模块的方式都差不多，但是与用户自己编写的应用程序模块的组织就不太相同了，特别是当插件作为单独的`Node.js`包分发时，问题就十分明显了。 

我们将学习如何构建一个`Node.js`插件，并如何把这些插件集成到主应用程序中。

在本章最后，对于`Node.js`如何组织模块就不再是晦涩难懂的话题了。

## 模块和依赖
每个应用程序都是多个模块组织在一起的结果，如同盖楼一样，随着应用程序日益迭代复杂，我们组织模块的方式将导致应用程序的成功或失败。这不仅与应用程序的拓展性相关，还是我们构建大型系统的重点关注点。过于复杂紊乱的模块依赖是一种灾难，它增加了我们项目的组织难度，在这种情况下，代码的任何修改和拓展都将会使我们付出巨大的代价。

最糟糕的情况是，这些模块严重耦合，导致我们不重写整个应用程序就不更改代码的任何一部分。当然，不必害怕，我们并不用从写第一个模块开始就开始全面规划我们的模块。但只要我们遵循应有的模式，就不会出现这样的问题。

`Node.js`提供了一个很好的工具来连接和组织应用程序。那就是`CommonJS`模块系统。但是，使用模块系统并不能够保证我们我们一定能解决模块依赖的问题，如果使用不当，将会使得耦合变得更加严重。在本节中，我们将讨论书写`Node.js`模块的基本模式。

### Node.js最常见的依赖
在一个软件体系结构中，我们在设计其的过程中就应该考虑到可能影响其中任何一个组件依赖关系的实体、状态、数据格式。例如，一个组件可能使用另一个组件的提供的服务，也可能依赖系统特定的一个全局状态，或者实现一个特定的通信协议，以便与其他组件交换信息等等。依赖的概念十分广泛，有时会显得难以评估。

但是，在`Node.js`中，我们可以确定一个最常见也最容易识别的最基本的依赖模型。当然，当我们在讨论模块之间的依赖关系，我们应该首先明确：模块是我们组织和构建代码的基本机制。不依赖模块系统构建的大型应用程序是十分不合理的。如果使用正确的方式来组织应用程序的各个模块单元，它会带来很多好处。实际上，一个模块的属性可以概括如下：

* 一个模块应该具有可读性和可理解性，因为它应该专注于一件事
* 一个模块被表示为一个单独的文件，使得其更容易被识别
* 模块可以更容易地在不同的应用程序中复用

一个模块代表的是一个完全私有的命名空间，并通过`module.exports`来公开访问这个模块的接口。

但是，对于一个成功的模块设计，只是简单地将应用程序或库的功能区分为不同的模块是完全不够的。最常见的错误会出现在我们创建了一个过于复杂的模块，那么想要替换或更改这个模块会对整个应用的架构产生巨大的影响。这时就能够意识到把代码组织成模块的优势了。我们需要在模块设计中找到一个平衡点。

### 内聚与耦合
评判创建的模块平衡性两个最重要的特征就是内聚度和耦合度。这两个特征可以应用于软件体系结构中的任何类型的组件或子系统。因此在构建`Node.js`模块时也可以把这两个特征作为重要的参考价值。这两个属性定义如下：

* 内聚度：用于度量模块内部功能之间的相关性。例如，对于一个只做一件事的模块，其中的所有部件都只对这一件事起作用，那说明这个模块具有很高的内聚度。举个例子，那种包含把任何类型的对象存储到数据库的函数内聚度就较低，如`saveProduct()`、`saveInvoice()`、`saveUser()`等。

* 耦合度：评判模块对系统其他模块的依赖程度。例如，当一个模块直接读取或修改另一个模块的数据时，该模块与另一个模块紧密耦合。另外，通过全局或共享状态交互的两个模块紧密耦合。另一方面，仅通过参数传递进行通信的两个模块耦合度较低。

理想情况下，一个模块应该具有较高的内聚度和较低的耦合度，这样的模块更易于理解、重用和扩展。

### 有状态模块