# 构建  F8  2016  App （四）：测试
这是为了介绍  React Native  和它的开源生态的一个系列教程，我们将以构建 F8 2016 开发者大会官方应用的  iOS  和  Android  版为主题。

在传统软件开发生命周期里，测试环节通常往往仅仅是作为一个临近开发结束时才开始进行的特殊环节。由于新推出开源框架发布时往往并未有任何相关测试技术支持，当使用这些框架进行开发时，这种说法便更接近事实真相。

幸运的是， Facebook 在 React Native 构建之初很早的就考虑到了测试策略。在这部分我们将介绍编码阶段如何使用  [Nuclide][1] ,  [Flow][2] ,  以及 [Jest][3] 改善 React Native 代码质量。

## Flow : 利用类型检查避免编写糟糕代码
[Flow][2]为JavaScript 提供以渐进方式工作的[静态类型检查][4]，允许我们将 Flow 特性逐步添加到代码中。这是个非常有用的设计。当我们仅仅想为部分代码引入类型检查时， 我们不必为兼容 Flow 而去重写整个 app 。

我们决定从一开始就完全采用 Flow 来配合 React Native 搭建 F8app，在一切必要的地方添加[类型注解(type annotations )][5] ，让 Flow 能在整个代码开发过程中一直尽情发挥力量。

下面我们以曾经在[数据教程][6]部分曾提及的一个简单 action 作为示例：
```javascript
/* from js/actions/login.js */

/*
 * @flow
 */

...

function skipLogin(): Action {
  return {
    type: 'SKIPPED_LOGIN',
  };
}
```
我们在文件顶部添加 @Flow 标签（[通知 Flow 需检查此段代码][7]）。我们使用 Flow 的[类型注解][5] 限定 skipLogin() 返回值类型必须是 type Action ，由于该Action类型并未在 React Native 及 Redux 里内置，因此在这里我们需要自己对该类型进行定义：
```javascript
export type Action =
    { type: 'LOADED_ABOUT', list: Array<ParseObject> }
  | { type: 'LOADED_NOTIFICATIONS', list: Array<ParseObject> }
  | { type: 'LOADED_MAPS', list: Array<ParseObject> }
  | { type: 'LOADED_FRIENDS_SCHEDULES', list: Array<{ id: string; name: string; schedule: {[key: string]: boolean}; }> }
  | { type: 'LOADED_CONFIG', config: ParseObject }
  | { type: 'LOADED_SESSIONS', list: Array<ParseObject> }
  | { type: 'LOADED_SURVEYS', list: Array<Object> }
  | { type: 'SUBMITTED_SURVEY_ANSWERS', id: string; }
  | { type: 'LOGGED_IN', data: { id: string; name: string; sharedSchedule: ?boolean; } }
  | { type: 'RESTORED_SCHEDULE', list: Array<ParseObject> }
  | { type: 'SKIPPED_LOGIN' }
  | { type: 'LOGGED_OUT' }
  | { type: 'SESSION_ADDED', id: string }
  | { type: 'SESSION_REMOVED', id: string }
  | { type: 'SET_SHARING', enabled: boolean }
  | { type: 'APPLY_TOPICS_FILTER', topics: {[key: string]: boolean} }
  | { type: 'CLEAR_FILTER' }
  | { type: 'SWITCH_DAY', day: 1 | 2 }
  | { type: 'SWITCH_TAB', tab: 'schedule' | 'my-schedule' | 'map' | 'notifications' | 'info' }
  | { type: 'TURNED_ON_PUSH_NOTIFICATIONS' }
  | { type: 'REGISTERED_PUSH_NOTIFICATIONS' }
  | { type: 'SKIPPED_PUSH_NOTIFICATIONS' }
  | { type: 'RECEIVED_PUSH_NOTIFICATION', notification: Object }
  | { type: 'SEEN_ALL_NOTIFICATIONS' }
  | { type: 'RESET_NUXES' }
  ;
```
这里我们创建了 [Flow 类型别名（ Flow type alias ）][8]，限定了 type Action 的样式范围。比如，SKIPPED_LOGIN Action 仅能包含一个自己的type标签， LOADED_SURVEYS Action 则会返回 type 标签以及一个 list 列表。相关 Action Creator 如下：
```javascript
/* from js/actions/surveys.js */
async function loadSurveys(): Promise<Action> {
  const list = await Parse.Cloud.run('surveys');
  return {
    type: 'LOADED_SURVEYS',
    list,
  };
}
```
我们在 app 里使用了大量不同 Action ，强类型检查除了会帮助我们发现类似 type 标签拼写错误这样的低级错误，还会发现如数据格式错误这样的比较重要的问题。

