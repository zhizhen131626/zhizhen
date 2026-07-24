<template>
	<view class="ai-page">
		<!-- ===== 背景层 ===== -->
		<view class="ai-bg-layer">
			<image class="ai-bg-img" src="/static/ai/bg.png" mode="scaleToFill" />
		</view>

		<!-- ===== 状态栏占位 ===== -->
		<view class="ai-status-bar" :style="{ height: statusBarHeight + 'px' }" />

		<!-- ===== 顶部导航栏 ===== -->
		<view class="ai-nav">
			<!-- 返回按钮 16x16 @x16 y65 -->
			<view class="ai-back-btn" @click="onBack">
				<view class="ai-back-arrow" />
			</view>
			<!-- 标题 "AI小助手" 16px Medium #000, 居中 -->
			<text class="ai-nav-title">{{ $t('ai.title') }}</text>
		</view>

		<!-- ===== 主体内容区 ===== -->
		<view class="ai-body">
			<!-- ===== 问候语 ===== -->
			<text class="ai-greeting">{{ $t('ai.greeting') }}</text>

			<!-- ===== 功能描述 ===== -->
			<view class="ai-desc">
				<text class="ai-desc-line">{{ $t('ai.desc1') }}</text>
				<text class="ai-desc-line">{{ $t('ai.desc2') }}</text>
			</view>

			<!-- ===== 「您可能想问」推荐卡片 ===== -->
			<view class="ai-ask-card">
				<!-- 标题 "您可能想问" 14px Medium #333 -->
				<text class="ai-ask-title">{{ $t('ai.youMayAsk') }}</text>

				<!-- 问题列表（超出可滚动） -->
				<scroll-view class="ai-ask-scroll" scroll-y :enhanced="true" :show-scrollbar="false">
					<view class="ai-ask-list">
						<view
							v-for="(q, idx) in suggestQuestions"
							:key="idx"
							class="ai-ask-item"
							@click="onAskQuestion(q)"
						>
							<!-- 左侧 # 图标 14x14 -->
							<image class="ai-ask-hash-img" src="/static/ai/icon_ticp.png" mode="aspectFit" />
							<!-- 问题文本 -->
							<text class="ai-ask-text">{{ q }}</text>
							<!-- 右侧箭头 -->
							<view class="ai-ask-arrow" />
						</view>
					</view>
					<!-- 底部留白（避免 input bar 遮挡） -->
					<view class="ai-body-bottom" />
				</scroll-view>
			</view>
		</view>

		<!-- ===== 底部输入栏（固定；录音时仅视觉隐藏，保留触摸节点） ===== -->
		<view class="ai-input-bar" :class="{ 'ai-input-bar-hidden': isRecording }">
			<!-- 语音按钮（文字模式）/ 键盘按钮（语音模式） -->
			<view class="ai-input-voice" @click="onToggleVoiceMode">
				<view class="ai-voice-circle" />
				<image
					v-if="!isVoiceMode"
					class="ai-voice-img"
					src="/static/ai/speek.png"
					mode="aspectFit"
				/>
				<image v-else class="ai-voice-img" src="/static/ai/write.png" mode="aspectFit" />
			</view>
			<!-- 文字模式：输入框 -->
			<input
				v-if="!isVoiceMode"
				v-model="inputText"
				class="ai-input-field"
				:placeholder="$t('ai.inputHint')"
				placeholder-class="ai-input-placeholder"
				:adjust-position="false"
				@confirm="onSend"
			/>
			<!-- 语音模式：「按住说话」按钮（触摸逻辑保持原实现） -->
			<view
				v-else
				class="ai-hold-btn"
				@touchstart="onVoiceTouchStart"
				@touchmove="onVoiceTouchMove"
				@touchend="onVoiceTouchEnd"
			>
				<text class="ai-hold-btn-text">{{ $t('ai.holdToSpeak') }}</text>
			</view>
			<!-- 发送按钮（仅文字模式展示） -->
			<view v-if="!isVoiceMode" class="ai-input-send" @click="onSend">
				<image class="ai-send-img" src="/static/ai/send.png" mode="aspectFit" />
			</view>
		</view>

		<!-- ===== 录音浮动提示层（仅样式层，不拦截触摸） ===== -->
		<view
			v-if="isRecording"
			class="ai-record-overlay"
			:class="{ 'ai-record-cancel': recordState === 'cancel' }"
		>
			<view class="ai-record-panel">
				<!-- 提示文案在上 -->
				<text class="ai-record-tip">
					{{ recordState === 'cancel' ? $t('ai.releaseToCancel') : $t('ai.releaseToSend') }}
				</text>
				<!-- 横向音量波形条 -->
				<view class="ai-record-wave">
					<view
						v-for="(bar, idx) in waveBars"
						:key="idx"
						class="ai-record-bar"
						:style="{
							height: bar.height + 'rpx',
							animationDuration: bar.duration + 's',
							animationDelay: bar.delay + 's',
						}"
					/>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
