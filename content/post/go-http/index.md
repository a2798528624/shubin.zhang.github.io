---
title: Go 是如何实现http 的超时控制的？
date: 2023-01-01
image: yang.jpg
slug: go-http
categories: 
    - Go
    - Http
---

### 摘要
前端时间在使用项目中的超时控制的时候看到直接根据 context.WithTimeout 来实现。感觉到十分困惑，这个方法难道还能够实现自己结束自己的"生命"吗？我感觉到十分困惑所以有了今天的这篇文章。

### 奇怪的代码
```go
ctx, _ := context.WithTimeout(context.Background(), 1*time.Second)
req, err := http.NewRequestWithContext(ctx, "GET", "http://localhost:8081/hello", nil)
res, err := client.Do(req)
```
上面就是大概抽象出来的当时看到的代码？ 如果按照我们正常的思维去实现一个超时控制，肯定会至少存在两个线程，一个线程用来发送请求，一个线程用来控制时间，如下图所示：

```go
type fn func(ctx context.Context) result

type result struct {
	Val interface{}
	Err error
}

func doWithTimeout(ctx context.Context, fn fn) result {
	ch := make(chan result)
	go func(ctx context.Context, ch chan<- result) {
		ch <- fn(ctx)
	}(ctx, ch)

	select {
	case <-ctx.Done(): // timeout
		go func() { <-ch }() // wait ch return...
		return result{Err: ctx.Err()}
	case res := <-ch: // normal case
		return res
	}
}
```
我感觉到十分的困惑，我感到只有两种可能造成这种情况。一种就是ctx.WithTimeout()中调用了操作系统的某个接口，让这个函数执行到一定时间自动的结束。另一种就是在http的内部存在自己实现的多个线程的控制，被封装了一层。我感觉第一种这种实现是十分危险的，它这种强行的结束，可能会造成程序结束在一个非常奇怪的地方，所以我觉得应该是第2种go语言的http包封装了一层实现的。带着这种猜测，我于是开始阅读http包的源码。

### Go的http包
```go
func setRequestCancel(req *Request, rt RoundTripper, deadline time.Time) (stopTimer func(), didTimeout func() bool) {
	if deadline.IsZero() {
		return nop, alwaysFalse
	}
	knownTransport := knownRoundTripperImpl(rt, req)
	oldCtx := req.Context()

	if req.Cancel == nil && knownTransport {
		// If they already had a Request.Context that's
		// expiring sooner, do nothing:
		if !timeBeforeContextDeadline(deadline, oldCtx) {
			return nop, alwaysFalse
		}

		var cancelCtx func()
		req.ctx, cancelCtx = context.WithDeadline(oldCtx, deadline)
		return cancelCtx, func() bool { return time.Now().After(deadline) }
	}
```

