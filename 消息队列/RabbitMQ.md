

## 一  消息可靠性

### 1. 生产者消息可靠性

#### 1.1 开启 confirm 机制

通过`cofirmSelect()`在 channel 上开启确认模式，添加监听，来监听 mq 的应答。

