## 1 Bucket

```go
type Bucket struct {
    cLock sync.RWMutex // 保护通道映射的读写锁
    chs map[int]*Channel // 用户ID到通道的映射
    bucketOptions BucketOptions // Bucket 配置选项
    rooms map[int]*Room // Bucket 中的房间映射
    routines []chan *proto.PushRoomMsgRequest // 工作协程通道
    routinesNum uint64 // 当前工作协程计数器
    // broadcast chan []byte // 广播消息通道
}
```
