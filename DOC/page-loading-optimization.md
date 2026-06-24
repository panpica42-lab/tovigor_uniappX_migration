# 页面白屏与加载优化结论

## 背景

首页点击「部位训练」后进入：

```text
pages/partTraining/part-training.uvue
```

用户感知问题：跳转后有明显整页白屏。

已确认：这不是课程接口慢，也不是课程图片加载慢。主要发生在 `navigateTo` 创建新页面、创建原生视图、排版、渲染、提交首帧的阶段。

## 诊断日志结论

实测三次进入：

```text
第一次：onReady 约 1600ms
第二次：onReady 约 970ms
第三次：onReady 约 820ms
继续多次进入后：稳定约 550-600ms
```

关键日志特征：

```text
setup/onLoad/onMounted 基本在 15-60ms 内完成
onShow 约 270-350ms
onReady 约 550-970ms
```

说明：

- JS 初始化不是主瓶颈。
- 课程数据不是主瓶颈。
- 图片加载不是主瓶颈。
- 慢点主要在页面路由、原生视图创建、排版渲染和首帧提交。

## 为什么越进越快

这是冷到热的过程：

- 页面模块被缓存。
- 组件和样式初始化路径变热。
- 图片解码和纹理缓存命中。
- Android/运行时调度进入活跃状态。
- 原生视图创建和渲染路径被预热。

所以第一次慢，后面逐步稳定到 550-600ms 是正常现象。

## 已废弃方案

### 首页 loading 遮罩

已废弃。

原因：loading 在首页显示，`navigateTo` 后首页被新页面覆盖。目标页首帧没出来时，loading 也看不到，结果变成：

```text
loading + 白屏
```

这不是优化，是增加等待感。

### 目标页骨架屏

已废弃。

原因：骨架屏只有目标页开始渲染后才可能显示，而当前白屏主要发生在目标页首帧之前。并且骨架屏存在时间太短，肉眼基本不可见。

## 当前保留的低风险优化

### 1. 关闭跳转动画

位置：

```text
pages/index/index.uvue
```

```ts
uni.navigateTo({
	url: '/pages/partTraining/part-training?traceTs=' + traceTs.toString(),
	animationType: 'none',
	animationDuration: 0
})
```

目的：减少跳转动画阶段的空白感。

### 2. 设置目标页背景色

位置：

```text
pages.json
```

```json
{
	"path": "pages/partTraining/part-training",
	"style": {
		"navigationStyle": "custom",
		"backgroundColor": "#F5F5F5",
		"backgroundColorTop": "#F5F5F5",
		"backgroundColorBottom": "#F5F5F5",
		"disableScroll": true
	}
}
```

目的：减少新页面容器默认白底暴露。

### 3. 保留诊断日志

位置：

```text
pages/index/index.uvue
pages/partTraining/part-training.uvue
```

日志前缀：

```text
[PartTrainingTrace]
```

用途：继续观察 `onShow`、`onReady`、页面统计耗时。

## 有效手段分级

| 手段 | 减少白屏 | 缩短总流程 | 当前建议 |
|---|---:|---:|---|
| 首页缓存课程图片 | 弱 | 弱 | 暂不优先 |
| 页面预加载 | 中 | 中 | 迁移完成后评估 |
| 首屏少挂组件 | 中/强 | 中 | 可作为低风险优化 |
| 轻首屏，后补筛选器 | 强 | 强 | 后续性能专项考虑 |
| 首页内状态切换，不走路由 | 最强 | 最强 | 迁移完成后再评估 |

## 当前决策

迁移阶段不改变页面模型。

原因：

- 当前 uni-app X 迁移还没完成。
- 过早把页面路由改成首页内状态切换，会偏离老项目结构。
- 老项目仍需要作为功能、流程、页面边界参考。

当前策略：

```text
先完成迁移
保留页面路由结构
只做低风险白屏缓解
迁移完成后再针对首页 -> 部位训练入口做性能专项
```

## 后续性能专项方向

迁移完成后重点评估：

- 「部位训练」是否应作为 App 主功能区，而不是普通二级页面。
- 是否将 `part-training` 主体抽成组件，保留独立页面，同时支持首页内快速切换。
- 是否延迟挂载侧栏筛选器、AI 弹窗等非首屏必需内容。
- 是否对首页到部位训练做页面预热。

目标：

```text
减少明显白屏
热态进入尽量接近 400ms
不破坏迁移阶段对老项目的参考关系
```