/**
 * AI 助手首页 — 严格按 Pixso 设计稿 1:1 还原
 * 设计稿: 375×812 (iPhone 13 mini)
 * 换算: 1px = 2rpx
 *
 * 页面结构（从上到下）：
 *   背景层 → 状态栏 → 导航栏 → 主体区(插画+问候+问题卡片) → 底部输入栏
 */
export default {
	data() {
		return {
			inputText: '', // 输入框文本
			isVoiceMode: false, // 是否处于语音输入模式
			isRecording: false, // 是否正在录音中
			recordState: 'send', // 录音状态: 'send' | 'cancel'
			/** 波形条高度/动画参数（模拟声波动效） */
			waveBars: [],
		}
	},
	computed: {
		statusBarHeight() {
			return this.$store.state.statusBar || 0
		},
		/** 推荐问题列表（按设计稿顺序 4 条） */
		suggestQuestions() {
			return [
				this.$t('ai.question1'),
				this.$t('ai.question2'),
				this.$t('ai.question3'),
				this.$t('ai.question3'),
			]
		},
	},
	created() {
		this.initWaveBars()
	},
	methods: {
		/** 初始化约 48 根波形条，高度呈中间高两侧低的包络 */
		initWaveBars() {
			const count = 48
			const bars = []
			for (let i = 0; i < count; i++) {
				// 正弦包络：中间高、两侧低
				const t = i / (count - 1)
				const envelope = Math.sin(t * Math.PI)
				const base = 12 + envelope * 36 // 12~48rpx
				const jitter = ((i * 7) % 11) - 5 // -5~5 轻微抖动
				const height = Math.max(8, Math.round(base + jitter))
				bars.push({
					height,
					duration: 0.6 + ((i * 13) % 10) / 20, // 0.6~1.05s
					delay: ((i * 17) % 20) / 40, // 0~0.475s
				})
			}
			this.waveBars = bars
		},
		/** 返回上一页 */
		onBack() {
			uni.navigateBack({ delta: 1 })
		},
		/** 点击推荐问题 → 填入输入框并发送 */
		onAskQuestion(q) {
			this.inputText = q
			this.onSend()
		},
		/** 切换语音 / 键盘输入模式 */
		onToggleVoiceMode() {
			this.isVoiceMode = !this.isVoiceMode
		},
		/**
		 * 语音按住开始
		 * 进入录音状态，显示 "松手发送，上移取消"
		 */
		onVoiceTouchStart(e) {
			this.isRecording = true
			this.recordState = 'send'
			/** 记录触摸起始点，用于判断上移偏移量 */
			this._voiceStartY = e.touches[0].clientY
			// TODO: 开始录音
		},
		/**
		 * 语音按住移动
		 * 手指上移超过阈值（80px）→ 切换为 "松手取消"
		 */
		onVoiceTouchMove(e) {
			if (!this.isRecording) return
			const touchY = e.touches[0].clientY
			const dy = this._voiceStartY - touchY
			// 上移超过 80px 判定为取消意图
			this.recordState = dy > 80 ? 'cancel' : 'send'
		},
		/**
		 * 语音按住结束
		 * send → 发送录音结果
		 * cancel → 取消录音
		 */
		onVoiceTouchEnd() {
			if (!this.isRecording) return
			if (this.recordState === 'send') {
				// TODO: 停止录音并发送语音识别结果
				uni.showToast({
					title: this.$t('all.developing'),
					icon: 'none',
					duration: 1500,
				})
			}
			// cancel 状态下直接关闭，不做任何处理
			this.isRecording = false
			this.recordState = 'send'
		},
		/** 发送消息 */
		onSend() {
			const text = this.inputText.trim()
			if (!text) return
			this.inputText = ''
			// TODO: 接入 AI 对话接口
			uni.showToast({
				title: this.$t('all.developing'),
				icon: 'none',
				duration: 1500,
			})
		},
	},
}
</script>

<style lang="scss" scoped>
/* ============================
   AI 助手首页 - Pixso 设计稿 1:1 还原
   设计尺寸: 375×812
   背景: #E7F7FF (rgba(231,247,255,1))
   ============================ */
