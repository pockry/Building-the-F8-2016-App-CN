# 构建 F8 2016 App 第二部分：设计跨平台App

这是为了介绍 React Native 和它的开源生态的一个系列教程，我们将以构建 F8 2016 开发者大会官方应用的 iOS 和 Android 版为主题。

React Native 的一大优势是：可以只用一种语法编写分别运行在 iOS 和 Android 平台上的程序，且可重用部分应用逻辑。

然而，与“一次编写，到处运行”的理念不同的是，React Native 的哲学是“一次学习，到处编写”。如此一来，即使用 React Native 编写不同平台的程序，也可以尽可能贴合每个平台的特性。

从 UI 的角度来看，每个平台都有自己独特的视觉风格、UI 范例甚或是技术层面的功能，那我们设计出一个统一的 UI 基础组件，然后再按照各平台特性进行调整岂不乐乎？

## 准备工作

在后续的所有教程中，我们会仔细解读 App 的源代码，请克隆一份[源代码][1]到本地可访问的路径，然后根据[配置说明][2]在本地运行 App。在本章的教程中，你只需要阅读相关源代码。

# React Native 思维模式

在你写任何 React 代码之前，请认真思考这个至关重要的问题：**如何才能尽可能多地重用代码？**

React Native 的理念是针对每个平台分而治之，代码重用的做法看起来与之相违背，好像我们就应该为每个平台定制其专属的视觉组件一样，但实际上我们仍需努力让每个平台上的代码尽可能多地统一。

构建一套 React Native 应用视觉组件的关键点在于如何最好地实现平台抽象。开发人员和设计师可以列出应用中需要重用的组件，例如按钮、容器、列表行，头部等等，只有在必要的时候才单独为每个平台设计特定的组件。

当然，有一些组件相对于其它组件而言更为复杂，我们先一起来看看 F8 应用中不同的组件有什么区别。

# 各种各样的小组件

请看 F8 应用的示例图：

