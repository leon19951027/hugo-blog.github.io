---
title: "Kubernetes List,Watch && informer"
description: "kubernetes 各组件通过此机制与api server 通信"
keywords: "kubernetes listwatch 机制"

date: 2022-11-15T15:45:38+08:00
lastmod: 2022-11-15T15:45:38+08:00

categories:
  - kubernetes
# tags:
#   - 
#   -

# 原文作者
# Post's origin author name
author: zzzjy
# 原文链接
# Post's origin link URL
#link:
# 图片链接，用在open graph和twitter卡片上
# Image source link that will use in open graph and twitter card
#imgs:
# 在首页展开内容
# Expand content on the home page
#expand: true
# 外部链接地址，访问时直接跳转
# It's means that will redirecting to external links
#extlink:
# 在当前页面关闭评论功能
# Disabled comment plugins in this post
#comment:
#  enable: false
# 关闭文章目录功能
# Disable table of content
toc: true
# 绝对访问路径
# Absolute link for visit
#url: "listwatch.html"
# 开启文章置顶，数字越小越靠前
# Sticky post set-top in home page and the smaller nubmer will more forward.
#weight: 1
# 开启数学公式渲染，可选值： mathjax, katex
# Support Math Formulas render, options: mathjax, katex
#math: mathjax
# 开启各种图渲染，如流程图、时序图、类图等
# Enable chart render, such as: flow, sequence, classes etc
#mermaid: true
---
<b>list ,watch and informer机制的研究，探讨</b>
<!--more-->
# listwatch设计理念and实现
从以下时序图中我们可以看到一个deployment是如何拉起pod的 ,各类controller都需要通过apiserver来获取对于pod的crud事件
![](/images/kube/event1.png)
那么这里就有一个问题了,这么多controller是如何维护自己需要的事件的，是apiserver去推送的，还是所有controller去poll的。如果所有组件，controller都去poll apiserver那么，不仅对apiserver本身的并发性能有很高的要求，而且对宿主机的网络要求也不低，所以，apiserver与各组件，controller采用长链接的通讯方式，如果拿kafka来类比的话，组件和controller会去订阅他们想要的topic。在k8s中，管这种机制叫watch。
    <iframe src="//player.bilibili.com/player.html?aid=249446524&bvid=BV1mv411E7Ey&cid=379090226&page=1"></iframe>
## how
  我们可以在源码中 apiserver/pkg/endpoints/handlers/get.go看一下watch的handler，和list接口在一起(ListResource)
  以及在apiserver/pkg/endpoints/handlers/watch.go中看一下watch的具体实现