我们在 Reducers 也进行了一样的强类型检查：mp
```javascript
/* from js/reducers/surveys.js */
function surveys(state: State = [], action: Action): State {
  if (action.type === 'LOADED_SURVEYS') {
    return action.list;
  }
  ...
  return state;
}
```
由于 **action** 参数被指定为与前面一样的 **Action** 类型，因此 Reducer 函数必须使用一个有效的 **action.type** 。我们通过类型别名为 Reducer **state** 树选项定义样式:
```javascript
/* from js/reducers/user.js */
export type State = {
  isLoggedIn: boolean;
  hasSkippedLogin: boolean;
  sharedSchedule: ?boolean;
  id: ?;
  name: ?string;
};

const initialState = {
  isLoggedIn: false,
  hasSkippedLogin: false,
  sharedSchedule: null,
  id: null,
  name: null,
};

function user(state: State = initialState, action: Action): State {
  ...
}
```  
我们在[数据教程][6]部分曾为你展示过 **initialState** ，现在你会看到，我们是怎样保证 **state** 树选项与 Flow 定义的类型相一致。当 Reducer 发送或者尝试返回任何不符合定义样式的 **state** 时，都会发生 Flow 类型检查错误。

请注意， Folw 检查仅在编译阶段运行， Recat Native packger 则会[自动去除][9]-这意味着在代码中使用 Flow 不会造成任何执行性能损失。

当然，目前每次想对某些代码进行测试时，我们仍然需要手动运行 Flow 类型检查（通过[ Flow 命令行接口][10]）。不过我们可以通过 Nuclide 在编码时进行这样的查验。
##Nuclide：React Native 开发环境
[Nuclide 网站][11]上有为 React Native 提供的定制功能的全面介绍。我必须得说， Neculite 是专为 React Native 的 Facebook 研发团队以及 Facebook app 专业研发人员而开发的顶级 React Native IDE 。

Nuclite 对 Flow 的集成格外让我们感兴趣。在这里让我们看看 user Reducer 的一段代码：
```javascript
if (action.type === 'SKIPPED_LOGIN') {
    return {
      isLoggedIn: false,
      hasSkippedLogin: true,
      sharedSchedule: null,
      id: null,
      name: null,
    };
  }
```
前面有提到，我们为 Reducer 函数定义了返回值样式。在 Nuclide ，我们可以实时看到错误发生：

<video width="1174" height="1002" autoplay="" loop="">
  <source src="static/videos/flow.mp4" type="video/mp4" />
  Your browser does not support the HTML5 video tag.
</video>

若我们有遗漏了 State 类型的任何部分，我们会瞬间收到内容为未返回正确对象类型的反馈。这种概率事件在快速构建 app 时会频繁发生。

Nuclite 对所有相关 Flow 类型检查都是如此。这意味着在代码还在编写时，类型错误便能暴露出来，同时我们还能对相关问题进行修正。而这一切无需再等到到开发接近完成时才进行。

这或许并不直观，但确实提高了开发速度。要想在代码中发现遗漏真的是件很困难的事。而在 app 接近开发完成时，这会更加棘手。
##Jest : 对可能造成 BUG 的改动进行单元测试
[jest][3] 是一个面向 JavaScipt 的单元测试框架，同时它在 [React Native app][12]下也表现不俗。

我们h会用这些单元测试（也被称作回归测试）来确保对已经开发完成的结构化代码的改动并未引入 bug 。

作为例子，下面我们准备通过一个 Jest 测试来保证 Reducer 能够继续按预期处理地图数据：
```javascript
jest.autoMockOff();

const Parse = require('parse');
const maps = require('../maps');

describe('maps reducer', () => {

  it('is empty by default', () => {
    expect(maps(undefined, {})).toEqual([]);
  });

  it('populates maps from server', () => {
    let list = [
      new Parse.Object({mapTitle: 'Day 1', mapImage: new Parse.File('1.png')}),
      new Parse.Object({mapTitle: 'Day 2', mapImage: new Parse.File('2.png')}),
    ];

    expect(
      maps([], {type: 'LOADED_MAPS', list})
    ).toEqual([{
      id: jasmine.any(String),
      title: 'Day 1',
      url: '1.png',
    }, {
      id: jasmine.any(String),
      title: 'Day 2',
      url: '2.png',
    }]);
  });

});
```
Jest 本身非常易于阅读（注意：我们甚至对 Jest 测试部分都使用了 Flow 来定义类型），不过我们还是会对此进行进一步的分解。在第 4 行，我们引入了地图的 Reducer 函数(**js/reducers/maps.js**) 这样便能在单元测试中直接调用（Reducer  函数作为 [pure functions][13] 能够很容易完成这些）。

