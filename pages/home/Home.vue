<template>
	<view class="mqtt">
		<view class="options">
			<view class="title" :style="'padding-top:' + systemInfo.statusBarHeight + 'px;'">
				<text>MQTT-调试</text>
				<text class="title-btn" @click="switchOldPage">回到旧版页面</text>
			</view>
			<view class="info">
				<view class="bar">
					<text class="label">clientId:</text>
					<input v-model="connectInfo.clientId" class="value" style="width:70%" :adjust-position="false" placeholder=""/>
				</view>
				<view class="flex-between">
					<view class="bar">
						<text class="label">username:</text>
						<input v-model="connectInfo.username" class="value" :adjust-position="false" placeholder=""/>
					</view>
					<view class="bar" style="margin-left:4px;">
						<text class="label">password:</text>
						<input v-model="connectInfo.password" class="value" :adjust-position="false" placeholder=""/>
					</view>
				</view>
				<view class="flex-between">
					<view class="btn" @click="startConnect">连接</view>
					<view class="btn" @click="endConnect">终止</view>
					<view class="btn" @click="reConnect">重新连接</view>
				</view>
				<view class="flex-between">
					<view class="bar" style="width:40%;">
						<text class="label">topic</text>
						<input v-model="subscribeInfo.topic" class="value" :adjust-position="false" placeholder=""/>
					</view>
					<view class="bar" @click="changeQos">
						<text class="label">qos</text>
						<text class="value">{{subscribeInfo.qos}}</text>
					</view>
					<view class="bar" @click="connectInfo.clean = !connectInfo.clean">
						<text class="label">clean</text>
						<text class="value">{{connectInfo.clean}}</text>
					</view>
				</view>
				<view class="flex-row">
					<view class="btn" @click="startSubscribe">订阅</view>
					<view class="btn" style="margin-left:6px;" @click="endSubscribe">取消订阅</view>
				</view>
				<view class="tips">
					<text class="tips-lab">操作日志：</text>
					<text class="tips-reset" @click="clearConfig">清空配置</text>
					<text class="tips-buffer" :style="isBuffer?'':'opacity:0.5'" @click="isBuffer=!isBuffer">打印Buffer</text>
					<text class="tips-clear" @click="cealrLog">清空日志</text>
				</view>
			</view>
		</view>
		<scroll-view class="logs" scroll-y>
			<view class="logs-list" v-for="(item,index) in logs" :key="index">
				{{item.option + item.log}}
			</view>
		</scroll-view>
		<view class="feet" :style="'bottom:' + (isSending?sendInfo.keyboard:0) + 'px;'">
			<view class="inpbar">
				<input v-model="sendInfo.msg" @focus="focusSend(true)" @blur="focusSend(false)" :adjust-position="false" :cursor-spacing="12" class="inp" type="text" placeholder="请输入内容" placeholder-style="color:#cccccc;"/>
				<view v-if="sendInfo.msg==''" class="send send-dark">发送</view>
				<view v-else @click="publish()" class="send">发送</view>
			</view>
		</view>
	</view>
</template>

<script>
	export default{
		data(){
			return{
				/* 连接信息 */
				connectInfo: {
					clientId: 'Uniapp' + Date.parse(new Date()),
					username: 'Mason',
					password: '123456',
					clean: false
				},
				/* 订阅信息 */
				subscribeInfo: {
					topic: 'testtopic',
					qos: 1
				},
				/* 日志信息 */
				isBuffer: false,
				logs: [
					{option:'环境配置：', log:'配置成功'},
				],
				/* 发送信息 */
				isSending: false,
				sendInfo: {
					msg: '',
					keyboard: 0
				}
			}
		},
		computed:{
			/* system */
			systemInfo(){
				return uni.getSystemInfoSync()
			},
		},
		onLoad(){
			/* 即时通讯类键盘元素高度处理 */
			uni.onKeyboardHeightChange(res => {
				this.sendInfo.keyboard = res.height
			})
		},
		onUnload(){
			uni.offKeyboardHeightChange();
		},
		methods:{
			/* 连接 */
			async startConnect(){
				var _this = this
				let opts = {
					url: 'wx://你的通讯地址:你的通讯端口号/mqtt',
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
					_this.logs.unshift({option:topic+' message：', log: message.toString()})
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
					this.logs.unshift({option:'Qos：', log:this.subscribeInfo.qos + ' Qos变更，订阅已取消，请重新发起订阅'})
					this.endSubscribe();
				}else{
					this.subscribeInfo.qos += 1
					this.logs.unshift({option:'Qos：', log:this.subscribeInfo.qos + ' Qos变更，订阅已取消，请重新发起订阅'})
					this.endSubscribe();
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
					topic: this.subscribeInfo.topic,
					message: this.sendInfo.msg,
				}
				this.$mqttTool.publish(opts).then(res =>{
					_this.sendInfo.msg = ''
					_this.logs.unshift({option:'发送：', log:res})
				})
			},
			/* 清空配置 */
			clearConfig(){
				this.endConnect();
				this.connectInfo = {
					clientId: '',
					username: '',
					password: ''
				}
				this.subscribeInfo = {
					topic: '',
					qos: 1,
					sendMsg: ''
				}
				this.isBuffer = false
			},
			/* 清空日志 */
			cealrLog(){
				this.logs = [{option:'环境配置：', log:'配置成功'}]
			},
			/* ***********************MQTT无关事件*********************** */
			/* 回到旧版 */
			switchOldPage(){
				uni.navigateTo({
					url: '../index/index'
				})
			},
			/* 聚焦 */
			focusSend(val){
				this.isSending = val
			},
		}
	}
