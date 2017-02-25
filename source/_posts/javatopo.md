---
layout: post
title:  java 实现的画网络拓扑插件
date:   2013-01-04 08:43:59
author: Lily
categories: java
tags:
- 网络拓扑
- 开源
---

附上源码链接 https://github.com/DoTalkLily/NetworkTopoPainter ，再看那时候写的代码不太好注释不全面，仅供参考，有时间和精力再重构。
是将之前（12年中）写的网络拓扑插件的第二版，增加了一些功能，同时将图标变小，可以迅速显示数百个网络元素，实现快速点击定位等操作，同时还可以在界面自定义添加一些如按钮菜单项等组件。

![topo1](http://dl.iteye.com/upload/attachment/0068/4650/0e697a0b-a9c3-36ef-9414-5f49970df324.png)

使用手册目录结构（每个标题下都带有代码例子）：

![readme](http://dl.iteye.com/upload/attachment/0068/4652/a191b690-53d6-3fc1-9bf5-b72f65799d4a.png)

还包括添加告警，告警级别设定，初始化指定方向箭头等功能。



这里贴一段 鼠标事件定义方法的说明(摘自“使用说明”)：

使用插件默认的鼠标事件类，则双击网络元素（链路或路由器交换机图标）会弹出一个窗口显示初始化时设定的名字或内容信息，右键元素会出现一个列表（有右键对象，添加告警，删除告警项）

1. 继承MyMouseAction 类，重写

 public void showMenu(MouseEvent e, Component com)方法来控制鼠标右键点击网络元素显示的内容。

例

```java

import java.awt.Component;

import java.awt.event.MouseEvent;

import javax.swing.JPopupMenu;


publicclass MyAction extends MyMouseAction

{

   publicvoid showMenu(MouseEvent e, Component com)//名字不能改变！

   {

        JPopupMenu popupMenu = new JPopupMenu();//定义一个弹出菜单

        if (com instanceof TopoLink) {  //如果传入对象是TopoLink实例

           TopoLink tl = (TopoLink) com;

           popupMenu.add("右键对象：" + tl.getLnode().getText() + "——"

                  + tl.getRnode().getText());

       } elseif (com instanceof TopoNode) { //如果传入对象是TopoNode实例

           popupMenu.add("右键对象：" + ((TopoNode) com).getText());

       }

       popupMenu.addSeparator();//分隔符

       popupMenu.show(e.getComponent(), e.getX(), e.getY());//显示弹出框

   }

}

```


注：然后要调用d.setMyAction(new MyAction());



2. 继承MyMouseAction 类，重写public void showDialog(MouseEvent e, Component com)，定义左键双击网络元素显示的内容，如弹框等，例子将在下面给出。



3. public void setMode(boolean mode)

//设置右上方tab切换时候显示的模式，针对各个界面都用一种网络拓扑的情况设计，如果设置//true，则各个界面显示的拓扑结构都与第一个界面相同，如各个界面共用同一拓扑结构，

//只是在拓扑结构上显示的路径不同，这种情况下只要将拓扑结构中的元素添加到第一个界面//即可（不用每个界面都加一遍）；默认为false，则各个界面的拓扑元素都要分别添加。

例：

```java

    MyMouseAction actions = new MyMouseAction ();

                     actions.setMode(true);//设置模式

      DrawGraph    topoView = new DrawGraph("窗口");

                     topoView.setMyAction(actions);

```

注：这里可能不好理解，一下举一个需要设置mode的情况，如图1：

![tu1](http://dl.iteye.com/upload/attachment/0068/4646/56ee202b-3c33-3009-85bc-9bceffcc90fe.png)

点击“界面2”图2：

![tu2](http://dl.iteye.com/upload/attachment/0068/4644/3bfe8f77-e75a-333c-96bd-174ad410079d.png)




两个界面三个元素位置相同（现网中拓扑结构应该比这个复杂很多），但是两个界面只是展示的路径不同。

多说一句：程序设计中可以定义一个类似

Map<Integer, ArrayList<TopoLink>> colorLinks;

的结构来保存不同面板号对应需要显示的连接对象列表，然后需要自己实现并覆盖MyMouseAction中的下述两个方法：

public void drawColorLines(int tabIndex)

public void clearColorLines(int tabIndex)

讲解如下：

以下两个方法是右上方tab切换时候执行的动作

4. 继承MyMouseAction 类，重写public void drawColorLines(int tabIndex)

传入右侧上方tab的索引值（从0起计数）则画出

Map<Integer, ArrayList<TopoLink>> colorLinks;（自定义）中tab索引值对应的链路列表。



5. 承MyMouseAction 类，重写public void clearColorLines(int tabIndex)

传入右侧上方tab的索引值（从0起计数），清除

Map<Integer, ArrayList<TopoLink>> colorLinks;（自定义）中tab索引值对应的链路列表。

注：这两个方法都是在public void setMode(boolean mode)模式设置为true时才会被执行到，先执行clearColorLines（lastTabIndex）;（即先清除上一个面板上内容），再

drawColorLines(currentTabIndex);(即再传入要展示的面板的索引值，画出相应内容)；

为了便于理解，以下贴出简单实现的代码：


```java

import java.awt.Component;

import java.awt.event.MouseEvent;

import java.util.ArrayList;

import java.util.HashMap;

import java.util.Map;



import javax.swing.JLayeredPane;



publicclass Demo extends MyMouseAction

{

//tab索引号——链路列表映射

Map<Integer, ArrayList<TopoLink>> colorLinks;



    public Demo()

    {

        colorLinks = new HashMap<Integer, ArrayList<TopoLink>>();

    }



/**

     * @return Returns the colorLinks.

     */

    public Map<Integer, ArrayList<TopoLink>> getColorLinks()

    {

        returncolorLinks;

    }



    /**

     * @param colorLinks The colorLinks to set.

     */

    publicvoid setColorLinks(int tabIndex, TopoLink link)

    {

        if (tabIndex < 0 || link == null)

        {

            System.out.print("自定义鼠标事件类的setColorLinks参数为空！");

            return;

        }

        if (this.colorLinks.containsKey(tabIndex))

        {

            this.colorLinks.get(tabIndex).add(link);

        }

        else

        {

            ArrayList<TopoLink> tempLink = new ArrayList<TopoLink>();

            tempLink.add(link);

            this.colorLinks.put(tabIndex, tempLink);

        }

    }



    //以下是重写的父类相关方法

    @Override

    publicvoid showDialog(MouseEvent e, Component com)

    {

        //这里自定义双击网络元素显示的内容

    }



       @Override

    publicvoid showMenu(MouseEvent e, Component com)

    {

       //这里自己定义右键网络元素需要显示的内容

    }



    @Override

    publicvoid drawColorLines(int tabIndex)

    {

        ArrayList<TopoLink> links = this.colorLinks.get(tabIndex);

        if (links != null)

        {

            int size = links.size();

            JLayeredPane temp = getCurrentPane();

            for (int i = 0; i < size; i++)

            {

                temp.add(links.get(i));

                temp.repaint();

            }

        }

    }



    @Override

    publicvoid clearColorLines(int tabIndex)

    {

        ArrayList<TopoLink> links = this.colorLinks.get(tabIndex);

        if (links != null)

        {

            int size = links.size();

            JLayeredPane temp = getCurrentPane();

            for (int i = 0; i < size; i++)

            {

                temp.remove(links.get(i));

                temp.repaint();

            }

        }

    }

}
```

