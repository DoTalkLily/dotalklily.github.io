---
layout: post
title:  React Native & IOS 动画效果实现
date:   2017-02-20 08:43:59
author: Lily
categories: frontend
tags:
- React Native
- 动画
---

开篇插播一条广告，欢迎大家下载和使用，狸猫相机，一个3D特效自拍、朋友圈短视频创作神器3D特效自拍变脸、朋友圈短视频创作神器。头条出品，必属精品~
下面进入正题~
产品主要用React Native(以下简称RN)和native开发，其中Story页面（包含联系人发的视频列表）和视频播放（涵盖很多复杂手势交互比如点赞动画，左右滑切视频，上拉出评论弹框等）以及消息页面等功能主要基于RN开发（部分小功能和模块用native封装供RN调用）。
最近接了个需求，pm想要实现如下的动效果：
点击列表页的任意头像逐渐放大然后自然过渡到播放页面，如下图所示：

<img src="https://wiki.bytedance.net/download/attachments/78356276/fangda.gif?version=1&modificationDate=1487865498779&api=v2" width=300/>

待视频播放完后渐渐缩小回到列表页原来的位置，如下图所示：

<img src="https://wiki.bytedance.net/download/attachments/78356276/suoxiao.gif?version=1&modificationDate=1487865513267&api=v2" width=300/>

如果列表和视频播放页面纯用native实现，那么可以用UIView的animateWithDuration或者一个CAAnimationGroup先将头像边放大边位移到屏幕中间，再让即将展现的UiViewController实现UIViewControllerTransitioningDelegate协议，在页面切入，退出的时候调用一个自定义的继承UIViewControllerAnimatedTransitioning的转场动画效果类，实现转场动画，展现第二个页面。网上也有一些类似的自定义转场动画，效果很酷，参见[Hero](https://github.com/lkzhao/Hero) 或[这个例子](https://github.com/wazrx/XWTrasitionPractice)。

那么回到我们的项目，如果这两个页面是纯RN实现的，并且页面路由使用一个第三方组件[react-native-router-flux](https://github.com/aksonov/react-native-router-flux),倒是可以添加一些简单的转场效果，水平垂直滑入、淡入等。但是较复杂的转场动画实现起来比较复杂，不能保证在ios和andoid平台动画的流畅度，而且要自己想办法实现比如动画结束的钩子函数，比较复杂。因此想办法利用native实现整个动画。那么如何在RN实现的页面上加native动画和转场呢？这里沿用上面native实现的思路，将整个过程分为两段，第一段用native控制头像和边框放大同时位移到中间位置，第二段与native不同的是，这里用rn组件实现的路由，因此无法通过自定义UIViewController转场效果切换动画（改造成本也比较大），因此这里用native获得当前navigation stack中最顶层的UIViewController，定义一个CABasicAnimation用于模拟转场扩散效果，然后加在上面得到的UIViewController的mask（遮罩）属性上来模拟纯native的转场效果。

介绍实现方法前，首先介绍下针对现有的业务，实现这个动画要考虑到的点：

+ 大小图切换：video第一帧要展示大图而story列表对应的后端裁剪后的小图。方法：点击瞬间将头像图替换成要播放的第一帧大图。
+ 列表页面的图不一定是点击要播放的图，比如点击要播放第一个未读的视频，但是列表中确要展示一个已读的视频图片。方法：和后端定义接口，加字段传来下一个要播放的视频第一帧。
+ 第一段动画有一层渐变的蒙层，方法：缩放过程加蒙层，用动画控制淡入淡出。
+ 位置计算，可以看到，圆框和里面的图片放大的速度不同，而且图片要放大到和要播放的封面一样大再淡出。方法：根据图片实际大小和当前屏幕大小以及圆框大小计算两者的缩放比例，分别缩放。
+ 两段动画如何流畅衔接，如果第一段动画放大到中间瞬间跳第二个页面同时加转场盖住，会有闪动。方法：考虑优化策略，下面会介绍。
+ 按照业务逻辑，视频播放结束回到列表页会立刻向后端拉取一次列表，如果缩放的圆框‘回来’时不在原来的位置了，或者被删除了怎么搞？方法：加回调等动画结束再通知RN再拉取。
+ 多个视频要播放，在任意视频播放过程中要回到story页面场景，这时候要缩小的图片和列表中的图片可能不一致。方法：缩小动画执行前将当前的封面url从rn传给native。

下面介绍一下具体实现：
用native实现动画，首先要保证动画目标元素是native的，因此首先将列表中渲染的头像用native封装成ui component暴露给rn，即下方红框区域：

<img src="https://wiki.bytedance.net/download/attachments/78356276/touxiang.png?version=1&modificationDate=1487865517733&api=v2" width=300 />

IOS部分代码如下：

![](https://wiki.bytedance.net/download/attachments/78356276/code2.png?version=1&modificationDate=1487865531084&api=v2)

用RN实现的列表部分渲染一个头像区域的代码如下（精简版）：

![](https://wiki.bytedance.net/download/attachments/78356276/code1.png?version=1&modificationDate=1487865517993&api=v2)

改成用刚封装的组件：

![](https://wiki.bytedance.net/download/attachments/78356276/code3.png?version=1&modificationDate=1487865531285&api=v2)

原来的头像点击事件是直接打开video页面，现在是：

![](https://wiki.bytedance.net/download/attachments/78356276/code4.png?version=1&modificationDate=1487865531612&api=v2)

点击调用这个元素的放大动画：
第一段动画代码：

![](https://wiki.bytedance.net/download/attachments/78356276/code5.png?version=1&modificationDate=1487865534177&api=v2)

效果：

<img src="https://wiki.bytedance.net/download/attachments/78356276/jieduan1.gif?version=1&modificationDate=1487865494247&api=v2" width=300/>

注意这里有个hack，为保证两个动画衔接，在第一段动画执行前有一句：
 [self performSelector:@selector(sendEvent) withObject:nil afterDelay:0.2f];
200ms后发送一个消息给RN唤起即将呈现的video页面，已保证第一段动画结束时遮罩底层已经是第二个页面了。

然后在第一段动画结束的回掉函数中调用转场效果动画：

![](https://wiki.bytedance.net/download/attachments/78356276/code7.png?version=1&modificationDate=1487865531763&api=v2)

看下转场动画实现方式：

![](https://wiki.bytedance.net/download/attachments/78356276/code8.png?version=1&modificationDate=1487865533854&api=v2)

第二段动画效果：

<img src="https://wiki.bytedance.net/download/attachments/78356276/jieduan2.gif?version=1&modificationDate=1487865506266&api=v2" width=300/>


关闭效果实现方式类似，也是两段动画拼接这里不再赘述。

还有些待调整的部分，比如动画执行时长，蒙层透明度变化，第二屏幕切换时机等，初步效果如下：

<img src="https://wiki.bytedance.net/download/attachments/78356296/xiaoguo.gif?version=1&modificationDate=1487866423000&api=v2" width=300/>


欢迎交流：）


