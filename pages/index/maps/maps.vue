<template>
	<view class="maps-container">
		<view class="search-bar">
			<view class="search-bar-form">
				<view class="search-bar-box">
					<icon class="icon-search-in-box" type="search" :size="16"></icon>
					<input confirm-type="search" class="search-bar-input" placeholder="请输入您的目的地" :value="inputVal" :focus="inputShowed"
					 @confirm="bindInput" @input="inputTyping" />
					<view class="icon-clear" v-if="inputVal" @tap="clearInput">
						<icon type="clear" size="14"></icon>
					</view>
				</view>
				<view class="search-bar-label" v-show="!inputShowed" @tap="showInput">
					<icon class="icon-search" type="search" :size="16"></icon>
					<view class="search-bar-text">请输入您的目的地</view>
				</view>
			</view>
			<view class="cancel-btn" @tap="hideInput" v-show="inputShowed">取消</view>
		</view>
		<map class="tui-map" :latitude="lat" :longitude="lng" :markers="covers" @markertap="marker" :scale="12"></map>
		<scroll-view scroll-y class="scrollView" :style="{height:scrollH +'px'}">
			<view class="tui-list">
				<view class="tui-list-cell" :class="[index==address.length-1?'tui-cell-last':'']" v-for="(item,index) in address"
				 :key="index">
					<view class="addr-title">
						<text>{{item.title}}</text>
					</view>
					<view class="addr-box ">
						<view class="addr-detail ">
							<text class="distance">{{item.distance}}m</text>
							{{item.address}}
						</view>
						<view class="addr-opera ">
							<view class="opera-box " hover-class="tui-opcity" :hover-stay-time="150 " @tap="call " :data-id="item.id" v-if="item.tel">
								<image src="/static/images/my/call.png" class="mini-img"></image>
								<view class="text">打电话</view>
							</view>
							<view class="opera-box " hover-class="tui-opcity" :hover-stay-time="150" @tap="go" :data-id="item.id">
								<image src="/static/images/my/go.png" class="mini-img"></image>
								<view class="text">到这里</view>
							</view>
						</view>
					</view>
				</view>
			</view>
		</scroll-view>
	</view>
</template>

<script>
	const amap = require('@/libs/amap-wx.js')
	export default {
		data() {
			return {
				inputShowed: false,
				inputVal: "",
				amapPlugin: null,
				key: "6799b5f6f88d3d9fb52ac244855a8759",
				lat: 22.63137,
				lng: 114.010857,
				covers: [],
				address: [],
				scrollH: 256
			}
		},
		onLoad(options) {
			this.amapPlugin = new amap.AMapWX({
				key: this.key
			});
			setTimeout(() => {
				let winHeight = uni.getSystemInfoSync().windowHeight;
				this.scrollH = winHeight - 44 - uni.upx2px(600);
				this.getLocation(() => {
					this.getPoiAround(options.key || "加油站")
				});
			}, 300)
		},
		methods: {
			trim: function(value) {
				return value ? value.toString().replace(/(^\s*)|(\s*$)/g, "") : value;
			},
			showInput() {
				this.inputShowed = true
			},
			hideInput() {
				this.inputVal = "";
				this.inputShowed = false;
				uni.hideKeyboard(); //强行隐藏键盘
			},
			clearInput() {
				this.inputVal = ""
			},
			inputTyping: function(e) {
				this.inputVal = e.detail.value
			},
			getLocation(callback) {
				const that = this
				this.amapPlugin.getRegeo({
					success: (data) => {
						that.lng = data[0].longitude;
						that.lat = data[0].latitude;
						callback();
					},
					fail: (info) => {
						callback();
					}
				})
			},
			getPoiAround(keywords) {
				//检索周边的POI	
				uni.showLoading({
					title: "加载中..."
				})
				const that = this;
				setTimeout(() => {
					this.amapPlugin.getPoiAround({
						querykeywords: keywords,
						location: '', //location： 经纬度坐标。 为空时， 基于当前位置进行地址解析。 格式： '经度,纬度'
						success: (data) => {
							let arr = [];
							let addr = [];
							for (let i = 0; i < data.markers.length; i++) {
								arr.push({
									id: i,
									latitude: data.markers[i].latitude,
									longitude: data.markers[i].longitude,
									title: data.markers[i].name
								})
								let tel = that.trim(data.poisData[i].tel);
								if (~tel.indexOf(";")) {
									tel = tel.split(";")[0]
								}
								addr.push({
									id: i,
									latitude: data.markers[i].latitude,
									longitude: data.markers[i].longitude,
									title: data.markers[i].name,
									address: data.markers[i].address,
									tel: tel,
									distance: data.poisData[i].distance
								})
							}

							this.address = addr;
							this.covers = arr;
							uni.hideLoading()
						},
						fail: (info) => {
							uni.showToast({
								title: '获取位置信息失败，请检查是否打开位置权限'
							})
							uni.hideLoading()
						}
					})
				}, 0);

			},
			bindInput: function(e) {
				const keywords = e.detail.value;
				this.getPoiAround(keywords);
			},
			marker: function(e) {
				const that = this
				const item = that.address[e.markerId || 0];
				const menu = item.tel ? ["打电话", "到这里"] : ["到这里"];

				uni.showActionSheet({
					itemList: menu,
					success(res) {
						if (res.tapIndex == 0 && item.tel) {
							uni.makePhoneCall({
								phoneNumber: item.tel
							})
						} else {
							const latitude = Number(item.latitude)
							const longitude = Number(item.longitude)
							uni.openLocation({
								name: item.title,
								address: item.address,
								latitude,
								longitude,
								scale: 18
							})
						}
					},
					fail(res) {
						console.log(res.errMsg)
					}
				})

			},
			call(event) {
				const index = Number(event.currentTarget.dataset.id);
				const tel = this.address[index].tel;
				if (tel) {
					uni.makePhoneCall({
						phoneNumber: tel
					})
				}

			},
			go(event) {
				const index = Number(event.currentTarget.dataset.id)
				const item = this.address[index];
				const latitude = Number(item.latitude)
				const longitude = Number(item.longitude)
				uni.openLocation({
					name: item.title,
					address: item.address,
					latitude,
					longitude,
					scale: 18
				})
			}
		}
	}
