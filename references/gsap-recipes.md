# GSAP 动效配方

以下参数来自已验证的 1080p / 30fps 精选版。它们是可靠基准，不是必须逐像素复制的模板。代码是可组合的配方；挂载、注册和媒体播放使用目标 HyperFrames 工程的适配接口。

## 时间线结构与初始状态

- 每个场景一个暂停的子时间线，注册到 window.__timelines[compositionId]。
- 根时间线控制字幕、片头和镜头交接；子时间线只控制本场景内部元素，不与根时间线争写同一元素的同一属性。
- 同一 DOM 中可能同时存在多个 `.title`、`.visual`。必须用元素引用或 `sceneRoot.querySelectorAll()` 限定范围，不能用全局同名选择器。
- 创建时间线前同步创建 DOM、准备字体与素材、建立确定的初始状态；不要在 onStart/onComplete 回调里挂载、换字幕文字或复位。直接 seek 可能跳过回调。
- 多个 fromTo 连续修改同一属性时，后续 tween 使用 immediateRender: false。不得用真实时钟驱动画面。

### 场景壳与运动层

每个 sceneRoot 是覆盖全画幅、不透明、无位移无缩放的场景壳，包含自身完整背景；`.scene-motion` 是内部内容运动层。合成容器裁切溢出，字幕和章节角标是场景壳之外的兄弟层。普通推镜可移动进入场景壳，但退出场景壳必须保持全幅覆盖；章节融合只淡入进入场景壳、移动其内部内容。不能把“容器有底色”当作内容连续性的保证。

静态布局示意：

~~~css
.composition { position: relative; overflow: hidden; }
.scene { position: absolute; inset: 0; overflow: hidden; opacity: 0; }
.scene-motion { position: absolute; inset: 0; transform-origin: 50% 50%; }
/* 各 .scene 自己指定不透明背景。字幕层位于场景外，z-index: 10000。 */
~~~

以下初始化每次挂载执行一次，不要每帧调用。DOM 被复用时，先销毁旧时间线，再初始化并重建；不允许带着上次退出状态重新入场。

~~~js
function initializeScene(sceneRoot, isFirst = false) {
  gsap.set(sceneRoot, {
    x: 0, y: 0, xPercent: 0, yPercent: 0, scale: 1,
    opacity: isFirst ? 1 : 0, visibility: 'inherit', zIndex: isFirst ? 1 : 0
  });
  gsap.set(sceneRoot.querySelectorAll('.scene-motion'), {
    x: 0, y: 0, scale: 1, opacity: 1
  });
}
~~~

### 预加载与时间映射

在切点 T 前约 0.10 秒挂载进入场景。挂载只负责创建 DOM 和确定初始状态，入场动画仍从 T 开始。若局部时间从挂载时刻计算，延后实际的 `T - mountAt`，而不是无条件延后 0.10 秒；若局部时间以切点为零，不再重复平移。首个正文从 0 秒已挂载。

最小接线示意（`mountScene`、`registerScene` 是项目适配接口，不是 GSAP API）：

~~~js
const PRELOAD = 0.10;
const mountAt = Math.max(0, T - PRELOAD);
const enterOffset = T - mountAt;
let sceneTl;
registerScene(nextScene, {
  start: mountAt,
  end: nextCutTime + exitDuration, // 覆盖本场景及它自己的退出转场
  onMount(root) {
    mountScene(root); // 仅同步创建 DOM 并调用 initializeScene
    sceneTl = buildSceneTimeline(root, enterOffset); // 唯一构建点，局部零点取挂载时刻
  }
});
// buildSceneTimeline 见下文。rootTl 也必须在所需外壳/片头/字幕 DOM 存在后构建。
function renderAt(globalTime) {
  // 仅在注册区间内调用；工程先确保 DOM 已挂载且 onMount 已完成，再 seek。
  rootTl.seek(globalTime, true);
  sceneTl.seek(Math.max(0, globalTime - mountAt), true);
}
~~~

