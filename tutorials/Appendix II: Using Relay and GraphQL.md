# 构建 F8 2016 App 附录 II 使用 Relay 和 GraphQL

在我们最初构建 app 时，[我们讨论了对数据层的选择](http://makeitopen.com/tutorials/building-the-f8-app/planning/#data-access-with-react-native)。并将我们最终使用的 [Redux](https://github.com/rackt/redux) 框架与 Facebook 的开源框架 [Relay](https://facebook.github.io/relay/) 进行了对比。

当时我们选择了 Redux，是因为与 Relay 相比，Redux 提供了更便捷的数据层实现，且与 Parse 的数据存储可以更快更容易的结合使用。

F8 的 IOS 和 Android 的应用开发完成上线后。我们再次回顾对数据层的框架选择，想尝试在我们的 App 中如何使用 Relay。

## 缓慢的转变

在传统原生 App 的开发中，更换数据层意味着整个应用的推翻重写，已有的功能都会被全部替换。

React Native 则不同。替换一个独立的View的数据层时，已有的绝大部份数据设置逻辑（Redux, Parse, 和相关的绑定），我们都可以保留。不需要重写或大量重构，我们可以只把 app 中的某个模块的数据层替换为新的数据层，其余模块继续使用已有数据层。

需要说明的是，这种逐步改善 app 的能力可以极大的降低维护和升级的开销，对应用的开发非常有利。

那么相比 Redux，使用 Relay 和 GraphQL 如果处理数据层呢？

## 介绍 Relay 和 GraphQL

首先，简单的说，[Relay](https://facebook.github.io/relay/) 是 app 中的一个数据框架。[GraphQL](http://graphql.org/) 是 Relay 中表示数据模型的查询语言。GraphQL 运行在服务器上，与 app 相隔离，给 Relay 提供交互所需要的数据源(我们之后的教程中会介绍 GraphQL 服务器的搭建，敬请期待)。

Relay 不是 Flux 构架的衍生物，且只能和 GraphQL 配合使用。这就直接意味着和 Redux 模型完全彻底不同。我们在[数据层教程](http://makeitopen.com/tutorials/building-the-f8-app/data/)中已经介绍的 Store/Reducer/Component 交互在 Relay 中是不存在的。它采用一种不一样的方法，并且移除了你和数据交互时需要做的一些构建工作。

在 Relay 中，各个 React component 具体依赖样的数据，取决于 GraphQL 的使用。Relay 处理一切有关数据获取，当数据改变后更新组件，客户端的数据缓存的事情。当客户端需要修改数据的时候，创建一个 [GraphQL Mutation](https://facebook.github.io/relay/docs/guides-mutations.html#content) 而不是像使用 Redux 时创建一个 Action.

## F8 App 中的一个的例子

鉴于具有逐渐改变 React Native 应用的一小部分的能力，我们选择把 F8 app 的 Info View 这个模块中的 Redux 替换为 Relay,作为一种概念证明.

![Info view of F8 iOS app](http://makeitopen.com/static/images/info_view.png)

Info 这个模块在 app 中和其余模块几乎时彻底分离的，而且大量内容都是非交互性的，用这个模块是最佳选择。

Info View 中包含一个非常简单的 `<InfoList>` 组件：
```javascript
/* from js/tabs/info/F8InfoView.js */
function InfoList({viewer: {config, faqs, pages}, ...props}) {
  return (
    <PureListView
      renderEmptyList={() => (
        <View>
          <WiFiDetails
            network={config.wifiNetwork}
            password={config.wifiPassword}
          />
          <CommonQuestions faqs={faqs} />
          <LinksList title="Facebook pages" links={pages} />
          <LinksList title="Facebook policies" links={POLICIES_LINKS} />
        </View>
      )}
      {...props}
    />
  );
}
```

这只是包含一些简单信息展示类组件的基本布局，但组件中的 `props` 和参数从哪来的呢？哈哈，在这个 js 文件中，我们有一个连接 GraphQL 的 `fragment`：

```javascript
/* from js/tabs/info/F8InfoView.js */
InfoList = Relay.createContainer(InfoList, {
  fragments: {
    viewer: () => Relay.QL`
      fragment on User {
        config {
          wifiNetwork
          wifiPassword
        }
        faqs {
          question
          answer
        }
        pages {
          title
          url
          logo
        }
      }
    `,
  },
});
```

这里，我们用一个 GraphSQL `fragment` 定义了 `<InfoList>` 组件需要展示的数据。它和我们在 GraphQL 服务器上定义的 GraphQL 对象是对应的：

```javascript
/* from server/schema/schema.js */
var F8UserType = new GraphQLObjectType({
  name: 'User',
  description: 'A person who uses our app',
  fields: () => ({
    id: globalIdField('User'),
    name: {
      type: GraphQLString,
    },
    ...
    faqs: {
      type: new GraphQLList(F8FAQType),
      resolve: () => new Parse.Query(FAQ).find(),
    },
    pages: {
      type: new GraphQLList(F8PageType),
      resolve: () => new Parse.Query(Page).find(),
    },
    config: {
      type: F8ConfigType,
      resolve: () => Parse.Config.get(),
    }
  }),
  ...
});
```

你可以看到数据是如何从 GraphQL 服务器上被获取的，然后 Relay 开始接手所有需要的且在 `fragment` 中指定的数据。这时 `viewer` 的参数就可以被 `<InfoList>` 组件访问到了，使用了一些 [destructuring assignments](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment)，反过来构建了在组件中使用的 `config`， `faq`，`pages` 变量。

归功于 Relay 的内建逻辑，我们不用担心数据变化的监听，或者数据本地储存等等。我们只需要告诉 Relay 组件中应该有什么数据，然后以标准的 React 方式来设计我们需要的组件。如果 GraphQL 服务器已经搭建好，以上就是我们需要做的所有事情。

我们的 Info View 中没有数据的变化，然而如果你想了解更多原理，请阅读在 [Relay 在 mutations 上面的文档](https://facebook.github.io/relay/docs/guides-mutations.html#content)。

原文地址：http://makeitopen.com/tutorials/building-the-f8-app/relay/
