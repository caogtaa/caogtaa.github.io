---
layout: post
title: "在Shader中处理Atlas的uv以及一点小优化"
subtitle: ""
date: 2020-08-05
author: "GT"
tags: []
---

<a name="2aeUg"></a>
## 背景
接上一篇<br />[https://forum.cocos.org/t/topic/95986](https://forum.cocos.org/t/topic/95986)<br />如何在最小改动shader的前提下，让shader支持atlas？<br />

<a name="PmkZK"></a>
## 方法
一个相对通用的方法，是将子纹理uv射到[0, 1]区间，其余部分保留shader原来的逻辑。<br />普通的映射函数如下， `Remap01` 将子纹理uv映射到[0, 1]区间， `Remap` 则支持任意两个区间的映射。

```c
// 将[a, b]区间映射到[0, 1]区间
// t是[a, b]区间内的值
// 函数返回t被映射后的值
float Remap01(float a, float b, float t) {
	return (t-a) / (b-a);
}

// 将[a, b]区间映射到[c, d]区间
// t是[a, b]区间内的值
// 函数返回t被映射后的值
float Remap(float a, float b, float c, float d, float t) {
    return Remap01(a, b, t) * (d - c) + c;
}
```
代码中的a、b参数，分别代表原区间的起始、结束，即子纹理在atlas中uv的坐标区间。这两个值需要通过顶点属性传入shader。<br />
![](/img/in-post/20200805/atlas_uv.png)
<br />在shader中通过以下代码实现映射
```c
// v_xrange、v_yrange分别表示子纹理在x轴、y轴上的uv区间
uv.x = Remap01(v_xrange.x, v_xrange.y, uv.x);
uv.y = Remap01(v_yrange.x, v_yrange.y, uv.y);
```

<a name="08yv8"></a>
## 优化Remap01方法
<a name="vBtKr"></a>
### 参数预计算

$(t-a) / (b-a) = t\frac{1}{b-a}+\frac{a}{a-b}$
<br />可以在CPU直接计算好$p=\frac{1}{b-a},q=\frac{a}{a-b}$代替v_xrange、v_yrange传入shader<br />`Remap01` 函数也可以简化为$tp+q$，一条MAD指令即可处理完。

<a name="2faef726"></a>
### 合并x、y轴运算
GPU被设计成可以高效处理四元数，一条指令里可以同时处理向量里的xyzw分量。<br />将x、y轴的p放在一个向量，x、y轴的q放在一个向量， `Remap01` 可以简化成如下形式
```c
// 正向映射
uv = uv * v_p + v_q;

// 反向映射也很方便
uv = (uv - v_q) / v_p
```
<br />
<a name="FJnpt"></a>
## 优化Remap方法
对 `Remap01` 的优化方法同样适用于任意区间映射的 `Remap` 函数，同时这个优化方法可以将原本a, b, c, d 4个参数简化为p, q 2个参数，在shader里并不用关心uv究竟被映射到了哪个区间。
