---
title: "WJ.14: Chromium Rendering Pipeline - Introduce"
date: 2022-08-14T13:37:47+08:00
categories: ['chromium']
draft: false
---

{{% admonition type="warning" title="Warning" %}}
如果您看到本提示，说明该文章目前仍未完成，会不定期迭代更新，该系列文章完成时会合稿发布博客。如果您有任何疑问或反馈，请联系 [@airing](https://ursb.me)。**Last update: August 20 2022**
{{% /admonition %}}

{{% admonition type="sourcecode" title="Purpose of this document" %}}
本文档系个人研究 Chromium 渲染管线的笔记记录，本文介绍 Chromium 的一些基础知识与渲染管线的主要流程。

本文分析的源码基于当前最新版本 `107`，每个版本类名/函数名和对应的具体实现或许有所差异，但主逻辑目前来看不会有所变动。

推荐阅读本系列后续文章：

- [Chromium Rendering Pipeline - Parsing](/posts/weekly-15)
{{% /admonition %}}

## Introduce to Blink

Blink 是 Web 的渲染引擎，Chromium 和 Android Webview 都是用 Blink 作为渲染引擎。

{{% aside %}}
[Blink Public API](https://chromium.googlesource.com/chromium/src/+/HEAD/third_party/blink/public)
{{% /aside %}}

{{<figure src="https://airing.ursb.me/image%2Fblog%2F717306-20180919114752433-1539598716.png?max_age=25920000" attr="Blink in Web platform" >}}

{{% aside %}}
你可以阅读 [How Blink works](https://docs.google.com/document/d/1aitSOucL0VHZa9Z2vbRJSyAIsAz24kX8LFByQ5xQnUg/edit?pli=1#) 获取更多关于 Blink 的知识。
{{% /aside %}}

Blink 包括以下特性：

- 实现了 Web 平台的规范（也就是 [HTML 标准](https://html.spec.whatwg.org/multipage/?)），包括 DOM，CSS 和 WebIDL；  
- 内嵌 V8 运行 JavaScript；  
- 通过底层的网络栈请求资源；  
- 构建 DOM 树；  
- 计算样式和排版；  
- 内嵌 [Chrome Compositor](https://chromium.googlesource.com/chromium/src/+/HEAD/cc/README.md) 用于绘图；  

{{% admonition type="sourcecode" title="Source to read" %}}

从代码结构的角度来看，"Blink" 指的是 [//third_party/blink/](https://chromium.googlesource.com/chromium/src/+/HEAD/third_party/blink/) 目录下的代码。

从项目的角度来看，"Blink" 指的是实现 Web 平台特性的项目，实现这些 Web 特性的代码分布在 [//third_party/blink/](https://chromium.googlesource.com/chromium/src/+/HEAD/third_party/blink/)， [//content/renderer/](https://chromium.googlesource.com/chromium/src/+/HEAD/content/renderer/)，[//content/browser/](https://chromium.googlesource.com/chromium/src/+/HEAD/content/browser/) 及其他相关依赖。
{{% /admonition %}}

### Renderer architecture overview

本文中我们重点介绍渲染管线，它的代码在 [//third_party/blink/renderer](https://chromium.googlesource.com/chromium/src/+/HEAD/third_party/blink/renderer/)下，目录结构如下所示：

- controller/: 一些使用 core/ 和 modules/ 的高级库。比如，devtools 的前端。
- core/ and bindings/core: core/ 实现了跟 DOM 紧密相关的特性。bindings/core 属于 core/ 的一部分，bindings 目录独立是因为它和 V8 强相关。
- modules/ and bindings/modules: modules/ 实现一些与 DOM 无关的特性，如 WebAudio，IndexDB 等。bindings/modules/ 属于 modules/ 的一部分, bindings 目录独立是因为它和 V8 强相关。
- platform/: Blink 的低阶功能集合，从庞大的 core/ 中分解出来，包括 geometry 和 graphics 等相关的功能。
- extensions/
- public/web
- public/platform
- public/common
- //base: chromium 基础库，基础库的源码解读可见 [Chromium //base](https://docs.google.com/document/d/13zOvNpBjUfI89HehtVl7CWQT-huJAUcvb7fNbse3fdc/edit#)
- //cc: chromium composite
- V8: JavaScript 解释器

{{<figure width="50%" src="https://airing.ursb.me/image%2Fblog%2F20220814145428%402x.png?max_age=25920000" attr="Dependencies in //third_party/blink/renderer" >}}

### Renderer processes

{{% aside %}}
过去进程间通信使用 [Chromium IPC](https://www.chromium.org/developers/design-documents/inter-process-communication)，现在还有很多代码仍旧在使用 [IPC](https://chromium.googlesource.com/chromium/+/refs/heads/trunk/ipc/)。
{{% /aside %}}

Chromium 拥有一个 browser processes 和 N 个沙盒 renderer processes，其中 Blink 运行在 renderer processes。browser 与 renderer 进程间通讯是由 [Mojo](https://chromium.googlesource.com/chromium/src/+/master/mojo/README.md)来实现的。

Blink renderer processes 模型如下图所示：

{{<figure src="https://airing.ursb.me/image%2Fblog%2FuIqf0QQZxF6mHPDWFEjz.png" attr="Renderer processes" attrlink="https://developer.chrome.com/blog/inside-browser-part3/">}}

{{% aside %}}
Renderer process with a main thread, worker threads, a compositor thread, and a raster thread inside.
{{% /aside %}}

Renderer process 拥有一个 main thread, 多个 worker threads, 一个 compositor thread, 以及一个 raster thread。

而 Blink 几乎所有的主要活动都发生在 main thread，如 JavaScript 调用，DOM 解析、CSS 样式和排版计算，因此 Blink 的架构被认为主要是单线程的。

## Rendering Pipeline

### Goals

渲染管线的目的是将 WebContent 渲染到屏幕上，它会将 HTML 的内容经过层层处理，最后通过 Open GL 操作显卡硬件驱动，将 HTML 内容光栅化输出到屏幕上。

需要注意的是 WebContent 指的是下图区域，即 HTML 的内容，不包括浏览器外壳：

{{<figure src="https://airing.ursb.me/image%2Fblog%2F20220814160559%402x.png" attr="WebContent" >}}

除此之外，渲染管线还设计了优良的数据结构，目的是高效处理渲染内容的更新。

### Pipeline

Blink 渲染管线如下图所示:

{{% aside %}}
"impl" = compositor thread. 
{{% /aside %}}

{{<figure src="https://airing.ursb.me/image%2Fblog%2Fimage_1660182336555_0.png?max_age=25920000" attr="Chromium Rendering Pipeline (by Life of a pixel)" >}}

这张图来自于 [Life of a pixel](https://docs.google.com/presentation/d/1boPxbgNrTU0ddsc144rcXayGA_WF53k96imRH8Mp34Y/edit#slide=id.ga884fe665f_64_45)，虽然描述了管线的关键流程，但不是很完整。此外也不是很准确，比如 Raster 阶段是运行在专门的 Raster thread 而不是 Compositor thread 中。

因此，结合 cc 与 viz 的工作流，重新绘制了下图：

{{<figure src="/images/chromium-rendering-pipeline/full-rendering-pipeline.png" attr="Chromium Rendering Pipeline" >}}

完整的渲染管线经历以下几个阶段:

1. Parsing
2. Style
3. Layout
4. Composting
5. Paint
6. Commit
7. Tiling
8. Raster
9. Activate
10. Submit
11. Draw
12. Display

渲染管线的前半程是在 Renderer process 中进行的，其中渲染管线的 1-5 阶段运行在 Main thread 中，这也是 Blink 负责的内容；而 6-10 阶段，除了 Raster 是运行在专门的 Raster thread 中，其它流程均是在 Compositor thread 中进行的，这也是 cc 的主要内容，其中 cc 的数据源由 Paint 产生。渲染管线的尾程则是在 GPU process 中进行的，跨进程的环节交给了 viz 负责。

在后续的文章，我们将逐个解析渲染管线的各个流程中 Chromium 是如何工作的。

## Resources

- [Inside look at modern web browser (part 3)](https://developer.chrome.com/blog/inside-browser-part3/)
- [Life of a pixel - Google Docs](https://docs.google.com/presentation/d/1boPxbgNrTU0ddsc144rcXayGA_WF53k96imRH8Mp34Y/edit#slide=id.ga884fe665f_64_45)