```go
if opts.Watch || forceWatch {
			if rw == nil {
				scope.err(errors.NewMethodNotSupported(scope.Resource.GroupResource(), "watch"), w, req)
				return
			}
			// TODO: Currently we explicitly ignore ?timeout= and use only ?timeoutSeconds=.
			timeout := time.Duration(0)
			if opts.TimeoutSeconds != nil {
				timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
			}
			if timeout == 0 && minRequestTimeout > 0 {
				timeout = time.Duration(float64(minRequestTimeout) * (rand.Float64() + 1.0))
			}
			klog.V(3).InfoS("Starting watch", "path", req.URL.Path, "resourceVersion", opts.ResourceVersion, "labels", opts.LabelSelector, "fields", opts.FieldSelector, "timeout", timeout)
			ctx, cancel := context.WithTimeout(ctx, timeout)
			defer cancel()
			watcher, err := rw.Watch(ctx, &opts)
			if err != nil {
				scope.err(err, w, req)
				return
			}
			requestInfo, _ := request.RequestInfoFrom(ctx)
			metrics.RecordLongRunning(req, requestInfo, metrics.APIServerComponent, func() {
				serveWatch(watcher, scope, outputMediaType, req, w, timeout)
			})
			return
		}
```
创建watcher
```go
watcher, err := rw.Watch(ctx, &opts)
```
rw是一个接口，注释说所有存储于etcd的资源都实现了这个接口
```go
// Watcher should be implemented by all Storage objects that
// want to offer the ability to watch for changes through the watch api.
type Watcher interface {
	// 'label' selects on labels; 'field' selects on the object's fields. Not all fields
	// are supported; an error should be returned if 'field' tries to select on a field that
	// isn't supported. 'resourceVersion' allows for continuing/starting a watch at a
	// particular version.
	Watch(ctx context.Context, options *metainternalversion.ListOptions) (watch.Interface, error)
}
```
所有资源的结构体都拥有Store结构体，Store实现了Watch，进而所有资源继承了Store的Watch接口
```go
type REST struct {
	*genericregistry.Store
}
```
```go
// Watch makes a matcher for the given label and field, and calls
// WatchPredicate. If possible, you should customize PredicateFunc to produce
// a matcher that matches by key. SelectionPredicate does this for you
// automatically.
func (e *Store) Watch(ctx context.Context, options *metainternalversion.ListOptions) (watch.Interface, error) {
	label := labels.Everything()
	if options != nil && options.LabelSelector != nil {
		label = options.LabelSelector
	}
	field := fields.Everything()
	if options != nil && options.FieldSelector != nil {
		field = options.FieldSelector
	}
	predicate := e.PredicateFunc(label, field)

	resourceVersion := ""
	if options != nil {
		resourceVersion = options.ResourceVersion
		predicate.AllowWatchBookmarks = options.AllowWatchBookmarks
	}
	return e.WatchPredicate(ctx, predicate, resourceVersion)
}
```
watcher是个接口类型，实现两个函数
实现者不少，eg:BroadCasterWatcher
```go
// Interface can be implemented by anything that knows how to watch and report changes.
type Interface interface {
	// Stop stops watching. Will close the channel returned by ResultChan(). Releases
	// any resources used by the watch.
	Stop()

	// ResultChan returns a chan which will receive all the events. If an error occurs
	// or Stop() is called, the implementation will close this channel and
	// release any resources used by the watch.
	ResultChan() <-chan Event
}
```
serverWatch
```go
	metrics.RecordLongRunning(req, requestInfo, metrics.APIServerComponent, func() {
				serveWatch(watcher, scope, outputMediaType, req, w, timeout)
	})
```
serveWatch函数最后构造WatchServer结构体，WatchServer处理http或者websocket
```go
func (s *WatchServer) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	kind := s.Scope.Kind

	if wsstream.IsWebSocketRequest(req) {
		w.Header().Set("Content-Type", s.MediaType)
		websocket.Handler(s.HandleWS).ServeHTTP(w, req)
		return
	}

	flusher, ok := w.(http.Flusher)
	if !ok {
		err := fmt.Errorf("unable to start watch - can't get http.Flusher: %#v", w)
		utilruntime.HandleError(err)
		s.Scope.err(errors.NewInternalError(err), w, req)
		return
	}

	framer := s.Framer.NewFrameWriter(w)
	if framer == nil {
		// programmer error
		err := fmt.Errorf("no stream framing support is available for media type %q", s.MediaType)
		utilruntime.HandleError(err)
		s.Scope.err(errors.NewBadRequest(err.Error()), w, req)
		return
	}

	var e streaming.Encoder
	var memoryAllocator runtime.MemoryAllocator

	if encoder, supportsAllocator := s.Encoder.(runtime.EncoderWithAllocator); supportsAllocator {
		memoryAllocator = runtime.AllocatorPool.Get().(*runtime.Allocator)
		defer runtime.AllocatorPool.Put(memoryAllocator)
		e = streaming.NewEncoderWithAllocator(framer, encoder, memoryAllocator)
	} else {
		e = streaming.NewEncoder(framer, s.Encoder)
	}

	// ensure the connection times out
	timeoutCh, cleanup := s.TimeoutFactory.TimeoutCh()
	defer cleanup()

	// begin the stream
	w.Header().Set("Content-Type", s.MediaType)
	w.Header().Set("Transfer-Encoding", "chunked")
	w.WriteHeader(http.StatusOK)
	flusher.Flush()

	var unknown runtime.Unknown
	internalEvent := &metav1.InternalEvent{}
	outEvent := &metav1.WatchEvent{}
	buf := &bytes.Buffer{}
	ch := s.Watching.ResultChan()
	done := req.Context().Done()

	embeddedEncodeFn := s.EmbeddedEncoder.Encode
	if encoder, supportsAllocator := s.EmbeddedEncoder.(runtime.EncoderWithAllocator); supportsAllocator {
		if memoryAllocator == nil {
			// don't put the allocator inside the embeddedEncodeFn as that would allocate memory on every call.
			// instead, we allocate the buffer for the entire watch session and release it when we close the connection.
			memoryAllocator = runtime.AllocatorPool.Get().(*runtime.Allocator)
			defer runtime.AllocatorPool.Put(memoryAllocator)
		}
		embeddedEncodeFn = func(obj runtime.Object, w io.Writer) error {
			return encoder.EncodeWithAllocator(obj, w, memoryAllocator)
		}
	}

	for {
		select {
		case <-done:
			return
		case <-timeoutCh:
			return
		case event, ok := <-ch:
			if !ok {
				// End of results.
				return
			}
			metrics.WatchEvents.WithContext(req.Context()).WithLabelValues(kind.Group, kind.Version, kind.Kind).Inc()

			obj := s.Fixup(event.Object)
			if err := embeddedEncodeFn(obj, buf); err != nil {
				// unexpected error
				utilruntime.HandleError(fmt.Errorf("unable to encode watch object %T: %v", obj, err))
				return
			}

			// ContentType is not required here because we are defaulting to the serializer
			// type
			unknown.Raw = buf.Bytes()
			event.Object = &unknown
			metrics.WatchEventsSizes.WithContext(req.Context()).WithLabelValues(kind.Group, kind.Version, kind.Kind).Observe(float64(len(unknown.Raw)))

			*outEvent = metav1.WatchEvent{}

			// create the external type directly and encode it.  Clients will only recognize the serialization we provide.
			// The internal event is being reused, not reallocated so its just a few extra assignments to do it this way
			// and we get the benefit of using conversion functions which already have to stay in sync
			*internalEvent = metav1.InternalEvent(event)
			err := metav1.Convert_v1_InternalEvent_To_v1_WatchEvent(internalEvent, outEvent, nil)
			if err != nil {
				utilruntime.HandleError(fmt.Errorf("unable to convert watch object: %v", err))
				// client disconnect.
				return
			}
			if err := e.Encode(outEvent); err != nil {
				utilruntime.HandleError(fmt.Errorf("unable to encode watch object %T: %v (%#v)", outEvent, err, e))
				// client disconnect.
				return
			}
			if len(ch) == 0 {
				flusher.Flush()
			}

			buf.Reset()
		}
	}
}
```

