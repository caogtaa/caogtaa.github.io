---
layout: default
title:  "Shader练习场"
date:   2020-07-17 05:34:35 +0800
categories: shader
tags:
  - shader
---

<!--<script type="text/javascript" src="/js/GlslCanvas.js"></script>
<link type="text/css" rel="stylesheet" href="/css/glslEditor.css">
<script type="application/javascript" src="/js/glslEditor.js"></script>
<link type="text/css" rel="stylesheet" href="/css/glslGallery.css">
<script type="application/javascript" src="/js/glslGallery.js"></script>


<canvas class="glslCanvas" data-fragment-url="/shader/hello_world.frag" width="500" height="500"></canvas>
-->

<!-- 
https://github.com/patriciogonzalezvivo/glslEditor
-->

<head>
    <link type="text/css" rel="stylesheet" href="/css/glslEditor.css">
    <script type="application/javascript" src="/js/glslEditor.js"></script>
</head>

<body>
    <div id="glsl_editor"></div>
</body>
<script type="text/javascript">
    const glslEditor = new GlslEditor('#glsl_editor', { 
        canvas_size: 500,
        canvas_draggable: true,
        theme: 'monokai',
        multipleBuffers: true,
        watchHash: true,
        fileDrops: true,
        menu: true
    });
</script>
