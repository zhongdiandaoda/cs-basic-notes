# groupcache

groupcache是一个kv缓存，用于在某些方面替代memcache。

因为groupcache只能get，不能update和delete，也不能设置过期时间，只能通过lru淘汰最近最少访问的数据；有些数据如果长时间不更改，那么可以用groupcache作为缓存；groupcache已经在dl.Google.com、Blogger、Google Code、Google Fiber、Google生产监视系统等项目中投入使用。

场景：静态文件缓存

但是groupcache还是有它的优点的，groupcache既是服务器，也是客户端，当在本地groupcache缓存中没有查找的数据时，通过一致性哈希，查找到该key所对应的peer服务器，在通过http协议，从该peer服务器上获取所需要的数据；还有一点就是当多个客户端同时访问memcache中不存在的键时，会导致多个客户端从mysql获取数据并同时插入memcache中，而在相同情况下，groupcache只会有一个客户端从mysql获取数据，其他客户端阻塞，直到第一个客户端获取到数据之后，再返回给多个客户端；

groupcache是一个缓存库，也就是说不是一个完整的软件，需要自己实现main函数。可以自己写个测试程序，跑跑groupcache，这个测试程序，客户端和groupcache通过rpc进行通信，而groupcache peer之间通过http协议进行通信；这是比较好的做法，因为如果客户端与服务器通信和groupcache之间通信采用的是同一个端口，那么在并发量上去的时候，会严重影响性能。

gl.google.com：

- HTTP服务区
- 服务google浏览器、google地球、安卓SDK的下载等
- 有些文件大，有些小
- 在缓存后，仍然有很大的流量

old code：

- 使用磁盘提供服务
- 单线程循环
- 使用零拷贝`sendfile`优化

# LRU

基于双端链表+map

maxBytes：最大可使用的字节数

nbytes：已使用的字节数

value类型（实现了Len接口的任意类型）：

```
type Value interface {
	Len() int
}
```

# cache

对LRU的封装

添加了mutex + lru的可使用bytes

使用mutex对get和put做并发控制

put和get均要加锁

get加锁原因：需要修改双端队列，将访问过的元素添加到队首

## 延迟初始化

```
func (c *cache) put(key string, value ByteView) {
	c.mu.Lock()
	defer c.mu.Unlock()
	// 延迟初始化
	if c.lru == nil {
		c.lru = lru.New(c.cacheBytes, nil)
	}
	c.lru.Put(key, value)
}
```

```
instant init duration : 938ns

lazy init duration : 891ns
```

# byteview

封装byte数组

选择 byte 类型是为了能够支持任意的数据类型的存储，例如字符串、图片等。

# gcache

用于支持集群的接口：

```
type PeerPicker interface {
	PickPeer(key string) (peer PeerGetter, ok bool)
}

// PeerGetter is the interface that must be implemented by a peer.
type PeerGetter interface {
	Get(group string, key string) ([]byte, error)
}
```

## Group结构体

```
type Group struct {
	name       string
	getter     Getter
	mainCache  cache
	peerPicker PeerPicker
}
```

创建与初始化：

```
var (
	mu     sync.RWMutex
	groups = make(map[string]*Group)
)

func NewGroup(name string, cacheBytes int64, getter Getter) *Group {
	if getter == nil {
		panic("nil Getter")
	}
	mu.Lock()
	defer mu.Unlock()
	g := &Group{
		name:      name,
		getter:    getter,
		mainCache: cache{cacheBytes: cacheBytes},
	}
	groups[name] = g
	return g
}
func (g *Group) RegisterPeers(peerPicker PeerPicker) {
	if g.peerPicker != nil {
		panic("RegisterPeerPicker called more than once")
	}
	g.peerPicker = peerPicker
}
```

peerPicker会返回一个peer的getter，从而可以通过getter远程获取数据。

## get流程