</script>

<style>
	page {
		height: 100%;
	}

	.tui-list {
		background-color: #fff;
		position: relative;
		width: 100%;
		display: flex;
		flex-direction: column;
		padding-bottom: env(safe-area-inset-bottom);
	}

	.tui-list-cell {
		position: relative;
		display: flex;
		flex-direction: row;
		justify-content: space-between;
		align-items: center;
	}

	.tui-list-cell::after {
		content: '';
		position: absolute;
		border-bottom: 1rpx solid #eaeef1;
		-webkit-transform: scaleY(0.5);
		transform: scaleY(0.5);
		bottom: 0;
		right: 0;
		left: 30rpx;
	}

	.tui-cell-last::after {
		border-bottom: 0;
	}

	.maps-container {
		width: 100%;
		height: 100%;
		overflow: hidden;
	}

	.tui-map {
		width: 100%;
		height: 600rpx;
		display: block;
	}

	.scrollView {
		width: 100%;
		padding-bottom: 100rpx;
		background-color: #fff;
	}

	.search-bar {
		position: relative;
		padding: 8px 10px;
		display: -webkit-flex;
		display: flex;
		box-sizing: border-box;
		align-items: center;
		background-color: #fff;
	}

	.icon-search {
		margin-right: 8px;
	}

	.icon-search-in-box {
		margin-right: 8rpx;
	}

	.search-bar-form {
		position: relative;
		-webkit-box-flex: 1;
		-webkit-flex: auto;
		flex: auto;
		border-radius: 5px;
		background-color: #f0f0f0;
	}

	.search-bar-text {
		display: inline-block;
		font-size: 14px;
		vertical-align: middle;
	}

	.search-bar-box {
		width: 100%;
		height: 30px;
		padding-left: 10px;
		padding-right: 30px;
		box-sizing: border-box;
		display: flex;
		align-items: center;
		flex-wrap: nowrap;
		position: relative;
		z-index: 1;
		
	}

	.search-bar-input {
		font-size: 14px;
		flex: 1;
	}

	.icon-clear {
		height: 100%;
		position: absolute;
		top: 0;
		right: 0;
		padding: 0 8px;
		font-size: 0;
		display: flex;
		align-items: center;
	}

	.search-bar-label {
		width: 100%;
		height: 100%;
		position: absolute;
		left: 0;
		top: 0;
		z-index: 2;
		border-radius: 3px;
		text-align: center;
		color: #9b9b9b;
		background: #f0f0f0;
		display: flex;
		align-items: center;
		justify-content: center;

	}

	.cancel-btn {
		margin-left: 10px;
		line-height: 30px;
		color: #5982fd;
		white-space: nowrap;
		font-size: 15px;
	}

	.tui-list-cell {
		display: initial;
		padding: 30rpx;
	}

	.addr-title {
		font-size: 30rpx;
		line-height: 40rpx;
		color: #000;
		font-weight: bold;
		width: 100%;
		padding-bottom: 20rpx;
	}

	.addr-box {
		display: flex;
		align-items: flex-end;
		justify-content: space-between;
	}

	.addr-detail {
		width: 450rpx;
		overflow: hidden;
		text-overflow: ellipsis;
		display: -webkit-box;
		-webkit-line-clamp: 2;
		/* autoprefixer: off */
		-webkit-box-orient: vertical;
		/* autoprefixer: on */
		font-size: 24rpx;
		color: #999;
	}

	.distance {
		color: #5677fc;
		padding-right: 6rpx;
	}

	.addr-opera {
		display: flex;
		align-items: center;
		justify-content: space-between;
	}

	.opera-box {
		text-align: center;
		margin-left: 26rpx;
	}

	.mini-img {
		width: 44rpx;
		height: 44rpx;
	}

	.text {
		color: #333;
		font-size: 24rpx;
		line-height: 30rpx;
	}
</style>
