<template>
	<!-- 对话消息气泡：用户靠右（绿色），AI 靠左（白色） -->
	<view
		class="chat-msg"
		:class="[
			message.role === 'user' ? 'chat-msg--user' : 'chat-msg--ai',
			message.status === 'streaming' ? 'chat-msg--streaming' : '',
		]"
	>
		<!-- 气泡主体 -->
		<view class="chat-msg-bubble" :class="{ 'chat-bubble-has-chart': hasCharts }">
			<!-- insight 文本 -->
			<text class="chat-msg-text">{{ insight }}</text>

			<!-- 图表区域：AI 消息包含 charts 时渲染 -->
			<view v-if="hasCharts" class="chat-msg-charts">
				<ChatChart v-for="(chart, ci) in charts" :key="ci" :chart="chart" :idx="ci" />
			</view>

			<!-- 建议区域：AI 消息包含 suggestion 时渲染（横向：图标 | 文案） -->
			<view v-if="hasSuggestion" class="chat-msg-suggestion">
				<view class="chat-sug-icon">
					<image class="chat-sug-image" src="/static/ai/suggestion.png" mode="aspectFit" />
				</view>
				<view class="chat-sug-divider" />
				<view
					class="chat-sug-text"
					:class="{ 'chat-sug-text--expand': sugExpanded }"
					@click="onToggleSuggestion"
				>
					{{ message.suggestion }}
				</view>
			</view>

			<!-- 存图 / 存 PDF 按钮（图表消息专属） -->
			<view v-if="hasCharts" class="chat-msg-actions-bar">
				<view class="chat-action-btn" @click="onSaveImage">
					<image class="chat-action-icon" src="/static/ai/image.png" mode="aspectFit" />
					<text class="chat-action-label">存图</text>
				</view>
				<view class="chat-action-btn" @click="onSavePdf">
					<image class="chat-action-icon" src="/static/ai/pdf.png" mode="aspectFit" />
					<text class="chat-action-label">存PDF</text>
				</view>
			</view>

			<!-- 流式输出闪烁光标 -->
			<text v-if="message.status === 'streaming'" class="chat-msg-cursor">|</text>
			<!-- 错误态：重试按钮 -->
			<view v-if="message.status === 'error'" class="chat-msg-error" @click="$emit('retry')">
				<text class="chat-msg-error-icon">!</text>
				<text class="chat-msg-error-text">{{ $t('ai.clickRetry') }}</text>
			</view>

			<!-- 底部操作：点赞 / 反馈（AI 消息 done 态，在气泡内部右下角） -->
			<view v-if="showActions" class="chat-msg-bottom-actions">
				<view class="chat-ba-btn" @click="onLike">
					<image
						class="chat-ba-icon"
						:src="liked ? '/static/ai/zan_active.png' : '/static/ai/zan.png'"
						mode="aspectFit"
					/>
				</view>
				<view class="chat-ba-btn" @click="onFeedback">
					<image
						class="chat-ba-icon"
						:src="feedbacked ? '/static/ai/suggestion.png' : '/static/ai/feedback.png'"
						mode="aspectFit"
					/>
				</view>
			</view>
		</view>

		<!-- 存图成功提示：胶囊样式（居中、半透明黑底、白字） -->
		<view v-if="saveToastVisible" class="chat-save-toast">
			<text class="chat-save-toast-text">已保存图片到相册</text>
		</view>
	</view>
</template>

<script>
import ChatChart from './ChatChart.vue'
import tui from '@/common/httpRequest.js'

