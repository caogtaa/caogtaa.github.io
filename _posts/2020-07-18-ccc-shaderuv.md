---
layout: post
title: "自定义渲染应用——图片遮罩合批"
subtitle: "性能和特效我全都要！"
date: 2020-07-18
author: "GT"
tags: [cocos-creator, render]
---


<a name="52XV3"></a>
## 背景
不少同学在用shader实现某个特效后都会遇到这样一个问题，在测试的时候效果完全正确，一集成到项目中后纹理就发生错位。这种情况大多是由于shader使用的纹理参与了合图导致uv发生了变化。<br />
<br />简单的处理方法是去掉纹理的packable属性，使其不参与合图。但是这样会带来一个问题: **不合图意味着渲染时无法参与合批**，如果有大量节点用到了这个shader，那么drawcall就会较高。<br />
<br />经过一段时间的论坛灌水，发现有不少关于 **图片遮罩**的话题，所以本文就以批量图片遮罩为例介绍合批处理方法。

先来看两张效果图，一张是用 **shader画圆做遮罩**，这里的遮罩效果可以任意替换为其他shader效果；另一张是 **自定义纹理做遮罩，**合批渲染后均只占一个draw call。<br />
<img src="/img/in-post/20200723/shader_circle.gif" style="display:inline" width="281" height="500"><img src="/img/in-post/20200723/sprite_mask.gif" style="display:inline" width="281" height="500"><br />[Demo快速传送门](https://github.com/caogtaa/CCBatchingTricks)<br />本文的实现基于[这篇分享](/cocos-creator/render/2020/07/17/ccc-custom-vfmt/)所介绍的 **自定义顶点格式**，想要了解实现原理的同学可以去回顾一下。<br />

<a name="Yn4RT"></a>
## 纹理uv坐标
> 注：为了简单起见，本文不对图片的 **透明裁剪** 和合图时的 **旋转** 进行讨论。实际的代码里会做相应细节处理
> 所有描述都以图片不做透明裁剪以及不旋转合图讨论

ccc中纹理uv坐标 **以左上角为原点**。 `cc.SpriteFrame` 的uv属性中，uv坐标按 **左下、右下、左上、右上**顺序排列，其值的含义是这个 `cc.SpriteFrame` 的四个顶点在 `cc.Texture` 纹理空间中的坐标。<br /><br />
<img src="/img/in-post/20200723/auto_atlas.png" width="690" height="487">
<a name="V7n4J"></a>
## shader画圆遮罩
如果要用shader对图片做一个圆形遮罩，通常要计算像素距离图片中心的距离，这里需要获取图片中心的uv坐标。<br />但是实际上在合图之后，基于图片中心uv的计算很难保证画出一个圆，即使当前的 `cc.SpriteFrame` 长宽是相同的，在合图后的大纹理内 **并不能保证相对的长宽比例相同** 。<br />个人的推荐做法是将 `cc.SpriteFrame` 的 **uv重新映射到[0,1]区间**，方便shader处理。

```c
  // 将[a, b]区间映射到[0, 1]区间
  // t是[a, b]区间内的值
  // 函数返回t被映射后的值
  float Remap01(float a, float b, float t) {
    return (t-a) / (b-a);
  }

  void main () {
    vec2 uv = v_uv0.xy;
    vec4 col = texture(texture, uv);

    // v_xrange.xy分别表示子纹理的x轴左右边缘坐标
    // v_yrange.xy分别表示子纹理的y轴上下边缘坐标
    // uv的xy轴分别映射到[0, 1]区间，之后shader的写法即可按照未合图前的方式处理
    uv.x = Remap01(v_xrange.x, v_xrange.y, uv.x);
    uv.y = Remap01(v_yrange.x, v_yrange.y, uv.y);

    // 画圆形遮罩，以(0.5, 0.5)为圆心
    float d = distance(uv, vec2(0.5, 0.5));
    float r = 0.5;
    float mask = smoothstep(r + 0.01, r - 0.01, d);
    col.a = mask;
 
    gl_FragColor = col;
  }
```
上面这段片元着色器中用到了 `v_xrange` ， `v_yrange` 两个变量，需要以参数形式输入。<br />为了 **不打断合批**，使用[这篇分享](/cocos-creator/render/2020/07/17/ccc-custom-vfmt/)所介绍的 **自定义顶点格式**方式传入。
> 这种映射不仅能处理画圆的场景，任何需要计算相对位置、距离的shader都可以用这种方式处理合图后的uv
> 实际的Demo代码中使用的是Remap01优化后的变种，但是原理是完全一致的


<br />

<a name="IVs0z"></a>
## 纹理遮罩
纹理遮罩合批需要保证底图和遮罩都参与合图。<br />可以选择 **底图合一张大图，遮罩合另一张大图**；也可以选择 **底图、遮罩都合到一起**。<br />遮罩的uv也是通过顶点属性传入shader。

```c
  in vec2 v_uv0;              // 底图uv
  uniform sampler2D texture;  // 底图纹理
  in vec2 v_mask_uv;          // 遮罩图uv，通过顶点属性传入
  uniform sampler2D mask;     // 遮罩图纹理，通过材质属性传入
  uniform UARGS {
    float enableMask;         // 遮罩图开关控制，通过材质属性传入
  };

  void main () {
    // 对底图采样
    vec4 col = texture(texture, v_uv0);

    // 对遮罩图采样
    vec4 maskCol = texture(mask, v_mask_uv);

    // 片元透明度使用遮罩图透明度
    // enableMask控制是否让遮罩生效
    col.a = mix(col.a, maskCol.a, enableMask);
    gl_FragColor = col;
  }
```

<br />

<a name="Apqv9"></a>
## 材质属性 or 顶点属性？
自定义顶点格式这么好用，是不是可以抛弃材质属性(uniform变量)了？<br />**NO!**<br />
<br />用顶点属性做传参是为了实现合批渲染的一种权宜的处理方式。

1. 顶点属性会在每个顶点上 **冗余一份数据**，即使这些数据是一模一样的，会少量增加内存使用和顶点数据拷贝时间。所以用顶点属性传参适合顶点数量少的渲染组件。一次draw call中统一的数据务必使用材质属性。
1. 如果某个特效只对少数几个渲染组件生效，甚至是否合批都无所谓，建议使用材质属性进行传参

两种传参方式可以结合，实际应用中根据项目需求灵活运用。<br />
<br />[Demo地址](https://github.com/caogtaa/CCBatchingTricks)