真实工程需让 HyperFrames 的合成时间驱动根时间线和子时间线。嵌入根时间线的子时间线须解除自身 paused；独立注册的暂停时间线由适配器显式 seek，不能同时由两个驱动器推进。不要把内部动画和外壳转场重复添加到两条时间线。至少以正序、倒序、随机顺序 seek T−0.10、T、T+0.20、T+D 及片头，比较同一时间点的状态；也检查 T 小于 0.10 的挂载截零情形。

## 片头：双页打开

本配方是默认必做的片头转场；仅在用户明确指定其他片头时替换。不能只保留文字淡入而省略展开交接。

实施前约定：首个正文场景从 0 秒起已挂载在片头下方；左右页各占半幅并裁切溢出，分别锚定左右边；字幕层高于片头层。`.intro-mark` 是首帧已可见的主题词或主题图形，放在 `.intro-lockup` 内，不能所有文字都从透明状态入场。正文入场与双页展开交叠，口播照常开始，不增加无信息等待。双页承载主题，不能退化成纯色幕布。

~~~js
function buildIntroTimeline(introRoot) {
  const q = (selector) => introRoot.querySelectorAll(selector);
  const tl = gsap.timeline({paused: true});
  gsap.set(introRoot, {autoAlpha: 1});
  gsap.set(q('.intro-mark'), {autoAlpha: 1});
  gsap.set(q('.intro-lockup'), {y: 0, autoAlpha: 1});
  gsap.set(q('.intro-left, .intro-right'), {xPercent: 0});
  tl.fromTo(q('.intro-word'),
    {y: 34, autoAlpha: 0},
    {y: 0, autoAlpha: 1, duration: 0.32, ease: 'power4.out'}, 0);
  tl.fromTo(q('.intro-line'),
    {scaleX: 0.001, autoAlpha: 0},
    {scaleX: 1, autoAlpha: 1, duration: 0.34, ease: 'power3.out'}, 0.10);
  tl.fromTo(q('.intro-kicker'),
    {y: 18, autoAlpha: 0},
    {y: 0, autoAlpha: 1, duration: 0.30, ease: 'power3.out'}, 0.16);
  tl.fromTo(q('.intro-lockup'),
    {y: 0, autoAlpha: 1},
    {y: -24, autoAlpha: 0, duration: 0.28, ease: 'power2.in', immediateRender: false}, 0.38);
  tl.fromTo(q('.intro-left'),
    {xPercent: 0},
    {xPercent: -100, duration: 0.68, ease: 'power3.inOut', immediateRender: false}, 0.30);
  tl.fromTo(q('.intro-right'),
    {xPercent: 0},
    {xPercent: 100, duration: 0.68, ease: 'power3.inOut', immediateRender: false}, 0.30);
  tl.set(introRoot, {autoAlpha: 0}, 0.99);
  return tl;
}
~~~

将片头时间线接入根时间线的 0 秒；首个正文内部进入可从约 0.30 秒开始，与展开同时发生，默认约 1 秒接稳。检查首帧、展开中点、正文落点和落点后一帧；再从正文回 seek 到 0 秒，确认主题和双页恢复且字幕未被遮挡。

## 普通切镜：前后画面重叠推镜

默认预加载 0.10 秒、转场 0.42 秒，缓动 power3.inOut。旧场景外壳保持不透明全幅覆盖，只让它的内部内容轻移与缩放，避免旧场景整体缩小后露出四边。进入场景由离屏推入，完全覆盖后才隐藏旧场景。

~~~js
function addPushTransition(tl, oldScene, nextScene, T) {
  const D = 0.42;
  const mountAt = Math.max(0, T - 0.10);
  const oldMotion = oldScene.querySelectorAll('.scene-motion');
  tl.set(oldScene, {x: 0, y: 0, xPercent: 0, yPercent: 0, scale: 1, opacity: 1, zIndex: 1}, T);
  tl.set(nextScene, {x: 0, y: 0, yPercent: 0, scale: 1, xPercent: 100, opacity: 1, zIndex: 2}, mountAt);
  tl.fromTo(nextScene,
    {xPercent: 100, opacity: 1},
    {xPercent: 0, opacity: 1, duration: D, ease: 'power3.inOut', immediateRender: false}, T);
  tl.fromTo(oldMotion,
    {x: 0, scale: 1},
    {x: -16, scale: 0.988, duration: D, ease: 'power3.inOut', immediateRender: false}, T);
  tl.set(oldScene, {opacity: 0, zIndex: 0}, T + D);
}
~~~

