---
title: 小常识-idea如何打jar包
categories: 技术
tags: [idea]
date: 2017/10/11 20:00:00
---

## start

&emsp;&emsp;昨晚在家玩游戏玩的正嗨,领导一通电话过来,让我写个连接zookeeper的demo,写就写吧,5分钟搞定,但是我竟然不知道怎么用idea把项目生成jar包,以前用Eclipse很简单,换成使用idea后,竟然不会弄了,硬生生的花了半个小时才搞定,真是无比尴尬,虽然又不难,但是觉得有必要记录下来.小常识也能难倒无数英雄好汉!

- - -

<!--more-->

- - -

## content

### 说明

&emsp;&emsp;我这里所说的idea生成jar包自然不是说的maven项目,maven项目的话很easy啦,`mvn clean install`/`mvn clean package`,轻轻松松生成war包/jar包.那么普通的java项目如何生成jar包呢,往下看就知道啦.

### 步骤

- 写完带有包含main函数的项目后,`File`->`ProjectStructure`进入项目设置界面
![idea-jar-1](http://okzr61x6y.bkt.clouddn.com/idea-jar-1.png)

- 创建jar的配置,选择main函数所在类,`META-INF/MANIFEST.MF`的目录选择项目的根目录
![idea-jar-2](http://okzr61x6y.bkt.clouddn.com/idea-jar-2.png)

- 修改jar包名称和jar包导出路径,并勾上`include in project build`,就OK啦
![idea-jar-3](http://okzr61x6y.bkt.clouddn.com/idea-jar-3.png)

- 如果有的小伙伴在进行到上面第二步时出现如下错误,那是因为项目下已经有`META-INF/MANIFEST.MF`文件了,如果已有该文件就不需要选择目录了.
![idea-jar-4](http://okzr61x6y.bkt.clouddn.com/idea-jar-4.png)

- 最后就是生成jar包,`Build`->`Build Artifacts`->`***.jar`->`Build`,然后就可以在之前设置的jar包生成路径里找到jar包啦

### end

&emsp;&emsp;以后可能不定期的写些这样简单的操作类型的文章,有时候不会弄真的很浪费时间,比代码写不出来都让人头疼.
