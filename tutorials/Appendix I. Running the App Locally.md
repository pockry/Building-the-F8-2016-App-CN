#构建 F8 2016 App 附录 1：本地运行 App

当你从[苹果商店](https://itunes.apple.com/us/app/f8/id853467066)或者[谷歌 Play](https://itunes.apple.com/us/app/f8/id853467066) 上下载了 F8 App，并可以在移动设备上运行它以后，阅读这些教程的同时你也会希望在本地运行 App 玩一玩。

这篇简短的文章，将指导你如何在 OSX 本地搭建和运行源代码（安卓版 React Native [可以支持](http://facebook.github.io/react-native/docs/linux-windows-support.html#content)在 Windows 和 Linux 运行）。

##要求

在开始之前，你需要安装一些必要的东西：

1. [React Native](http://facebook.github.io/react-native/docs/getting-started.html) (根据 iOS 和 Android 的引导说明)
2. [CocoaPods](http://cocoapods.org/) 1.0+ (仅 iOS)
3. [MongoDB](https://www.mongodb.org/downloads) (需要在本地运行 Parse Server)



##搭建

####1. 克隆仓库
```
$ git clone https://github.com/fbsamples/f8app.git
$ cd f8app
```

####2. 安装依赖（npm v3+）
```
$ npm install
$ (cd ios; pod install)        # only for iOS version
```

####3. 确认 MongoDB 运行正常
```
$ lsof -iTCP:27017 -sTCP:LISTEN
```

或者使用外部 MongoDB 服务，设置 `DATABASE_URI`

```
$ export DATABASE_URI=mongodb://example-mongo-hosting.com:1337/my-awesome-database
```

####4. 启动 Parse/GraphQL 服务
```
$ npm start
```

####5. 导入样本数据（本地的 Parse 服务已经运行）
```
$ npm run import-data
```

访问以下地址确认一切正常运行：

- Parse Dashboard: http://localhost:8080/dashboard
- GraphiQL: http://localhost:8080/graphql

![](http://makeitopen.com/static/images/screenshot-server@2x.png)



####6. 运行 Android 版 App
```
$ react-native run-android
$ adb reverse tcp:8081 tcp:8081   # required to ensure the Android app can
$ adb reverse tcp:8080 tcp:8080   # access the Packager and GraphQL server
```

####7. 运行 iOS 版 App
```
$ react-native run-ios
```

原文地址： http://makeitopen.com/tutorials/building-the-f8-app/local-setup/