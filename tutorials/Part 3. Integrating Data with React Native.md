# 构建 F8 2016 App 第三部分：React Native的数据交互

这是为了介绍 React Native 和它的开源生态的一个系列教程，我们将以构建 F8 2016 开发者大会官方应用的 iOS 和 Android 版为主题。

[React](http://facebook.github.io/react/) 和 其扩展 [React Native](http://facebook.github.io/react-native/)，允许在你构建应用程序，而无需担心你的数据来自哪里，这样你就可以专注于创建应用的UI和逻辑。

在[第一章](http://makeitopen.com/tutorials/building-the-f8-app/planning/)，我们提到如何采用 Parse Server 来持有数据，在 app 中我们将用 Redux 来处理它。在这一部分，我们将解释 Redux 在 React Native 应用中如何工作，以及连接 Parse Server 的简单过程。

在我们讨论 Redux 之前，让我们先看看 React 的数据交互是如何创建 Redux 的。

## 首先，React 应用如何同数据交互？

在MVC应用程序架构中， React 被经常性的视为 ‘View’ 层，但是这样的说法显然欠妥，React 实际上是对 MVC 模式的重新构想。

让我们先看下 MVC 架构的基本思想：

* 模型层就是数据层。
* 视图层负责整个应用程序数据的展现。
* 控制层在应用程序中负责提供数据处理的逻辑操作。

React 可以做到当你创建多个组件组合形成一个视图的时候，每个组件依然可以处理自有逻辑，且只需要提供一个控制器。

```java
class Example extends React.Component {
    render() {
        // Code that renders the view of the existing data, and
        // potentially a form to trigger changes to that data through
        // the handleSubmit function
    }

    handleSubmit(e) {
        // Code that modifies the data, like a controller's logic
    }
};
```
在一个 React 应用中，每个组件都有两种不同的数据类型，每一个都有不同的角色：

* `props` 当一个组件被创建的时候， `props` 会被作为参数传递给该组件。如果你有一个按钮，默认的 `prop` 将会是该按钮的文本。组件并不能改变它们的 props (即它们是不可变的)。

* `state` 是一种可以被任意组件改变任意次数的数据。如果上面的按钮是登录/注销按钮，那么该 `state` 将记录用户当前的登录状态，并且该按钮也可以访问它，修改它当用户点击了按钮改变其状态。

为了让 React 应用 减少重复， `state` 被设计为组件库中的最高级父组件所拥有，即前面提到过的`container 组件`。换句话说，在上个按钮组件例子中，实际上该按钮并没有拥有该属性，你可以在该按钮的父视图中拥有它，然后使用 `props` 传递相关的 `state` 数据给子组件。正因为如此，数据仅仅流向任何给定的应用，这会使得 React 更快和模块化。

如果你愿意，你可以从 `thinking-in-react` 中阅读更多关于制定这些决策的背后想法和原因。

##存储状态

为了进一步解释在 React 应用中的数据使用技术， Facebook 推荐了 `Flux architecture`,该架构是一种模式来实现你的应用，而不是一种实际的框架供你使用。我们不在在你的应用中使用 `Flux library` ,但是我们会使用 Redux 框架，其源于 Flux architecture，所以让我们深入进去吧。

Flux 通过引入 Stores 概念，应用的 `state` 对象容器，新的工作流用来修改 `state` 等来扩展 React 的 数据关系：

* 在 Flux 应用中每个在 Dispatcher 中注册过的 Store 都有一个回调方法。
* Views (基于 React 组件) 可以触发 Actions ，基本上每个对象，都包含了一堆刚刚发生事情的数据（例如，可能它包括一些将会被输入到应用中的新数据）和 action type (实质上就是 Action 动作的描述类型常量)。
* Action 被发送到 Dispatcher 。
* Dispatcher 传递该 Action 到所有 Store 的注册回调。
* 如果 Store 能告知其被一个 Action影响（因为 action 类型和数据相关），其会自动更新，当然其包含的 `state` 也会被更新。一旦更新，其会发出变化事件。
* 特殊的视图被叫做 Controller Views(container components的优雅术语)将会监听这些变化事件，当其抓获某事件，它们知道应该获取新的 Store 数据。
* 一旦获取到新数据后，它们会调用 `setState()` ，其会致使在该视图里的组件重新 render 。

你可以看到在 React 应用中 Flux 如何帮助 React 强制执行 one-way flow 策略 ，并且其会使得 React 的数据部分更加优雅和有结构。

我们现在不使用 Flux ,所以在这篇文章里不会再有更多该细节，但是如果你想了解更多，这儿有一些在 Flux 网站的[教程](https://facebook.github.io/flux/docs/todo-list.html)可以阅读。

所以 Redux 关 Flux 什么事，在我们应用中实际使用框架 Redux 呢？

## Flux 到 Redux
Redux 是 Flux 架构的框架实现，但是其也可以剥离开来，react-redux package 提供的 official bindings provided 可以让其和 React 应用更方便的融合。

在 Redux 中没有 dispatcher ,并且针对整个应用程序只有一个 Store。

在 Redux中，数据流如何工作，在后续我们有更详细的解释，但是这儿我们先介绍几个基本概念：

* React 组件可以导致 Actions 被触发，例如按钮的点击。
* Actions 是一个通过 `dispatch` 方法 被发送到 Store 的对象（包括一个 `type` 标签和 Action相关的其他数据）。
* 然后 Store 承载着相关 Action ,同当前 `state` 树（ `state` 树是一个单独的对象包含了所有的 `state` 数据）将其发送到 Reducers。
* Reducer 是一个纯函数，它维持着之前的状态和动作，然后返回一个基于任何变化的动作的 `state`。 Redux 应用只能包含一个 Reducer，但是大部分应用最后都会有几个，并且每个都会处理 `state` 的不同部分（我们将讨论这个）
* Store 接收到新的 `state` ，然后替换掉旧的。这里值得注意的是，当我们讨论 `state` 更新时，这其实是技术上的取代。
* 当 `state` 发生变化时， Store 触发[变化事件](http://redux.js.org/docs/api/Store.html#subscribe)。
* 任何[订阅了该变化事件](http://redux.js.org/docs/api/Store.html#subscribe)的 React 组件都会[收到来自 Store 的新 `state`](http://redux.js.org/docs/api/Store.html#getState)。
* 组件随 `state` 而更新。

该工作流可以通过下图简单总结：


![redux_flowchart](http://makeitopen.com/static/images/redux_flowchart.png）

你可以看到数据是如何遵循一个明确的单向通道，没有重叠或相反的方向流动。这也说明 Redux 可以将 app 的每一部分细分。 Store 只关注 `state` ,视图中的组件只关注展示数据和触发事件， Actions 只关注 `state` 的变化和其内部的数据。 Reducers 只关注融合旧的 `state` 和 actions 到新的 `state` 。当你阅读器源代码并理解它时，你会发现一切都是模块化，优雅和具有明确目的的。

Flux 有一些其他的好处：

* Actions 是触发 `state` 改变的唯一方式，且该过程不经过 UI 组件，并且因为 Reducer ，让其很有秩序，远离竞争。
* `state` 变得不可变，并且有一系列的 `state` 。每一个都因为代一个独特的变化而被创建。这让你在应用程序中能更简单和清晰的追踪 `state` 的状态。

##将它们融合在一起

所以现在，我们已经讨论完抽象的数据流，现在让我们来看看我们的 React Native 应用如何使用它们，以及在这个过程中我们学到的东西。

### Store

[Redux 文档](http://redux.js.org/docs/basics/Store.html) 非常好的解释了如何创建简单的 Store,所以我们将会将设您已经阅读那里的基本知识，并跳过一点点内容，包括 Store 的几个额外参数。

#### Store 的离线同步

我们讨论过本地离线存储的必需性，这样应用程序才可以在低信号或无信号条件下工作（在技术会议上尤为重要）。幸运的是，因为我们正在使用 Redux，在我们的 app 中我们可以使用一些很简单的模块，其被叫做 [Redux Persist](https://www.npmjs.com/package/redux-persist) 。

在我们的 Store 中我们也会用到 [中间件](http://redux.js.org/docs/advanced/Middleware.html) ,我们将更多地讨论一些在测试阶段我们用的一些东西。但是总的来说，中间件让你能够在 Action 被分发和到达 Reducer 之前添加一些额外的逻辑操作（这对日志，崩溃报告，异步 APIs 等很有帮助）。

```javascript
/* js/store/configureStore.js */
var createF8Store = applyMiddleware(...)(createStore);

function configureStore(onComplete: ?() => void) {
  const store = autoRehydrate()(createF8Store)(reducers);
  persistStore(store, {storage: AsyncStorage}, onComplete);
  ...
  return store;
}
```
在这里，嵌套函数的语法可能有些让人迷惑（其中的一些函数返回将另一个函数作为参数），所以在这里扩展下：

```javascript
/* js/store/configureStore.js */

// 1
var middlewareWrapper = applyMiddleware(...);
// 2
var createF8Store = middlewareWrapper(createStore(reducers));

function configureStore(onComplete: ?() => void) {
  // 3
  const rehydrator = autoRehydrate();
  const store = rehydrator(createF8Store);
  // 4
  persistStore(store, {storage: AsyncStorage}, onComplete);
  ...
  return store;
}
```

在第一行代码中，我们使用 Redux 的 appluMiddleware() 启动中间件（如果你想知道更多，请阅读 [Redux appluMiddleware 文档](http://redux.js.org/docs/api/applyMiddleware.html) 该函数用于实例 Store 对象）。

所以在第二行代码中，我们我们将 Redux的 createStore() 方法包裹在了 middlewareWrapper . createStore() 方法返回一个 Store 对象 ，middlewareWrapper() 通过中间件扩展它，最终结果保存在 createF8Store 中。

然后我们配置我们的 Store 对象。 Persist's autoRehydrate() 是另外一个扩展 Store方法（和 applyMiddleware() 一样，其返回一个函数），我们更新现有的 Store 对象（第三行代码）。 autoRehydrate() 方法事先将一个 Store 对象保存到本地，然后根据 `state` 自动更新其状态。

第四行的 Persist package's persistStore() （我们使用到了 React Native 的 [异步存储系统](https://facebook.github.io/react-native/docs/asyncstorage.html)） 函数是实际的将 app 的 Store 存储到本地。 autoRehydrate() 和 persistStore()  函数就是我们需要离线同步功能的全部代码。

现在，不管何时应用程序断开网络连接，Store 最近的副本依然会被存储到本地（包括通过 API 来抓取解析的数据），从用户角度来看，我们的应用仍然可以正常工作。
关于更多信息，你可以阅读 [technical details of how the Redux Persist package works](https://www.npmjs.com/package/redux-persist#basic-usage) ,但本质上我们已经完成了构建我们的 Store 。

## Reduers

在上一节，我们解释了 Redux ,我们也因此引入了一个 Reducer 对象。然而在每个应用中，会有很多个 Reducers 对象。每个对象都对应着 `state` 的不同部分。举个例子，在一个可评论的应用中，你可能需要一个 reducer 关联到登录状态，其他的关联到其他的评论数据。

在 F8 应用中，我们将 reducers 存储在 js/reducers 中。这是 user.js 的简写：

```javascript
/* js/reducers/user.js */
...
import type {Action} from '../actions/types';
...

// 1
const initialState = {
  isLoggedIn: false,
  hasSkippedLogin: false,
  sharedSchedule: null,
  id: null,
  name: null,
};

// 2
function user(state: State = initialState, action: Action): State {
  // 3
  if (action.type === 'LOGGED_IN') {
    // 6
    let {id, name, sharedSchedule} = action.data;
    if (sharedSchedule === undefined) {
      sharedSchedule = null;
    }
    return {
      isLoggedIn: true,
      hasSkippedLogin: false,
      sharedSchedule,
      id,
      name,
    };
  }
  if (action.type === 'SKIPPED_LOGIN') {
    return {
      isLoggedIn: false,
      hasSkippedLogin: true,
      sharedSchedule: null,
      id: null,
      name: null,
    };
  }
  // 4
  if (action.type === 'LOGGED_OUT') {
    return initialState;
  }
  // 5
  if (action.type === 'SET_SHARING') {
    return {
      ...state,
      sharedSchedule: action.enabled,
    };
  }
  if (action.type === 'RESET_NUXES') {
    return {...state, sharedSchedule: null};
  }
  return state;
}

module.exports = user;
```
正如你所知， 这个 reducer 包含了 登录/登出 操作，以及特定的用户参数修改。让我们一点一点分析。

注意：在第六段，我们使用了 ES2015 的 destructuring assignment ，其将左边的变量分配给 `action.data`

1. 初始状态

在我们定义初始状态之前，这符合流动式命名习惯（我们将会在 React Native 应用的测试中详细解释）。 `initialState` 定义了 `state` 的一部分值，该值被 Reducer 处理，应用会首次加载它或者以前在其上的任何同步存储。

2. Reducer 方法

然后，我们编写 Reducer 的实例，其实相对来说还是很简单。 `state` 和 Action (之后我们会讨论) 被作为参数， `initialState` 作为 `state` 的默认值（我们将参数使用流式类型注解，同样的，我们会在 React Native 应用的测试部分提到）。然后，我们使用接收到的 Action ,具体为 ‘type’ 标签，返回一个新的改变了的 `state` 。

举个例子，如果第四段的 `LOGGED_OUT` Action 被分派（由于用户点击了登出按钮），我们将会重置 `state` 的一部分值为 `initialState` 。 如果第三段  `LOGGED_IN` Action 发生，你会看到应用将会使用余下数据，返回新的 `state` ，该 `state` 和 常规变化例如 `isLoggedIn`,或者用户输入数据变化 例如 `name` 都将冲突。

还有个方法我们也可以看下，这便是第五段的 `SET_SHARING` Action 类型。 其非常有趣因为 `...state` 注解被使用。在 React 中，注解让对象的分配更具兼容性和可读性，并且其会创建一个对象，然后拷贝已经存在的 `state` ,最后单独更新 `sharedSchedule` 的值。

你可以看到 Reducer 的结构是如此的简单和可读 - 定义一个 `initialState` ，构建一个传入 `state` 和 Action ,返回一个新的 `state` 的函数。

reducers 还有一个大的规则，我们引用 [Redux 文档](http://redux.js.org/docs/basics/Reducers.html#handling-actions)的内容：

> "记住，reducer 必须是单纯的。提供相同的参数，其应该计算下一个状态然后返回它。没有意外，没有副作用，没有 API 调用，没有突变，只有计算。"

另外一件事需要注意的是：看看 `js/reducers/notifications.js` ,其有针对 `LOGGED_OUT` 动作类型的另一个引用。我们之前提到过，但是现在依然需要重复下 - 每一个 reducer 通常是在一个动作被派发后被调用，所以多个 reducers 可能会基于相同的动作来更新不同的 `state` 。

* Actions

让我们仔细看下登录相关的动作，看看它们的代码位置：

```javascript
/* from js/actions/login.js */
function skipLogin(): Action {
  return {
    type: 'SKIPPED_LOGIN',
  };
}
```

这是一个简单的Action creator (这个 creator 函数返回的对象实际上是一个 Action)，但是这让你看到了最基本的结构 - 每个动作可以简单的视为一个包含了 `type` 标签的对象。然后 reducers 可以使用该 `type` 来更新 `state` 。

同样，我们可以为 `type` 添加一些数据：

```javascript
/* from js/actions/filter.js */
function applyTopicsFilter(topics): Action {
  return {
    type: 'APPLY_TOPICS_FILTER',
    topics,
  };
}
```

在这儿，动作创造者接收了一个参数，并且将其插入到了动作对象中。

然后，我们有一些动作制造者会执行额外的逻辑以及返回 Action 对象。在这个例子中，我们也会使用到一个叫做 ThunkAction 的 动作（[Redux 建议你创建点类似这样的东西来减少模板](http://redux.js.org/docs/recipes/ReducingBoilerplate.html)） - 这种特殊类型的动作制造者返回的是一个函数而不是一个动作。在这种情况下，登出动作制造者返回一个执行一些登出相关的逻辑函数，然后分配到一个 Action 。

```javascript
/* from js/actions/login.js */
function logOut(): ThunkAction {
  return (dispatch) => {
    Parse.User.logOut();
    FacebookSDK.logout();
    ...

    return dispatch({
      type: 'LOGGED_OUT',
    });
  };
}
```

(注意，在这个例子中，我们也会使用到 [Arrow function](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Functions/Arrow_functions) 语法)

### 异步动作

举个例子，如果你同任何 APIs 交互，你需要一些异步的动作制造者。 Redux 在这方面有个相当复杂的方式来实现异步，但是由于我们在使用 React Native , 我们可以使用到 ES7 的 `await` 函数，这极大简化了异步过程：

```JavaScript
/* from js/actions/config.js */
async function loadConfig(): Promise<Action> {
  const config = await Parse.Config.get();
  return {
    type: 'LOADED_CONFIG',
    config,
  };
}
```
在这里，我们使用 一个 API 调用去解析去抓取一些应用程序的配置参数。任何像这样的 API 调用获取网络资源都需要耗费一定的时间。动作不会被立即分配，动作创造者首先等待 API 调用的结果，然后一旦数据有效，动作对象（负载着 API 数据）会被立即返回。

其异步调用的一个好处是，由于我们在等待 Parse.Config 调用的结果，其他的异步操作可以做其自己的工作，这样我们可以同时又多个操作，其会自动提高效率。

## 绑定到组件

现在，在我们应用的 setup 函数中链接 Redux 逻辑到 React ：

```javascript
/* from js/setup.js */
function setup(): React.Component {
  // ... other setup logic

  class Root extends React.Component {
    constructor() {
      super();
      this.state = {
        isLoading: true,
        store: configureStore(() => this.setState({isLoading: false})),
      };
    }
    render() {
      if (this.state.isLoading) {
        return null;
      }
      return (
        // 1
        <Provider store={this.state.store}>
          <F8App />
        </Provider>
      );
    }
  }

  return Root;
}
```

我们使用官方的 [React 和 Redux 绑定](https://github.com/reactjs/react-redux), 因此我们可以使用 [<Provider> 组件](http://redux.js.org/docs/basics/UsageWithReact.html#passing-the-store) 。 这个 Provider 让我们的 Store 可以和任何我们创建的组件通信：

```javascript
/* from js/F8App.js */
var F8App = React.createClass({
  ...
})

// 1
function select(state) {
  return {
    notifications: state.notifications,
    isLoggedIn: state.user.isLoggedIn || state.user.hasSkippedLogin,
  };
}

// 2
module.exports = connect(select)(F8App);
```
在上面，我们展示了一部分 <F8App> 组件的代码 - 我们整个应用的父组件。

上面第一个方被用来获取 Redux Store，然后从中获取一些数据，针对我们的 `<F8App>` 组件插入到 `props`。在这种情况下，我们希望通知的数据和用户登录状态的数据成为组件的 `props`，这样其会根据每一次 Store 的改变而更新状态。

我们可以使用 React-Redux 的 [`connect()` 方法](https://github.com/reactjs/react-redux/blob/master/docs/api.md#connectmapstatetoprops-mapdispatchtoprops-mergeprops-options)来实现它 - `connect()` 有一个参数叫做 [`mapStateToProps`](https://github.com/reactjs/react-redux/blob/master/docs/api.md#arguments)，其会负载一个方法，当一个 Store 更新，该方法就会被调用。

所以当我们应用的 Store 更新时，`select()` 将会和被作为参数的新 `state` 一起被调用。`select()`返回一个包含了我们想从新 `state`中获取的数据（自在这个例子中，是`notifications`和`isLoggedIn`）,然后 第二段的`connect()` 融合数据到  `<F8App>` 组件的 `props`。

```javascript
/* from js/F8App.js */
var F8App = React.createClass({
  ...
  componentDidMount: function() {
    ...
    // 1
    if (this.props.notifications.enabled && !this.props.notifications.registered) {
      ...
    }
    ...
  },
  ...
})
```  

现在我们有个 `<F8App>` 组件，当任何新的已经通过 `select()` 方法订阅了的 `state`数据，都会被更新，其可以通过我们的 props 访问。但是我们如何从一个组件里分派动作？

### 从组件中分派动作

为了知道我们如何连接动作到组件，让我们看看 `<GeneralScheduleView>` 相关部分：

```javascript
/* from js/tabs/schedule/GeneralScheduleView.js */
class GeneralScheduleView extends React.Component {
  props: Props;

  constructor(props) {
    super(props);
    this.renderStickyHeader = this.renderStickyHeader.bind(this);
    ...
    this.switchDay = this.switchDay.bind(this);
  }

  render() {
    return (
      <ListContainer
        title="Schedule"
        backgroundImage={require('./img/schedule-background.png')}
        backgroundShift={this.props.day - 1}
        backgroundColor={'#5597B8'}
        data={this.props.data}
        renderStickyHeader={this.renderStickyHeader}
        ...
      />
    );
  }

  ...

  renderStickyHeader() {
    return (
      <View>
        <F8SegmentedControl
          values={['Day 1', 'Day 2']}
          selectedIndex={this.props.day - 1}
          selectionColor="#51CDDA"
          onChange={this.switchDay}
        />
        ...
      </View>
    );
  }

  ...

  switchDay(page) {
    this.props.switchDay(page + 1);
  }
}

// 1

module.exports = GeneralScheduleView;
```

同样，这段代码已经为了便于学习很大程度上简化了，但是我们现在可以在第一段添加和修改一些代码来连接这个 容器组件到 Redux store：

```javascript
/* from js/tabs/schedule/GeneralScheduleView.js */
function select(store) {
  return {
    day: store.navigation.day,
    data: data(store),
  };
}

function actions(dispatch) {
  return {
    switchDay: (day) => dispatch(switchDay(day)),
  }
}

module.exports = connect(select, actions)(GeneralScheduleView);

```

这一次有一些不同 - 我们提供了 React-Redux 的 `connect()` 方法。总的来说，执行 `actions()` 内部的动作创造者到组件的 props,然后将其包裹在 `dispatch()`方法，这样它们才能立即分发一个 Action 。

### 如何工作

让我们看看实际的组件:

![](http://makeitopen.com/static/images/redux_flowchart.png）

  利用 ’Day 1' 在 `renderStickyHeader()` 中触发 `onChange` 事件，然后在组件内部的 `switchDay()` 方法被调用，该方法分发 `this.props.switchDay()` 动作创造者。在我们的动作文件内，我们可以看到这样一个动作创造者：

  ```javascript
  /* from js/actions/index.js */
  switchDay: (day): Action => ({
    type: 'SWITCH_DAY',
    day,
  })
  ```

在导航 Reducer 内部，我们可以看到一个修改的 `day` 值产生一个新的 `state` 树：

```javascript
/* from js/reducers/navigation.js */
  if (action.type === 'SWITCH_DAY') {
    return {...state, day: action.day};
  }
```

这个 Reducer（以及任何其他那些可能监视 `SWITCH_DAY` 动作的 Reducers ）返回新的 `state` 到 Store中，其更新自己并且发送一个改变事件。

而由于通过连接 `<GeneralScheduleView>` 连接到 Redux Store,我们还订阅了 Store 的变化状态。

## 什么是解析服务器

在这个教程中，你希望你能够获取到大量新的信息，所以让我们快速展示我们如何连接 React Native 应用到我们的解析服务器数据后端，以及相关 API:

```javascript
Parse.initialize(
    'PARSE_APP_ID',
    'PARSE_JAVASCRIPT_KEY'
  );
  Parse.serverURL = 'http://exampleparseserver.com:1337/parse'
```
就这样，在 React Native 中我们通过 解析 API 建立连接。

是的，因为我们现在使用的是 [Parse + React](https://github.com/ParsePlatform/ParseReact) SDK ,我们有非常简单的SDK接入。

### 解析和动作

当然，我们希望能够查询（例如，在我们的Actions中）......很多的查询。对于那些动作创造者来说没什么特别的，它们是我们之前提到的相同的异步操作。然而，因为有非常多的简单的 Parse API 查询需要初始化应用，我们想大量减少样板。在我们的基本动作文件中，我们创建基本的动作构建者：

```javascript
/* from js/actions/index.js */
function loadParseQuery(type: string, query: Parse.Query): ThunkAction {
  return (dispatch) => {
    return query.find({
      success: (list) => dispatch(({type, list}: any)),
      error: logError,
    });
  };
}
```
然后我们就可以简单的多次复用该代码：

```java
loadMaps: (): ThunkAction =>
  loadParseQuery('LOADED_MAPS', new Parse.Query(Maps)),
```

`loadMaps()` 变成了一个动作创建器，其将会运行一个简单的解析查询针对所有存储的地图数据，然后将其单独传递当查询结束时。`loadMaps()` 以及其他解析数据操作的动作可以在整个应用的 `componentDidMount（）` 方法中找到，这意味着当应用第一次打开时，其需要拉去所有的解析数据。

### 解析和 Reducers

我们已经减少了重复的动作，但是同时我们也想减少在我们 Reducers 内部的模板。这些模板将会从 动作负载中收到一些解析 API 的数据，并且必须将其映射到 `state`。我们针对解析数据创建单个基本 Reducer ：

```javascript
/* from js/reducers/createParseReducer.js */
function createParseReducer<T>(
  type: string,
  convert: Convert<T>
): Reducer<T> {
  return function(state: ?Array<T>, action: Action): Array<T> {
    if (action.type === type) {
      // Flow can't guarantee {type, list} is a valid action
      return (action: any).list.map(convert);
    }
    return state || [];
  };
}
```

这是个简单的 Reducer (有大量流式类型注解)，但是让我们看看其如何同基于关闭它的子 Reducers工作。

```javascript
/* from js/reducers/faqs.js */
const createParseReducer = require('./createParseReducer');

export type FAQ = {
  id: string;
  question: string;
  answer: string;
};

function fromParseObject(map: Object): FAQ {
  return {
    id: map.id,
    question: map.get('question'),
    answer: map.get('answer'),
  };
}

module.exports = createParseReducer('LOADED_FAQS', fromParseObject);

```

所以不用每次去重复创建一份 `createParseReducer` 代码，我们只需要简单的传递一个对象给基本的 Reducer
，其会将 API 数据映射到我们的 `state` 上。

现在，我们的应用，拥有一个结构良好且易于理解的数据流，可以连接到我们的解析服务器，甚至能够离线同步我们的 Store 到本地存储。