左右方向可按阅读流交替。纵向转场只在层级推进、步骤升级或章节前使用，避免每镜随机换方向。变体也必须保留全幅不透明底层，不能直接缩小退出外壳。

关键不变量：T 到 T+D 的中间帧有新旧内容连续交接；nextScene 已预加载，T 时仍离屏，不要求在端点同时看见两者；T+D 后无旧画面残留。

## 章节转场

新场景在固定全幅外壳上淡入；旧场景外壳保持 opacity: 1、无缩放位移，直到新场景达到 opacity: 1 后才隐藏。位移和缩放只作用于 `.scene-motion`。这样即使新层半透明，底层仍是完整旧场景，而不是合成容器底色。不得让两个全屏壳同时淡到 0.5。

~~~js
function addChapterTransition(tl, oldScene, nextScene, chapterBadge, T) {
  const D = 0.52;
  const mountAt = Math.max(0, T - 0.10);
  const oldMotion = oldScene.querySelectorAll('.scene-motion');
  const nextMotion = nextScene.querySelectorAll('.scene-motion');
  tl.set(oldScene, {x: 0, y: 0, xPercent: 0, yPercent: 0, scale: 1, opacity: 1, zIndex: 1}, T);
  tl.set(nextScene, {x: 0, y: 0, xPercent: 0, yPercent: 0, scale: 1, opacity: 0, zIndex: 2}, mountAt);
  tl.set(nextMotion, {x: 36, scale: 1.025}, mountAt);
  tl.fromTo(nextScene,
    {opacity: 0},
    {opacity: 1, duration: D, ease: 'power3.inOut', immediateRender: false}, T);
  tl.fromTo(nextMotion,
    {x: 36, scale: 1.025},
    {x: 0, scale: 1, duration: D, ease: 'power3.inOut', immediateRender: false}, T);
  tl.fromTo(oldMotion,
    {x: 0, scale: 1},
    {x: -36, scale: 0.975, duration: D, ease: 'power3.inOut', immediateRender: false}, T);
  tl.set(oldScene, {opacity: 0, zIndex: 0}, T + D);
  // 每次章节使用独立角标元素，初始 autoAlpha: 0，位于场景外、字幕下。
  tl.fromTo(chapterBadge,
    {y: -28, autoAlpha: 0},
    {y: 0, autoAlpha: 1, duration: 0.38, ease: 'power3.out', immediateRender: false}, T + 0.12);
  tl.fromTo(chapterBadge,
    {y: 0, autoAlpha: 1},
    {y: -18, autoAlpha: 0, duration: 0.32, ease: 'power2.in', immediateRender: false}, T + 1.25);
}
~~~

章节角标使用画面一角的小组件，不占满屏幕，不遮住字幕。检查中间帧与四角四边，尤其是内层缩小和水平位移时；旧壳只能在新壳覆盖完成后退出。若需要连背景一起移动，另做覆盖几何验证，不能仅靠补纯色背景掩盖透底。

## 场景内部进入

标题、界面框架、主素材和节点有不同层级。每个场景各自构建，所有选择器由 sceneRoot 限定；同名 class 不会串改别的场景。

~~~js
function buildSceneTimeline(sceneRoot, enterOffset = 0) {
  const q = (selector) => sceneRoot.querySelectorAll(selector);
  const sceneTl = gsap.timeline({paused: true});
  // 挂载时显式设初值，避免未来 tween 立即渲染或第一次远距离 seek 污染状态。
  gsap.set(q('.title'), {y: 22, autoAlpha: 0});
  gsap.set(q('.chapter, .rule, .footer'), {y: 10, autoAlpha: 0});
  gsap.set(q('.visual'), {x: 34, autoAlpha: 0});
  sceneTl.addLabel('scene-enter', enterOffset);
  sceneTl.fromTo(q('.title'),
    {y: 22, autoAlpha: 0},
    {y: 0, autoAlpha: 1, duration: 0.38, ease: 'power3.out', immediateRender: false},
    'scene-enter');
  sceneTl.fromTo(q('.chapter, .rule, .footer'),
    {y: 10, autoAlpha: 0},
    {y: 0, autoAlpha: 1, duration: 0.34, stagger: 0.05, ease: 'power2.out', immediateRender: false},
    'scene-enter+=0.08');
  sceneTl.fromTo(q('.visual'),
    {x: 34, autoAlpha: 0},
    {x: 0, autoAlpha: 1, duration: 0.48, ease: 'power3.out', immediateRender: false},
    'scene-enter+=0.08');
  return sceneTl;
}
~~~

