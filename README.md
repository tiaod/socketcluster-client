# 修改详情
改写了`./lib/sctransport.js`
1. 重写`SCTransport.prototype.open`，将`createWebSocket`函数替换成`wx.connectSocket({url})`。
```javascript
SCTransport.prototype.open = function () {
  //……
  wx.connectSocket({
    url:uri
  })
  //……
};
```
同时为了避免onopen，onclose，onmessage，onerror被重复调用（因为之前是每个socket都会单独监听。而小程序只有一个socket，这些事件都是全局的）。将这些监听转移到constructor：
```javascript
var SCTransport = function (authEngine, codecEngine, options){
  //……
  wx.onSocketOpen(function () {
    self._onOpen()
  })
  wx.onSocketClose(function (res) {
    var code;
    if (res.code == null) {
      code = 1005;
    } else {
      code = res.code;
    }
    self._onClose(code, res.reason);
  })
  wx.onSocketMessage(function(res) {
    self._onMessage(res.data)
  })
  wx.onSocketError(function (res) {
    if (self.state === self.CONNECTING) {
      self._onClose(1006);
    }
  })
}
```
3. 继续搜索关键字`.socket`，找到所有和`self.socket`，`this.socket`相关的API，能注释的则注释掉。不能注释的使用小程序相关API替换。例如替换所有的`self.socket.onopen=CALLBACK`为`wx.onSocketOpen(CALLBACK)`。（看了一下，基本都是能注释的）

# 2017年8月7日09:10:40更新，解决pingTimeout问题
注释所有.socket.close逻辑。对于socket需要关闭的时一律不调用`wx.closeSocket`，因为微信小程序只允许建立一个websocket连接，建立了新的，旧的会自动关闭。如果写了`wx.closeSocket`的话，可能会导致旧的旧的SCTransport调用wx.closeSocket，从而把新的连接也关闭了，导致pingTimeout。