```
                            是
接收 key --> 检查是否被缓存 -----> 返回缓存值 ⑴
                |  否                         是
                |-----> 是否应当从远程节点获取 -----> 与远程节点交互 --> 返回缓存值 ⑵
                            |  否
                            |-----> 调用`回调函数`，获取值并添加到缓存 --> 返回缓存值 ⑶
```

```
func (g *Group) Get(key string) (ByteView, error) {
	if key == "" {
		return ByteView{}, fmt.Errorf("key is required")
	}
    // 首先检查本地
	if v, ok := g.mainCache.get(key); ok {
		log.Println("[GCache] hit")
		return v, nil
	}
    // load
	return g.load(key)
}

func (g *Group) load(key string) (value ByteView, err error) {
    // 如果一致性哈希到远程，则远程获取
	if g.peerPicker != nil {
		if peerGetter, ok := g.peerPicker.PickPeer(key); ok {
			if value, err := g.getFromPeer(peerGetter, key); err == nil {
				return value, nil
			}
			log.Println("[gcache] Failed to get from peer", err)
		}
	}
    // 否则本地执行Getter回调函数
	return g.getLocally(key)

}
```

## 接口型函数

```
type Getter interface {
	Get(key string) ([]byte, error)
}

type GetterFunc func(key string) ([]byte, error)

func (f GetterFunc) Get(key string) ([]byte, error) {
	return f(key)
}
```

可以有多种方式调用该函数：

- 方式一：GetterFunc 类型的函数作为参数

```
GetFromSource(GetterFunc(func(key string) ([]byte, error) {
	return []byte(key), nil
}), "hello")
```

支持匿名函数，也支持普通的函数：

```
func test(key string) ([]byte, error) {
	return []byte(key), nil
}

func main() {
    GetFromSource(GetterFunc(test), "hello")
}
```

将 test 强制类型转换为 GetterFunc，GetterFunc 实现了接口 Getter，是一个合法参数。这种方式适用于逻辑较为简单的场景。

- 方式二：实现了 Getter 接口的结构体作为参数

```
type DB struct{ url string}

func (db *DB) Query(sql string, args ...string) string {
	// ...
	return "hello"
}

func (db *DB) Get(key string) ([]byte, error) {
	// ...
	v := db.Query("SELECT NAME FROM TABLE WHEN NAME= ?", key)
	return []byte(v), nil
}

func main() {
	GetFromSource(new(DB), "hello")
}
```

DB 实现了接口 Getter，也是一个合法参数。这种方式适用于逻辑较为复杂的场景，如果对数据库的操作需要很多信息，地址、用户名、密码，还有很多中间状态需要保持，比如超时、重连、加锁等等。这种情况下，更适合封装为一个结构体作为参数。

这样，既能够将普通的函数类型（需类型转换）作为参数，也可以将结构体作为参数，使用更为灵活，可读性也更好，这就是接口型函数的价值。

# httpPool

## 初始化

```
type HTTPPool struct {
	self       string
	basePath   string
	mu         sync.Mutex
	peers      *consistenthash.Map
	httpGetter map[string]*httpGetter
}

func NewHTTPPool(self string) *HTTPPool {
	return &HTTPPool{
		self:     self,
		basePath: defaultBasePath,
	}
}

// peer: ip:port
// peers中包括自己
func (p *HTTPPool) SetPeers(peers ...string) {
	p.mu.Lock()
	defer p.mu.Unlock()
	p.peers = consistenthash.New(defaultReplicas, nil)
	p.peers.Add(peers...)
	p.httpGetter = make(map[string]*httpGetter, len(peers))
	for _, peer := range peers {
		p.httpGetter[peer] = &httpGetter{baseURL: p.getPeerURL(peer)}
	}
}
```

选择peer：

```
func (p *HTTPPool) PickPeer(key string) (PeerGetter, bool) {
	p.mu.Lock()
	defer p.mu.Unlock()
	if peer := p.peers.Get(key); peer != "" && peer != p.self {
		p.Log("Pick peer %s", peer)
		return p.httpGetter[peer], true
	}
	return nil, false
}
```

# server

读config，获取所有client的地址