流程节点、插画中的中文标注和动作按已核验关键词的相对时间进入。普通节点参考 y:22、0.22 秒；短卡片仅在信息适合时使用。仅同一语义落点的一组元素可使用 stagger，分别对应不同关键词的元素必须按各自锚点排布；连线使用 strokeDashoffset 在约 0.25 秒内匀速画出。同一场景重复进入须用新的实例或重建时间线，不能复用被退出动画改过的隐式状态。

## 插画关键词锚点

从制作蓝图的关键词同步表读取已核验的绝对时间；场景时间包含预加载偏移时，统一用 `关键词绝对时间 - 子时间线全局起点` 得到局部时间，不能额外再加一次预加载延迟。中文标注保持独立文字层并靠近目标对象，关键词起点直接设为可读状态，再动画指向或解释动作，避免长淡入导致读到词时标签还看不清。

用 `addLabel` 与 position parameter 安排同一关键词的标注、指向和动作；结果用自己的发音锚点揭示，不把整句中的事件都放在字幕起点。任意 seek 到关键词前、中、后都应得到正确状态。转场提前交接，使锚点帧的主体已可读。音效使用同表的绝对入点交给音轨混合，不通过 GSAP 回调或定时器实时播放，以保证导出与 seek 一致。

## 字幕

字幕属于根时间线和最高视觉层，使用真实口播的独立时间轴，不能等待转场结束才换句。参考层级：字幕 10000、片头 9500、章节角标 9200、转场场景临时为 1 和 2；这些层应处在同一根堆叠上下文，不能被父级层叠困住。

每条字幕独立 DOM 元素，文字在创建时间线前固定。下面用时间线 set 保证短句边界精确、随机 seek 后文字正确；需要 0.14 秒进入或 0.10 秒退出时，将动画安排在该句的时间窗内，不能延迟下一句或造成双句重叠。

~~~js
function addSubtitleCues(rootTl, cues) {
  for (const {element, start, end} of cues) {
    // start/end 来自已核验口播，且 0 <= start < end；相邻句不重叠。
    // 首次 seek(0) 不一定触发零时刻 set，首句显式初始化为可见。
    gsap.set(element, {autoAlpha: start === 0 ? 1 : 0, y: 0});
    rootTl.set(element, {autoAlpha: 1, y: 0}, start);
    rootTl.set(element, {autoAlpha: 0}, end);
  }
}
~~~

切镜不能重置或遮住字幕。专门检查一句结束落在 T 到 T+D 内的情况，确认转场途中正常换句，且回 seek 后仍恢复对应短句。

## 性能和稳定性

- 运动优先使用 x、y、xPercent、yPercent、scale、rotation、autoAlpha。
- 不动画 top、left、width、height、margin 或 padding。
- 只给实际运动的全屏场景和关键元素设置 will-change。
- 避免大面积模糊、复杂滤镜、持续 3D 和无种子随机值。
- 不使用 CSS animation、transition、Web Animations API、requestAnimationFrame 或定时器制作画面运动。
- 最终验证必须针对编码后的 MP4，不只看浏览器预览；范围按蓝图与验收表执行。

手绘辅助文字的布局和触发时机遵循 [文字预留与口播联动规范](illustration-text-sync.md)。文字层由当前 sceneRoot 限定的时间线驱动；与字幕、动作和音效共用关键词时间锚点，并检查无文字、单项、多项和退场状态的往返 seek。