$ai-bg: #e7f7ff;
$text-dark: #333333;
$text-hint: #999999;
$card-shadow: rgba(0, 110, 175, 0.1);
$send-blue: #4ea5f1;
$wave-green: #52d273;
$wave-red: #ff5a5f;
$record-green-bg: rgba(180, 236, 200, 0.55);
$record-red-bg: rgba(255, 196, 186, 0.55);

.ai-page {
	width: 750rpx;
	height: 100vh;
	background: $ai-bg;
	position: relative;
	overflow: hidden;
	display: flex;
	flex-direction: column;
}

/* ===== 背景层：底色 + 装饰图 + 渐变遮罩 ===== */
.ai-bg-layer {
	position: absolute;
	top: 0;
	left: 0;
	width: 750rpx;
	height: 1624rpx; /* 812px */
	z-index: 0;
	overflow: hidden;
}

/* 背景装饰图：占满整个页面 */
.ai-bg-img {
	position: absolute;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
}

/* 渐变遮罩：68% opaque→81.5% transparent，从下到上
   Pixso: LINEAR gradient, stops=[68%: #F6FAFA, 81.5%: transparent]
   transform=[0,1,0,-1,0,1] 翻转 Y → gradient from bottom to top */
.ai-bg-mask {
	position: absolute;
	top: 0;
	left: 0;
	width: 750rpx;
	height: 1624rpx; /* 812px */
	background: linear-gradient(
		to top,
		rgba(246, 250, 250, 1) 0%,
		rgba(246, 250, 250, 1) 68%,
		rgba(246, 250, 250, 0) 81.5%,
		rgba(246, 250, 250, 0) 100%
	);
}

/* ===== 状态栏 ===== */
.ai-status-bar {
	width: 750rpx;
	flex-shrink: 0;
	position: relative;
	z-index: 5;
}

/* ===== 导航栏 =====
   高度 ≈ 44px = 88rpx，含返回按钮 + 标题 + Pill */
.ai-nav {
	width: 750rpx;
	height: 88rpx;
	position: relative;
	z-index: 5;
	flex-shrink: 0;
	display: flex;
	flex-direction: row;
	align-items: center;
	justify-content: center;
}

/* 返回按钮：16x16 @x16 y65（nav 内 y=65-44-状态栏 ≈ 相对居中） */
.ai-back-btn {
	position: absolute;
	left: 32rpx; /* 16px */
	top: 50%;
	transform: translateY(-50%);
	width: 48rpx;
	height: 48rpx;
	display: flex;
	align-items: center;
	justify-content: center;
}

/* 左箭头：纯 CSS 实现 16x16 */
.ai-back-arrow {
	width: 16rpx;
	height: 16rpx;
	border-left: 3rpx solid #333;
	border-bottom: 3rpx solid #333;
	transform: rotate(45deg);
	position: relative;
	left: 4rpx; /* 视觉居中微调 */
}

/* 标题 "AI小助手" 16px Medium #000 */
.ai-nav-title {
	font-family: 'PingFang SC', sans-serif;
	font-size: 32rpx; /* 16px */
	font-weight: 500;
	color: #000;
	letter-spacing: 0;
}

/* ===== 主体内容区 ===== */
.ai-body {
	flex: 1;
	position: relative;
	z-index: 3;
	width: 750rpx;
	display: flex;
	flex-direction: column;
	overflow: hidden;
}

/* ===== 推荐问题滚动区 ===== */
.ai-ask-scroll {
	flex: 1;
	min-height: 0; /* flex 子元素允许收缩 */
}

/* ===== 问候语 =====
   "Hello，老师好" 20px Bold, Alimama FangYuanTi VF, #000
   距离页面顶部 294px（588rpx），减去 header 约 88px（176rpx）→ margin-top: 412rpx */
.ai-greeting {
	display: block;
	width: 750rpx;
	padding: 0 32rpx;
	box-sizing: border-box;
	font-family: 'Alimama FangYuanTi VF', 'PingFang SC', sans-serif;
	font-size: 40rpx; /* 20px */
	font-weight: 700;
	color: #000;
	text-align: center;
	line-height: 48rpx;
	margin-top: 412rpx;
}

/* ===== 功能描述 =====
   两行文本，12px Regular PingFang SC #999, line-height 20px, 居中 */
.ai-desc {
	width: 456rpx; /* 228px */
	margin: 24rpx auto 0; /* greeting y294→desc y330, gap=36px → 72rpx */
	display: flex;
	flex-direction: column;
	align-items: center;
}

