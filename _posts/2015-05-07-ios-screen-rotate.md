---
layout: post
title: iOS Screen Rotate
date: 2015-05-07
category: "ios"
---
###前言
默认情况下,UIKit framework通过监听UIDeviceOrientationDidChangeNotification来自动的更新界面方向 
当UIKit接收到屏幕旋转的通知后,它会利用UIApplication和root view controller来确定是否支持新方向的旋转.如果支持则window改变尺寸以适应新的方向,然后window通知root view controller调整尺寸以适应新的(屏幕)尺寸，最后该尺寸会被逐级的传递给各个子级的view.

###事件的响应链
当屏幕旋转时，会触发以下响应:

**1.** window调用root view controller的方法检测是否支持旋转<br />
*•对于IOS 5.0之前的系统*：<br />
调用`shouldAutorotateToInterfaceOrientation` (来判断是否支持该方向的旋转)。
<br />*•对于IOS 6.0以上系统*：<br />
调用`supportedInterfaceOrientations` 返回viewcontroller支持的旋转方向；<br />
调用`shouldAutorotate` 动态控制屏幕是否支持旋转,返回YES表示支持旋转,返回NO表示不支持旋转

**2.** 如果root view controller支持该方向的旋转,则会调用当前显示的viewcontroller的`willRotateToInterfaceOrientation:duration:`方法
viewcontroller容器(root view controller)会将该消息转发给当前正在显示的viewcontroller中.你可以重写当前正在显示的viewcontroller的该方法用于隐藏views或做一些其他改变.

**3.** window调整viewcontroller中view的bounds。这会引起view重新布局它的subviews,触发viewcontroller的`viewWillLayoutSubviews`方法.
当该方法运行后,你可以访问应用的`statusBarOrientation`属性来判断当前用户界面的方向,以对view进行相应的布局处理.

**4.** 当前正在显示的viewcontroller的`willAnimateRotationToInterfaceOrientation:duration:`方法被调用,准备开始旋转动画

**5.** 执行旋转动画

**6.** viewcontroller的`didRatateFromInterfaceOrientation:`方法被调用.

![](/images/post/2015-05-07-ios-screen-rotate-img01.png)

对于push的view controller所支持的旋转方向是由navigationController的回调控制的<br />
对于present的view controller所支持的选装方向是由本view controller的回调控制的

###<font color="#ff0000">注意事项：</font>

**1.** 尽量使用autoresizingMask进行相对布局,这样无需改变view的frame,减少性能消耗<br />
**2.** 屏幕旋转过程中禁止视图中事件的传递.(旋转过程中屏蔽点击事件)<br />
**3.** 如果旋转的视图中包含地图视图,可以在开始旋转的时候保存显示区域所指向的值(中心点经纬度),旋转完成后恢复显示区域,使旋转后的地图显示的区域和显示前的地图显示区域大致相同.<br />
**4.** 对相对复杂的视图(多个层级的视图)执行旋转动画时,可能引起性能问题,可以先用截图覆盖在复杂视图上,旋转完成后重新布局视图后将截图移除.<br />
**5.** 旋转完成后重新加载可见tableview的数据，使得tableview显示的行数适合屏幕尺寸<br />
**6.** 监听旋转消息去更新应用的状态信息,可以通过viewcontroller的回调方法或设备方向的notification来记录当前设备的方向,从而做出必要的调整(其他viewcontroller显示的时候需要根据当前设备方向做出对应的调整).

####[ScreenRotate](https://github.com/zrongl/ScreenRotate)