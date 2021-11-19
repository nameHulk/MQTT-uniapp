<template>
	<view class="mqtt-page">
		<view class="book-bar">
			<input class="inp-id" v-model="connectInfo.clientId" placeholder="clientId"></input>
			<input class="inp-id" v-model="connectInfo.username" placeholder="username"></input>
			<input class="inp-id" v-model="connectInfo.password" placeholder="password"></input>
		</view>
		<view class="book-bar">
			<view class="btn-book" @click="startConnect">连接</view>
			<view class="btn-book" @click="endConnect">终止</view>
			<view class="btn-book" @click="reConnect">重连</view>
			<view class="btn-qos" @click="connectInfo.clean = !connectInfo.clean">清除{{connectInfo.clean?' Yes':' No'}}</view>
		</view>
		<view class="book-bar">
			<input class="inp-tpc" v-model="subscribeInfo.topic" placeholder="topic"></input>
			<view class="btn-qos" @click="changeQos">Qos {{subscribeInfo.qos}}</view>
			<view class="btn-book" @click="startSubscribe">订阅</view>
			<view class="btn-book" @click="endSubscribe">取订</view>
		</view>
		<view class="book-bar">
			<input class="inp-tpc" v-model="publishInfo.topic" placeholder="topic"></input>
			<input class="inp-msg" v-model="publishInfo.message" placeholder="message"></input>
			<view class="btn-book" @click="publish">发送</view>
		</view>
		<view class="tips-bar">
			<text>操作日志：</text>
			<view class="text-btn">
				<text :style="isBuffer?'':'opacity:0.6'" @click="isBuffer=!isBuffer">打印Buffer</text>
				<text @click="clearConfig">清空配置</text>
				<text @click="cealrLog">清空日志</text>
			</view>
		</view>
		<scroll-view scroll-y class="tips-panel">
			<view class="panel-list" v-for="(item,index) in logs" :key="index">
				{{item.option + item.log}}
			</view>
		</scroll-view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				connectInfo: {
					clientId: 'App_0109',
					username: '',
					password: '',
					clean: false
				},
				subscribeInfo: {
					topic: '',
					qos: 1,
				},
				publishInfo: {
					topic: '',
					message: ''
				},
				isBuffer: false,
				logs: [
					{option:'环境配置：', log:'配置成功'},
				]
			}
		},
		onLoad() {
			
		},
		methods: {
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
			/* 终止连接 */
			endConnect(){
				var _this = this
				this.$mqttTool.end().then(res =>{
					_this.logs.unshift({option:'终止：', log:res})
				})
			},
			/* 重新连接 */
			reConnect(){
				var _this = this
				this.$mqttTool.reconnect().then(res =>{
					_this.logs.unshift({option:'重连：', log:res})
				})
			},
			/* 更改Qos */
			changeQos(){
				var _this = this
				if(this.subscribeInfo.qos >= 2){
					this.subscribeInfo.qos = 0
					this.logs.unshift({option:'Qos：', log:this.subscribeInfo.qos})
					this.startSubscribe();
				}else{
					this.subscribeInfo.qos += 1
					this.logs.unshift({option:'Qos：', log:this.subscribeInfo.qos})
					this.startSubscribe();
				}
			},
			/* 订阅 */
			startSubscribe(){
				if(this.subscribeInfo.topic == ''){
					uni.showToast({
						icon: 'none',
						title: '输入topic'
					})
					return;
				}
				var _this = this
				let opts = {
					topic: this.subscribeInfo.topic,
					qos: this.subscribeInfo.qos,
				}
				this.$mqttTool.subscribe(opts).then(res =>{
					_this.logs.unshift({option:'订阅' + opts.topic + '：', log:res})
				})
			},
			/* 取消订阅 */
			endSubscribe(){
				var _this = this
				let opts = {
					topic: this.subscribeInfo.topic
				}
				this.$mqttTool.unsubscribe(opts).then(res =>{
					_this.logs.unshift({option:'取消订阅：', log:res})
				})
			},
			/* 发送消息 */
			publish(){
				var _this = this
				let opts = {
					topic: this.publishInfo.topic,
					message: this.publishInfo.message,
				}
				this.$mqttTool.publish(opts).then(res =>{
					_this.logs.unshift({option:'发送：', log:res})
				})
			},
			/* 清空配置 */
			clearConfig(){
				this.endSubscribe();
				this.connectInfo = {
					clientId: '',
					username: '',
					password: ''
				}
				this.subscribeInfo = {
					topic: '',
					qos: 1,
				}
				this.publishInfo = {
					topic: '',
					message: ''
				}
				this.isBuffer = false
			},
			/* 清空日志 */
			cealrLog(){
				this.logs = [{option:'环境配置：', log:'配置成功'}]
			}
		}
	}
</script>

<style lang="less" scoped>
	.mqtt-page {
		width: 100%;
		height: 100%;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		.book-bar{
			width: 710upx;
			height: 70upx;
			margin: 0 auto 20upx auto;
			display: flex;
			flex-direction: row;
			align-items: center;
			justify-content: flex-start;
			.btn-book{
				width: 150upx;
				height: 70upx;
				border-radius: 10px;
				background-color: #3F536E;
				text-align: center;
				line-height: 70upx;
				color: #ffffff;
				margin-right: 20upx;
			}
			.btn-qos{
				width: 150upx;
				height: 70upx;
				border-radius: 10px;
				background-color: #4c6b99;
				text-align: center;
				line-height: 70upx;
				color: #ffffff;
				margin-right: 20upx;
			}
			.inp-id{
				width: 160upx;
				height: 60upx;
				border-radius: 10px;
				border: 2px solid #3F536E;
				color: #3F536E;
				padding: 0upx 20upx;
				margin-right: 20upx;
			}
			.inp-tpc{
				width: 120upx;
				height: 60upx;
				border-radius: 10px;
				border: 2px solid #3F536E;
				color: #3F536E;
				padding: 0upx 20upx;
				margin-right: 20upx;
			}
			.inp-msg{
				width: 260upx;
				height: 60upx;
				border-radius: 10px;
				border: 2px solid #3F536E;
				color: #3F536E;
				padding: 0upx 20upx;
				margin-right: 20upx;
			}
		}
		.tips-bar{
			width: 710upx;
			height: 30upx;
			margin: 20upx auto;
			display: flex;
			flex-direction: row;
			align-items: center;
			justify-content: space-between;
			font-size: 14px;
			.text-btn{
				font-size: 12px;
				width: 320upx;
				display: flex;
				align-items: center;
				justify-content: space-between;
				color: #3F536E;
			}
		}
		.tips-panel{
			width: 100%;
			margin: 0 auto;
			height: calc(100vh - 430upx);
			font-size: 13px;
			color: #3e3d3e;
			word-break: break-all;
			.panel-list{
				padding: 10upx 20upx;
				border-bottom: 1px solid #eeeeee;
			}
		}
	}
</style>