.ai-desc-line {
	font-family: 'PingFang SC', sans-serif;
	font-size: 24rpx; /* 12px */
	font-weight: 400;
	color: $text-hint;
	line-height: 40rpx; /* 20px */
	text-align: center;
}

/* ===== 「您可能想问」推荐卡片 =====
   卡片：343x282 @x16 y394, 圆角 20, 毛玻璃背景
   desc bottom=330+40=370, card y=394, gap=24px=48rpx */
.ai-ask-card {
	width: 686rpx; /* 343px */
	margin: 48rpx auto 0; /* gap 24px */
	padding: 32rpx; /* 16px inner padding */
	box-sizing: border-box;
	border-radius: 40rpx; /* 20px */
	border: 2rpx solid rgba(255, 255, 255, 1);
	background: linear-gradient(90deg, rgba(255, 255, 255, 0.6) 0%, rgba(243, 251, 255, 0.6) 100%);
	backdrop-filter: blur(20rpx);
	display: flex;
	flex-direction: column;
	max-height: 564rpx; /* 282px */
	overflow: hidden;
}

/* "您可能想问" 14px Medium #333, letter-spacing 0.5 */
.ai-ask-title {
	font-family: 'PingFang SC';
	font-size: 28rpx; /* 14px */
	font-weight: 500;
	color: $text-dark;
	letter-spacing: 1rpx; /* 0.5px */
	line-height: 40rpx; /* 20px */
	margin-bottom: 24rpx; /* title bottom y36→item top y48: 12px */
}

/* 问题列表 */
.ai-ask-list {
	display: flex;
	flex-direction: column;
	gap: 12rpx; /* item 间距: 6px */
}

/* 每个问题行：311x50, 白底, 圆角 25, 水平布局 */
.ai-ask-item {
	width: 622rpx; /* 311px */
	height: 100rpx; /* 50px */
	background: #fff;
	color: $text-dark;
	font-size: 28rpx; /* 14px */
	border: 2rpx solid #fff;
	border-radius: 50rpx; /* 25px */
	padding: 0 24rpx; /* 左右内边距 */
	box-sizing: border-box;
	display: flex;
	flex-direction: row;
	align-items: center;
}

/* # 图标：14x14 */
.ai-ask-hash-img {
	width: 28rpx; /* 14px */
	height: 28rpx; /* 14px */
	flex-shrink: 0;
	margin-right: 18rpx;
}

/* 问题文本 14px Regular #333 */
.ai-ask-text {
	flex: 1;
	font-family: 'PingFang SC', sans-serif;
	font-size: 28rpx; /* 14px */
	font-weight: 400;
	color: $text-dark;
	line-height: 40rpx; /* 20px */
	overflow: hidden;
	text-overflow: ellipsis;
	white-space: nowrap;
}

/* 右侧箭头：11.6x7.1 纯 CSS */
.ai-ask-arrow {
	width: 14rpx;
	height: 14rpx;
	border-top: 3rpx solid #999;
	border-right: 3rpx solid #999;
	transform: rotate(45deg);
	flex-shrink: 0;
}

/* 底部留白 */
.ai-body-bottom {
	height: 120rpx; /* input bar + 安全区 */
	flex-shrink: 0;
}

/* ===== 底部输入栏（固定） =====
   343x50 @x16 y722, radius 25, shadow */
.ai-input-bar {
	position: absolute;
	bottom: 80rpx; /* 安全区底部 */
	left: 32rpx; /* 16px */
	width: 686rpx; /* 343px */
	height: 100rpx; /* 50px */
	background: #fbfcfd;
	border: 2rpx solid #fff;
	border-radius: 50rpx; /* 25px */
	box-shadow: 0 4rpx 20rpx rgba(0, 110, 175, 0.1);
	display: flex;
	flex-direction: row;
	align-items: center;
	padding-left: 10rpx; /* 语音按钮 x=5 */
	padding-right: 8rpx; /* 发送按钮 x=297, bar_end=343, gap=343-297-42=4px */
	box-sizing: border-box;
	z-index: 10;
}

/* 录音中仅隐藏外观，节点保留以确保 touchmove/touchend 不中断 */
.ai-input-bar-hidden {
	opacity: 0;
}

/* 语音按钮：40x40 圆底 + 麦克风图标 */
.ai-input-voice {
	position: relative;
	width: 80rpx; /* 40px */
	height: 80rpx; /* 40px */
	flex-shrink: 0;
}