第一段测试代码位于第 8 行，目的是确保 Recucer 函数返回一个空数组。观察  **js/reducers/maps** 处 Reducer 代码，你会发现 state 并未做任何初始化，因此我们会期待期待单元测试结果为返回空数组。

第二段测试代码位于第 12 行，目的是确保地图数据被解析 API 检索到的时候能够被 Reducer 函数转化成 **state** 树中对应的正确结构。在这个测试中，我们使用假数据完全模拟真实 API 返回数据结构，这将避免API连接问题导致的测试失败。

现在我们已经令跑 Jest 测试成为开发工作流程（比如每次提交 git 前）的一部分。这样我们能确保对现有代码的改动不会令 app 默默阵亡。

由于 Redux Reducers 会改变 **state** 树，不引入 bug 变得绝对至关重要。由 **state** 改变引发的 bug 很容易被忽略。因为它们并 不会造成功能不可用，而仅仅只是造成往 Parse Server 发送错误数据的问题。 Reducer 函数的 pure function 特性令它们成为回归测试的理想对象，因为每次我们都可以准确预知它们会如何执行。
##调试
在你试图对 bug 定位或者修复时，手边有一些调试工具的话。我们已经介绍了[如何搭建app可视元素调试系统][14]，那这么处理数据呢？

我们通过 [Nuclite][1] 和 [Redux Logger middleware][15] 来调用[ Chrome 开发者工具][16]（ Chrome Developer Tools ），控制台上展示类似 Actions 或 Reducers 中 state 变动这样的新增 Redux 上下文（ context ）:
![iOS and Android Segmented Controls Comparison](http://makeitopen.com/static/images/redux_logger.png)
你可以看到我们是如何通过**configurestore**来启动这个的：
```javascript
/* from js/store/configureStore.js */
var createLogger = require('redux-logger');
...
var isDebuggingInChrome = __DEV__ && !!window.navigator.userAgent;
var logger = createLogger({
  predicate: (getState, action) => isDebuggingInChrome,
  collapsed: true,
  duration: true,
});

var createF8Store = applyMiddleware(thunk, promise, array, logger)(createStore);

function configureStore(onComplete: ?() => void) {
  const store = autoRehydrate()(createF8Store)(reducers);
  persistStore(store, {storage: AsyncStorage}, onComplete);
  if (isDebuggingInChrome) {
    window.store = store;
  }
  return store;
}
```
在第 5 行，我们[通过一些选项][17]创建了 Logger middleware 。 接着，在第 10 行我们调用 Redux 的[ **applyMiddleware()** 函数][18]开始应用 Logger middleware 。这样，我们便可以在控制台上看到日志输出了。

在第4行我们使用[全局变量][19] **__DEV__**  来触发调试功能，通过布尔值的改动来在调试模式的开关间进行切换。这一举措除了会决定 Logger middleware 创建后是否记录日志（通过[**断言**选项][20]），还会在第17行 拷贝当前 Store 到 [Window object][21] ，如此就可以更容易的通过控制台来直接查看 Store 对象。

[1]:http://nuclide.io/
[2]:http://flowtype.org/
[3]:http://facebook.github.io/jest/
[4]:http://flowtype.org/docs/about-flow.html#_
[5]:http://flowtype.org/docs/type-annotations.html#_
[6]:http://makeitopen.com/tutorials/building-the-f8-app/data/
[7]:http://flowtype.org/docs/getting-started.html#_
[8]:http://flowtype.org/docs/type-aliases.html#_
[9]:https://github.com/facebook/react-native/blob/master/babel-preset/configs/main.js#L32
[10]:http://flowtype.org/docs/cli.html#_
[11]:http://nuclide.io/docs/platforms/react-native/
[12]:http://facebook.github.io/react-native/docs/testing.html#content
[13]:https://en.wikipedia.org/wiki/Pure_function
[14]:http://makeitopen.com/tutorials/building-the-f8-app/design/#the-design-iteration-cycle
[15]:http://fcomb.github.io/redux-logger/
[16]:https://facebook.github.io/react-native/docs/debugging.html#chrome-developer-tools
[17]:https://github.com/fcomb/redux-logger#options-1
[18]:http://redux.js.org/docs/api/applyMiddleware.html
[19]:http://www.w3schools.com/js/js_scope.asp
[20]:https://github.com/fcomb/redux-logger#predicate--getstate-function-action-object--boolean
[21]:http://www.w3schools.com/jsref/obj_window.asp

