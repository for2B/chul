---
layout:     post
title:      "filebeat使用tcp作为input方式源码分析"
subtitle:   "es学习记录"
date:       2019-07-22
author:     "CHuiL"
header-img: "img/elk-bg.png"
tags:
    - elk
---


### 配置文件

```
filebeat.inputs:
- type: tcp
  max_message_size: 10MiB
  host: "0.0.0.0:9000"
  fields:
    doc_type: tcp
output.console:
    pretty: true

```
这里使用控制台输出，主要用来测试看看filebeat是如何使用tcp作为输入的。由于官方文档没有关于这部分的详细介绍，所以只能自己尝试看看了。

### filebeat源码
filebeat是使用GO语言开发的。也比较容易看懂


##### beats\filebeat\input\tcp\input.go
```
...
// Run start a TCP input
func (p *Input) Run() {
	p.Lock()
	defer p.Unlock()

	if !p.started {
		p.log.Info("Starting TCP input")
		err := p.server.Start() //开启服务
		if err != nil {
			p.log.Errorw("Error starting the TCP server", "error", err)
		}
		p.started = true
	}
}
...
```

##### beats\filebeat\inputsource\tcp\server.go
```
...
// Start listen to the TCP socket.
func (s *Server) Start() error {
	var err error
	s.Listener, err = s.createServer() //创建监听服务
	if err != nil {
		return err
	}

	s.log.Info("Started listening for TCP connection")

	s.wg.Add(1)
	go func() {
		defer s.wg.Done()
		s.run() //开始监听
	}()
	return nil
}
...

// Run start and run a new TCP listener to receive new data
func (s *Server) run() {
	for {
		conn, err := s.Listener.Accept() //监听
		if err != nil {
			select {
			case <-s.done:
				return
			default:
				s.log.Debugw("Can not accept the connection", "error", err)
				continue
			}
		}

		client := newClient(
			conn,
			s.log,
			s.callback,
			s.splitFunc,
			uint64(s.config.MaxMessageSize),
			s.config.Timeout,
		)

		s.wg.Add(1)
		go func() {
			defer logp.Recover("recovering from a tcp client crash")
			defer s.wg.Done()
			defer conn.Close()

			s.registerClient(client)
			defer s.unregisterClient(client)
			s.log.Debugw("New client", "remote_address", conn.RemoteAddr(), "total", s.clientsCount())

			err := client.handle() //对客户连接进行处理
			if err != nil {
				s.log.Debugw("Client error", "error", err)
			}

			defer s.log.Debugw(
				"Client disconnected",
				"remote_address",
				conn.RemoteAddr(),
				"total",
				s.clientsCount(),
			)
		}()
	}
}
```
##### beats\filebeat\inputsource\tcp\client.go
```
func (c *client) handle() error {
	r := NewResetableLimitedReader(NewDeadlineReader(c.conn, c.timeout), c.maxMessageSize)
	buf := bufio.NewReader(r)
	scanner := bufio.NewScanner(buf)
	scanner.Split(c.splitFunc)

	for scanner.Scan() {
		err := scanner.Err()
		if err != nil {
			// we are forcing a close on the socket, lets ignore any error that could happen.
			select {
			case <-c.done:
				break
			default:
			}
			// This is a user defined limit and we should notify the user.
			if IsMaxReadBufferErr(err) {
				c.log.Errorw("client error", "error", err)
			}
			return errors.Wrap(err, "tcp client error")
		}
		r.Reset()
		c.callback(scanner.Bytes(), c.metadata) //这里接受到一次完整数据后会调用回调。
	}

	// We are out of the scanner, either we reached EOF or another fatal error occurred.
	// like we failed to complete the TLS handshake or we are missing the client certificate when
	// mutual auth is on, which is the default.
	if err := scanner.Err(); err != nil {
		return err
	}

	return nil
}
```
##### beats\filebeat\input\tcp\input.go
```
// NewInput creates a new TCP input
func NewInput(
	cfg *common.Config,
	outlet channel.Connector,
	context input.Context,
) (input.Input, error) {

    ...
    //这里是回调函数
	cb := func(data []byte, metadata inputsource.NetworkMetadata) {
		event := createEvent(data, metadata) //使用传输过来的数据创建一个新事件。
		forwarder.Send(event)
	}
	...
}

func createEvent(raw []byte, metadata inputsource.NetworkMetadata) *util.Data {
	data := util.NewData()
	data.Event = beat.Event{
		Timestamp: time.Now(),
		Fields: common.MapStr{
			"message": string(raw),
			"log": common.MapStr{
				"source": common.MapStr{
					"address": metadata.RemoteAddr.String(),
				},
			},
		},
	}
	return data
}
```

所以通过学习Filebeat的源码，发现只要建立起tcp连接，并直接发送数据就可以了。filebeat最后输出的信息除了我们输入的被赋值再字段“message”外，还会有诸如source,address等其他信息。

#### 实践
直接建立tcp连接，然后往其中发送数据，一开始总是发送了之后没有反应。后面仔细翻看源码才发现，他需要按照分隔符进行分割，默认是'\n'符号，所以没有该符号的情况下他会继续接受，直到遇到分隔符。可以使用参数`line_delimiteredit`再配置文件中指定分割符号。