export default {
	name: 'ChatMessage',
	components: { ChatChart },
	props: {
		message: { type: Object, required: true },
		idx: { type: Number, default: 0 },
	},
	data() {
		return {
			liked: false, // 点赞状态
			feedbacked: false, // 反馈状态
			sugExpanded: false, // 建议区域是否展开全量文本
			saveToastVisible: false, // 存图成功提示是否展示
			_saveToastTimer: null, // 存图提示自动隐藏定时器
		}
	},
	beforeDestroy() {
		if (this._saveToastTimer) {
			clearTimeout(this._saveToastTimer)
			this._saveToastTimer = null
		}
	},
	computed: {
		/** 提取消息中的图表数据（AI 消息可能携带 charts 数组） */
		charts() {
			return this.message.charts || []
		},
		/** insight 文本：优先 insight 字段，兼容旧数据 content */
		insight() {
			return this.message.insight || this.message.content || ''
		},
		/** 是否有图表数据 */
		hasCharts() {
			return this.message.role === 'assistant' && this.charts.length > 0
		},
		/** 是否有建议 */
		hasSuggestion() {
			return this.message.role === 'assistant' && !!this.message.suggestion
		},
		/** 是否展示底部操作栏（点赞/反馈） */
		showActions() {
			return this.message.role === 'assistant' && this.message.status === 'done'
		},
	},
	methods: {
		/** 点赞切换 */
		onLike() {
			this.liked = !this.liked
		},
		/** 反馈切换 */
		onFeedback() {
			this.feedbacked = !this.feedbacked
		},
		/** 建议文本展开/收起（小程序 text 不支持 line-clamp，用 view + 点击切换） */
		onToggleSuggestion() {
			this.sugExpanded = !this.sugExpanded
		},
		/** 展示存图成功胶囊提示 */
		_showSaveToast() {
			if (this._saveToastTimer) {
				clearTimeout(this._saveToastTimer)
				this._saveToastTimer = null
			}
			this.saveToastVisible = true
			this._saveToastTimer = setTimeout(() => {
				this.saveToastVisible = false
				this._saveToastTimer = null
			}, 2000)
		},
		/** 存图：将图表区域截图保存到相册 */
		async onSaveImage() {
			// 使用 uni.createSelectorQuery 获取图表容器 → canvas.toTempFilePath → wx.saveImageToPhotosAlbum
			const that = this
			// #ifdef MP-WEIXIN
			const query = uni.createSelectorQuery().in(that)
			query
				.select('.chat-msg-charts')
				.boundingClientRect()
				.exec(res => {
					if (!res || !res[0]) {
						uni.showToast({ title: '暂无可保存的图表', icon: 'none' })
						return
					}
					// 对整个 scroll-view 截图方案：使用 wx.createSelectorQuery 选取气泡容器再画到 canvas
					that._captureChatBubble()
				})
			// #endif
			// #ifndef MP-WEIXIN
			uni.showToast({ title: '存图功能仅支持微信小程序', icon: 'none' })
			// #endif
		},

		/** 保存 PDF：调用后端 Puppeteer 渲染 PDF */
		async onSavePdf() {
			const sessionId = this.message.sessionId
			try {
				uni.showLoading({ title: '生成报告中...' })
				await tui.requestPost('ai/report/pdf', {
					sessionId,
					msgId: this.message.id,
				})
				uni.hideLoading()
				uni.showToast({ title: '报告生成成功', icon: 'success' })
			} catch {
				uni.hideLoading()
				uni.showToast({ title: '生成失败，请稍后重试', icon: 'none' })
			}
		},

		/** 截图气泡容器并保存到相册 — 路径 A（存图） */
		_captureChatBubble() {
			// #ifdef MP-WEIXIN
			const that = this
			// 取 ChatChart 子组件中第一个 canvas 类型的图表截图
			const chartRefs = that.$children?.filter(c => c.$options.name === 'ChatChart') || []
			if (chartRefs.length === 0) {
				uni.showToast({ title: '表格暂不支持存图', icon: 'none' })
				return
			}
			const firstCanvas = chartRefs[0]
			if (firstCanvas.canvasId) {
				const ctx = uni.createCanvasContext(firstCanvas.canvasId, that)
				ctx.draw(false, () => {
					setTimeout(() => {
						uni.canvasToTempFilePath(
							{
								canvasId: firstCanvas.canvasId,
								success: res => {
									uni.saveImageToPhotosAlbum({
										filePath: res.tempFilePath,
										success: () => {
											// 胶囊样式提示：已保存图片到相册
											that._showSaveToast()
										},
										fail: err => {
											if (err.errMsg?.includes('auth deny')) {
												uni.showToast({ title: '请授权相册权限', icon: 'none' })
											} else {
												uni.showToast({ title: '保存失败', icon: 'none' })
											}
										},
									})
								},
								fail: () => {
									uni.showToast({ title: '截图失败', icon: 'none' })
								},
							},
							that
						)
					}, 300)
				})
			}
			// #endif
		},
	},
}
</script>

<style lang="scss" scoped>
/* ===== 消息行布局 ===== */
.chat-msg {
	display: flex;
	flex-direction: row;
	align-items: flex-start;
	padding: 16rpx 32rpx;
	box-sizing: border-box;
}

/* 用户消息：靠右 */
.chat-msg--user {
	flex-direction: row-reverse;
}

/* AI 消息：靠左 */
.chat-msg--ai {
	flex-direction: row;
}

/* ===== 气泡 ===== */
.chat-msg-bubble {
	max-width: 540rpx;
	padding: 16rpx 24rpx;
	border-radius: 24rpx;
	box-sizing: border-box;
	position: relative;
}

/* AI 气泡：白色 */
.chat-msg--ai .chat-msg-bubble {
	background: #ffffff;
	border-top-left-radius: 8rpx;
}

/* AI 消息带图表时放大气泡宽度，溢出裁剪防止 canvas/表格越界 */
.chat-msg--ai .chat-bubble-has-chart {
	max-width: 640rpx;
	overflow: hidden;
}

/* 用户气泡：绿色 #00C684（对齐 Pixso 设计稿） */
.chat-msg--user .chat-msg-bubble {
	background: #00c684;
	border-top-right-radius: 8rpx;
}

/* ===== 文本 ===== */
.chat-msg-text {
	font-family: 'PingFang SC', sans-serif;
	font-size: 28rpx;
	font-weight: 400;
	line-height: 40rpx;
	word-break: break-all;
}

.chat-msg--ai .chat-msg-text {
	color: #333333;
}

