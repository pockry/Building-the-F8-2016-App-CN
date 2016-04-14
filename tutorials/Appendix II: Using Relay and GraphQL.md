# 构建 F8 2016 App 附录 II 使用Relay和GraphQL

在我们最初构建app时，[我们讨论了对数据层的选择](http://makeitopen.com/tutorials/building-the-f8-app/planning/#data-access-with-react-native)。并将我们最终使用的[Redux](https://github.com/rackt/redux)框架与Facebook的开源框架[Relay](https://facebook.github.io/relay/)进行了对比。

当时我们选择了Redux，是因为与Relay相比，Redux提供了更便捷的数据层实现，且与Parse的数据存储可以更快更容易的结合使用。

F8的IOS和Android的应用开发完成上线后。我们再次回顾对数据层的框架选择，想尝试在我们的App中如何使用Relay。

## 缓慢的转变

在传统原生App的开发中，更换数据层意味着整个应用的推翻重写，已有的功能都会被全部替换。

React Native则不同。替换一个独立的View的数据层时，已有的绝大部份数据设置逻辑（Redux, Parse, 和相关的绑定），我们都可以保留。不需要重写或大量重构，我们可以只把app中的某个模块的数据层替换为新的数据层，其余模块继续使用已有数据层。

需要说明的是，这种逐步改善app的能力可以极大的降低维护和升级的开销，对应用的开发非常有利。

那么相比Redux，使用Relay和GraphQL如果处理数据层呢？

## 介绍Relay和GraphQL

首先，简单的说，[Relay](https://facebook.github.io/relay/)是app中的一个数据框架。[GraphQL](http://graphql.org/)是Relay中表示数据模型的查询语言。GraphQL运行在服务器上，与app相隔离，给Relay提供交互所需要的数据源(我们之后的教程中会介绍GraphQL服务器的搭建，敬请期待)。

Relay不是Flux构架的衍生物，且只能和GraphQL配合使用。这就直接意味着和Redux模型完全彻底不同。我们在[数据层教程](http://makeitopen.com/tutorials/building-the-f8-app/data/)中已经介绍的Store/Reducer/Component交互在Relay中是不存在的。它采用一种不一样的方法，并且移除了你和数据交互时需要做的一些构建工作。

在Relay中，各个React component具体依赖样的数据，取决于GraphQL的使用。Relay处理一切有关数据获取，当数据改变后更新组件，客户端的数据缓存的事情。当客户端需要修改数据的时候，创建一个 [GraphQL Mutation](https://facebook.github.io/relay/docs/guides-mutations.html#content) 而不是像使用Redux时创建一个Action.

## F8 App中的一个的例子

鉴于具有逐渐改变React Native应用的一小部分的能力，我们选择把F8 app的Info View这个模块中的Redux替换为Relay,作为一种概念证明.

![Info view of F8 iOS app](http://makeitopen.com/static/images/info_view.png)

Info这个模块在app中和其余模块几乎时彻底分离的，而且大量内容都是非交互性的，用这个模块是最佳选择。

Info View中包含一个非常简单的 <InfoList> 组件：
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

这只是包含一些简单信息展示类组件的基本布局，但组件中的props和参数从哪来的呢？哈哈，在这个js文件中，我们有一个连接GraphQL的fragment：

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

这里，我们用一个GraphSQL fragment定义了<InfoList>组件需要展示的数据。它和我们在GraphQL服务器上定义的GraphQL对象是对应的：

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

你可以看到数据是如何从GraphQL服务器上被获取的，然后Relay开始接手所有需要的且在Fragment中指定的数据。这时｀viewer｀的参数就可以被<InfoList>组件访问到了，使用了一些[destructuring assignments](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment)，反过来构建了在组件中使用的`config`，`faq`，`pages`变量。

归功于Relay的内建逻辑，我们不用担心数据变化的监听，或者数据本地储存等等。我们只需要告诉Relay 组件中应该有什么数据，然后以标准的React方式来设计我们需要的组件。如果GraphQL服务器已经搭建好，以上就是我们需要做的所有事情。

Info View中没有数据的变化，然而如果你想了解更多原理，请阅读在[Relay在mutations上面的文档](https://facebook.github.io/relay/docs/guides-mutations.html#content)。




