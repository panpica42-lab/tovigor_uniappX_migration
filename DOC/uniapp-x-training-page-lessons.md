# uni-app X 训练页迁移经验记录

## 背景

本记录来自 `pages/partTraining` 训练流程页面迁移排查，主要涉及：

- `warm-up-page.uvue`
- `formal-training.uvue`
- `cool-down-page.uvue`
- `components/coach/coach-box-entry.uvue`

这些页面都包含全屏视频背景、透明背景点击层、AI 教练气泡、控制面板、训练阶段跳转等原生 App 交互。uni-app X / UTS 在这些场景里和普通 Vue / H5 行为差异很大，不能直接照搬 WebView 经验。

## 关键结论

### 1. 背景视频不能只依赖 autoplay

App 端 `video autoplay loop` 不一定稳定，表现可能是黑屏、停在首帧或循环不连续。

推荐做法：

- 给视频稳定 `id`
- 保留 `autoplay` / `loop` / `muted`
- 页面挂载后延迟调用 `uni.createVideoContext(id).play()`
- `@ended` 时手动 `seek(0)` 再 `play()`
- `@error` 打完整 `event.detail`

示例：

```uts
function playBackgroundVideo() {
	const videoContext = uni.createVideoContext('coolDownVideo')
	if (videoContext != null) {
		videoContext.play()
	}
}

function restartBackgroundVideo() {
	const videoContext = uni.createVideoContext('coolDownVideo')
	if (videoContext != null) {
		videoContext.seek(0)
		videoContext.play()
	}
}
```

注意：`uni.createVideoContext()` 返回值需要判空后再调用方法。

### 2. UTS 不要依赖函数提升

在 `<script setup lang="uts">` 中，`computed`、顶层常量、事件回调如果引用 helper 函数，helper 函数必须先声明。

错误风险：

```uts
const setText = computed<string>(() => pad2(currentSet.value))

function pad2(value: number): string {
	return value.toString()
}
```

推荐：

```uts
function pad2(value: number): string {
	return value.toString()
}

const setText = computed<string>(() => pad2(currentSet.value))
```

### 3. 全屏透明点击层会吃掉上层组件事件

训练页为了“点击空白处唤起控制面板”，通常有一层透明背景层。这个层如果铺满全屏，即使视觉上在下方，也可能导致 AI 教练气泡区域无法收到触摸事件。

有效处理方式：

- 背景点击层不要覆盖教练气泡、顶部导航、底部控制区
- 用 `top` / `bottom` 限制命中范围
- 必要时给目标交互区增加页面级命中层

示例：

```css
.tap-layer {
	position: absolute;
	left: 0;
	right: 0;
	top: 320rpx;
	bottom: 340rpx;
	width: 750rpx;
}
```

formal 页同样避免使用不可用的 `uni.upx2px`，直接使用 rpx 定位。

### 4. 自定义组件点击不可靠时，用页面级命中层兜底

AI 教练组件内部 `@click` / `@tap` 在训练页视频背景和透明层组合下可能完全收不到事件。控制台无日志时，说明事件没有进入组件。

最终可行方案是在页面上放一个透明命中层，覆盖教练气泡可点击区域，由页面直接打开 `CoachDetailModal`。

示例：

```vue
<view class="coach-hit-layer" @tap="openCoachModal" @click="openCoachModal"></view>
```

```css
.coach-hit-layer {
	position: absolute;
	left: 38rpx;
	top: 162rpx;
	width: 440rpx;
	height: 170rpx;
	z-index: 420;
	background-color: rgba(0,0,0,0.001);
}
```

页面级打开逻辑：

```uts
function openCoachModal() {
	selectedCoach.value = getSelectedCoach()
	coachModalVisible.value = true
}
```

这比继续调整子组件 z-index 更可靠。

### 5. 控制面板触摸结构要简单

控制面板推荐保持简单层级：

- overlay
- mask
- buttons
- exit button

避免在中间再加一个全屏 content 层并绑定点击事件，否则可能影响按钮点击。

推荐：

```vue
<view v-if="panelVisible" class="control-panel-overlay">
	<view class="control-panel-mask" @click="hidePanel"></view>
	<view class="control-buttons">
		...
	</view>
	<view class="exit-btn" @click="exitTraining">
		<text>退出训练</text>
	</view>
</view>
```

按钮层和退出按钮需要压在 mask 上方：

```css
.control-panel-overlay { z-index: 500; }
.control-panel-mask { z-index: 0; }
.control-buttons { position: relative; z-index: 2; }
.exit-btn { position: absolute; z-index: 2; }
```

### 6. 老项目 nvue 是很好的参照，但不能机械复制

老项目 `nvue` 页面有几个值得保留的经验：

- 视频使用页面内上下文管理
- 背景点击由单独透明层处理
- 教练弹窗打开时暂停计时
- 控制面板 mask 和按钮分层
- 页面尺寸使用系统窗口尺寸

但迁移到 uni-app X 时，需要注意：

- UTS 类型和声明顺序更严格
- 部分 uni API 可能不存在或类型不同
- 自定义组件事件和原生层命中可能不如 nvue 稳定
- 不要直接引入老项目 JS mixin，要优先保留 X 项目已有 `serialRuntime.uts`

## 排查方法

遇到“点了没反应”时，不要先猜弹窗坏了，按下面顺序查：

1. 在点击函数第一行加 `console.log`
2. 如果无日志，说明事件没进入该组件或函数
3. 检查全屏透明层、video、mask、overlay 的命中范围
4. 将透明点击层改成避开目标区域
5. 如果自定义组件仍收不到事件，用页面级透明命中层兜底
6. 如果有日志但无弹窗，再查弹窗 `visible`、层级和 props

遇到“视频黑屏”时：

1. 确认资源路径存在
2. 给 video 加 `id`
3. 使用 `createVideoContext(id).play()`
4. `@ended` 手动续播
5. `@error` 打印 `event.detail`
6. 排查 Android 基座 ABI 和 video so 支持

## 后续建议

- 新增训练页时，直接复用这套视频与背景命中层结构。
- 不要让全屏透明层覆盖教练气泡、顶部导航、底部控制区。
- 在 `tovigor_yei_unIappX/AGENTS.md` 中保留并继续补充 UTS / uni-app X 特有规则。
- UI 可先照老项目 nvue 对齐，但交互命中和生命周期以真机表现为准。
