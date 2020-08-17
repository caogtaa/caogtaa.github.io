---
layout: post
title: "Cocos Creator流体渲染效率优化"
subtitle: "一种粒子渲染优化方法"
date: 2020-08-17 16:44:35 +0800
author: "GT"
tags:
  - cocos-creator
  - optimize
  - metaballs
---

![](/img/in-post/20200817/pipe_and_water.gif)
<br />流体效果相信大家都不陌生，一种常用的实现方式是基于metaballs，可以参考[这里](https://www.shadertoy.com/view/tsXGDM)和[这里](https://www.shadertoy.com/view/4sBXRy)获取metaballs的shader代码。<br />如果要渲染有一定体积的流体，需要实时渲染几十上百甚至上千个metaball，此时渲染的效率就需要考虑进来。<br />论坛里已经有不少相关的实现，本文将对两种不同的实现进行学习分析并提出优化方案。<br />优化方案只针对运行效率，本文不讨论metaballs渲染的视觉效果优化。<br />

<a name="qEgVc"></a>
## 方案1：box2d + shader
<a name="90ed305f"></a>
#### 实现原理
通过box2d产生一批粒子， `将N个粒子的坐标通过uniform变量一次性传入shader` 。<br />shader的片元着色器中，累积计算每个粒子对当前片元产生的“势能”（势能和距离相关），“势能”大于阈值时输出颜色，否则透明。<br />

<a name="bw54n"></a>
#### 示例代码（shader）
```c

uniform ARGS{
    // ...
    // ts中从box2d直接获取每个粒子的坐标，进行简单处理后传入shader
    vec4 metaballs[500];
};

void main () {
    // ...
    float v = 0.0;			// 所有metaball对当前片元的影响将累加到变量v中
    for(int i = 0; i < 500; i++){
        vec4 mb = metaballs[i];
        // ...
        // ss是第i个metaball 对当前片元影响的“势能”, (cx, cy)为metaball坐标，r为半径
        float ss = r * r / (cx * cx + cy * cy);
        
        // 累加势能
        v += ss;
    }
    
    // 势能超过阈值则渲染为水流颜色，否则渲染为透明
    // ... 略
}
```
<a name="YHo1z"></a>
#### 分析
片元着色器程序需要遍历所有metaball，500个metaball的情况下会产生上千条指令。每一帧每个片元都需要执行这么长的程序，并且往往这种渲染方式需要全屏幕渲染，GPU压力将非常大。<br />
<br />

<a name="rb5Dt"></a>
## 方案2：PhysicsManager
<a name="354bv"></a>
#### 实现原理
每个粒子是一个cc.Node，并挂上 `物理碰撞组件` 。<br />每个粒子用一张圆形渐变图渲染到内存纹理，纹理的半透明部分将叠加，相当于metaball里的势能叠加效果。然后用一个简单的shader按阈值处理颜色。<br />

<a name="1nNxV"></a>
#### 示例代码
略<br />

<a name="ZVNAq"></a>
#### 分析
cc自带的碰撞检测的性能相对于box2d的粒子组碰撞检测效率要差一些，前者算法时间复杂度是O(N^2)，[后者](https://docs.google.com/presentation/d/1fEAb4-lSyqxlVGNPog3G1LZ7UgtvxfRAwR0dwd19G4g/htmlpresent)在渲染同半径的粒子组时可以优化为O(NlogN)。在metaball数量较多的情况下差异会显现出来。<br />另外由于使用cc.Node包装了粒子，对引擎带来一定的overhead，如render-flow遍历时需要逐粒子做RenderData更新（相对于碰撞检测来说这部分可以忽略）。<br />
<br />

<a name="ebca0d21"></a>
## 方案3：box2d + assembler
<a name="2z693"></a>
#### 实现原理

1. 跟方案1一样，使用box2d产生粒子组。
1. 在assembler里获取所有粒子坐标，批量组装成 `RenderData` 。<br />针对每个粒子生成一个四边形，附带它在世界坐标里的原心位置，同时省略了uv和color属性。<br />可以学习方案2对每个粒子使用圆形纹理图，本方案为了简单起见直接在shader里画圆。<br />顶点格式如下：
```typescript
var vfmtPosCenter = new gfx.VertexFormat([
    { name: gfx.ATTR_POSITION, type: gfx.ATTR_TYPE_FLOAT32, num: 2 },   // 粒子顶点（1个粒子有3个或4个顶点）
    { name: "a_center", type: gfx.ATTR_TYPE_FLOAT32, num: 2 }           // 原粒子中心（每个顶点相同数据）
]);
```


<a name="34577a27"></a>
#### 示例代码
见[Demo](https://github.com/caogtaa/CCBatchingTricks)内的 `SceneMetaBalls` <br />

<a name="72fa7c88"></a>
#### 分析
避开了方案1的GPU瓶颈和方案2的CPU碰撞计算瓶颈。<br />缺点是代码量相对较高，在自定义assembler内需要处理box2d坐标空间到屏幕空间的换算。<br />
<br />

<a name="9c0db15b"></a>
## 性能对比
测试环境：华为P9手机，chrome访问，开发模式<br />测试数据均来自cc自带调试面板数据的目测。

|  |  | 方案1<br />（box2d+shader） | 方案2<br />（cc.Node+物理） | 方案3<br />（box2d+assembler） |
| --- | --- | --- | --- | --- |
| 500粒子 | 帧率 | < 10fps | < 20fps | 60fps |
|  | Game Logic | 7~15ms | > 60ms | < 14ms |
| 1000粒子 | 帧率 | < 10fps | < 10fps | 53~60fps<br />流动时60fps；<br />在1000粒全部积压时fps达到最低，此时contact数量最多，为7000+ |
|  | Game Logic | 未测试 | > 150ms | < 25ms |

<a name="Dkmqp"></a>
#### 数据解释
WebGL的数据不知道为啥一直是0，以上没有记录。<br />方案1的Game Logic很低，但是帧率较低，结合不难推断出是GPU压力影响了整体帧率；<br />方案2的Game Logic偏高，主要是物理碰撞检测导致的CPU压力；<br />进一步对方案3进行profile，可以发现，最耗时的仍然是粒子碰撞检测部分，其中最耗时的部分是box2d里寻找粒子之间的连接点，见下图。<br />
![](/img/in-post/20200817/box2d_profile.png)
<br />经实际运行统计，在1000个粒子的场景下将产生7000+个碰撞点，函数内部循环次数加到14000+。这是一帧的运算量，所以内部循环是热点代码。`利用临时变量合并一些公共表达式` 可以实现少量性能提升，不过需要改引擎内部。<br />
![](/img/in-post/20200817/findContacts.png)
<br />

<a name="1wvkA"></a>
## Demo地址
[https://github.com/caogtaa/CCBatchingTricks](https://github.com/caogtaa/CCBatchingTricks)<br />

<a name="9PDZW"></a>
## 参考
[https://forum.cocos.org/t/topic/92305](https://forum.cocos.org/t/topic/92305)<br />[https://forum.cocos.org/t/shader/92906](https://forum.cocos.org/t/shader/92906)<br />[https://forum.cocos.org/t/happy-glass/72468/46](https://forum.cocos.org/t/happy-glass/72468/46)<br />[https://docs.google.com/presentation/d/1fEAb4-lSyqxlVGNPog3G1LZ7UgtvxfRAwR0dwd19G4g/htmlpresent](https://docs.google.com/presentation/d/1fEAb4-lSyqxlVGNPog3G1LZ7UgtvxfRAwR0dwd19G4g/htmlpresent)