可以在这里面看到，如果你在clint中配置了timeout这个参数，在这里面也会把其中转换为context.WithDeadline()。它会在 clint中配置的参数 与 ctx.WiithTimeout中选择一个较小的时间作为，我们所设置的deadline时间。
### http.persistConn.roundTrip
连接实际上也实现了RoundTripper接口。我们先看一看http/1.1的[实现](https://github.com/golang/go/blob/9baddd3f21230c55f0ad2a10f5f20579dcf0a0bb/src/net/http/transport.go#L2524)，限于篇幅，删掉了大量代码，保留我们感兴趣的：
```go
func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
	// deleted...
	if !pc.t.replaceReqCanceler(req.cancelKey, pc.cancelRequest) {
		pc.t.putOrCloseIdleConn(pc)
		return nil, errRequestCanceled
	}
	// deleted...
	defer func() {
		if err != nil {
			pc.t.setReqCanceler(req.cancelKey, nil)
		}
	}()
	// deleted...
	writeErrCh := make(chan error, 1)
	pc.writech <- writeRequest{req, writeErrCh, continueCh}
	// deleted...
	cancelChan := req.Request.Cancel
	ctxDoneChan := req.Context().Done()
	for {
		select {
		case err := <-writeErrCh:
			// deleted...
		case <-pcClosed:
			// deleted...
		case <-respHeaderTimer:
			// deleted...
		case re := <-resc:
			// deleted...
			return re.res, nil
		case <-cancelChan:
			canceled = pc.t.cancelRequest(req.cancelKey, errRequestCanceled)
			cancelChan = nil
		case <-ctxDoneChan:
			canceled = pc.t.cancelRequest(req.cancelKey, req.Context().Err())
			cancelChan = nil
			ctxDoneChan = nil
		}
	}
}
```
首先会判定连接是否是有效的，如果有效则会设置canceler，这是一个[函数](https://github.com/golang/go/blob/9baddd3f21230c55f0ad2a10f5f20579dcf0a0bb/src/net/http/transport.go#L1966)，在检测到超时或者取消的时候将会被调用：
```go
func (t *Transport) cancelRequest(key cancelKey, err error) bool {
	t.reqMu.Lock()
	cancel := t.reqCanceler[key]
	delete(t.reqCanceler, key)
	t.reqMu.Unlock()
	if cancel != nil {
		cancel(err)
	}

	return cancel != nil
}

func (pc *persistConn) cancelRequest(err error) {
	pc.mu.Lock()
	defer pc.mu.Unlock()
	pc.canceledErr = err
	pc.closeLocked(errRequestCanceled)
}
```
cancel的逻辑，相当粗暴，直接将连接断掉，不过http/1.1也确实没有别的办法。那么什么时候这函数会被调用到呢？接下来会通过writech发送写request的请求到连接的一个wirteloop中，然后就是select等待任意一个条件满足：

- 写请求失败
- 连接断开
- 读取响应超时
- 响应返回
- request.Cancel收到了信号
- context到期

其中最后两者会就会调用到canceler将tcp连接断开。
以上就是http/1.1请求是如何超时/取消的具体实现。最后我们来看一看http/2的，代码框架其实也类似。h2连接的细节这里略过了，只看最后取消的[地方](https://github.com/golang/go/blob/9baddd3f21230c55f0ad2a10f5f20579dcf0a0bb/src/net/http/h2_bundle.go#L7535)：
```go
func (cc *http2ClientConn) roundTrip(req *Request) (res *Response, gotErrAfterReqBodyWrite bool, err error) {
	// deleted...
	for {
		select {
		case re := <-readLoopResCh:
			return handleReadLoopResponse(re)
		case <-respHeaderTimer:
			// deleted...
			return nil, cs.getStartedWrite(), http2errTimeout
		case <-ctx.Done():
			if !hasBody || bodyWritten {
				cc.writeStreamReset(cs.ID, http2ErrCodeCancel, nil)
			} else {
				bodyWriter.cancel()
				cs.abortRequestBodyWrite(http2errStopReqBodyWriteAndCancel)
				<-bodyWriter.resc
			}
			cc.forgetStreamID(cs.ID)
			return nil, cs.getStartedWrite(), ctx.Err()
		case <-req.Cancel:
			// deleted...
		case <-cs.peerReset:
			// processResetStream already removed the
			// stream from the streams map; no need for
			// forgetStreamID.
			return nil, cs.getStartedWrite(), cs.resetErr
		case err := <-bodyWriter.resc:
			// deleted...
		}
	}
}
```
这里模型还是http/1.1一样的，select等待超时条件，request.Cancel和context实现基本一致，所以我们只看context到期的，区别于http/1.1，http/2提供了RST_STREM桢来将一个stream的request给cancel掉，所以这也是它的一个优化吧，毕竟连接建立比较耗时。

![Photo by yang on Unsplash](yang.jpg)


参考 ：
1.  [https://www.xiaolongtongxue.com/articles/2021/how-does-go-handle-http-request-timeout-and-cancel](https://www.xiaolongtongxue.com/articles/2021/how-does-go-handle-http-request-timeout-and-cancel)

2[https://www.xiaolongtongxue.com/articles/2021/how-does-go-handle-http-request-timeout-and-cancel](https://www.xiaolongtongxue.com/articles/2021/how-does-go-handle-http-request-timeout-and-cancel)

3 [https://leeshengis.com/archives/4399](https://leeshengis.com/archives/4399)
s://leeshengis.com/archives/4399