![iOS and Android Segmented Controls Comparison](http://makeitopen.com/static/images/iOS%20vs%20Android%20Segmented%20Controls@3x.png)

在 iOS 版本中，我们用 iOS 系统中很常见的圆角边框风格来切分 Tab 控制；在 Android 版本中，我们用下划线的风格来标示这个组件。而这两个控制组件的功能其实完全相同。

所以，即使两者样式稍有不同，但是实现的功能相同，所以我们可以用同一套代码抽象此处的逻辑，从而可以尽可能多地重用代码。

我们针对像这样的小组件做了很多跨平台重用逻辑代码的案例，比如一个简单的文本按钮，在每个平台上我们都会设计不同的 hover 和 active 状态的样式，但是除开这些视觉上的细微的差异外，逻辑功能完全相同。所以我们总结了一个抽象 React Native 视觉组件的最佳实践方法：设计一套相同的逻辑代码，然后在控制语句中编写其余需要差异化的部分。

以下是这个组件的示例代码（来自 `<F8SegmentedControl>`）：

```JavaScript
/* from js/common/F8SegmentedControl.js */
class Segment extends React.Component {
  props: {
    value: string;
    isSelected: boolean;
    selectionColor: string;
    onPress: () => void;
  };

render() {
    var selectedButtonStyle;
    if (this.props.isSelected) {
      selectedButtonStyle = { borderColor: this.props.selectionColor };
    }
    var deselectedLabelStyle;
    if (!this.props.isSelected && Platform.OS === 'android') {
      deselectedLabelStyle = styles.deselectedLabel;
    }
    var title = this.props.value && this.props.value.toUpperCase();

    var accessibilityTraits = ['button'];
    if (this.props.isSelected) {
      accessibilityTraits.push('selected');
    }

    return (
      <TouchableOpacity
        accessibilityTraits={accessibilityTraits}
        activeOpacity={0.8}
        onPress={this.props.onPress}
        style={[styles.button, selectedButtonStyle]}>
        <Text style={[styles.label, deselectedLabelStyle]}>
          {title}
        </Text>
      </TouchableOpacity>
    );
  }
}
```

在这段代码中，我们为每一种平台分别应用了不同的样式（用到了 React Native 的 [Platform 模块][3]）。各平台中的 Tab 按钮都应用了相同的通用样式，同时也根据各平台特性定制了独占样式（同样出自 `<F8SegmentedControl>`）：

```JavaScript
/* from js/common/F8SegmentedControl.js */
var styles = F8StyleSheet.create({
  container: {
    flexDirection: 'row',
    backgroundColor: 'transparent',
    ios: {
      paddingBottom: 6,
      justifyContent: 'center',
      alignItems: 'center',
    },
    android: {
      paddingLeft: 60,
    },
  },
  button: {
    borderColor: 'transparent',
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: 'transparent',
    ios: {
      height: HEIGHT,
      paddingHorizontal: 20,
      borderRadius: HEIGHT / 2,
      borderWidth: 1,
    },
    android: {
      paddingBottom: 6,
      paddingHorizontal: 10,
      borderBottomWidth: 3,
      marginRight: 10,
    },
  },
  label: {
    letterSpacing: 1,
    fontSize: 12,
    color: 'white',
  },
  deselectedLabel: {
    color: 'rgba(255, 255, 255, 0.7)',
  },
});
```

在这段代码中我们使用了一个改编自 [React Native `StyleSheet` API][4] 的函数 F8StyleSheet，它可以针对各平台分别进行样式转换操作：

```
export function create(styles: Object): {[name: string]: number} {
  const platformStyles = {};
  Object.keys(styles).forEach((name) => {
    let {ios, android, ...style} = {...styles[name]};
    if (ios && Platform.OS === 'ios') {
      style = {...style, ...ios};
    }
    if (android && Platform.OS === 'android') {
      style = {...style, ...android};
    }
    platformStyles[name] = style;
  });
  return StyleSheet.create(platformStyles);
}
```

在这个 `F8StyleSheet` 函数中我们解析了前面示例代码中的 `styles` 对象，如果我们发现了匹配当前平台的 `ios` 或 `android` 键值，就会应用相应的样式，如果都没有，则应用默认样式。以此看来，减少代码重复的另一种做法是：尽可能多地重用通用样式代码。

现在，我们已经可以在我们的App中重用这个组件了，它也可以根据不同的平台自动匹配相应的样式。

# 分离复杂差异

如果一个组件在各平台上的差异不仅仅是样式的不同，也存在大量的逻辑代码差异，那我们就需要换一种方式了。正如下图所示，iOS 和 Android 平台中最高阶的菜单导航组件就有非常大的差异：

![iOS and Android Main Navigation Comparison](http://makeitopen.com/static/images/iOS%20vs.%20Android@3x.png)

正如你所见，在 iOS 版本中我们在屏幕底部放了一个固定的 Tab，而在 Android 版本中，我们却实现了一种可划出的侧边栏。这两种组件其实是本质上的不同，况且一般来说，在 Android 应用中，这种侧边栏通常还会包含更多的菜单选项，例如：退出登录。

你当然可以将这两种菜单模式写到一个组件中去，但是这个组件会变得异常臃肿，所有的差异不得不通过大量的分支语句来实现，你一定会在不久的将来对这段代码感到陌生难懂。

其实，我们可以用 React Native 内建的平台特定的扩展来解决这个问题。我们可以创建两个独立的应用，在下面的示例中我们会创建两个组件，分别命名为：`F8TabsView.ios.js` 和 `F8TabsView.android.js`。React Native 会自动检测当前平台并根据扩展命名加载相应的组件。

# 内建UI组件

在每一个 `FBTabsView` 组件中，我们也可以重用一些内建的 React Native UI 组件，Android 版本使用的是 [`DrawerLayoutAndroid`][5]（很显然，只在 Android 中可用）：

```JavaScript
/* from js/tabs/F8TabsView.android.js */
render() {
  return (
    <DrawerLayoutAndroid
      ref="drawer"
      drawerWidth={300}
      drawerPosition={DrawerLayoutAndroid.positions.Left}
      renderNavigationView={this.renderNavigationView}>
      <View style={styles.content} key={this.props.activeTab}>
        {this.renderContent()}
      </View>
    </DrawerLayoutAndroid>
  );
}
```

在第8行代码中，我们在当前的类中显式地为 drawer 组件指定了 `renderNavigationView()` 函数。这个函数会返回 drawer 中渲染出来的内容。在这个示例中，我们渲染的是一个包含在自定义 `MenuItem` 组件(点击查看 [`MenuItem.js`][6])中的 [`ScrollView`][7] 组件：

```JavaScript
/* from js/tabs/F8TabsView.android.js */
renderNavigationView() {
  ...
  return(
    <ScrollView style={styles.drawer}>
      <MenuItem
        title="Schedule"
        selected={this.props.activeTab === 'schedule'}
        onPress={this.onTabSelect.bind(this, 'schedule')}
        icon={scheduleIcon}
        selectedIcon={scheduleIconSelected}
      />
      <MenuItem
        title="My F8"
        selected={this.props.activeTab === 'my-schedule'}
        onPress={this.onTabSelect.bind(this, 'my-schedule')}
        icon={require('./schedule/img/my-schedule-icon.png')}
        selectedIcon={require('./schedule/img/my-schedule-icon-active.png')}
      />
      <MenuItem
        title="Map"
        selected={this.props.activeTab === 'map'}
        onPress={this.onTabSelect.bind(this, 'map')}
        icon={require('./maps/img/maps-icon.png')}
        selectedIcon={require('./maps/img/maps-icon-active.png')}
      />
      <MenuItem
        title="Notifications"
        selected={this.props.activeTab === 'notifications'}
        onPress={this.onTabSelect.bind(this, 'notifications')}
        badge={this.state.notificationsBadge}
        icon={require('./notifications/img/notifications-icon.png')}
        selectedIcon={require('./notifications/img/notifications-icon-active.png')}
      />
      <MenuItem
        title="Info"
        selected={this.props.activeTab === 'info'}
        onPress={this.onTabSelect.bind(this, 'info')}
        icon={require('./info/img/info-icon.png')}
        selectedIcon={require('./info/img/info-icon-active.png')}
      />
    </ScrollView>
  );
}
```

相比之下，iOS 版本直接在 `render()` 函数中使用了一个不同的内建组件，[`TabBarIOS`][8]：

```JavaScript
/* from js/tabs/F8TabsView.ios.js */
render() {
  var scheduleIcon = this.props.day === 1
    ? require('./schedule/img/schedule-icon-1.png')
    : require('./schedule/img/schedule-icon-2.png');
  var scheduleIconSelected = this.props.day === 1
    ? require('./schedule/img/schedule-icon-1-active.png')
    : require('./schedule/img/schedule-icon-2-active.png');
  return (
    <TabBarIOS tintColor={F8Colors.darkText}>
      <TabBarItemIOS
        title="Schedule"
        selected={this.props.activeTab === 'schedule'}
        onPress={this.onTabSelect.bind(this, 'schedule')}
        icon={scheduleIcon}
        selectedIcon={scheduleIconSelected}>
        <GeneralScheduleView
          navigator={this.props.navigator}
          onDayChange={this.handleDayChange}
        />
      </TabBarItemIOS>
      <TabBarItemIOS
        title="My F8"
        selected={this.props.activeTab === 'my-schedule'}
        onPress={this.onTabSelect.bind(this, 'my-schedule')}
        icon={require('./schedule/img/my-schedule-icon.png')}
        selectedIcon={require('./schedule/img/my-schedule-icon-active.png')}>
        <MyScheduleView
          navigator={this.props.navigator}
          onJumpToSchedule={() => this.props.onTabSelect('schedule')}
        />
      </TabBarItemIOS>
      <TabBarItemIOS
        title="Map"
        selected={this.props.activeTab === 'map'}
        onPress={this.onTabSelect.bind(this, 'map')}
        icon={require('./maps/img/maps-icon.png')}
        selectedIcon={require('./maps/img/maps-icon-active.png')}>
        <F8MapView />
      </TabBarItemIOS>
      <TabBarItemIOS
        title="Notifications"
        selected={this.props.activeTab === 'notifications'}
        onPress={this.onTabSelect.bind(this, 'notifications')}
        badge={this.state.notificationsBadge}
        icon={require('./notifications/img/notifications-icon.png')}
        selectedIcon={require('./notifications/img/notifications-icon-active.png')}>
        <F8NotificationsView navigator={this.props.navigator} />
      </TabBarItemIOS>
      <TabBarItemIOS
        title="Info"
        selected={this.props.activeTab === 'info'}
        onPress={this.onTabSelect.bind(this, 'info')}
        icon={require('./info/img/info-icon.png')}
        selectedIcon={require('./info/img/info-icon-active.png')}>
        <F8InfoView navigator={this.props.navigator} />
      </TabBarItemIOS>
    </TabBarIOS>
  );
}
```

显而易见，尽管 iOS 菜单接受了相同的数据，但是它的结构略有不同。我们并没有用一个独立的函数创建菜单元素，而是将这些元素作为父级菜单的子元素插入进来，正如 [`TabBarItemIOS`][9] 组件这样。
这里的 `TabBarItem` 与 Android 中 的 `MenuItem` 本质上是相同的，唯一的区别是在 Android 组件中我们会定义一个独立的主 [`View` 组件][10]：

```JavaScript
<View style={styles.content} key={this.props.activeTab}>
  {this.renderContent()}
</View>
```

然后当一个 tab 改变时改变这个组件（通过 `renderContent()` 函数），而 iOS 组件则会有多个分离的 `View` 组件，例如：

```JavaScript
<GeneralScheduleView
  navigator={this.props.navigator}
  onDayChange={this.handleDayChange}
/>
```

这是 `TabBarItem` 的一部分，可以点击使它们可见。

# 设计迭代周期

当你构建任何应用，无论是在移动平台还是 web 环境下，调整适配的 UI 元素是非常痛苦的。如果工程师和设计师共同协作，会使整个过程慢下来。

React Native 包含了一个[实时重载][11]的 debug 功能，可以当 JavaScript 改变时触发刷新应用。这可以在极大程度上减少设计迭代过程，一旦改变了组件样式并保存后，你会立即看到更新的样式。

但是如果组件在不同条件下看起来不同怎么办呢？举个例子，一个按钮组件可能有一个默认样式，也分别包含按下、执行任务中、执行任务完成时的样式。

为了避免每次都与应用交互，我们内建了一个用于 debug 视觉效果的 `Playgroud` 组件：

```JavaScript
/* from js/setup.js */
class Playground extends React.Component {
  constructor(props) {
    super(props);
    const content = [];
    const define = (name: string, render: Function) => {
      content.push(<Example key={name} render={render} />);
    };

    var AddToScheduleButton = require('./tabs/schedule/AddToScheduleButton');
    AddToScheduleButton.__cards__(define);
    this.state = {content};
  }

  render() {
    return (
      <View style=>
        {this.state.content}
      </View>
    );
  }
}
```

其实我们只是创建了一个可交换加载的空视图，将其与一些示例定义整合到其中一个 UI 组件中，正如下面这段 `AddToScheduleButton.js` 所示：

```JavaScript
/* from js/tabs/schedule/AddToScheduleButton.js */
module.exports.__cards__ = (define) => {
  let f;
  setInterval(() => f && f(), 1000);

  define('Inactive', (state = true, update) =>
    <AddToScheduleButton isAdded={state} onPress={() => update(!state)} />);

  define('Active', (state = false, update) =>
    <AddToScheduleButton isAdded={state} onPress={() => update(!state)} />);

  define('Animated', (state = false, update) => {
    f = () => update(!state);
    return <AddToScheduleButton isAdded={state} />;
  });
};
```

我们可以将这个应用转化为一个 UI 预览工具：

![UI preview playground in action with a button and three different states](http://makeitopen.com/static/images/button-playground.gif)

在这个示例中为这个按钮定义了按下和抬起两种状态，第三个按钮在这两者的状态之间不断循环，以此来预览过渡的动画效果。

现在，我们可以跟设计师一起快速调整基础组件的视觉样式了。

如果想用这个功能，`<Playground>` 组件必须在任何 React Native 应用中都可用，我们需要在 `setup()` 函数中交换一些代码来加载 `<Playground>` 组件：

```JavaScript
/* from js/setup.js */
render() {
  ...
  return (
    <Provider store={this.state.store}>
      <F8App />
    </Provider>
  );
}
```

变为

```JavaScript
/* in js/setup.js */
render() {
  ...
  return (
    <Provider store={this.state.store}>
      <Playground />
    </Provider>
  );
}
```

当然，你也可以修改 `<Playground>` 组件，使其能够改变引入的其它组件。


  [1]: https://github.com/fbsamples/f8app
  [2]: http://makeitopen.com/tutorials/building-the-f8-app/local-setup/
  [3]: https://facebook.github.io/react-native/docs/platform-specific-code.html#platform-module
  [4]: https://facebook.github.io/react-native/docs/stylesheet.html
  [5]: http://facebook.github.io/react-native/docs/drawerlayoutandroid.html
  [6]: https://github.com/fbsamples/f8app/blob/master/js/tabs/MenuItem.js
  [7]: http://facebook.github.io/react-native/docs/scrollview.html
  [8]: http://facebook.github.io/react-native/docs/tabbarios.html
  [9]: http://facebook.github.io/react-native/docs/tabbarios-item.html#content
  [10]: http://facebook.github.io/react-native/docs/view.html#content
  [11]: http://facebook.github.io/react-native/docs/debugging.html#live-reload
