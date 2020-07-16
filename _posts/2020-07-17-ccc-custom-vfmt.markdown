---
layout: post
title:  "Cocos Creator自定义渲染 —— 自定义顶点格式"
date:   2020-07-17 01:20:35 +0800
categories: cocos-creator render
tags:
  - cocos-creator
  - render
---


<a name="1ap0j"></a>
## 背景
自定义渲染可以实现很多酷炫的shader特效，目前常用的有两种方法：

1. **创建自定义材质，给材质增加参数。这个参数会作为uniform变量传入shader**<br />由于渲染合批要求材质参数保持一致，所以如果大量对象使用自定义材质时，并且材质参数各不相同，是无法进行合批渲染的，一个对象占一个draw call
1. **创建自定义assembler，在顶点数据输入渲染管道前修改它的值**<br />这种方式比较灵活，如果需要输入更多自定义参数，标准的顶点格式就不够用了


<br />本文介绍另一种方法，即能让shader获得自定义参数，又能让自定义材质合批渲染。这种方法就是 **自定义顶点格式**<br />
<br />

<a name="czlaS"></a>
## Assembler详解

Assembler是实现本文相关功能的核心类，先简单回顾一下官方文档里介绍的内容<br />[https://docs.cocos.com/creator/manual/zh/advanced-topics/custom-render.html#%E8%87%AA%E5%AE%9A%E4%B9%89-assembler](https://docs.cocos.com/creator/manual/zh/advanced-topics/custom-render.html#%E8%87%AA%E5%AE%9A%E4%B9%89-assembler)<br />
![](/img/in-post/20200717/rendercomponent.png)
> Assembler 中必须要定义 updateRenderData 及 fillBuffers 方法
> 前者需要更新准备顶点数据，后者则是将准备好的顶点数据填充进 VetexBuffer 和 IndiceBuffer 中


<br />2D渲染中，Assember2D类是一个重要的基础类，最常用的cc.Sprite的各种模式（Simple，平铺，九宫格）在内部都对应了不同的Assembler派生类。同样是一个四边形的节点，不同的Assembler可以将其转化成不同数量的顶点实现不同的渲染效果

- Simple模式下是常规的四边形，有4个顶点
- 平铺模式下Assembler根据纹理的重复次数对节点进行“拆碎”，相当于每重复一次就产生1个四边形
- 九宫格模式下Assembler将节点拆分为9个四边形，每个四边形对应纹理上的一个“格子”



<a name="Y0rKU"></a>
### fillBuffers源码解读

先看看Assembler2D是如何实现 `fillBuffers` 的<br />源码位置：[https://github.com/cocos-creator/engine/blob/master/cocos2d/core/renderer/assembler-2d.js](https://github.com/cocos-creator/engine/blob/master/cocos2d/core/renderer/assembler-2d.js)

```typescript
fillBuffers (comp, renderer) {
    // 如果节点的世界坐标发生变化，重新从当前节点的世界坐标计算一次顶点数据
    if (renderer.worldMatDirty) {
        this.updateWorldVerts(comp);
    }

    // 获取准备好的顶点数据
    // vData包含pos、uv、color数据
    // iData包含三角剖分后的顶点索引数据
    let renderData = this._renderData;
    let vData = renderData.vDatas[0];
    let iData = renderData.iDatas[0];

    // 获取顶点缓存
    // getBuffer()方法后面会被我们重载，以便获得支持自定义顶点格式的缓存
    let buffer = this.getBuffer(renderer);
    
    // 获取当前节点的顶点数据对应最终buffer的偏移量
    // 可以简单理解为当前节点和其他同格式节点的数据，都将按顺序追加到这个大buffer里
    let offsetInfo = buffer.request(this.verticesCount, this.indicesCount);

    // fill vertices
    let vertexOffset = offsetInfo.byteOffset >> 2,
        vbuf = buffer._vData;

    // 将准备好的vData拷贝到VetexBuffer里。
    // 这里会判断如果buffer装不下了，vData会被截断一部分。
    // 通常不会出现装不下这种情况，因为buffer.request中会分配足够大的空间；如果出现，则当前组件只能被渲染一部分
    if (vData.length + vertexOffset > vbuf.length) {
        vbuf.set(vData.subarray(0, vbuf.length - vertexOffset), vertexOffset);
    } else {
        vbuf.set(vData, vertexOffset);
    }

    // 将准备好的iData拷贝到IndiceBuffer里
    let ibuf = buffer._iData,
        indiceOffset = offsetInfo.indiceOffset,
        vertexId = offsetInfo.vertexOffset;
    for (let i = 0, l = iData.length; i < l; i++) {
        ibuf[indiceOffset++] = vertexId + iData[i];
    }
}
```
<a name="vBrlw"></a>
### 思考

**Q**: 为什么要需要准备顶点数据，而不是在fillBuffer()方法内直接计算后填入buffer?<br />**A**: 因为fillBuffer()每帧都会被调用，是热点代码，需要关注效率。但是顶点数据不是每一帧都会更新，可以预先计算<br />
<br />**Q**: 实现自定义顶点格式需要修改fillBuffer()方法吗？<br />**A**: 不需要，fillBuffer()是简单的字节流拷贝，只关心数据长度，不关心数据内容<br />
<br />**Q**: 顶点数据包含哪些内容？如何计算？<br />**A**: 见下文<br />

<a name="kRccu"></a>
### 顶点数据格式描述

最常用的顶点格式是 `vfmtPosUvColor` ，也是Assembler2D默认使用的格式。<br />[https://github.com/cocos-creator/engine/blob/master/cocos2d/core/renderer/webgl/vertex-format.js](https://github.com/cocos-creator/engine/blob/master/cocos2d/core/renderer/webgl/vertex-format.js)

```typescript
var vfmtPosUvColor = new gfx.VertexFormat([
    // 节点的世界坐标，占2个float32
    { name: gfx.ATTR_POSITION, type: gfx.ATTR_TYPE_FLOAT32, num: 2 },
  
    // 节点的纹理uv坐标，占2个float32
    // 如果节点使用了独立的纹理（未合图），这里的uv值通常是0或1
    // 合图后的纹理，这里的uv对应其在图集里的相对位置，取值范围在[0,1)内
    { name: gfx.ATTR_UV0, type: gfx.ATTR_TYPE_FLOAT32, num: 2 },
  
    // 节点颜色值，cc.Sprite组件上可以设置。占4个uint8 = 1个float32
    { name: gfx.ATTR_COLOR, type: gfx.ATTR_TYPE_UINT8, num: 4, normalize: true },
]);
```
顶点格式和shader顶点着色器的attribute变量对应关系如下。

```C
CCProgram vs %{
  precision highp float;

  #include <cc-global>
  #include <cc-local>

  // 对应vfmtPosUvColor结构里的3个字段
  // 注意这里a_position是vec3类型，但是vfmtPosUvColor对其自定义了2个float长度。所以a_position.z = 0
  in vec3 a_position;		        // gfx.ATTR_POSITION
  in vec2 a_uv0;			// gfx.ATTR_UV0
  in vec4 a_color;			// gfx.ATTR_COLOR

  // ...

  void main () {
      // ...
  }
}%

```

官方定义了一些常用的attribute变量名<br />[https://github.com/cocos-creator/engine/blob/master/cocos2d/renderer/gfx/enums.js](https://github.com/cocos-creator/engine/blob/master/cocos2d/renderer/gfx/enums.js)
用户可以自定义attribute变量名，只需要 **顶点格式中的变量名和shader中的匹配上**即可。

再来看下Assembler2D里的属性和顶点格式的对应关系<br />源码位置：[https://github.com/cocos-creator/engine/blob/master/cocos2d/core/renderer/assembler-2d.js](https://github.com/cocos-creator/engine/blob/master/cocos2d/core/renderer/assembler-2d.js)

```typescript
cc.js.addon(Assembler2D.prototype, {
    // vfmtPosUvColor 结构占5个float32
    floatsPerVert: 5,

    // 一个四边形4个顶点
    verticesCount: 4,
  
    // 一个四边形按照对角拆分成2个三角形，2*3 = 6个顶点索引
    indicesCount: 6,

    // uv的值在vfmtPosUvColor结构里下标从2开始算
    uvOffset: 2,
  
    // color的值在vfmtPosUvColor结构里下标从4开始算
    colorOffset: 4,
});
```
<a name="k75Nz"></a>
### 
<a name="4Rmm9"></a>
### 顶点数据计算


![](/img/in-post/20200717/vertexformat.png)<br />了解了上面的顶点格式之后，顶点数据无非就是计算 pos、uv、color几个值。<br />在Assembler里分别有 `updateVerts()`、 `updateUVs()`、 `updateColor()` 方法来准备这几个值，并且临时存储在Assembler自己分配的数组里。<br />
顶点数据存在RenderData中，源码位置：[https://github.com/cocos-creator/engine/blob/master/cocos2d/core/renderer/assembler-2d.js](https://github.com/cocos-creator/engine/blob/master/cocos2d/core/renderer/assembler-2d.js)

```typescript
export default class Assembler2D extends Assembler {
    constructor () {
        super();

      	// renderData.vDatas用来存储pos、uv、color数据
        // renderData.iDatas用来存储顶点索引数据
        this._renderData = new RenderData();
        this._renderData.init(this);
        
        this.initData();
        this.initLocal();
    }

    get verticesFloats () {
      	// 当前节点的所有顶点数据总大小
        return this.verticesCount * this.floatsPerVert;
    }

    initData () {
        let data = this._renderData;
      	// 创建一个足够长的空间用来存储顶点数据 & 顶点索引数据
      	// 这个方法内部会初始化顶点索引数据
        data.createQuadData(0, this.verticesFloats, this.indicesCount);
    }
    
    // ...
}
```
`updateUVs()` 方法解读<br />源码位置：[https://github.com/cocos-creator/engine/blob/master/cocos2d/core/renderer/webgl/assemblers/sprite/2d/simple.js](https://github.com/cocos-creator/engine/blob/master/cocos2d/core/renderer/webgl/assemblers/sprite/2d/simple.js)

```typescript
updateUVs (sprite) {
    // 获取当前cc.Sprite组件设置的spriteFrame对应的uv
    // uv数组长度=8，分别表示4个顶点的uv.x, uv.y
    // 按照左下、右下、左上、右上的顺序存储，注意这里的顺序和顶点索引的数据需要对应上
    let uv = sprite._spriteFrame.uv;
    let uvOffset = this.uvOffset;		// 之前提到过vfmtPosUvColor结构里uvOffset = 2
    let floatsPerVert = this.floatsPerVert; // floatsPerVert = vfmtPosUvColor结构大小 = 5
    let verts = this._renderData.vDatas[0];
    for (let i = 0; i < 4; i++) {
        // 2个1组取uv数据，写入renderData.vDatas对应位置
        let srcOffset = i * 2;
        let dstOffset = floatsPerVert * i + uvOffset;
        verts[dstOffset] = uv[srcOffset];
        verts[dstOffset + 1] = uv[srcOffset + 1];
    }
}
```
`updateColor()` 和 `updateVerts()` 的具体实现这里不再分析。<br />

<br />
<br />由于上面多次提到了顶点索引，对于不了解它的同学需要再单独解释一下。
<a name="mYNs3"></a>
### 理解顶点索引

除了pos、uv、color数据之外，为什么还需要计算顶点索引数据？<br />我们发送给GPU的数据，实际上表示的是三角形，而不是四边形。一个四边形需要剖分成2个三角形然后传给GPU。<br />在4个顶点数据的基础上，三角形的描述信息单独存在 **IndiceBuffer (即renderData.iDatas)**里，IndiceBuffer里的每个值表示其对应顶点数据的下标。通过索引可以合并掉多个三角形中相同的顶点数据，减少总数据大小。<br />
![](/img/in-post/20200717/subdivision.png)
<br />常规四边形的索引数据准备，源码位置：[https://github.com/cocos-creator/engine/blob/master/cocos2d/core/renderer/webgl/render-data.js](https://github.com/cocos-creator/engine/blob/master/cocos2d/core/renderer/webgl/render-data.js)

```typescript
initQuadIndices(indices) {
    // 按照上述剖分方式得到的下标: [0,1,2] [1,3,2]
    // 6个一组（对应1个四边形）生成索引数据
    let count = indices.length / 6;
    for (let i = 0, idx = 0; i < count; i++) {
        let vertextID = i * 4;
        indices[idx++] = vertextID;
        indices[idx++] = vertextID+1;
        indices[idx++] = vertextID+2;
        indices[idx++] = vertextID+1;
        indices[idx++] = vertextID+3;
        indices[idx++] = vertextID+2;
    }
}
```


<a name="PpXkb"></a>
## 顶点格式自定义

现在进入正题，基于上面对Assembler以及相关类的解读，顶点格式自定义需要做这么几件事

1. 定义新的格式
1. 用新的格式准备足够长的renderData
1. 在renderData对应位置写入自定义数据
1. 在fillBuffers()方法内将renderData数据正确刷入buffer

```typescript
// 自定义顶点格式，在vfmtPosUvColor基础上，加入gfx.ATTR_UV1，去掉gfx.ATTR_COLOR
let gfx = cc.gfx;
var vfmtCustom = new gfx.VertexFormat([
    { name: gfx.ATTR_POSITION, type: gfx.ATTR_TYPE_FLOAT32, num: 2 },
    { name: gfx.ATTR_UV0, type: gfx.ATTR_TYPE_FLOAT32, num: 2 },        // texture纹理uv
    { name: gfx.ATTR_UV1, type: gfx.ATTR_TYPE_FLOAT32, num: 2 }         // 自定义数据
]);

const VEC2_ZERO = cc.Vec2.ZERO;

export default class MovingBGAssembler extends GTSimpleSpriteAssembler2D {
    // 根据自定义顶点格式，调整下述常量
    verticesCount = 4;
    indicesCount = 6;
    uvOffset = 2;
    uv1Offset = 4;
    floatsPerVert = 6;

    // 自定义数据，将被写入uv1的位置
    public moveSpeed: cc.Vec2 = VEC2_ZERO;
    initData() {
        let data = this._renderData;
        // createFlexData支持创建指定格式的renderData
        data.createFlexData(0, this.verticesCount, this.indicesCount, this.getVfmt());

        // createFlexData不会填充顶点索引信息，手动补充一下
        let indices = data.iDatas[0];
        let count = indices.length / 6;
        for (let i = 0, idx = 0; i < count; i++) {
            let vertextID = i * 4;
            indices[idx++] = vertextID;
            indices[idx++] = vertextID+1;
            indices[idx++] = vertextID+2;
            indices[idx++] = vertextID+1;
            indices[idx++] = vertextID+3;
            indices[idx++] = vertextID+2;
        }
    }

    // 自定义格式以getVfmt()方式提供出去，除了当前assembler，render-flow的其他地方也会用到
    getVfmt() {
        return vfmtCustom;
    }

    // 重载getBuffer(), 返回一个能容纳自定义顶点数据的buffer
    // 默认fillBuffers()方法中会调用到
    getBuffer() {
        return cc.renderer._handle.getBuffer("mesh", this.getVfmt());
    }

    // pos数据没有变化，不用重载
    // updateVerts(sprite) {
    // }

    updateUVs(sprite) {
        // uv0调用基类方法写入
        super.updateUVs(sprite);
        // 填入自己的uv1数据
        // ...
        // 方法类似uv0写入，详见Demo
      	// https://github.com/caogtaa/CCBatchingTricks
    }

    updateColor(sprite) {
        // 由于已经去掉了color字段，这里重载原方法，并且不做任何事
    }
}
```
<br />上面用到的 **GTSimpleSpriteAssembler2D**基类代码大部分参考官方cc.Sprite的实现。<br />

<br />

<a name="Jnc5H"></a>
## 双uv坐标shader案例

这里将通过额外的一组uv数据，实现纹理滚动的方向 & 速度控制。<br />
![](/img/in-post/20200717/movingbg.gif)
<br />用材质参数的方法同样能够实现这个效果，但是无法做到合批渲染。<br />基于上面给出的Assembler类，继续完善一下其他辅助类<br />

<a name="mvcIP"></a>
### 材质

材质只用于关联effect，没有额外逻辑，也不需要新建uniform变量<br />

<a name="CjSDJ"></a>
### RenderComponent (渲染组件)

Assembler可以理解为渲染组件的渲染数据装配器，渲染组件需要关联对应的Assembler才能进行渲染数据的更新和提交。本例基于常规的cc.Sprite组件加入自定义数据`moveSpeed`用于控制纹理移动。

```typescript
@ccclass
export default class MovingBGSprite extends cc.Sprite {
    @property(cc.Vec2)
    moveSpeed: cc.Vec2 = cc.Vec2.ZERO;

    // 将自定义数据传递给assembler，在设置完所有参数后调用
    // 也可以在moveSpeed setter方法内主动传值，需要调用setVertsDirty()使顶点数据重算
    public FlushProperties() {
        let assembler: MovingBGAssembler = this._assembler;
        if (!assembler)
            return;

        assembler.moveSpeed = this.moveSpeed;
        this.setVertsDirty();
    }

    _resetAssembler () {
        this.setVertsDirty();
        let assembler = this._assembler = new MovingBGAssembler();
        this.FlushProperties();

        assembler.init(this);
    }
}
```
<a name="kLElQ"></a>
### Effect (shader)

滚动效果非常简单，这里只贴出片元着色器代码<br />纹理滚动通过v_uv1.xy控制方向和速度
```C
CCProgram fs %{
  precision highp float;

  #include <cc-global>
  #include <cc-local>

  in vec2 v_uv0;
  in vec2 v_uv1;

  uniform sampler2D texture;

  void main()
  {
    vec2 uv = v_uv0.xy;
    float tx = cc_time.x * v_uv1.x;
    float ty = cc_time.x * v_uv1.y;

    uv.x = fract(uv.x - tx);
    uv.y = fract(uv.y + ty);
    vec4 col = texture(texture, uv);

    gl_FragColor = col;
  }
}%

```
将RenderComponent组件挂在到对应节点上并赋予上述材质即可使用。
至此，一个简单的自定义顶点格式达到合批目的的功能就实现了！


<a name="hjUGd"></a>
## Demo地址

demo基于Cocos Creator 2.4.0 （2.4会是Cocos Creator 2D的最后一个版本，也是LTS版本，大家赶紧用起来吧！）<br />
<br />如果小伙伴觉得这个Demo对自己有帮助，记得star哦~^_^~<br />[https://github.com/caogtaa/CCBatchingTricks](https://github.com/caogtaa/CCBatchingTricks)<br />

<a name="3ExgM"></a>
## 写在后面

实际项目中可以灵活利用自定义顶点格式，达到给shader传参的目的，同时不会打断合批。<br />当然想要实现合批渲染，还有其他前置条件要满足，包括节点层级关系、合图、纹理状态等，这些在论坛其他帖子有详细讨论。<br />
<br />有错误的地方欢迎指正


---------- 20200703更新 ----------
* Demo 扩展了顶点格式，用于处理动纹理动态合图之后的uv变化

---------- 20200706更新 ----------
* Demo 增加了分层合批渲染演示

[论坛讨论贴](https://forum.cocos.org/t/postrender-demo/95201)
