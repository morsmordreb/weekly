---
title: "WJ.15: Chromium Rendering Pipeline - Parsing"
date: 2022-08-20T00:00:47+08:00
categories: ['chromium']
draft: false
---

{{% admonition type="warning" title="Warning" %}}
如果您看到本提示，说明该文章目前仍未完成，会不定期迭代更新，该系列文章完成时会合稿发布博客。如果您有任何疑问或反馈，请联系 [@airing](https://ursb.me)。**Last update: August 21 2022**
{{% /admonition %}}

{{% admonition type="sourcecode" title="Purpose of this document" %}}
本文档系个人研究 Chromium 渲染管线的笔记记录，本文介绍 Chromium 渲染管线的 Parsing 阶段。

本文分析的源码基于当前最新版本 `107`，每个版本类名/函数名和对应的具体实现或许有所差异，但主逻辑目前来看不会有所变动。

如果你没有看过前文，推荐阅读：

- [Chromium Rendering Pipeline - Introduce](/posts/weekly-14)
{{% /admonition %}}

## Overview

Parsing 的目的是将 HTML 文档解析为 [HTML 标准](https://html.spec.whatwg.org/)中的 DOM 树结构和 CSSOM 树结构。

这个环节的核心逻辑是 Render process 接收到网络字节或来自 Service Worker 缓存之后，Parsing HTML 并构建 DOM Tree 和 CSSOM Tree。对于 DOM Tree 的解析，模型如下图所示：

{{<figure src="/images/chromium-rendering-pipeline/bytes-conversion.png" attr="The main thread parsing HTML and building a DOM tree" >}}

Parsing 涉及到的数据流为 bytes → characters → token → nodes → object model，对应到 DOM(document object model) Tree 的构建，如下转换过程所示：

1. Conversion: HTMLParser 将 bytes 转为 characters
2. Tokenizing: 将 characters 转为 W3C 标准的 token
3. Lexing: 通过词法分析将 token 转为 Element 对象
4. DOM construction: 使用构建好的 Element 对象构建 DOM Tree

而对于 CSSOM (css object model tree) 规则树的生成也是类型，流程为：

1. Conversion
2. Tokenizing
3. Lexing
4. CSSOM construction: 使用构建好的 Node 对象构建 CSSOM Tree

以下逐步分析 Parsing 的这四个子环节 conversion → tokenizing → Lexing → DOM/CSSOM construction，不过在开始之前先介绍 Loading，了解一下 Document Loader 是如何接收 bytes 的。

{{% admonition type="tryit" title="Read" %}}
推荐更多阅读材料：

- DOM 标准：[https://dom.spec.whatwg.org/](https://dom.spec.whatwg.org/)
- Blink 中的 DOM 实现：`renderer/core/dom`  
{{% /admonition %}}

## Loading

在 `HTMLDocumentParser::AppendBytes` 断点可以了解 Blink 是如何接收 bytes 的，堆栈如下：

```
#0	0x00000002d238276c in blink::HTMLDocumentParser::AppendBytes(char const*, unsigned long) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_document_parser.cc:1351
#1	0x00000002d1795144 in blink::DocumentLoader::CommitData(char const*, unsigned long) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/loader/document_loader.cc:1259
#2	0x00000002d1792d0c in blink::DocumentLoader::ProcessDataBuffer(char const*, unsigned long) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/loader/document_loader.cc:1479
#3	0x00000002d1792a54 in blink::DocumentLoader::BodyDataReceived(base::span<char const, 18446744073709551615ul>) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/loader/document_loader.cc:969
#4	0x00000002e17c96a8 in blink::NavigationBodyLoader::ReadFromDataPipe() at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/platform/loader/fetch/url_loader/navigation_body_loader.cc:288
#5	0x00000002e17c80a0 in blink::NavigationBodyLoader::OnReadable(unsigned int) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/platform/loader/fetch/url_loader/navigation_body_loader.cc:244
#6	0x00000002e17c8a94 in blink::NavigationBodyLoader::BindURLLoaderAndStartLoadingResponseBodyIfPossible() at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/platform/loader/fetch/url_loader/navigation_body_loader.cc:349
#7	0x00000002e17c835c in blink::NavigationBodyLoader::StartLoadingBody(blink::WebNavigationBodyLoader::Client*, blink::CodeCacheHost*) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/platform/loader/fetch/url_loader/navigation_body_loader.cc:150
#8	0x00000002d1799a08 in blink::DocumentLoader::StartLoadingBodyWithCodeCache() at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/loader/document_loader.cc:1758
#9	0x00000002d1799018 in blink::DocumentLoader::CreateParserPostCommit() at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/loader/document_loader.cc:2598
#10	0x00000002d1797f0c in blink::DocumentLoader::StartLoadingResponse() at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/loader/document_loader.cc:1662
#11	0x00000002d179e158 in blink::DocumentLoader::CommitNavigation() at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/loader/document_loader.cc:2533
#12	0x00000002d17f08e8 in blink::FrameLoader::CommitDocumentLoader(blink::DocumentLoader*, blink::HistoryItem*, blink::CommitReason) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/loader/frame_loader.cc:1365
#13	0x00000002d17f5980 in blink::FrameLoader::CommitNavigation(std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >, std::Cr::unique_ptr<blink::WebDocumentLoader::ExtraData, std::Cr::default_delete<blink::WebDocumentLoader::ExtraData> >, blink::CommitReason) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/loader/frame_loader.cc:1199
#14	0x00000002d073e558 in blink::WebLocalFrameImpl::CommitNavigation(std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >, std::Cr::unique_ptr<blink::WebDocumentLoader::ExtraData, std::Cr::default_delete<blink::WebDocumentLoader::ExtraData> >) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/frame/web_local_frame_impl.cc:2507
#15	0x000000013a5347f4 in content::RenderFrameImpl::CommitNavigationWithParams(mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> >, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >) at /Users/airing/Files/code/chromium/src/content/renderer/render_frame_impl.cc:2823
#16	0x000000013a580af8 in void base::internal::FunctorTraits<void (content::RenderFrameImpl::*)(mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> >, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >), void>::Invoke<void (content::RenderFrameImpl::*)(mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> >, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >), base::WeakPtr<content::RenderFrameImpl>, mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> >, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> > >(void (content::RenderFrameImpl::*)(mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> >, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >), base::WeakPtr<content::RenderFrameImpl>&&, mojo::StructPtr<blink::mojom::CommonNavigationParams>&&, mojo::StructPtr<blink::mojom::CommitNavigationParams>&&, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >&&, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >&&, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>&&, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>&&, mojo::PendingRemote<network::mojom::URLLoaderFactory>&&, mojo::PendingRemote<blink::mojom::CodeCacheHost>&&, mojo::StructPtr<content::mojom::CookieManagerInfo>&&, mojo::StructPtr<content::mojom::StorageInfo>&&, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> >&&, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >&&) at /Users/airing/Files/code/chromium/src/base/bind_internal.h:562
#17	0x000000013a5805c4 in void base::internal::InvokeHelper<true, void>::MakeItSo<void (content::RenderFrameImpl::*)(mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> >, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >), base::WeakPtr<content::RenderFrameImpl>, mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> >, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> > >(void (content::RenderFrameImpl::*&&)(mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> >, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >), base::WeakPtr<content::RenderFrameImpl>&&, mojo::StructPtr<blink::mojom::CommonNavigationParams>&&, mojo::StructPtr<blink::mojom::CommitNavigationParams>&&, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >&&, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >&&, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>&&, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>&&, mojo::PendingRemote<network::mojom::URLLoaderFactory>&&, mojo::PendingRemote<blink::mojom::CodeCacheHost>&&, mojo::StructPtr<content::mojom::CookieManagerInfo>&&, mojo::StructPtr<content::mojom::StorageInfo>&&, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> >&&, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >&&) at /Users/airing/Files/code/chromium/src/base/bind_internal.h:746
#18	0x000000013a5804d4 in void base::internal::Invoker<base::internal::BindState<void (content::RenderFrameImpl::*)(mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> >, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >), base::WeakPtr<content::RenderFrameImpl>, mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> > >, void (std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >)>::RunImpl<void (content::RenderFrameImpl::*)(mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> >, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >), std::Cr::tuple<base::WeakPtr<content::RenderFrameImpl>, mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> > >, 0ul, 1ul, 2ul, 3ul, 4ul, 5ul, 6ul, 7ul, 8ul, 9ul, 10ul, 11ul>(void (content::RenderFrameImpl::*&&)(mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> >, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >), std::Cr::tuple<base::WeakPtr<content::RenderFrameImpl>, mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> > >&&, std::Cr::integer_sequence<unsigned long, 0ul, 1ul, 2ul, 3ul, 4ul, 5ul, 6ul, 7ul, 8ul, 9ul, 10ul, 11ul>, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >&&) at /Users/airing/Files/code/chromium/src/base/bind_internal.h:799
#19	0x000000013a580244 in base::internal::Invoker<base::internal::BindState<void (content::RenderFrameImpl::*)(mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> >, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >), base::WeakPtr<content::RenderFrameImpl>, mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, std::Cr::unique_ptr<content::DocumentState, std::Cr::default_delete<content::DocumentState> > >, void (std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >)>::RunOnce(base::internal::BindStateBase*, std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >&&) at /Users/airing/Files/code/chromium/src/base/bind_internal.h:768
#20	0x000000013a534b30 in base::OnceCallback<void (std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >)>::Run(std::Cr::unique_ptr<blink::WebNavigationParams, std::Cr::default_delete<blink::WebNavigationParams> >) && at /Users/airing/Files/code/chromium/src/base/callback.h:145
#21	0x000000013a532ebc in content::RenderFrameImpl::CommitNavigation(mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, mojo::StructPtr<network::mojom::URLResponseHead>, mojo::ScopedHandleBase<mojo::DataPipeConsumerHandle>, mojo::StructPtr<network::mojom::URLLoaderClientEndpoints>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, base::UnguessableToken const&, std::Cr::vector<blink::ParsedPermissionsPolicyDeclaration, std::Cr::allocator<blink::ParsedPermissionsPolicyDeclaration> > const&, mojo::StructPtr<blink::mojom::PolicyContainer>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, base::OnceCallback<void (mojo::StructPtr<content::mojom::DidCommitProvisionalLoadParams>, mojo::StructPtr<content::mojom::DidCommitProvisionalLoadInterfaceParams>)>) at /Users/airing/Files/code/chromium/src/content/renderer/render_frame_impl.cc:2723
#22	0x000000013a523260 in content::NavigationClient::CommitNavigation(mojo::StructPtr<blink::mojom::CommonNavigationParams>, mojo::StructPtr<blink::mojom::CommitNavigationParams>, mojo::StructPtr<network::mojom::URLResponseHead>, mojo::ScopedHandleBase<mojo::DataPipeConsumerHandle>, mojo::StructPtr<network::mojom::URLLoaderClientEndpoints>, std::Cr::unique_ptr<blink::PendingURLLoaderFactoryBundle, std::Cr::default_delete<blink::PendingURLLoaderFactoryBundle> >, absl::optional<std::Cr::vector<mojo::StructPtr<blink::mojom::TransferrableURLLoader>, std::Cr::allocator<mojo::StructPtr<blink::mojom::TransferrableURLLoader> > > >, mojo::StructPtr<blink::mojom::ControllerServiceWorkerInfo>, mojo::StructPtr<blink::mojom::ServiceWorkerContainerInfoForClient>, mojo::PendingRemote<network::mojom::URLLoaderFactory>, base::UnguessableToken const&, std::Cr::vector<blink::ParsedPermissionsPolicyDeclaration, std::Cr::allocator<blink::ParsedPermissionsPolicyDeclaration> > const&, mojo::StructPtr<blink::mojom::PolicyContainer>, mojo::PendingRemote<blink::mojom::CodeCacheHost>, mojo::StructPtr<content::mojom::CookieManagerInfo>, mojo::StructPtr<content::mojom::StorageInfo>, base::OnceCallback<void (mojo::StructPtr<content::mojom::DidCommitProvisionalLoadParams>, mojo::StructPtr<content::mojom::DidCommitProvisionalLoadInterfaceParams>)>) at /Users/airing/Files/code/chromium/src/content/renderer/navigation_client.cc:53
```

{{% aside %}}
关于 Mojo 的介绍可见本文文末最后一个章节。<br/>`DocumentLoader` 的缓冲区使用 `data_buffer_` 成员变量存储，只有一些特殊情况会用到缓存区逻辑，如 FTP、mhtml 解析等。
{{% /aside %}}

Browser process 一边下载网页的内容，一边将下载回来的网页交给 Render process 的 Content 模块，`content::` 中使用 Mojo 从 Browser process 向 Renderer process 发送 network 线程的数据，而 `blink::NavigationBodyLoader` 会绑定 Mojo Data Pipe 接收 Content 模块传递过来的网页内容，之后交由 `blink::DocumentLoader` 做缓冲处理，`DocumentLoader` 最后会将缓冲区的 bytes 提交给 `HTMLDocumentParser` 做解析处理。

{{<figure src="/images/chromium-rendering-pipeline/loading.png" attr="IPC between the browser and the renderer processes, requesting to render the page" >}}

{{% aside %}}
Conversion: bytes → characters
{{% /aside%}}


## Conversion

根据编码格式做 Decode 处理，我们从 `HTMLDocumentParser::AppendBytes` 往下，最后走到 `HTMLDocumentParser::Append` 发现它的参数列表已经是 `WTF::String` 了，那么 Conversion 就是这两个函数中间堆栈做的事情，断点堆栈如下所示：

```
#0	0x00000002d2380488 in blink::HTMLDocumentParser::Append(WTF::String const&) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_document_parser.cc:1037
#1	0x00000002cfec278c in blink::DecodedDataDocumentParser::UpdateDocument(WTF::String&) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/dom/decoded_data_document_parser.cc:98
#2	0x00000002cfec268c in blink::DecodedDataDocumentParser::AppendBytes(char const*, unsigned long) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/dom/decoded_data_document_parser.cc:71
#3	0x00000002d2382778 in blink::HTMLDocumentParser::AppendBytes(char const*, unsigned long) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_document_parser.cc:1351
```

`HTMLDocumentParser::AppendBytes` 会检查入参数据和当前是否为主线程，之后调用 `DecodedDataDocumentParser::AppendBytes`。

{{% admonition type="sourcecode" title="Source Code" %}} 
HTMLDocumentParser::AppendBytes

```cpp
void HTMLDocumentParser::AppendBytes(const char* data, size_t length) {
  TRACE_EVENT2("blink", "HTMLDocumentParser::appendBytes", "size",
               (unsigned)length, "parser", (void*)this);

  DCHECK(Thread::MainThread()->IsCurrentThread());

  if (!length || IsStopped())
    return;

  DecodedDataDocumentParser::AppendBytes(data, length);
}
```
{{% /admonition %}}

DecodedDataDocumentParser 类的成员变量 `decoder_` 指向一个 TextResourceDecoder 对象，这个TextResourceDecoder 对象负责对下载回来的网页数据进行解码(`TextResourceDecoder::Decode`)，解码后得到网页数据的字符串表示，这个字符串将会交给由另外一个成员函数 UpdateDocument 进行处理。

{{% admonition type="sourcecode" title="Source Code" %}} 
DecodedDataDocumentParser::AppendBytes

```cpp
void DecodedDataDocumentParser::AppendBytes(const char* data, size_t length) {
  // ...
  String decoded = decoder_->Decode(data, length);
  UpdateDocument(decoded);
}
```
{{% /admonition %}}

{{% admonition type="sourcecode" title="Source Code" %}} 

DecodedDataDocumentParser::UpdateDocument

```cpp
void DecodedDataDocumentParser::UpdateDocument(String& decoded_data) {
  // A Document created from XSLT may have changed the encoding of the data
  // before feeding it to the parser, so don't overwrite the encoding data XSLT
  // provided about the original encoding.
  if (!DocumentXSLT::HasTransformSourceDocument(*GetDocument()))
    GetDocument()->SetEncodingData(DocumentEncodingData(*decoder_.get()));

  if (!decoded_data.IsEmpty())
    Append(decoded_data);
}
```
{{% /admonition %}}


最后将解析后的数据交给 `HTMLDocumentParser::Append` 处理，至此完成了 bytes → characters 的转换，接下来便是解析 token。

{{% aside %}}
Tokenizing: characters → token
{{% /aside%}}

## Tokenizing

{{% aside %}}
HTMLInputStream input_
{{% /aside %}}

接上节所述，调用栈到了 `HTMLDocumentParser::Append`，该方法首先将 characters 附加在成员变量 `input_` 描述的一个输入流中，接下来再调用 `HTMLDocumentParser::PumpTokenizerIfPossible` 对该输入流 `input_` 中的 characters 进行解析。

{{% admonition type="sourcecode" title="Source Code" %}} 

HTMLDocumentParser::Append

```cpp
void HTMLDocumentParser::Append(const String& input_source) {
  TRACE_EVENT2("blink", "HTMLDocumentParser::append", "size",
               input_source.length(), "parser", (void*)this);

  if (IsStopped())
    return;

  const SegmentedString source(input_source);

  ScanInBackground(input_source);

  if (!background_scanner_ && !preload_scanner_ && preloader_ &&
      GetDocument()->Url().IsValid() &&
      (!task_runner_state_->IsSynchronous() ||
       GetDocument()->IsPrefetchOnly() || IsPaused())) {
    // If we're operating with a budget, we need to create a preload scanner to
    // make sure that parser-blocking Javascript requests are dispatched in
    // plenty of time, which prevents unnecessary delays.
    // When parsing without a budget (e.g. for HTML fragment parsing), it's
    // additional overhead to scan the string unless the parser's already
    // paused whilst executing a script.
    preload_scanner_ =
        CreatePreloadScanner(TokenPreloadScanner::ScannerType::kMainDocument);
  }

  if (GetDocument()->IsPrefetchOnly()) {
    if (preload_scanner_) {
      preload_scanner_->AppendToEnd(source);
      // TODO(Richard.Townsend@arm.com): add test coverage of this branch.
      // The crash in crbug.com/1166786 indicates that text documents are being
      // speculatively prefetched.
      ScanAndPreload(preload_scanner_.get());
    }

    // Return after the preload scanner, do not actually parse the document.
    return;
  }
  if (preload_scanner_) {
    preload_scanner_->AppendToEnd(source);
    if (task_runner_state_->GetMode() == kAllowDeferredParsing &&
        (IsPaused() || !task_runner_state_->SeenFirstByte())) {
      // Should scan and preload if the parser's paused waiting for a resource,
      // or if we're starting a document for the first time (we want to at least
      // prefetch anything that's in the <head> section).
      ScanAndPreload(preload_scanner_.get());
    }
  }

  input_.AppendToEnd(source);
  task_runner_state_->MarkSeenFirstByte();

  // Add input_source.length() to "file size" metric.
  if (metrics_reporter_)
    metrics_reporter_->AddInput(input_source.length());

  if (task_runner_state_->InPumpSession()) {
    // We've gotten data off the network in a nested write. We don't want to
    // consume any more of the input stream now.  Do not worry.  We'll consume
    // this data in a less-nested write().
    return;
  }

  // If we are preloading, FinishAppend() will be called later in
  // CommitPreloadedData().
  if (IsPreloading())
    return;

  FinishAppend();
}

void HTMLDocumentParser::FinishAppend() {
  // ...
  PumpTokenizerIfPossible();
}
```
{{% /admonition %}}


根据 `HTMLDocumentParser::Append` 的实现，我们可以发现在这个过程中会处理一些 Preload 的资源进行预请求。网站通常会使用外链资源，比如 img、css 和 JavaScript 资源，这些文件需要从网络或缓存加载。Main thread 在解析构建 DOM 时逐个请求它们，为了加快网站的加载速度，preload scanner 是并发执行的。即如果在 `Append` 过程中发现有 `<img>` 或 `<link>` 之类的内容，preload scanner(`HTMLParserScriptRunner`)会直接获取 HTML parser 解析出来的资源地址，并将请求发送到 browser process 中的 network thread 以获取远程的网络资源。

{{<figure src="/images/chromium-rendering-pipeline/parsing-html.png" attr="The main thread parsing HTML and building a DOM tree" >}}

在 `Append` 函数的最后会调用成员函数 `PumpTokenizerIfPossible`，尝试进行 Tokenizing。我们走进 `HTMLDocumentParser::PumpTokenizerIfPossible`，先单步断点看看调用栈的情况:

```
#0	0x00000002d23dc084 in blink::HTMLTreeBuilder::ProcessToken(blink::AtomicHTMLToken*) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_tree_builder.cc:363
#1	0x00000002d23dadc8 in blink::HTMLTreeBuilder::ConstructTree(blink::AtomicHTMLToken*) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_tree_builder.cc:329
#2	0x00000002d237eff0 in blink::HTMLDocumentParser::ConstructTreeFromHTMLToken() at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_document_parser.cc:954
#3	0x00000002d237d3a0 in blink::HTMLDocumentParser::PumpTokenizer() at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_document_parser.cc:837
#4	0x00000002d237b734 in blink::HTMLDocumentParser::PumpTokenizerIfPossible() at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_document_parser.cc:703
```

{{% admonition type="sourcecode" title="Source Code" %}} 

HTMLDocumentParser::PumpTokenizerIfPossible

```cpp
void HTMLDocumentParser::PumpTokenizerIfPossible() {
  TRACE_EVENT1("blink", "HTMLDocumentParser::PumpTokenizerIfPossible", "parser",
               (void*)this);

  bool yielded = false;
  CheckIfBlockingStylesheetAdded();
  if (!IsStopped() &&
      (!IsPaused() || task_runner_state_->ShouldEndIfDelayed())) {
    yielded = PumpTokenizer();
  }

  if (yielded) {
    DCHECK(!task_runner_state_->ShouldComplete());
    SchedulePumpTokenizer();
  } else if (task_runner_state_->ShouldAttemptToEndOnEOF()) {
    // Fall into this branch if ::Finish has been previously called and we've
    // just finished asynchronously parsing everything.
    if (metrics_reporter_)
      metrics_reporter_->ReportMetricsAtParseEnd();
    AttemptToEnd();
  } else if (task_runner_state_->ShouldEndIfDelayed()) {
    // If we did not exceed the budget or parsed everything there was to
    // parse, check if we should complete the document.
    if (task_runner_state_->ShouldComplete() || IsStopped() || IsStopping()) {
      if (metrics_reporter_)
        metrics_reporter_->ReportMetricsAtParseEnd();
      EndIfDelayed();
    } else {
      ScheduleEndIfDelayed();
    }
  }
}
```
{{% /admonition %}} 

{{% aside %}}
因此我们常说 CSS 加载会阻塞渲染。
{{% /aside %}}

函数体内的 `CheckIfBlockingStylesheetAdded` 调用会检查当前是否有 Stylesheet 追加，如果有的话就会阻止继续 parsing。

{{% aside %}}
task_runner_state 用于追踪 HTML Parser 内部的状态，可以自行阅读 `HTMLDocumentParserState` 类的实现了解，这里不赘述。
{{% /aside %}}

{{% admonition type="warning" title="Warning" %}}
需要后续补充 `HTMLDocumentParser::SchedulePumpTokenizer` 异步解析的相关内容。
{{% /admonition %}}

`HTMLDocumentParser::PumpTokenizerIfPossible` 中还存在异步解析的逻辑，逻辑分支较多，但不管怎样都会调用到 `HTMLDocumentParser::PumpTokenizer` 进行具体的 Tokenizing。

{{% admonition type="sourcecode" title="Source Code" %}} 
HTMLDocumentParser::PumpTokenizer
`
``cpp
bool HTMLDocumentParser::PumpTokenizer() {
  
  // ...
  while (!should_yield) {
    if (task_runner_state_->ShouldProcessPreloads())
      FlushPendingPreloads();

    const auto next_token_status = CanTakeNextToken();
    if (next_token_status == kNoTokens) {
      // No tokens left to process in this pump, so break
      break;
    } else if (next_token_status == kHaveTokensAfterScript &&
               task_runner_state_->HaveExitedHeader()) {
      // Just executed a parser-blocking script in the body. We'd probably like
      // to yield at some point soon, especially if we're in "extended budget"
      // mode. So reduce the budget back to at most the default.
      budget = std::min(budget, kDefaultMaxTokenizationBudget);
      if (TimedParserBudgetEnabled()) {
        timed_budget = std::min(timed_budget, chunk_parsing_timer_.Elapsed() +
                                                  GetDefaultTimedBudget());
      }
    }
    {
      RUNTIME_CALL_TIMER_SCOPE(
          V8PerIsolateData::MainThreadIsolate(),
          RuntimeCallStats::CounterId::kHTMLTokenizerNextToken);
      if (!tokenizer_->NextToken(input_.Current(), Token()))
        break;
      budget--;
      tokens_parsed++;
    }
    ConstructTreeFromHTMLToken();
    // ...
  }
  // ...
}
```
{{% /admonition %}} 

这段代码首先 `CanTakeNextToken` 判断是否能继续获取 token，如果是就调用 `NextToken` 获取下一个 token，之后调用 `HTMLDocumentParser::ConstructTreeFromHTMLToken`。

{{% aside %}}
CanTakeNextToken 函数内部调用 HasParserBlockingScript 判断当前是否有 JavaScript 脚本，如果有就暂停解析，并调用 RunScriptsForPausedTreeBuilder 在 main thread 优先执行 JavaScript 逻辑。
{{% /aside %}}

不过这段代码中，我们先需要注意的是 RUNTIME_CALL_TIMER_SCOPE(V8PerIsolateData::MainThreadIsolate(), RuntimeCallStats::CounterId::kHTMLTokenizerNextToken) 这段逻辑，在 parse 的过程中如果遇到 V8 中有 JavaScript 需要运行，或者解析的网页内容超过一定量时，如果当前线程花在解析网页内容的时间超过预设的阀值(`TimedParserBudgetEnabled`)，那么当前线程就会自动放弃 CPU，通过一个定时器等待一小段时间后再继续解析剩下的网页内容。

{{% admonition type="tryit" title="Note" %}}
JavaScript can block the parsing

当 HTML Parser 解析到 `<script>` 标签时，Parser 会暂停 HTML 解析并优先执行 `<script>` 中的 JavaScript 代码。

因为 JavaScript 代码逻辑（如 `document.write()`）可能会改变 DOM 的布局，这就是为什么在继续解析 HTML Document 之前，HTML Parser 必须等待 JavaScript 运行。

{{% /admonition %}}

所以完整的 Parsing 模型应如下所示: 

{{<figure height="600" src="/images/chromium-rendering-pipeline/parsing-model-overview.svg" attr="Parsing Model Overview">}}


回到 `HTMLDocumentParser::PumpTokenizer`，我们分析下 `NextToken` 究竟是如何获取 token 的，这是 Tokenizing 的核心。

{{% aside %}}
Tokenization 的实现依照 W3C 规范 13.2.5，可阅读了解： [https://html.spec.whatwg.org/#tokenization](https://html.spec.whatwg.org/#tokenization)
{{% /aside %}}

`NextToken` 是 HTMLTokenizer 的对象 `tokenizer_` 的成员函数，`HTMLTokenizer::NextToken` 逻辑很简单且冗长(1600行)，根据当前字符判断下个字符是否可能属于某个 token，源码在 third_party/blink/renderer/core/html/parser/html_tokenizer.cc 文件下，可自行阅读。

{{% admonition type="sourcecode" title="Source Code" %}} 
HTMLTokenizer::NextToken

```cpp
bool HTMLTokenizer::NextToken(SegmentedString& source, HTMLToken& token) {
    // ...
    switch (state_) {
        HTML_BEGIN_STATE(kDataState) {
        if (cc == '&')
            HTML_ADVANCE_PAST_NON_NEWLINE_TO(kCharacterReferenceInDataState);
        else if (cc == '<') {
            if (token_->GetType() == HTMLToken::kCharacter) {
            // We have a bunch of character tokens queued up that we
            // are emitting lazily here.
            return true;
            }
            HTML_ADVANCE_PAST_NON_NEWLINE_TO(kTagOpenState);
        } else if (cc == kEndOfFileMarker)
            return EmitEndOfFile(source);
        else {
            return EmitData(source, cc);
        }
        }
        END_STATE()
        // ...
    }
    // ...
}
```
{{% /admonition %}} 

至此 Tokenizing 逻辑结束，我们获取到了一个个独立的 token。

{{% aside %}}
Lexing: token → Element
{{% /aside%}}

## Lexing

在 `HTMLDocumentParser::PumpTokenizer` 的最后调用了 `ConstructTreeFromHTMLToken` 来构建 DOM Tree，我们接着看 `HTMLDocumentParser::ConstructTreeFromHTMLToken`，源码如下:

{{% admonition type="sourcecode" title="Source Code" %}} 

HTMLDocumentParser::ConstructTreeFromHTMLToken

```cpp
void HTMLDocumentParser::ConstructTreeFromHTMLToken() {
  DCHECK(!GetDocument()->IsPrefetchOnly());

  AtomicHTMLToken atomic_token(Token());

  // Check whether we've exited the header.
  if (!task_runner_state_->HaveExitedHeader()) {
    if (GetDocument()->body()) {
      task_runner_state_->SetExitedHeader();
    }
  }

  // We clear the token_ in case ConstructTree() synchronously re-enters the
  // parser.
  Token().Clear();

  tree_builder_->ConstructTree(&atomic_token);
  CheckIfBlockingStylesheetAdded();
}
```
{{% /admonition %}} 

如果解析完 `<head>` 部分需要修改 `task_runner_state_` 的状态，之后调用 `HTMLTreeBuilder::ConstructTree` 并传入解析到的 token。这个过程中依然会调用 `CheckIfBlockingStylesheetAdded` 检查是否有样式表插入。

{{% admonition type="sourcecode" title="Source Code" %}} 
HTMLTreeBuilder::ConstructTree

```cpp
void HTMLTreeBuilder::ConstructTree(AtomicHTMLToken* token) {
  RUNTIME_CALL_TIMER_SCOPE(V8PerIsolateData::MainThreadIsolate(),
                           RuntimeCallStats::CounterId::kConstructTree);
  if (ShouldProcessTokenInForeignContent(token))
    ProcessTokenInForeignContent(token);
  else
    ProcessToken(token);

  if (parser_->IsDetached())
    return;

  bool in_foreign_content = false;
  if (!tree_.IsEmpty()) {
    HTMLStackItem* adjusted_current_node = AdjustedCurrentStackItem();
    in_foreign_content =
        !adjusted_current_node->IsInHTMLNamespace() &&
        !HTMLElementStack::IsHTMLIntegrationPoint(adjusted_current_node) &&
        !HTMLElementStack::IsMathMLTextIntegrationPoint(adjusted_current_node);
  }

  parser_->Tokenizer()->SetForceNullCharacterReplacement(
      insertion_mode_ == kTextMode || in_foreign_content);
  parser_->Tokenizer()->SetShouldAllowCDATA(in_foreign_content);

  tree_.ExecuteQueuedTasks();
  // We might be detached now.
}
```
{{% /admonition %}}

{{% aside %}}
Foreign Content: 即不是 HTML 标签相关的内容，如 MathML 和 SVG 这种外部标签相关的内容。对应 W3C 规范可查询 [http://www.whatwg.org/specs/web-apps/current-work/multipage/tree-construction.html#tree-construction](http://www.whatwg.org/specs/web-apps/current-work/multipage/tree-construction.html#tree-construction)
{{% /aside %}}

`HTMLTreeBuilder::ConstructTree` 首先调用 `ShouldProcessTokenInForeignContent` 判断参数 `token` 描述的 Token 是否为Foreign Content，如果是，就调用 `ProcessTokenInForeignContent` 对它进行处理，函数内部会判断这些 token 属于什么类型，如注释、开标签、闭标签、内容等，解析完之后生成 Node 并将其推入到 HTMLElementStack 中。在后文另一个分支 `ProcessToken` 的调用里会介绍 Node 的生成与压栈逻辑，此处不赘述，这部分源码可见：third_party/blink/renderer/core/html/parser/html_tree_builder.cc。

而如果 `token` 描述的是一个 HTML 标签相关的内容，那么调用 `HTMLTreeBuilder::ProcessToken` 对它进行处理，后文着重介绍 `ProcessToken`。

{{% aside %}}
HTML 具有很强的健壮性，Lexing 阶段的这个环节遇到异常时会根据规范做对应的异常处理，具体规范可见[An introduction to error handling and strange cases in the parser - HTML spec](https://html.spec.whatwg.org/multipage/parsing.html#an-introduction-to-error-handling-and-strange-cases-in-the-parser)
{{% /aside %}}

我们先回到 `HTMLTreeBuilder::ConstructTree` 看看后续逻辑，这里的 `tree_` 是一个 `HTMLConstructionSite` 类型的成员变量后续分析 DOM Tree 构建的时候再来看它，我们重点先看看 `tree_.ExecuteQueuedTasks()`。该方法会执行保存在 `HTMLConstructionSite` 内部的一个事件队列的任务，这些任务都是在 Tokenizing 的过程中添加到事件队列中去的，主要是为了处理在网页中没有正确闭合格式化的 Tag。即在 Lexing 阶段，Parser 保证了 HTML 结构的健壮性。


接着我们来看 Lexing 的核心函数 `HTMLTreeBuilder::ProcessToken` 的实现。

{{% admonition type="sourcecode" title="Source Code" %}} 
HTMLTreeBuilder::ProcessToken

```cpp
void HTMLTreeBuilder::ProcessToken(AtomicHTMLToken* token) {
  if (token->GetType() == HTMLToken::kCharacter) {
    ProcessCharacter(token);
    return;
  }

  // Any non-character token needs to cause us to flush any pending text
  // immediately. NOTE: flush() can cause any queued tasks to execute, possibly
  // re-entering the parser.
  tree_.Flush(kFlushAlways);
  should_skip_leading_newline_ = false;

  switch (token->GetType()) {
    case HTMLToken::kUninitialized:
    case HTMLToken::kCharacter:
      NOTREACHED();
      break;
    case HTMLToken::DOCTYPE:
      ProcessDoctypeToken(token);
      break;
    case HTMLToken::kStartTag:
      ProcessStartTag(token);
      break;
    case HTMLToken::kEndTag:
      ProcessEndTag(token);
      break;
    case HTMLToken::kComment:
      ProcessComment(token);
      break;
    case HTMLToken::kEndOfFile:
      ProcessEndOfFile(token);
      break;
  }
}
```
{{% /admonition %}}


如果 Token 的类型是 `HTMLToken::Character`，就表示该 Token 代表的是一个普通文本，这些内容不会马上进行处理，而是先调用 `ProcessCharacter` 将其保存在内部的一个 `PendingText` 缓冲区中。

{{% admonition type="sourcecode" title="Source Code" %}} 
HTMLTreeBuilder::ProcessCharacter

```cpp
void HTMLTreeBuilder::ProcessCharacter(AtomicHTMLToken* token) {
  // ...
  CharacterTokenBuffer buffer(token);
  ProcessCharacterBuffer(buffer);
}
```
{{% /admonition %}}

等到遇到下一个要处理的 Token 的类型不是 `HTMLToken::Character` 时，才会调用 `HTMLConstructionSite::Flush` 对缓冲区里的 characters 进行处理，它会把这些文本挂载到最近的 DOM 节点下。

{{% aside %}}
这部分解析遵循 HTML Standard 规范，可自行阅读: [https://html.spec.whatwg.org/multipage/syntax.html](https://html.spec.whatwg.org/multipage/syntax.html)
{{% /aside %}}

而对于非 `HTMLToken::Character` 类型的 Token，会根据不同类型调用不同函数对 Token 做处理。在处理的过程中，就会使用栈结构存储 Node (HTML Tag)，以便后续构造 DOM Tree —— 例如对于 `HTMLToken::StartTag` 类型的 Token，就会调用 `ProcessStartTag` 执行一个压栈操作，而对于`HTMLToken::EndTag` 类型的 Token，就会调用 `ProcessEndTag` 执行一个出栈操作。

{{% admonition type="tryit" title="Note" %}} 
针对如下所示的 DOM Tree

```html
<div>
  <p>
    <div></div>
  </p>
  <span></span>
</div>
```

各 Node 压榨与出栈流程如下：

![](/images/chromium-rendering-pipeline/node-stack.png)

{{% /admonition %}}

本文具体讲解下压栈流程，先看看 `ProcessStartTag` 的实现，源码解析 token 会针对不同名字对应到 Tag 做分支处理，每个分支会调用自己的函数做进一步处理，代码比较繁琐有将近千行，我们择取一个分支看看。

{{% aside %}}
Insertion Mode 是解析 HTML 时的根据所处理的不同内容设置的状态变量，规范可见：[https://html.spec.whatwg.org/multipage/parsing.html#the-insertion-mode](https://html.spec.whatwg.org/multipage/parsing.html#the-insertion-mode)
{{% /aside %}}

{{% admonition type="sourcecode" title="Source Code" %}} 
HTMLTreeBuilder::ProcessStartTag

```cpp
void HTMLTreeBuilder::ProcessStartTag(AtomicHTMLToken* token) {
  switch (GetInsertionMode()) {
      // ...
      case kInBodyMode:
      DCHECK_EQ(GetInsertionMode(), kInBodyMode);
      ProcessStartTagForInBody(token);
      break;
      // ...
  }
  // ...
}
```
{{% /admonition %}}

当处理到 `<body>` 里面的内容时，Insertion Mode 就设置为 `kInBodyMode`，命中状态之后进一步执行 `ProcessStartTagForInBody` 对 tag 做处理。

{{% aside %}}
这个示例对应的规范可见: [https://html.spec.whatwg.org/multipage/parsing.html](https://html.spec.whatwg.org/multipage/parsing.html#:~:text=An%20start%20tag%20whose%20tag%20name%20is%20one%20of%3A%20%22address%22%2C)
{{% /aside %}}

`HTMLTreeBuilder::ProcessStartTagForInBody` 的源码又是几百行，会判断当前在 `<body>` 内各种的 tag 可能的情况，我们继续择取一个分支看看，假设要处理的是 `<body>` 下的 `<div>`。

{{% admonition type="sourcecode" title="Source Code" %}} 
HTMLTreeBuilder::ProcessStartTagForInBody

```cpp
void HTMLTreeBuilder::ProcessStartTagForInBody(AtomicHTMLToken* token) {
  // ...
  if (token->GetName() == html_names::kAddressTag ||
    token->GetName() == html_names::kArticleTag ||
    token->GetName() == html_names::kAsideTag ||
    token->GetName() == html_names::kBlockquoteTag ||
    token->GetName() == html_names::kButtonTag ||
    token->GetName() == html_names::kCenterTag ||
    token->GetName() == html_names::kDetailsTag ||
    token->GetName() == html_names::kDialogTag ||
    token->GetName() == html_names::kDirTag ||
    token->GetName() == html_names::kDivTag ||
    token->GetName() == html_names::kDlTag ||
    token->GetName() == html_names::kFieldsetTag ||
    token->GetName() == html_names::kFigcaptionTag ||
    token->GetName() == html_names::kFigureTag ||
    token->GetName() == html_names::kFooterTag ||
    token->GetName() == html_names::kHeaderTag ||
    token->GetName() == html_names::kHgroupTag ||
    token->GetName() == html_names::kListingTag ||
    token->GetName() == html_names::kMainTag ||
    token->GetName() == html_names::kMenuTag ||
    token->GetName() == html_names::kNavTag ||
    token->GetName() == html_names::kOlTag ||
    token->GetName() == html_names::kPreTag ||
    token->GetName() == html_names::kSectionTag ||
    token->GetName() == html_names::kSummaryTag ||
    token->GetName() == html_names::kUlTag) {
    ProcessFakePEndTagIfPInButtonScope();
    if (tree_.CurrentStackItem()->IsNumberedHeaderElement()) {
      ParseError(token);
      tree_.OpenElements()->Pop();
    }
    tree_.InsertHTMLElement(token);
    return;
  }
  // ...
}
```
{{% /admonition %}}

{{% aside %}}
HTMLElementStack 规范可见: [http://www.whatwg.org/specs/web-apps/current-work/multipage/parsing.html#the-stack-of-open-elements](http://www.whatwg.org/specs/web-apps/current-work/multipage/parsing.html#the-stack-of-open-elements) <br/> <br/> 根据 token 创建 element 的规范可见: [https://html.spec.whatwg.org/C/#create-an-element-for-the-token](https://html.spec.whatwg.org/C/#create-an-element-for-the-token)
{{% /aside %}}

核心是传入 `<div>` token 调用 `HTMLConstructionSite::InsertHTMLElement`，在该函数内调用 `CreateElement` 根据 token 创建一个 HTMLElement，之后将其压入 `HTMLElementStack` 栈中：


{{% admonition type="sourcecode" title="Source Code" %}} 
HTMLConstructionSite::InsertHTMLElement

```cpp
void HTMLConstructionSite::InsertHTMLElement(AtomicHTMLToken* token) {
  Element* element = CreateElement(token, html_names::xhtmlNamespaceURI);
  AttachLater(CurrentNode(), element);
  open_elements_.Push(HTMLStackItem::Create(element, token));
}

void HTMLConstructionSite::CreateElement(AtomicHTMLToken* token) {
  Element* element = CreateElement(token, html_names::xhtmlNamespaceURI);
  AttachLater(CurrentNode(), element);
  open_elements_.Push(HTMLStackItem::Create(element, token));
}
```

`HTMLConstructionSite::CreateElement`
```cpp
// "create an element for a token"
Element* HTMLConstructionSite::CreateElement(
    AtomicHTMLToken* token,
    const AtomicString& namespace_uri) {
  // "1. Let document be intended parent's node document."
  Document& document = OwnerDocumentForCurrentNode();

  // "2. Let local name be the tag name of the token."
  QualifiedName tag_name =
      ((token->IsValidHTMLTag() &&
        namespace_uri == html_names::xhtmlNamespaceURI)
           ? static_cast<const QualifiedName&>(
                 html_names::TagToQualifedName(token->GetHTMLTag()))
           : QualifiedName(g_null_atom, token->GetName(), namespace_uri));
  // "3. Let is be the value of the "is" attribute in the given token ..." etc.
  const Attribute* is_attribute = token->GetAttributeItem(html_names::kIsAttr);
  const AtomicString& is = is_attribute ? is_attribute->Value() : g_null_atom;
  // "4. Let definition be the result of looking up a custom element ..." etc.
  auto* definition = LookUpCustomElementDefinition(document, tag_name, is);
  // "5. If definition is non-null and the parser was not originally created
  // for the HTML fragment parsing algorithm, then let will execute script
  // be true."
  bool will_execute_script = definition && !is_parsing_fragment_;

  Element* element;

  if (will_execute_script) {
    // "6.1 Increment the document's throw-on-dynamic-insertion counter."
    ThrowOnDynamicMarkupInsertionCountIncrementer
        throw_on_dynamic_markup_insertions(&document);

    // "6.2 If the JavaScript execution context stack is empty,
    // then perform a microtask checkpoint."

    // TODO(dominicc): This is the way the Blink HTML parser performs
    // checkpoints, but note the spec is different--it talks about the
    // JavaScript stack, not the script nesting level.
    if (0u == reentry_permit_->ScriptNestingLevel())
      Microtask::PerformCheckpoint(V8PerIsolateData::MainThreadIsolate());

    // "6.3 Push a new element queue onto the custom element
    // reactions stack."
    CEReactionsScope reactions;

    // "7. Let element be the result of creating an element given document,
    // localName, given namespace, null, and is. If will execute script is true,
    // set the synchronous custom elements flag; otherwise, leave it unset."
    // TODO(crbug.com/1080673): We clear the CreatedbyParser flag here, so that
    // elements get fully constructed. Some elements (e.g. HTMLInputElement)
    // only partially construct themselves when created by the parser, but since
    // this is a custom element, we need a fully-constructed element here.
    element = definition->CreateElement(
        document, tag_name,
        GetCreateElementFlags().SetCreatedByParser(false, nullptr));

    // "8. Append each attribute in the given token to element." We don't use
    // setAttributes here because the custom element constructor may have
    // manipulated attributes.
    for (const auto& attribute : token->Attributes())
      element->setAttribute(attribute.GetName(), attribute.Value());

    // "9. If will execute script is true, then ..." etc. The CEReactionsScope
    // and ThrowOnDynamicMarkupInsertionCountIncrementer destructors implement
    // steps 9.1-3.
  } else {
    if (definition) {
      DCHECK(GetCreateElementFlags().IsAsyncCustomElements());
      element = definition->CreateElement(document, tag_name,
                                          GetCreateElementFlags());
    } else {
      element = CustomElement::CreateUncustomizedOrUndefinedElement(
          document, tag_name, GetCreateElementFlags(), is);
    }
    // Definition for the created element does not exist here and it cannot be
    // custom, precustomized, or failed.
    DCHECK_NE(element->GetCustomElementState(), CustomElementState::kCustom);
    DCHECK_NE(element->GetCustomElementState(),
              CustomElementState::kPreCustomized);
    DCHECK_NE(element->GetCustomElementState(), CustomElementState::kFailed);

    // TODO(dominicc): Move these steps so they happen for custom
    // elements as well as built-in elements when customized built in
    // elements are implemented for resettable, listed elements.

    // 10. If element has an xmlns attribute in the XMLNS namespace
    // whose value is not exactly the same as the element's namespace,
    // that is a parse error. Similarly, if element has an xmlns:xlink
    // attribute in the XMLNS namespace whose value is not the XLink
    // Namespace, that is a parse error.

    // TODO(dominicc): Implement step 10 when the HTML parser does
    // something useful with parse errors.

    // 11. If element is a resettable element, invoke its reset
    // algorithm. (This initializes the element's value and
    // checkedness based on the element's attributes.)
    // TODO(dominicc): Implement step 11, resettable elements.

    // 12. If element is a form-associated element, and the form
    // element pointer is not null, and there is no template element
    // on the stack of open elements, ...
    auto* html_element = DynamicTo<HTMLElement>(element);
    FormAssociated* form_associated_element =
        html_element ? html_element->ToFormAssociatedOrNull() : nullptr;
    if (form_associated_element && document.GetFrame() && form_.Get()) {
      // ... and element is either not listed or doesn't have a form
      // attribute, and the intended parent is in the same tree as the
      // element pointed to by the form element pointer , associate
      // element with the form element pointed to by the form element
      // pointer , and suppress the running of the reset the form owner
      // algorithm when the parser subsequently attempts to insert the
      // element.

      // TODO(dominicc): There are many differences to the spec here;
      // some of them are observable:
      //
      // - The HTML spec tracks whether there is a template element on
      // the stack both for manipulating the form element pointer 
      //   and using it here.
      // - FormAssociated::AssociateWith implementations don't do the
      //   "same tree" check; for example
      //   HTMLImageElement::AssociateWith just checks whether the form
      //   is in *a* tree. This check should be done here consistently.
      // - ListedElement is a mixin; add IsListedElement and skip
      //   setting the form for listed attributes with form=. Instead
      //   we set attributes (step 8) out of order, after this step,
      //   to reset the form association.
      form_associated_element->AssociateWith(form_.Get());
    }
    // "8. Append each attribute in the given token to element."
    SetAttributes(element, token, parser_content_policy_);
  }

  return element;
}
```

{{% /admonition %}}

{{% admonition type="warning" title="Warning" %}}

待完善对 createElement 的规范讲解。

{{% /admonition %}}

创建 Element 的流程比较复杂，因为[规范](https://html.spec.whatwg.org/C/#create-an-element-for-the-token)相当繁琐，我们这里直接看堆栈:

```
#0	0x00000002d09b0ea4 in blink::HTMLDivElement::HTMLDivElement(blink::Document&) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/html_div_element.cc:32
#1	0x00000002d0129ee4 in blink::HTMLDivElement* cppgc::MakeGarbageCollectedTrait<blink::HTMLDivElement>::Call<blink::Document&>(cppgc::AllocationHandle&, blink::Document&) at /Users/airing/Files/code/chromium/src/v8/include/cppgc/allocation.h:242
#2	0x00000002d0125100 in blink::HTMLDivElement* cppgc::MakeGarbageCollected<blink::HTMLDivElement, blink::Document&>(cppgc::AllocationHandle&, blink::Document&) [inlined] at /Users/airing/Files/code/chromium/src/v8/include/cppgc/allocation.h:280
#3	0x00000002d01250f0 in blink::HTMLDivElement* blink::MakeGarbageCollected<blink::HTMLDivElement, blink::Document&>(blink::Document&) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/platform/heap/garbage_collected.h:34
#4	0x00000002d21a16f8 in blink::HTMLDivConstructor(blink::Document&, blink::CreateElementFlags) at /Users/airing/Files/code/chromium/src/out/gn/gen/third_party/blink/renderer/core/html_element_factory.cc:259
#5	0x00000002d219fdbc in blink::HTMLElementFactory::Create(WTF::AtomicString const&, blink::Document&, blink::CreateElementFlags) at /Users/airing/Files/code/chromium/src/out/gn/gen/third_party/blink/renderer/core/html_element_factory.cc:851
#6	0x00000002d22021c8 in blink::Document::CreateRawElement(blink::QualifiedName const&, blink::CreateElementFlags) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/dom/document.cc:1031
#7	0x00000002d2337cf0 in blink::Element* blink::CustomElement::CreateUncustomizedOrUndefinedElementTemplate<(blink::CustomElement::CreateUUCheckLevel)0>(blink::Document&, blink::QualifiedName const&, blink::CreateElementFlags, WTF::AtomicString const&) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/custom/custom_element.cc:158
#8	0x00000002d2337c68 in blink::CustomElement::CreateUncustomizedOrUndefinedElement(blink::Document&, blink::QualifiedName const&, blink::CreateElementFlags, WTF::AtomicString const&) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/custom/custom_element.cc:180
#9	0x00000002d2367774 in blink::HTMLConstructionSite::CreateElement(blink::AtomicHTMLToken*, WTF::AtomicString const&) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_construction_site.cc:988
#10	0x00000002d2368230 in blink::HTMLConstructionSite::InsertHTMLElement(blink::AtomicHTMLToken*) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_construction_site.cc:729
#11	0x00000002d23e28a8 in blink::HTMLTreeBuilder::ProcessStartTagForInBody(blink::AtomicHTMLToken*) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_tree_builder.cc:648
#12	0x00000002d23dced0 in blink::HTMLTreeBuilder::ProcessStartTag(blink::AtomicHTMLToken*) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_tree_builder.cc:1209
#13	0x00000002d23dc10c in blink::HTMLTreeBuilder::ProcessToken(blink::AtomicHTMLToken*) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_tree_builder.cc:372
#14	0x00000002d23dadc8 in blink::HTMLTreeBuilder::ConstructTree(blink::AtomicHTMLToken*) at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_tree_builder.cc:329
#15	0x00000002d237eff0 in blink::HTMLDocumentParser::ConstructTreeFromHTMLToken() at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_document_parser.cc:954
#16	0x00000002d237d3a0 in blink::HTMLDocumentParser::PumpTokenizer() at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_document_parser.cc:837
#17	0x00000002d237b734 in blink::HTMLDocumentParser::PumpTokenizerIfPossible() at /Users/airing/Files/code/chromium/src/third_party/blink/renderer/core/html/parser/html_document_parser.cc:703

```

可以发现这里层层调用，最后到 `HTMLElementFactory::Create` 的工厂方法里去找 `<div>` 映射对应的 `HTMLDivConstructor` 去构造 `<div>`。构造器处理了 GC 逻辑，并处理了 `<div>` 部分行内样式的绑定。

`HTMLElementStack` 对象的栈结构如下所示:

{{<figure src="/images/chromium-rendering-pipeline/open_elements.png" attr="open_elements_">}}

{{% aside %}}
为保证健壮性，这里也会异常情况做自闭合处理。
{{% /aside %}}

最后看看 `</div>` 的出栈，根据下述源码可见，这个时候会调用 `HTMLConstructionSite::OpenElements` 获取缓存的 `HTMLElementStack` 对象，并将 `<div>` 组合出栈。

{{% aside %}}
该源码对应的规范可见: [https://html.spec.whatwg.org/multipage/parsing.html](https://html.spec.whatwg.org/multipage/parsing.html#:~:text=An%20end%20tag%20whose%20tag%20name%20is%20one%20of%3A%20%22address%22%2C)
{{% /aside %}}

{{% admonition type="sourcecode" title="Source Code" %}} 
HTMLTreeBuilder::ProcessEndTagForInBody

```cpp
void HTMLTreeBuilder::ProcessEndTagForInBody(AtomicHTMLToken* token) {
  // ...
  if (token->GetName() == html_names::kAddressTag ||
      token->GetName() == html_names::kArticleTag ||
      token->GetName() == html_names::kAsideTag ||
      token->GetName() == html_names::kBlockquoteTag ||
      token->GetName() == html_names::kButtonTag ||
      token->GetName() == html_names::kCenterTag ||
      token->GetName() == html_names::kDetailsTag ||
      token->GetName() == html_names::kDialogTag ||
      token->GetName() == html_names::kDirTag ||
      token->GetName() == html_names::kDivTag ||
      token->GetName() == html_names::kDlTag ||
      token->GetName() == html_names::kFieldsetTag ||
      token->GetName() == html_names::kFigcaptionTag ||
      token->GetName() == html_names::kFigureTag ||
      token->GetName() == html_names::kFooterTag ||
      token->GetName() == html_names::kHeaderTag ||
      token->GetName() == html_names::kHgroupTag ||
      token->GetName() == html_names::kListingTag ||
      token->GetName() == html_names::kMainTag ||
      token->GetName() == html_names::kMenuTag ||
      token->GetName() == html_names::kNavTag ||
      token->GetName() == html_names::kOlTag ||
      token->GetName() == html_names::kPreTag ||
      token->GetName() == html_names::kSectionTag ||
      token->GetName() == html_names::kSummaryTag ||
      token->GetName() == html_names::kUlTag) {
    if (!tree_.OpenElements()->InScope(token->GetName())) {
      ParseError(token);
      return;
    }
    tree_.GenerateImpliedEndTags();
    if (!tree_.CurrentStackItem()->MatchesHTMLTag(token->GetName()))
      ParseError(token);
    tree_.OpenElements()->PopUntilPopped(token->GetName());
    return;
  }
  // ...
}
```
{{% /admonition %}}

至此 Lexing 环节结束，后续流程会基于该环节存储 Element 使用到的数据结构来进行 DOM Tree 的构建。

{{% aside %}}
DOM Construction: Element → DOM Tree
{{% /aside%}}


## DOM Construction

DOM Tree 构建流程如下图所示：

{{<figure src="/images/chromium-rendering-pipeline/parsing.png" attr="DOM Construction">}}

Element 在创建时会 attach 到 `HTMLConstructionSite` 的成员变量 `_document` (`blink::Document`) 上，得益于 Element 数据结构的设计，当 HTML 文档所有 bytes 通过以上流程解析结束时，会得到一个完整 `_document` 对象，其中的 Tree 结构就是 DOM Tree。

{{<figure src="/images/chromium-rendering-pipeline/dom-tree.png" attr="blink::Document">}}

{{% aside %}}
Element 数据结构在前文介绍 `HTMLElementStack` 时有断点截图，可自行翻阅前文查看。
{{% /aside %}}

{{% admonition type="sourcecode" title="Node Tree" %}} 
Node Tree 结构如下图所示

![](/images/chromium-rendering-pipeline/next-sibling.svg)

Node 是所有 node tree 最基础的数据结构（DOM Tree, CSSOM Tree, etc.，每个 Node 至少有以下三个指针：

1. parent_or_shadow_host_node_: 指向父节点 (但如果是 shadow root 会指向 shadow host)
2. previous_: 指向上一个兄弟节点
3. next_: 指向下一个兄弟节点


每个 Element 都集成自 ContainerNode, 而 ContainerNode 还有额外的两个指针：

- first_child_
- last_child_

兄弟节点是相互关联的，但这个结构意味着从父节点去寻找任意一个子节点需要 O(N) 复杂度。

{{% /admonition %}}


至此，渲染管线的第一阶段 `Parsing` 已经完成，完善本节的具体流程，我们得到更完整的渲染管线结构图：

{{<figure src="/images/chromium-rendering-pipeline/pipeline-1.png" attr="Rendering Pipeline">}}


## CSSOM Construction

{{% admonition type="warning" title="Warning" %}}
This section will be written next.
{{% /admonition %}}

## [Appendix] Mojo

{{% admonition type="warning" title="Warning" %}}
这里插播简单介绍一下 Mojo，后续丰富内容之后会独立成篇具体介绍 Mojo。
{{% /admonition %}}

{{% aside %}}
Mojo 也被用于 ChromeOS。
{{% /aside %}}

Mojo 是一个平台无关的跨平台 IPC 框架，在 Chromium 项目中用来实现 chromium 进程内/进程间的通信。

Mojo 的分层如下图所示：

{{<figure src="/images/chromium-rendering-pipeline/mojo-overview.png" attr="Mojo" >}}

1. Mojo Core: Mojo 的实现层，不能独立使用，由 C++ 实现；
2. Mojo System API(C): Mojo 的 C API 层，它和 Mojo Core 对接，可以在程序中独立使用；
3. Mojo System API(C++/Java/JS): Mojo 的各种语言包装层，它将 Mojo C API 包装成多种语言的库，让其他语言可以使用。这一层也可以在程序中独立使用；
4. Mojo Bindings: 这一层引入一种称为 Mojom 的 IDL（接口定义）语言，通过它可以定义通信接口，这些接口会生成接口类，使用户只要实现这些接口就可以使用 Mojo 进行通信，这一层使得IPC两端不需要通过原始字节流进行通信，而是通过接口进行通信，有点类似 Protobuf 和 Thrift 。

{{% admonition type="sourcecode" title="Note" %}}
在 Chromium 中，还有两个基础模块使用 Mojo，分别是 Services 和 IPC::Channel。因此 Chromium 中 Mojo 完整的架构图如下所示：

![](/images/chromium-rendering-pipeline/chromium-mojo-layer.png)

{{% /admonition %}}

{{% aside %}}
这点类似 TCP/IP 中的 IP 地址和端口的关系。
{{% /aside %}}

Mojo 支持在多个进程之间互相通信，由 Mojo 组成的这些可以互相通信的进程形成了一个网络，在这个网络内的任意两个进程都可以进行通信，并且每个进程只能处于一个 Mojo 网络中，在这个网络内每一个进程内部有且只有一个 Node，每一个 Node 可以提供多个 Port，每个 Port 对应一种服务，一个 Node:Port 对可以唯一确定一个服务。

`Node` 和 `Node` 之间通过 `Channel` 来实现通信，在不同平台上 `Channel` 有不同的实现方式  
- 在 Linux上 是 domain socket  
- 在 windows 上是 name pipe  
- 在 Mac OS 平台上是 Mach Port  

{{<figure src="/images/chromium-rendering-pipeline/mojo-data-flow.png" attr="Mojo" >}}

{{% aside %}}
类似在 TCP 上封装了 HTTP，SMTP，FTP。
{{% /aside %}}

在 Port 上一层，Mojo 封装了 3 个“应用层协议”，分别为：  
1. `MessagePipe`: 应用层接口，用于进程间的双向通信，类似 UDP，消息是基于数据报的，底层使用 Channel 通道
2. `DataPipe`: 应用层接口，用于进程间单向块数据传递，类似 TCP，消息是基于数据流的，底层使用系统的 Shared Memory 实现
3. `SharedBuffer`: 应用层接口，支持双向块数据传递，底层使用系统 Shared Memory 实现。

{{% admonition type="sourcecode" title="Source to read" %}}
你可以阅读官方文档 [Mojo docs (go/mojo-docs)](https://chromium.googlesource.com/chromium/src/+/master/mojo/README.md) 以了解更多关于 Mojo 的内容。
{{% /admonition %}}

{{% admonition type="tryit" title="Create a new message pipe" %}}

有两种方式创建 `MessagePipe`，可以直接使用构造函数：

```cpp
mojo::MessagePipe pipe;

// NOTE: Because pipes are bi-directional there is no implicit semantic
// difference between |handle0| or |handle1| here. They're just two ends of a
// pipe. The choice to treat one as a "client" and one as a "server" is entirely
// the API user's decision.
mojo::ScopedMessagePipeHandle client = std::move(pipe.handle0);
mojo::ScopedMessagePipeHandle server = std::move(pipe.handle1);
```

或者调用 `CreateMessagePipe`:

```cpp
mojo::ScopedMessagePipeHandle client;
mojo::ScopedMessagePipeHandle server;
mojo::CreateMessagePipe(nullptr, &client, &server);
```
{{% /admonition %}}

{{% admonition type="tryit" title="Create a new data pipe" %}}

同创建 Message Pipe 类似，也有两种方式：

```cpp
mojo::DataPipe pipe;
mojo::ScopedDataPipeProducerHandle producer = std::move(pipe.producer_handle);
mojo::ScopedDataPipeConsumerHandle consumer = std::move(pipe.consumer_handle);

// Or alternatively:
mojo::ScopedDataPipeProducerHandle producer;
mojo::ScopedDataPipeConsumerHandle consumer;
mojo::CreateDataPipe(nullptr, producer, consumer);
```

{{% /admonition %}}

{{% admonition type="tryit" title="Create a new shared buffer" %}}

创建 Shared buffer 方法如下：

```cpp
mojo::ScopedSharedBufferHandle buffer =
    mojo::SharedBufferHandle::Create(4096);
```

这个 handle 可以被任意 `Clone` 多次:

```cpp
mojo::ScopedSharedBufferHandle another_handle = buffer->Clone();
mojo::ScopedSharedBufferHandle read_only_handle =
    buffer->Clone(mojo::SharedBufferHandle::AccessMode::READ_ONLY);
```
{{% /admonition %}}

## Resources

- [Inside look at modern web browser (part 3)](https://developer.chrome.com/blog/inside-browser-part3/)
- [Life of a pixel - Google Docs](https://docs.google.com/presentation/d/1boPxbgNrTU0ddsc144rcXayGA_WF53k96imRH8Mp34Y/edit#slide=id.ga884fe665f_64_45)
- [Mojo docs (go/mojo-docs)](https://chromium.googlesource.com/chromium/src/+/master/mojo/README.md)

{{% admonition type="warning" title="Warning" %}}
TODO:

1. 补充 custom element 与 shadow tree
2. 补充 flat tree traversal
3. 补充 CSSOM Tree 的构建流程

{{% /admonition %}}