.chat-msg--user .chat-msg-text {
	color: #ffffff;
}

/* ===== 流式输出光标 ===== */
.chat-msg-cursor {
	font-family: 'PingFang SC', sans-serif;
	font-size: 28rpx;
	color: #333333;
	animation: cursor-blink 0.8s step-end infinite;
}

@keyframes cursor-blink {
	0%,
	100% {
		opacity: 1;
	}
	50% {
		opacity: 0;
	}
}

/* ===== 错误态 ===== */
.chat-msg-error {
	display: flex;
	flex-direction: row;
	align-items: center;
	margin-top: 8rpx;
}

.chat-msg-error-icon {
	width: 32rpx;
	height: 32rpx;
	border-radius: 50%;
	background: #ff5a5f;
	color: #fff;
	font-size: 20rpx;
	font-weight: 700;
	text-align: center;
	line-height: 32rpx;
	margin-right: 8rpx;
	flex-shrink: 0;
}

.chat-msg-error-text {
	font-size: 24rpx;
	color: #ff5a5f;
	text-decoration: underline;
}

/* ===== 图表区域 ===== */
.chat-msg-charts {
	width: 100%;
	display: flex;
	flex-direction: column;
	gap: 20rpx;
	margin-top: 16rpx;
}

/* ===== 建议区域（横向：绿图标 + 分隔线 + 文案） ===== */
.chat-msg-suggestion {
	width: 100%;
	margin-top: 16rpx;
	padding: 16rpx 20rpx;
	border-radius: 12rpx;
	background: rgba(247, 248, 249, 1);
	box-sizing: border-box;
	display: flex;
	flex-direction: row;
	align-items: center;
}

/* 左侧绿色聊天气泡图标 */
.chat-sug-icon {
	width: 64rpx;
	height: 64rpx;
	border-radius: 16rpx;
	background: linear-gradient(135deg, rgba(10, 179, 122, 1) 0%, rgba(85, 206, 144, 1) 100%);
	display: flex;
	align-items: center;
	justify-content: center;
	flex-shrink: 0;
	position: relative;
}

.chat-sug-image {
	width: 32rpx;
	height: 32rpx;
}

/* 中间青色分隔线 */
.chat-sug-divider {
	width: 2rpx;
	height: 24rpx;
	margin: 0 16rpx;
	background: linear-gradient(135deg, rgba(10, 179, 122, 1) 0%, rgba(85, 206, 144, 1) 100%);
	flex-shrink: 0;
}

/* 右侧建议文案（默认最多两行，点击展开） */
.chat-sug-text {
	flex: 1;
	min-width: 0;
	font-size: 24rpx;
	font-weight: 400;
	color: #333333;
	line-height: 36rpx;
	word-break: break-all;
	overflow: hidden;
	text-overflow: ellipsis;
	display: -webkit-box;
	-webkit-box-orient: vertical;
	-webkit-line-clamp: 2;
}

/* 展开态：取消行数限制 */
.chat-sug-text--expand {
	overflow: visible;
	text-overflow: clip;
	display: block;
	-webkit-line-clamp: unset;
}

/* ===== 存图 / 存 PDF 按钮栏 ===== */
.chat-msg-actions-bar {
	display: flex;
	flex-direction: row;
	align-items: center;
	margin-top: 16rpx;
	gap: 24rpx;
}

.chat-action-btn {
	display: flex;
	flex-direction: row;
	align-items: center;
	justify-content: center;
	height: 64rpx;
	padding: 0 24rpx;
	border-radius: 12rpx;
	// background: rgba(0, 198, 132, 0.08);
	border: 2rpx solid rgba(235, 238, 241, 1);
	gap: 8rpx;
}

.chat-action-icon {
	width: 32rpx;
	height: 32rpx;
	flex-shrink: 0;
}

.chat-action-label {
	font-size: 24rpx;
	color: #666666;
	font-weight: 500;
}

/* ===== 底部点赞 / 反馈（气泡内部右下角）===== */
.chat-msg-bottom-actions {
	display: flex;
	flex-direction: row;
	align-items: center;
	justify-content: flex-end;
	margin-top: 20rpx;
	gap: 16rpx;
}

.chat-ba-btn {
	width: 48rpx;
	height: 48rpx;
	display: flex;
	align-items: center;
	justify-content: center;
}

.chat-ba-icon {
	width: 36rpx;
	height: 36rpx;
	flex-shrink: 0;
}

/* ===== 存图成功胶囊提示（对齐设计稿） ===== */
.chat-save-toast {
	position: fixed;
	left: 50%;
	top: 50%;
	transform: translate(-50%, -50%);
	z-index: 9999;
	padding: 18rpx 40rpx;
	border-radius: 999rpx;
	background: rgba(0, 0, 0, 0.75);
	box-sizing: border-box;
	pointer-events: none;
}

.chat-save-toast-text {
	font-family: 'PingFang SC', sans-serif;
	font-size: 28rpx;
	font-weight: 400;
	line-height: 40rpx;
	color: #ffffff;
	white-space: nowrap;
}
</style>
