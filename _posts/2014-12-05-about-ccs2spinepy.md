---
layout: post    
title: "ccs2spine.py开发笔记"
description: ""    
category:     
tags: [No Coco Studio, Spine, Cocos2d-x]

---
{% include JB/setup %}
##千万不要用Coco Studio
项目之初由于对工具的考察工作做得不够到位，让美术同学用[Coco Studio]来绘制角色骨骼动画，由此陷进了Coco Studio这个大坑。至于Coco Studio有多坑就不多说了，后来赶紧换工具。    
经过对比[DragonBones]和[Spine]，最后还是决定用Spine。虽然Flash用于做动画很成熟，但是Spine对于骨骼动画有更多非常强大的功能，比如Mesh、IK动画等，具体可以参见[Spine Features]。

由于美术同学已经用Coco Studio做了一批角色动画，需要一个工具把原来Coco Studio的动画格式（ExportJson文件）导出为Spine可编辑的json文件，于是有了这个格式转换的工具：[ccs2spine]。

##踩坑记录
具体的工作大部分是繁琐的细节，但其中遇到了不少问题，在这里记录一下。

###概念差异
1. 在Spine中，除了Bone（骨骼），还有Slot（槽位？），Coco Studio只有Bone的概念。    
   在Spine中Bone只用在动画中控制于平移、旋转、缩放，而Slot用于控制渲染层级，即类似于Coco Studio中"z"属性的概念。
2. 关键帧的颗粒度不同：Coco Studio创建关键帧的单位是Bone，而Spine中中创建关键帧的单位是「属性」。这是Spine文件比Coco Studio小很多的原因之一。
3. 在Coco Studio中，旋转正方向是逆时针方向，Spine是顺时针方向。
4. Coco Studio中可以改变图片的锚点；Spine中没有锚点的概念，图片的锚点一直是在中心。
5. Coco Studio的骨骼支持「斜切」（Skew）功能，旋转只是斜切的一种特殊情况；而Spine除了可以设定旋转之外，只有更为强大也更为复杂的Mesh功能。

###drawOrders排序
Coco Studio中，在形体模式中通过设定"z"属性来设定骨骼的初始渲染次序，动画模式中再通过"z"来设置骨骼渲染次序的。

Spine中，SetupUp模式同样可以设定Slot的初始渲染顺序，Animation模式中可以通过「拖拉」来改变渲染次序。最后Spine输出的描述文件中，是以偏移值来描述渲染次序的变化的，在运行时会挨条顺序执行。

    {'slot':'slot1', 'offset':1}    
    {'slot':'slot2', 'offset':2}
    
具体算法实现可以参见源文件。

###优化
由于关键帧的颗粒度不同，Spine关键帧和Coco Studio关键帧无法一一对应，因此转化为Spine时，有大量属性的关键帧可以忽略。    
另外，由于Spine属性支持默认值，因此也可以省略输出很多东西。

###斜切处理
**此功能尚未实现**    
需要将Coco Studio中的「斜切」功能转化为Spine中Mesh

###锚点处理
**此功能尚未实现**    





[Coco Studio]:http://www.cocos2d-x.org/wiki/Cocos_Studio
[Spine]:http://esotericsoftware.com
[Spine Features]:http://esotericsoftware.com/spine-in-depth
[DragonBones]:http://dragonbones.effecthub.com
[ccs2spine]:https://github.com/mobiuschen/ccs2spine
