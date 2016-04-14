# 构建 F8 2016 App 第一部分：开发计划

这是为了介绍 React Native 和它的开源生态的一个系列教程，我们将以构建 F8 2016 开发者大会官方应用的 iOS 和 Android 版为主题。

在第一部分，我们将介绍我们是如何计划的，在后面的部分，我们将分享示例代码，讨论多平台设计需要考虑的事情，分析应用的数据层，最后解释我们所选择的测试策略。

## 使用 React Native

在 2015 年的 F8 大会，我们使用 React Native 开发了官方应用的 iOS 版，但 Android 版使用的是原生代码；在更早的大会上，我们在两个平台使用的都是原生代码。去年大会之后，React Native 发布了 Android 版本，这给我们提供了机会来削减应用的逻辑和 UI 代码。部分 Facebook 团队在使用 React Native 时发现[达到了 85% 的代码共用](https://code.facebook.com/posts/1189117404435352/react-native-for-android-how-we-built-the-first-cross-platform-react-native-app/)。

React Native 在原型设计方面也有优势，它可以在你和 UI 设计师讨论的时候快速构建视觉组件原型，这些我们将在[第二部分]()讨论。

所以，如果我们选择使用 React Native，还有什么需要考虑的？让我们先从内容开始。

## 选择一个数据层

2014 和 15 年的 F8 应用都使用了[Parse](https://parse.com/)作为数据后端。因此，在我们计划开发 2016 年的应用时，使用 Parse 将允许我们重用现有的数据结构，以及能够快速开发开发。

选择 Parse 也有其它的原因 - 从大会准备到举办的期间，应用所展现的内容需要经常更新，并且内容更新应该像更新表格一样容易，而不需要工程师来协助。Parse 提供的 dashboard 能完美的满足这些需求。

由于以上原因，选择 Parse 作为后端顺理成章。不过，由于 [Parse 宣布将停止服务](http://blog.parse.com/announcements/moving-on/)，我们转而使用开源的 [Parse Server](http://blog.parse.com/announcements/introducing-parse-server-and-the-database-migration-tool/) 和 [Parse Dashboard](https://github.com/ParsePlatform/parse-dashboard) 来替代它。

由于 React Native 并不需要时刻与数据层保持紧密连接，比如 UI 和应用逻辑能使用模拟数据完成开发。这意味着我们可以仅以极少的代价替换整个数据层。比如在开发完 F8 应用后，我们能够非常容易的从 Parse 转移到开源的 Parse Server，我们将在[第三部分]()进一步讨论。

## React Native 应用的数据接入

为了让 Parse 和 React Native 协同工作，我们已经有了 [Parse + React Native package](https://github.com/ParsePlatform/ParseReact) 来提供必需的绑定工具，不过这里有一个问题 - 考虑到会场 wi-fi 和连通性并不总是表现良好，F8 应用需要支持离线工作。因为 Parse + React 并不提供离线数据同步功能，我们只能自己开发。

这里还有另一个决定因素 - 团队大小。比如，提供类似功能的 Relay 更适合大型团队开发大规模应用，但 F8 应用的开发者只有一个人，以及少数其它人提供设计支持。这将极大影响你在开发应用中使用的数据接入方法。

那么 [GraphQL](http://graphql.org/) 和 [Relay](https://facebook.github.io/relay/) 呢？虽然它们与 React Native 工作良好，但[目前为止](https://github.com/facebook/relay/wiki/Roadmap#in-progress)，Relay 不提供内建的离线使用，而 Parse 对 GraphQL 的支持并不完美。使用它们开发，我们需要构建 GraphQL 到 Parse 的 API，并且开发 Relay 的离线存储方法。

GraphQL 服务器的设置对于面临紧迫工期的个人来说也相对复杂。我需要开发应用并发布到应用商店，只能选择最简单并且最快的方式，除此之外我还能怎么做呢？

由于以上原因，[Redux](https://github.com/rackt/redux) 也是应用的一个最佳选择。Redux 提供了 [Flux 架构](https://facebook.github.io/flux/)的一个简单实现，对数据的存储和缓存提供更多控制，并最终让应用能从 Parse 进行单向同步。

对应用的存储部分，Redux 提供了功能性与易用性之间的平衡。在应用发布之后，我们重新梳理了这一部分，并且使用 Relay 和 GraphQL 实现了一遍，我们将在 [Relay 和 GraphQL 附加部分](http://makeitopen.com/tutorials/building-the-f8-app/relay/)讨论这些。

## 我们的开发技术栈

将 React Native 作为我们的应用框架，以及 Redux 作为数据层，我们需要一些其它的支持技术和工具：

* 开源的 [Parse Server](https://github.com/ParsePlatform/parse-server) 提供数据存储 - 运行在 [Node.js](https://nodejs.org/) 上。
* 我们使用了 [Flow](http://flowtype.org/) 来检查 React Native JavaScript 代码中的类型错误。
* [Jest framework](http://facebook.github.io/jest/) 用来对复杂的函数进行单元测试。
* 使用 [React Native Facebook SDK](https://github.com/facebook/react-native-fbsdk) 来快速集成 Facebook 接入功能。
* 使用 OSX 平台的 [Nuclide](http://nuclide.io/) 编辑器，以及它[内建的 React Native 支持](http://nuclide.io/docs/platforms/react-native/)。
* 我们还使用 Git 做版本管理，将开发的过程都存储到了 [Github](https://github.com/fbsamples/f8app) 上。

我们还使用了一些小的附加工具包，当我们在教程中碰到时会重点提到。

在你阅读系列教程的下一章节之前，我们推荐你先学习 [React.js 的官方教程](http://facebook.github.io/react/docs/tutorial.html) - 特别是它的[模块化组件概念](http://facebook.github.io/react/docs/thinking-in-react.html#step-1-break-the-ui-into-a-component-hierarchy)以及 [JSX 语法](http://facebook.github.io/react/docs/jsx-in-depth.html)。然后阅读 [React Native 的引导教程](http://facebook.github.io/react-native/docs/tutorial.html#content)，它将展示将其用于移动应用开发的一些基础知识。

原文链接：http://makeitopen.com/tutorials/building-the-f8-app/planning/