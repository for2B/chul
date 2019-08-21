---
layout:     post
title:      "Minio-GetObject实时传输实践记录"
subtitle:   "Minio实践记录"
date:       2019-07-26
author:     "CHuiL"
header-img: "/img/linux.jpg"
tags:
    - minio
---

## minio介绍
分布式对象存储工具。用法很简单。

## 实现DownLoad接口来下载Minio上的对象
使用GoSDK，其中的GetObject接口如下
```
object, err := minioClient.GetObject("mybucket", "myobject", minio.GetObjectOptions{})
```
传入桶名和对象名，Object类型实例
起初关于这部分内容，起初参考官方的使用用例，先将它存储到本地。
```
object, err := minioClient.GetObject("mybucket", "myobject", minio.GetObjectOptions{})
if err != nil {
    fmt.Println(err)
    return
}
localFile, err := os.Create("/tmp/local-file.jpg")
if err != nil {
    fmt.Println(err)
    return
}
if _, err = io.Copy(localFile, object); err != nil {
    fmt.Println(err)
    return
}
```
因为服务使用Beego框架，Output有Download这个接口。所以就顺势直接拿来用，就有如下代码
```
object, err := minioClient.GetObject("mybucket", "myobject", minio.GetObjectOptions{})
if err != nil {
    fmt.Println(err)
    return
}
localFile, err := os.Create("/tmp/local-file.jpg")
if err != nil {
    fmt.Println(err)
    return
}
if _, err = io.Copy(localFile, object); err != nil {
    fmt.Println(err)
    return
}

this.Ctx.Output.Download(""/tmp/local-file.jpg"")
```
访问接口，浏览器能够正常访问下载我们的文件。看起来好像没什么问题。但是仔细想一下，每次都将文件先拷贝到本地，然后在从本地文件系统中发送给客户端。是不是有点绕？不仅浪费内存还占用磁盘空间。重点是这里对于首次下载的对象是需要先下载到服务器本地，然后客户端才会有响应。小对象小文件可能感受不出来，对于大文件，那就相当于客户要先等你下载好了，它才能下载。这样也太不科学了吧。

所以以上代码不可行。转换思路，不存到本地，获取到object后直接发送给前端客户。所以这里就直接将object的数据copy给response。
```
object, err := minioClient.GetObject("mybucket", "myobject", minio.GetObjectOptions{})
if err != nil {
    fmt.Println(err)
    return
}
//激活浏览器文件下载对话框
oc.Ctx.Output.Header("Content-Disposition","attachment; filename=local-file.jpg")

if _, err = io.Copy(this.Ctx.ResponseWriter, object); err != nil {
    fmt.Println(err)
    return
}
```

然后重新访问接口，直接就能出现下载窗口进行下载。而且也不用临时存储到服务端的磁盘上。   
  
 
但还是总感觉不对，虽然效果上看是实现了，但是object是一个结构体，数据应该是存放在内中的。那么这又是如何实现的？数据难道不是先全部存放到object然后copy给response？而且假如我有一个几gb大小的对象，那样的话内存不会一下子就满了？

我开启beego的内存监控，然后传输一个较大的文件；但是却发现在传输的过程中，内存并没有我预想中的会多出一个文件的大小。而且跟没传输时的内存几乎差不多，好像根本没有一个大文件正在传输的感觉。


所以还是得好好看一下源码，查看一下object的结构体。看到
```
// Object represents an open object. It implements
// Reader, ReaderAt, Seeker, Closer for a HTTP stream.
type Object struct {
	// Mutex.
	mutex *sync.Mutex

	// User allocated and defined.
	reqCh      chan<- getRequest
	resCh      <-chan getResponse
	doneCh     chan<- struct{}
	currOffset int64
	objectInfo ObjectInfo

	// Ask lower level to initiate data fetching based on currOffset
	seekData bool

	// Keeps track of closed call.
	isClosed bool

	// Keeps track of if this is the first call.
	isStarted bool

	// Previous error saved for future calls.
	prevErr error

	// Keeps track of if this object has been read yet.
	beenRead bool

	// Keeps track of if objectInfo has been set yet.
	objectInfoSet bool
}
```