.ai-voice-circle {
	position: absolute;
	left: 0;
	top: 0;
	width: 80rpx;
	height: 80rpx;
	background: #f2f6f9;
	border-radius: 50%;
}

.ai-voice-img {
	position: absolute;
	left: 50%;
	top: 50%;
	transform: translate(-50%, -50%);
	z-index: 1;
	width: 48rpx; /* 24px */
	height: 48rpx; /* 24px */
}

/* 文本输入域 */
.ai-input-field {
	flex: 1;
	height: 80rpx;
	padding: 0 20rpx;
	font-family: 'PingFang SC', sans-serif;
	font-size: 28rpx; /* 14px */
	font-weight: 400;
	color: $text-dark;
}

.ai-input-placeholder {
	font-family: 'PingFang SC', sans-serif;
	font-size: 28rpx;
	font-weight: 400;
	color: $text-hint;
}

/* 发送按钮：42x42, 蓝色渐变圆底 + 纸飞机 */
.ai-input-send {
	position: relative;
	width: 84rpx; /* 42px */
	height: 84rpx; /* 42px */
	flex-shrink: 0;
}

.ai-send-img {
	width: 84rpx; /* 42px */
	height: 84rpx; /* 42px */
}

/* ===== 语音模式：「按住说话」按钮 ===== */
.ai-hold-btn {
	flex: 1;
	height: 80rpx;
	display: flex;
	flex-direction: row;
	align-items: center;
	justify-content: center;
}

.ai-hold-btn-text {
	font-family: 'PingFang SC', sans-serif;
	font-size: 32rpx; /* 16px */
	font-weight: 500;
	color: $text-dark;
	line-height: 40rpx; /* 20px */
}

/* ===== 录音底部提示层 =====
   设计稿：全宽底部渐变区域 + 上方文案 + 下方横向波形
   send → 薄荷绿底 + 绿色波形
   cancel → 浅粉红底 + 红色波形 */
.ai-record-overlay {
	position: fixed;
	left: 0;
	right: 0;
	bottom: 0;
	z-index: 9; /* 低于输入栏，避免截获按住说话的触摸事件 */
	width: 750rpx;
	height: 360rpx; /* 底部录音区域高度 */
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: flex-end;
	/* 自下而上：薄荷绿 → 透明 */
	background: linear-gradient(
		to top,
		rgba(170, 230, 195, 0.72) 0%,
		rgba(200, 240, 215, 0.45) 35%,
		rgba(230, 248, 235, 0.18) 65%,
		rgba(255, 255, 255, 0) 100%
	);
	padding-bottom: calc(40rpx + env(safe-area-inset-bottom));
	box-sizing: border-box;
	pointer-events: none; /* 纯展示层，触摸继续落在 hold 按钮上 */
	transition: background 0.2s ease;
}

/* cancel：浅红/桃色渐变 */
.ai-record-overlay.ai-record-cancel {
	background: linear-gradient(
		to top,
		rgba(255, 180, 170, 0.75) 0%,
		rgba(255, 205, 195, 0.48) 35%,
		rgba(255, 230, 225, 0.2) 65%,
		rgba(255, 255, 255, 0) 100%
	);
}

.ai-record-panel {
	width: 750rpx;
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: flex-end;
	padding: 0 48rpx 24rpx;
	box-sizing: border-box;
}

/* 提示文案：12~14px 中灰，居中，位于波形上方 */
.ai-record-tip {
	font-family: 'PingFang SC', sans-serif;
	font-size: 24rpx; /* 12px */
	font-weight: 400;
	color: #999999;
	line-height: 36rpx;
	text-align: center;
	margin-bottom: 28rpx;
}

/* 横向波形容器 */
.ai-record-wave {
	width: 654rpx; /* ~327px */
	height: 64rpx;
	display: flex;
	flex-direction: row;
	align-items: center;
	justify-content: center;
	gap: 6rpx;
}

/* 单根波形条 */
.ai-record-bar {
	width: 6rpx;
	min-height: 8rpx;
	border-radius: 6rpx;
	background: $wave-green;
	flex-shrink: 0;
	transform-origin: center center;
	animation-name: ai-wave-pulse;
	animation-timing-function: ease-in-out;
	animation-iteration-count: infinite;
	animation-direction: alternate;
	transition: background-color 0.2s ease;
}

.ai-record-cancel .ai-record-bar {
	background: $wave-red;
}

@keyframes ai-wave-pulse {
	0% {
		transform: scaleY(0.45);
		opacity: 0.75;
	}
	100% {
		transform: scaleY(1);
		opacity: 1;
	}
}
</style>
