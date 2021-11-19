# MQTT 演示及调试demo

仅作为供MQTT初试者参考范例

---

## 如何使用

- 下载项目代码
- npm install 安装mqtt.js包
- HbuilderX运行项目即可（请先替换配置中mqtt的connect地址为你个人地址）

### 1. 代码示例[mqttTool.js]

- mqttTool为个人处理工具

```js
var mqtt = require('mqtt/dist/mqtt.js')

let mqttTool = {
	client: null
}

mqttTool.connect = function(params){
	let options = {
		clientId: params.clientId,
		username: params.username,
		password: params.password,
		clean: params.clean,
		connectTimeout: 600000, 
		cleanSession: false
	}
	let client = mqtt.connect(params.url, options);
	mqttTool.client = client
	return client;
}

mqttTool.end = function(){
	return new Promise((resolve, reject) => {
		if(mqttTool.client == null){
			resolve('未连接')
			console.log('App_text' + ":end 未连接")
			return;
		}
		mqttTool.client.end()
		mqttTool.client = null
		resolve('连接终止')
	})
}

mqttTool.reconnect = function(){
	return new Promise((resolve, reject) => {
		if(mqttTool.client == null){
			resolve('未连接')
			console.log('App_text' + ":reconnect 未连接")
			return;
		}
		mqttTool.client.reconnect()
	})
}

mqttTool.subscribe = function(params){
	return new Promise((resolve, reject) => {
		if(mqttTool.client == null){
			resolve('未连接')
			console.log('App_text' + ":unconnect 未连接")
			return;
		}
		mqttTool.client.subscribe(params.topic, {qos:params.qos}, function(err,res) {
			console.log(err,res)
			if (!err && res.length>0) {
				resolve('订阅成功')
				console.log('App_text' + ":subscribe success 订阅成功")
			}else{
				resolve('订阅失败')
				console.log('App_text' + ":subscribe failed 订阅失败")
				return;
			} 
		})  
	})
}

mqttTool.unsubscribe = function(params){
	return new Promise((resolve, reject) => {
		if(mqttTool.client == null){
			resolve('未连接')
			console.log('App_text' + ":unconnect 未连接")
			return;
		}
		mqttTool.client.unsubscribe(params.topic, function(err) {
			if (!err) {
				resolve('取消订阅成功')
				console.log('App_text' + ":unsubscribe success 取消订阅成功")
			}else{
				resolve('取消订阅失败')
				console.log('App_text' + ":unsubscribe failed 取消订阅失败")
				return;
			} 
		})  
	})
}

mqttTool.publish = function(params){
	return new Promise((resolve, reject) => {
		if(mqttTool.client == null){
			resolve('未连接')
			console.log('App_text' + ":unconnect 未连接")
			return;
		}
		mqttTool.client.publish(params.topic, params.message, function(err){
			if (!err) {
				resolve(params.topic + '-' + params.message + '-发送成功')
				console.log('App_text' + ":publish success 发送成功")
			}else{
				resolve(params.topic + '-' + params.message + '-发送失败')
				console.log('App_text' + ":publish failed 发送失败")
				return;
			} 
		})
	})
}

export default mqttTool
```

### 2. 代码示例[main.js]

- 使用范例
  ```js
  import mqttTool from './lib/mqttTool.js'
  Vue.prototype.$mqttTool = mqttTool
  ```


### 3. 代码示例[index.vue]

- 使用范例
  ```js
  /* 连接 */
    		async startConnect(){
    			var _this = this
    			let opts = {
    				url: 'wx://192.168.1.131:8083/mqtt',
    				clientId: this.connectInfo.clientId,
    				username: this.connectInfo.username,
    				password: this.connectInfo.password,
    				clean: this.connectInfo.clean
    			}
    			var client = await this.$mqttTool.connect(opts);
    			client.on('connect', function(res) {
    				_this.logs.unshift({option:'mqtt：', log:'连接成功'})
    			})
    			client.on('reconnect', function(res) {
    				_this.logs.unshift({option:'mqtt：', log:'重新连接'})
    			})
    			client.on('error', function(res) {
    				_this.logs.unshift({option:'mqtt：', log:'连接错误'})
    			})
    			client.on('close', function(res) {
    				_this.logs.unshift({option:'mqtt：', log:'关闭成功'})
    			})
    			client.on('message', function(topic, message, buffer) {
    				if(_this.isBuffer){
    					_this.logs.unshift({option:topic+' buffer：', log: JSON.stringify(buffer)})
    				}
    				_this.logs.unshift({option:topic+' message：' + buffer.messageId, log: message.toString()})
    			})
    		},
  ```