可以看到有可以看到这里有两个管道，reqCh和resCh。看到这两个管道，多多少少应该能猜出来是怎么一回事了。
查看他的部分源码
```
func (c Client) getObjectWithContext(ctx context.Context, bucketName, objectName string, opts GetObjectOptions) (*Object, error) {
	// Input validation.
	if err := s3utils.CheckValidBucketName(bucketName); err != nil {
		return nil, err
	}
	if err := s3utils.CheckValidObjectName(objectName); err != nil {
		return nil, err
	}

	var httpReader io.ReadCloser
	var objectInfo ObjectInfo
	var err error

	// Create request channel.
	reqCh := make(chan getRequest)
	// Create response channel.
	resCh := make(chan getResponse)
	// Create done channel.
	doneCh := make(chan struct{})

	// This routine feeds partial object data as and when the caller reads.
	go func() {//单独起一个Goroutine去等待数据请求
		defer close(reqCh)
		defer close(resCh)

		// Used to verify if etag of object has changed since last read.
		var etag string

		// Loop through the incoming control messages and read data.
		for {
			select {
			// When the done channel is closed exit our routine.
			case <-doneCh:
				// Close the http response body before returning.
				// This ends the connection with the server.
				if httpReader != nil {
					httpReader.Close()
				}
				return

			// Gather incoming request.
			case req := <-reqCh: //接受res请求
				// If this is the first request we may not need to do a getObject request yet.
				if req.isFirstReq {...} else if req.settingObjectInfo {...} else {
					// Offset changes fetch the new object at an Offset.
					// Because the httpReader may not be set by the first
					// request if it was a stat or seek it must be checked
					// if the object has been read or not to only initialize
					// new ones when they haven't been already.
					// All readAt requests are new requests.
					if req.DidOffsetChange || !req.beenRead {...}
					
					// Read at least req.Buffer bytes, if not we have
					// reached our EOF.
					
					size, err := io.ReadFull(httpReader, req.Buffer)
					if err == io.ErrUnexpectedEOF {
						// If an EOF happens after reading some but not
						// all the bytes ReadFull returns ErrUnexpectedEOF
						err = io.EOF
					}
					// Reply back how much was read.
					resCh <- getResponse{ //请求到的数据写入响应
						Size:       int(size),
						Error:      err,
						didRead:    true,
						objectInfo: objectInfo,
					}
				}
			}
		}
	}()

	// Create a newObject through the information sent back by reqCh.
	return newObject(reqCh, resCh, doneCh), nil
}
```

我们最先获得的Object仅仅只是获得到包含几个管道的实例而已。然后后面在执行具体Copy的时候才会利用管道去进行文件数据的传输。
```
func (o *Object) Read(b []byte) (n int, err error) {
    ...
	// Create a new request.
	readReq := getRequest{
		isReadOp: true,
		beenRead: o.beenRead,
		Buffer:   b,
	}

    ...
	// Ask to establish a new data fetch routine based on seekData flag
	readReq.DidOffsetChange = o.seekData
	readReq.Offset = o.currOffset

	// Send and receive from the first request.
	response, err := o.doGetRequest(readReq) //关键在这里。
	if err != nil && err != io.EOF {
		// Save the error for future calls.
		o.prevErr = err
		return response.Size, err
	}

	// Bytes read.
	bytesRead := int64(response.Size)

	// Set the new offset.
	oerr := o.setOffset(bytesRead)
	if oerr != nil {
		// Save the error for future calls.
		o.prevErr = oerr
		return response.Size, oerr
	}

	// Return the response.
	return response.Size, err
}
```
doGetRequest代码
```
func (o *Object) doGetRequest(request getRequest) (getResponse, error) {
	o.reqCh <- request //发送请求
	response := <-o.resCh //接受响应

	// Return any error to the top level.
	if response.Error != nil {
		return response, response.Error
	}

	// This was the first request.
	if !o.isStarted {
		// The object has been operated on.
		o.isStarted = true
	}
	// Set the objectInfo if the request was not readAt
	// and it hasn't been set before.
	if !o.objectInfoSet && !request.isReadAt {
		o.objectInfo = response.objectInfo
		o.objectInfoSet = true
	}
	// Set beenRead only if it has not been set before.
	if !o.beenRead {
		o.beenRead = response.didRead
	}
	// Data are ready on the wire, no need to reinitiate connection in lower level
	o.seekData = false

	return response, nil
}
```

最后在看一看Copy的代码
```
func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	// If the reader has a WriteTo method, use it to do the copy.
	// Avoids an allocation and a copy.
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// Similarly, if the writer has a ReadFrom method, use it to do the copy.
	if rt, ok := dst.(ReaderFrom); ok {
		return rt.ReadFrom(src)
	}
	if buf == nil {
		size := 32 * 1024  //每次32kb
		if l, ok := src.(*LimitedReader); ok && int64(size) > l.N {
			if l.N < 1 {
				size = 1
			} else {
				size = int(l.N)
			}
		}
		buf = make([]byte, size)
	}
	for {
		nr, er := src.Read(buf)  //调用src.read
		if nr > 0 {
			nw, ew := dst.Write(buf[0:nr]) //写入到dst，内部每次都会Flush。
			if nw > 0 {
				written += int64(nw)
			}
			if ew != nil {
				err = ew
				break
			}
			if nr != nw {
				err = ErrShortWrite
				break
			}
		}
		if er != nil {
			if er != EOF {
				err = er
			}
			break
		}
	}
	return written, err
}
```

通过以上源码分析，我们可以总结出，Minio在获取Object的时候只返回包含三个管道的实例，只有在真正需要获取数据的时候才会去创建一个请求并获取数据。而且又借助了Copy这个函数，Copy是流传输的，每次从src中获取32kb的字节数据然后在刷到dst上，即不会再内存中逗留。所以也就不会发送我一开始所设想的再传输大数据的时候会占用很多内存。