---
layout: post    
title: "对Spine Runtime的扩展"
description: ""    
category:     
tags: [Spine, Root Motion, Cocos2d-x]

---
{% include JB/setup %}

Spine是目前最优秀的2D骨骼动画编辑软件应该没有之一，有几个非常好用的功能：

* Meshes：可能应该翻译成「自由变形」
* Skins：蒙皮
* Inverse Kinematics：IK动画

![img2]


具体实例可以参照安装目录下自带的`examples`目录下的示例。

虽然Spine编辑器很优秀，但是其Runtime功能却略显简单，只提供了比较基本的接口，因此有些功能自己去实现。

比如，Spine的Skins功能，虽然可以很方便地实现换肤，但是在编辑器中导出时，它会将所有皮肤素材都导出为一个atlas，即一整张大材质。这样导致每次加载人物只能把所有皮肤素材一次性加载，而游戏中大多数时候是需要按需加载和释放的。    
大多数游戏如果人物可以「换装」，我想都应该是把皮肤\服装作为单独的素材管理的。因此我对Spine Runtime做了些扩展，使其可以实现动态增删AtlasPage和Skin。

另外，我还加入了对「根骨骼动画」的支持，这对于动作非常多的动作类游戏是非常方便。

该项目我已经Fork到我的Github上：[Spine runtimes]。    

本文依据的Spine Runtime版本是2.0 Cocos2d-x，Spine编辑器版本是2.1。

##Runtime架构
Spine Runtime的实现分为两层：

1. 基础层：使用C语言编写，包括格式解析、动画数据计算等。
2. 胶水层：接入具体框架（Cocos2d-x就是用C++写的），主要根据基础层提供的数据进行绘制。

![img1]
这是官方提供类关系图，分为几部分：

* Loading：负责加载、解析动作文件。
* Atlas：负责加载、解析atlas文件，将
* **Rendering**：负责具体渲染部分。在Cocos2d-x中，这部分就是胶水层，与具体框架相关。
* Setup Pose Data：动作json文件的数据结构化。
* InstanceData：当前角色的动作状态
* Animation：动画相关的数据结构

##功能扩展1：支持根骨骼动画
「根骨骼动画」（Root Motion）是指以角色根骨骼的变换（移动/旋转/缩放）来代表整个角色的变换，用在3D动作编辑器中会比较多。使用该功能可以制作一些由程序不太好控制位移的动作，典型的比如「后撤步」动作。    
这篇文章中对于「根骨骼动画」有比较详细的解释：[Unreal 根骨骼动画][link1]。

修改`SkeletonAnimation`类中播放动画接口，加入`rootMotion`参数：

	spTrackEntry* setAnimation (int trackIndex, const std::string& name, bool loop, bool rootMotion = false);
    spTrackEntry* addAnimation (int trackIndex, const std::string& name, bool loop, float delay = 0, bool rootMotion = false);


##功能扩展2：动态加载AtlasPage
`SkeletonRenderer`类中增加以下接口：

    void addAtlasPagesWithFile(const std::string& atlasFile);
    
##功能扩展3：动态增删Skin
`SkeletonRendrer`类中增加以下接口：

    bool setSkin (const std::string& skinName, bool cleanOld = false);
    const std::string getSkinName();
    bool hasSkin(const std::string& skinName);
    bool addSkinWithFile(const std::string& file);
    bool removeSkinByName(const std::string& skinName);

##功能扩展4：增加动画控制的接口
`SkeletonAnimatioin`类中加入以下接口：

    float getAnimationDuration(const std::string& name);
    void pauseAnimation();
    void resumeAnimation();
    bool isPlaying(){return _isPlaying;}
    const cocos2d::Vec2 getBoneWorldPosition(const std::string& boneName);



[Spine runtimes]:https://github.com/mobiuschen/spine-runtimes

[link1]:https://docs.unrealengine.com/latest/CHN/Engine/Animation/RootMotion/index.html

[img1]: /image/spine-runtimes-diagram.png
[img2]: /image/spine-goblin-mesh.png