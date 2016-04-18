##将 F8 应用移植到 windows 平台—使用 React Native 开发

大家可能已经了解到，[微软正努力将 React Native 引入到 windows 通用应用平台](https://blogs.windows.com/buildingapps/)。对于 React Native 开发者来说，这是一个可以让自己的应用到达 2.7 亿 windows 10 用户（包括手机、电脑、xbox 甚至是 hololens VR眼镜）的绝佳机会。基于同 facebook 的合作以及将 React Native 引入到 windows 所要作出的努力，我们在 windows 应用商店发布了[ F8 开发者大会应用](https://www.microsoft.com/en-us/store/apps/f8-developer-conference/9nblggh4ntvn)。 所有代码的移植基于最近开源的[ F8 github 代码库](https://github.com/fbsamples/f8app)。

下面这个视频演示了在UWP( windows 通用应用平台)上运行的 F8 应用中源自 React Native 的部分特性：
https://www.youtube.com/watch?v=51_M9Dp5X80&feature=youtu.be
 (国内用户需翻墙才能查看)

将 F8 应用移植到 windows 平台占用了一个 3 人小组 80/100 的时间，历时 3 周完成。我们将此数据完全透明的公布给大家，主要是基于这样的考虑：决策是否使用 React Native 这样的技术来进行开发，开发效率是一个很重要的参考指标。

虽然我们揭开了 React Native 在 windows 平台开发的序幕，但当前 React Native 的部分核心视图管理器和原生模块在 windows 上仍不可用， React Native 的第三方依赖库也都还不支持 windows 平台。具体就这个应用来说，缺少针对菜单和过滤器的 SplitView 视图管理器；缺少切换 tab 和会话页的FlipView视图管理器; 在滚动视图管理器里面，缺少合适的运行事件用来处理拖拽和内容视图更新。我们也缺少一个剪贴板模块-用来拷贝粘贴 wifi 的详情；缺少用作导航状态存储的异步存储模块；缺少用作登出和其他警告处理的对话框模块；也缺少一个用来处理应用的信息tab页中的链接行为的启动器模块。对于第三方模块，我们缺少[线性渐变视图管理器](https://github.com/brentvatne/react-native-linear-gradient)、[facebook sdk 模块](https://github.com/facebook/react-native-fbsdk)、 [React Native 分享模块](https://github.com/EstebanFuentealba/react-native-share)。其中一部分，例如启动器模块，花了我们小半天时间搞定。其他更复杂的模块，比如 facebook sdk 模块，花了我们超过 1 天的时间；这里主要的工作量在于找到恰当的原生 api 依赖，然后实现功能并且测试。

当我们将要把应用发布到应用商店时，发现有一些细节还没有考虑到：发布到商店的应用必须使用 .net 原生环境编译。我们运气不错，很快搞定了这件事。因为其中只有很少的一部分 api 是 .net 原生环境所不支持的（主要是关于反射方面的）；我们只需要[解决这部分反射相关的问题](https://github.com/ReactWindows/react-native/commit/9420ff92f7ccc910ceeb6bb88568675da3abcada)。

我们对应用作了一些设计和风格上的调整以让它在 windows phone 上看起来更棒。这里我不会给出太多的细节，因为 [facebook 已经详细的阐述了如何用 React Native 为 android 和iOS 作平台定制开发](http://makeitopen.com/), 而这些定制开发的原则也同样适用于 windows 平台。**剔除内核适配、第三方组件适配开发以及发布到应用商店这些工作量，通过 js 语言实现的 windows 平台定制开发和风格调整仅花了 1 名开发者不到 1 周的时间。**请大家留意这个工作量估计，因为在可期待的未来—当 React Native 在 Windows 平台上的成熟度接近 iOS 和 android 的时候，这就是 React Native 应用移植到 windows 时开发者的唯一工作量。下面我给出了一些例子来展现如何让 windows 应用变得不同（与 iOS 和 android 应用相比）。

F8 ListContainer 组件的平台特定风格设置代码:
```
var styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: 'white',
  },
  listView: {
    ios: {
      backgroundColor: 'transparent',
    },
    android: {
      backgroundColor: 'white',
    },
    windows: {
      backgroundColor: 'white',
    }
  },
  headerTitle: {
    color: 'white',
    fontWeight: 'bold',
    fontSize: 20,
  },
});
```
F8TabsView.window.js 中：

```
class F8TabsView extends React.Component {
  ...
 
  render() {
    return (
      <F8SplitView
        ref="splitView"
        paneWidth={290}
        panePosition="left"
        renderPaneView={this.renderPaneView}>
        <View style={styles.content} key={this.props.tab}>
          {this.renderContent()}
        </View>
      </F8SplitView>
    );
  }
 
  ...
}
```
对比 F8TabsView.android.js：

```
class F8TabsView extends React.Component {
  ...
 
  render() {
    return (
      <F8DrawerLayout
        ref="drawer"
        drawerWidth={290}
        drawerPosition="left"
        renderNavigationView={this.renderNavigationView}>
        <View style={styles.content} key={this.props.tab}>
          {this.renderContent()}
        </View>
      </F8DrawerLayout>
    );
  }
 
  ...
}
```
对比 F8TabsView.ios.js：

```
class F8TabsView extends React.Component {
  ...
 
  render() {
    return (
      <TabBarIOS tintColor={F8Colors.darkText}>
        <TabBarItemIOS>
          ...
        </TabBarItemIOS>
        ...
      </TabBarIOS>
    );
  }
 
  ...
}
```
 React Native 的理念是做一个“水平的平台“，意思是更倾向于"learn once, write anywhere"，以区别于 java 的"write once, run everywhere"。虽然我们的 F8 windows 应用体验接近于 Android，但如果有更多的开发时间，我们很可能会修改其界面和菜单让它看起来更像是一个 windows 应用。例如在 XAML 中，SplitView 支持一种紧凑的显示模式，其 panel 关闭时该视图仅从一个下拉菜单里显示所有的 icon 图片。对应用的桌面版本和 [continuum](https://msdn.microsoft.com/en-us/library/windows/hardware/dn917883%28v=vs.85%29.aspx) 特性来说，这个体验很棒。同时，在 XAML 中 pivot 被广泛用于翻页，拥有 pivot 风格的页面和会话标题对 windows 用户来说将会是更熟悉的体验。

总的来说，我们基于 React Native 将 F8 开发者大会应用移植到 windows 的过程让人感觉很好； React Native 应用移植到 windows 平台也将会变得越来越容易。我们希望我们所作出的努力可以证明在 windows 平台上用 React Native 开发绝不仅仅是实验性质的。通过社区的强大支持，它可能为你的 React Native 应用带来更广泛的受众（来自 windows 平台的用户）；这是一个属于 React Native 开发者的绝好机会。

在 5 月 13 日爱尔兰都柏林举行的 [DECODED Conference](http://www.decodedshow.com/events/) 上，我们将继续讨论此次移植的经验以及其他一些关于在 windows 上使用 React Native 的议题。此外，这里有一篇文章，分享了另一个微软的团队[在 UWP 平台上使用 CodePush 实现 React Native 代码热更新](http://microsoft.github.io/code-push/articles/ReactNativeWindows.html)的经验。
特别感谢 [Matt Podwysocki](https://github.com/mpodwysocki) 和 [Eero Bragge](https://github.com/ebragge) ，是他们的努力工作才得以让 F8 windows 应用如期到来。

原文链接：https://ericroz.wordpress.com/2016/04/11/f8-app-on-windows-10-mobile/