</script>

<style lang="less" scoped>
	.flex-row{
		display: flex;
		flex-direction: row;
		align-items: center;
		justify-content: flex-start;
	}
	.flex-between{
		display: flex;
		flex-direction: row;
		align-items: center;
		justify-content: space-between;
	}
	.mqtt{
		width: 100vw;
		height: 100vh;
		background-color: #f8f8f8;
		.options{
			width: 100%;
			height: 316px;
			background-color: #ffffff;
			box-shadow: 0px 0px 8px rgba(0, 0, 0, 0.1);
			font-size: 13px;
			color: #666666;
			.title{
				width: 710upx;
				margin: 0 auto;
				font-size: 15px;
				font-weight: 500;
				color: #333333;
				text-align: center;
				padding-bottom: 8px;
				display: flex;
				flex-direction: row;
				align-items: center;
				justify-content: space-between;
				.title-btn{
					font-size: 14px;
					font-weight: 400;
					color: #3F536E;
				}
			}
			.info{
				width: 710upx;
				margin: 0 auto;
				.bar{
					min-width: 160upx;
					margin-bottom: 8px;
					height: 30px;
					border-radius: 8px;
					box-shadow: inset 0px 0px 6px rgba(0, 0, 0, 0.1);
					border: 1px solid #cccccc;
					display: flex;
					flex-direction: row;
					align-items: center;
					justify-content: flex-start;
					font-size: 14px;
					.label{
						height: 30px;
						background-color: #f8f8f8;
						padding: 0px 10px;
						line-height: 30px;
						border-right: 1px solid #eeeeee;
						border-radius: 15px 0px 0px 15px;
					}
					.value{
						height: 30px;
						padding: 0px 12px;
						line-height: 30px;
						font-size: 14px;
						color: #3F536E;
					}
				}
				.btn{
					width: 30%;
					height: 32px;
					background-color: #3F536E;
					color: #ffffff;
					border-radius: 8px;
					margin-bottom: 8px;
					text-align: center;
					line-height: 32px;
				}
				.tips{
					width: 100%;
					height: 30px;
					margin-top: 6px;
					display: flex;
					flex-direction: row;
					align-items: center;
					position: relative;
					.tips-lab{
						font-size: 14px;
						color: #333333;
					}
					.tips-reset{
						font-size: 14px;
						font-weight: 500;
						color: #3F536E;
						position: absolute;
						right: 150px;
					}
					.tips-buffer{
						font-size: 14px;
						font-weight: 500;
						color: #3F536E;
						position: absolute;
						right: 70px;
					}
					.tips-clear{
						font-size: 14px;
						font-weight: 500;
						color: #3F536E;
						position: absolute;
						right: 0px;
					}
				}
			}
		}
		.logs{
			width: 100%;
			height: calc(100% - 396px);
			.logs-list{
				padding: 14upx 20upx;
				border-bottom: 1px solid #eeeeee;
				font-size: 13px;
				color: #3e3d3e;
				word-break: break-all;
			}
		}
		.feet{
			width: 100%;
			height: 80px;
			background-color: #ffffff;
			position: absolute;
			bottom: 0px;
			transition: all 0.35s;
			-webkit-transition: all 0.35s;
			&::before{
				position: absolute;
				content: '';
				top: 0px;
				background: rgba(0,0,0,0.2);
				width: 100%;
				height: 1px;
				transform: scaleY(0.5);
				transform-origin: 0 0;
				-webkit-transform: scaleY(0.5);
				-webkit-transform-origin: 0 0;
			}
			.inpbar{
				width: 710upx;
				height: 44px;
				background-color: #eeeeee;
				border-radius: 10px;
				position: absolute;
				left: 20upx;
				top: 10px;
				display: flex;
				flex-direction: row;
				align-items: center;
				justify-content: space-between;
				.img{
					width: 30px;
					height: 30px;
					margin-left: 12px;
				}
				.inp{
					width: calc(100% - 138px);
					height: 40px;
					margin: 0px 12px;
					font-size: 14px;
					color: #000000;
				}
				.send-dark{
					opacity: 0.7;
				}
				.send{
					width: 60px;
					height: 32px;
					background-color: #3F536E;
					border-radius: 6px;
					font-size: 14px;
					color: #ffffff;
					display: flex;
					align-items: center;
					justify-content: center;
					margin-right: 12px;
				}
			}
		}
	}
</style>