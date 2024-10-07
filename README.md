# 项目目的
基于成熟的 https://github.com/mqttjs  mqtt.js `4.3.8`版本进行改造。 提升它对uniapp等环境的适配。这里不使用5.x版本的原因是因为5.x版本全面使用ts进行编写，uniapp对ts的支持有大坑（如：不支持??语法和三目运算符。非常离谱）。

# 修改点
针对uniapp环境的独特性，它不能使用常规的方式进行创建websocket，所以本次改造将允许自定义websocket
修改了`dist\mqtt.js`的`createWebSocket` 和 `createBrowserWebSocket`方法：
``` javascript
function createWebSocket (client, url, opts) {
  debug('createWebSocket')
  debug('protocol: ' + opts.protocolId + ' ' + opts.protocolVersion)
  const websocketSubProtocol =
    (opts.protocolId === 'MQIsdp') && (opts.protocolVersion === 3)
      ? 'mqttv3.1'
      : 'mqtt'

  debug('creating new Websocket for url: ' + url + ' and protocol: ' + websocketSubProtocol)
  let socket;
  if (opts.createWebsocket) {//允许自定义websocket
	socket = opts.createWebsocket(url, [websocketSubProtocol], opts);
  } else {
	socket = new WS(url, [websocketSubProtocol], opts.wsOptions)
  }
  
  return socket
}

function createBrowserWebSocket (client, opts) {
  const websocketSubProtocol =
  (opts.protocolId === 'MQIsdp') && (opts.protocolVersion === 3)
    ? 'mqttv3.1'
    : 'mqtt'

  const url = buildUrl(opts, client)
  /* global WebSocket */
  let socket;
  if (opts.createWebsocket) {//允许自定义websocket
	socket = opts.createWebsocket(url, [websocketSubProtocol], opts);
  } else {
	socket = new WebSocket(url, [websocketSubProtocol]);
  }
  
  socket.binaryType = 'arraybuffer'
  return socket
}
```

# 安装方法
`npm install mqtt-expand`

# 引用方法
`import mqtt, { MqttClient, IClientOptions } from 'mqtt-expand/dist/mqtt.js'`

# uniapp调用示例
``` typescript
......
// 自定义 createWebsocket 函数，使用 Uniapp 的 WebSocket
const createWebUnisocket = (url, websocketSubProtocols, options) => {
  const socketTask = uni.connectSocket({
    url: url,
    protocols: websocketSubProtocols,  // 使用传入的子协议
    success: () => {
      console.log('WebSocket connected via UniApp');
    },
    fail: (error) => {
      console.error('WebSocket connection failed: ', error);
    }
  });

  // 模拟 WebSocket 对象的接口
  const uniAppWebSocket = {
    send: (data) => {
      socketTask.send({
        data: data,
        success: () => console.log('Data sent successfully'),
        fail: (err) => console.error('Send failed: ', err)
      });
    },
    close: () => {
      socketTask.close();
    },
    onopen: null,
    onmessage: null,
    onclose: null,
    onerror: null,
  };

  // 绑定 Uniapp WebSocket 事件
  socketTask.onOpen((res) => {
    if (uniAppWebSocket.onopen) {
      uniAppWebSocket.onopen(res);
    }
  });

  socketTask.onMessage((res) => {
    if (uniAppWebSocket.onmessage) {
      uniAppWebSocket.onmessage(res);
    }
  });

  socketTask.onClose((res) => {
    if (uniAppWebSocket.onclose) {
      uniAppWebSocket.onclose(res);
    }
  });

  socketTask.onError((res) => {
    if (uniAppWebSocket.onerror) {
      uniAppWebSocket.onerror(res);
    }
  });

  return uniAppWebSocket;
};

//传入createWebsocket 。自定义websocket的创建
......
const optionsTempWorkAround = Object.assign(
  { 
    createWebsocket: createWebUnisocket,
  }
)
console.log(url)
const curConnectClient: MqttClient = mqtt.connect(url, optionsTempWorkAround)
......
```
