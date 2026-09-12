# 3D 模型表情系统：流程与参考文档

本文档详细说明 Kawaii_Agent 中 **VRM** 与 **MMD** 两种模型的表情获取、传递、应用流程，以及涉及的组件、函数、参数与调用关系。

---

## 目录

1. [VRM 表情名/列表](#1-vrm-表情名列表)
2. [VRM 情感权重](#2-vrm-情感权重)
3. [VRM 口型/眨眼](#3-vrm-口型眨眼)
4. [VRM 发话表情（若启用）](#4-vrm-发话表情若启用)
5. [MMD 顶点表情名](#5-mmd-顶点表情名)
6. [MMD 材质表情](#6-mmd-材质表情)
7. [MMD 口型](#7-mmd-口型)
8. [MMD 发话表情](#8-mmd-发话表情)
9. [总览与调用关系图](#9-总览与调用关系图)
10. [详细调用链（按场景）](#10-详细调用链按场景)
11. [参数速查表](#11-参数速查表)
12. [关键函数签名与参数一览](#12-关键函数签名与参数一览)

---

## 1. VRM 表情名/列表

### 1.1 说明

VRM 的「表情」由 VRM 规范定义，对应 BlendShape（变形目标）。程序不自己维护表情名列表，而是**从已加载的 VRM 对象上读取**由 `@pixiv/three-vrm` 解析出的 Expression 信息。

### 1.2 数据来源

| 项目 | 说明 |
|------|------|
| **来源** | VRM 文件 → GLTFLoader + VRMLoaderPlugin → `gltf.userData.vrm` |
| **承载对象** | `vrm.expressionManager`（或旧版 `vrm.blendShapeProxy`） |
| **表情名列表** | `expressionManager.expressions`（每项含 `expressionName`）或 `expressionManager.expressionMap` 的键（VRM 1.0） |

### 1.3 涉及组件与调用

| 层级 | 组件/文件 | 函数/位置 | 说明 |
|------|------------|-----------|------|
| 加载 | `VRMViewer.jsx` → `VRMModel` | `useEffect` 内 `loader.load(url, (gltf) => { ... })` | GLTFLoader 加载 VRM，得到 `vrm = gltf.userData.vrm` |
| 读取 | `VRMViewer.jsx` → `VRMModel` | 同一 `loader.load` 回调内 | `vrm.expressionManager` 存在时，读取 `expressions?.map(e => e.expressionName)` 与 `Object.keys(expressionManager.expressionMap)` |
| 常量 | `src/utils/vrmMotions.js` | `VRM_EXPRESSIONS` | 程序内约定使用的 BlendShape 名（neutral, happy, angry, sad, relaxed, surprised, blink, aa, ih, ou, ee, oh 等），用于写代码时引用；实际模型是否包含由 VRM 文件决定 |

### 1.4 关键代码位置

- **VRM 加载与 Expression 确认**：`src/components/VRMViewer.jsx` 约 376–420 行（`VRMModel` 内 `loader.load` 回调）。
- **表情名常量**：`src/utils/vrmMotions.js` 约 27–46 行 `VRM_EXPRESSIONS`。

### 1.5 参数与数据结构

- `vrm.expressionManager.expressions`：数组，元素形状含 `expressionName: string`。
- `vrm.expressionManager.expressionMap`：对象，键为表情名（string），值为内部 Expression 对象。
- 应用时使用：`expressionManager.setValue(expressionName: string, value: number)`，`value` 通常为 0–1。

### 1.6 代码与函数

**VRM 加载与 Expression 读取（VRMViewer.jsx，VRMModel 内 loader.load 回调）：**

```javascript
loader.load(url, (gltf) => {
  const vrm = gltf.userData.vrm;
  scene.add(vrm.scene);
  vrmRef.current = vrm;

  if (vrm.expressionManager) {
    const expressionNames = vrm.expressionManager.expressions?.map(e => e.expressionName);
    if (vrm.expressionManager.expressionMap) {
      console.log('Expression map keys:', Object.keys(vrm.expressionManager.expressionMap));
    }
  }

  if (vrm.expressionManager) {
    const expressions = ['happy', 'angry', 'sad', 'relaxed', 'neutral', 'surprised'];
    expressions.forEach(exp => {
      if (vrm.expressionManager.setValue) {
        vrm.expressionManager.setValue(exp, 0);
      }
    });
  }
});
```

**表情名常量（src/utils/vrmMotions.js）：**

```javascript
export const VRM_EXPRESSIONS = {
  neutral: 'neutral',
  happy: 'happy',
  angry: 'angry',
  sad: 'sad',
  relaxed: 'relaxed',
  surprised: 'surprised',
  blink: 'blink',
  blinkLeft: 'blinkLeft',
  blinkRight: 'blinkRight',
  lookUp: 'lookUp',
  lookDown: 'lookDown',
  lookLeft: 'lookLeft',
  lookRight: 'lookRight',
  aa: 'aa',
  ih: 'ih',
  ou: 'ou',
  ee: 'ee',
  oh: 'oh'
};
```

**VRMModel 组件签名（VRMViewer.jsx）：**

```javascript
function VRMModel({
  url,
  onLoad,
  enableMouseFollow = true,
  enableInteraction = true,
  onMotionReady,
  emotion = 'neutral',
  emotionIntensity = 0.5,
  isTyping = false,
  gesture = null,
  isSpeaking = false,
  currentSpeechText = '',
  cameraConfig = { position: [0,1.4,2.5], fov: 50, lookAt: [0,1,0] },
  onInteraction,
  enableCameraFollow = false,
  onCameraChange,
  overlayBlendRatio = 1.0,
  onTapEffect,
  vrmScale = 1.0,
  enableManualCamera = true
}) { ... }
```

---

## 2. VRM 情感权重

### 2.1 说明

「情感」指 happy / sad / angry / surprised / thinking / neutral / sleeping 等。其**类型**和**强度**由 App 层根据「当前播放的动作名」或「AI 回复文本」推断，通过 props 下发给 VRMViewer，再在每帧根据 `currentEmotion` 和 `emotionIntensity` 计算权重并写入 `expressionManager.setValue(...)`。

### 2.2 数据流（简要）

```
App.jsx (currentEmotion, emotionIntensity)
  → VRMViewer (props: emotion, emotionIntensity)
    → VRMModel (props: emotion, emotionIntensity)
      → useFrame: currentEmotion.current, targetEmotionValue.current, emotionIntensity
        → applyEmotion(emotion, emotionValue)
          → expressionManager.setValue('happy'|'sad'|'angry'|... , value)
```

### 2.3 情感来源（谁设置 currentEmotion / emotionIntensity）

| 场景 | 位置 | 方式 |
|------|------|------|
| 初始/回退待机 | `App.jsx` | `guessEmotionFromMotion(targetMotion)` → `setCurrentEmotion(resolvedEmotion)`、`setEmotionIntensity(intensity)` |
| 定时待机更新 | `App.jsx` → `updateIdleEmotion` | 同上，根据 `pickFallbackMotion(null)` 得到的动作名推断 |
| AI 回复文本 | `App.jsx` | `detectEmotionFromText(assistantMessage)` → 若检测到则 `setCurrentEmotion(finalEmotion)`，否则用 `fallbackEmotion` |
| 播放特定动作 | `App.jsx` | 如播放 wave → `setCurrentEmotion('happy')`、`setCurrentGesture('wave')` 等 |
| 思考/打字中 | `App.jsx` | `setCurrentEmotion('thinking')` |

### 2.4 涉及组件与参数

| 组件/文件 | 类型 | 名称 | 说明 |
|-----------|------|------|------|
| `App.jsx` | state | `currentEmotion` | 当前情感键名，如 `'neutral'`、`'happy'` |
| `App.jsx` | state | `emotionIntensity` | 情感强度，0–1，默认 0.5 |
| `App.jsx` | 函数 | `guessEmotionFromMotion(motionName)` | 根据动作名返回情感；见约 73–119 行 |
| `App.jsx` | 函数 | `detectEmotionFromText(text)` | 根据 AI 回复文本关键词返回情感；见约 123–159 行 |
| `VRMViewer` | props | `emotion`, `emotionIntensity` | 从 App 传入，再传给 VRMModel |
| `VRMModel` | props | `emotion`, `emotionIntensity` | 同上 |
| `VRMModel` | ref | `currentEmotion` | 与 props `emotion` 同步，用于 useFrame |
| `VRMModel` | ref | `targetEmotionValue` | 情感值缓动目标，在 useFrame 中衰减 |
| `VRMModel` | useFrame | `applyEmotion(emotion, value)` | 根据 emotion 分支调用 `setValue('happy'|'sad'|...)` |

### 2.5 情感 → BlendShape 映射（VRMModel 内）

- `happy` → `setValue('happy', value)` 或回退 `setValue('ih', value*0.6)`
- `sad` → `setValue('sad', value)` 或 `setValue('ou', value*0.4)`
- `angry` → `setValue('angry', value)`
- `surprised` → `setValue('surprised', value)`、`setValue('aa', value*0.8)`
- `thinking` → `setValue('relaxed', value)`
- `sleeping` → `setValue('blink', 1)`、`setValue('ih', 0.3*value)`
- `neutral` → 仅做「全预设表情归 0」，不额外 setValue

每帧会先把 `['happy','sad','angry','surprised','relaxed','neutral']` 设为 0，再根据 `currentEmotion.current` 调用上述映射。

### 2.6 关键代码位置

- App 状态与传递：`App.jsx` 约 492–493、4956–4957、1888–1899、1943–1950、2735–2762、2845–2852 等。
- VRM 应用：`VRMViewer.jsx` 约 1610–1673（VRMModel 的 useFrame 内 `applyEmotion` 及调用）。

### 2.7 代码与函数

**App.jsx 状态声明：**

```javascript
const [currentEmotion, setCurrentEmotion] = useState('neutral');
const [emotionIntensity, setEmotionIntensity] = useState(0.5);
```

**guessEmotionFromMotion（App.jsx，约 73–119 行）：**

```javascript
// モーション名から表情を推測する関数
const guessEmotionFromMotion = (motionName = '') => {
  const lower = motionName.toLowerCase();
  if (lower.includes('happy') || lower.includes('joy') || lower.includes('excited') ||
      lower.includes('smile') || lower.includes('laugh') || lower.includes('cheer') ||
      lower.includes('clap') || lower.includes('wave') || lower.includes('waving') ||
      lower.includes('greeting') || lower.includes('sing')) return 'happy';
  if (lower.includes('sad') || lower.includes('cry') || lower.includes('tear') || lower.includes('depress')) return 'sad';
  if (lower.includes('think') || lower.includes('confus') || lower.includes('typing') || lower.includes('looking_files') || lower.includes('looking_through')) return 'thinking';
  if (lower.includes('surprise') || lower.includes('shock') || lower.includes('amaze') || lower.includes('jump')) return 'surprised';
  if (lower.includes('angry') || lower.includes('mad') || lower.includes('rage')) return 'angry';
  if (lower.includes('sleep') || lower.includes('sleeping') || lower.includes('laying')) return 'sleeping';
  if (lower.includes('relax') || lower.includes('calm') || lower.includes('lazy') || lower.includes('tired') || lower.includes('drunk')) return 'thinking';
  return 'neutral';
};
```

**detectEmotionFromText（App.jsx，约 123–159 行）：**

```javascript
const detectEmotionFromText = (text = '') => {
  if (!text) return null;
  const lower = text.toLowerCase();
  if (lower.includes('恥ずかし') || lower.includes('照れ') || lower.includes('///')) return 'happy';
  if (lower.includes('やめて') || lower.includes('ダメ') || lower.includes('怒') || lower.includes('！！')) return 'angry';
  if (lower.includes('え') || lower.includes('びっくり') || lower.includes('驚') || lower.includes('！？')) return 'surprised';
  if (lower.includes('嬉し') || lower.includes('ありがと') || lower.includes('やった') || lower.includes('♪') || lower.includes('♡')) return 'happy';
  if (lower.includes('なんで') || lower.includes('どうして') || lower.includes('？') || lower.includes('不思議')) return 'thinking';
  return null;
};
```

**VRMModel 内 emotion 同步与 applyEmotion（VRMViewer.jsx）：**

```javascript
// emotion 变化时同步到 ref
useEffect(() => {
  if (emotion !== currentEmotion.current) {
    currentEmotion.current = emotion;
    targetEmotionValue.current = 1;
    // ... gesture 触发等
  }
}, [emotion]);

// useFrame 内
const amplifiedIntensity = Math.min(emotionIntensity * 1.3, 1.0);
const emotionValue = targetEmotionValue.current * amplifiedIntensity;
['happy', 'sad', 'angry', 'surprised', 'relaxed', 'neutral'].forEach(exp => {
  try { expressionManager.setValue(exp, 0); } catch(_) {}
});
const applyEmotion = (emotion, value) => {
  if (value <= 0) return;
  switch(emotion) {
    case 'happy':
      try { expressionManager.setValue('happy', value); } catch(_) { try { expressionManager.setValue('ih', value * 0.6); } catch(_) {} }
      break;
    case 'sad':
      try { expressionManager.setValue('sad', value); } catch(_) { try { expressionManager.setValue('ou', value * 0.4); } catch(_) {} }
      break;
    case 'angry': try { expressionManager.setValue('angry', value); } catch(_) {} break;
    case 'surprised':
      try { expressionManager.setValue('surprised', value); } catch(_) {}
      try { expressionManager.setValue('aa', value * 0.8); } catch(_) {}
      break;
    case 'thinking': try { expressionManager.setValue('relaxed', value); } catch(_) {} break;
    case 'sleeping':
      try { expressionManager.setValue('blink', 1.0); } catch(_) {}
      try { expressionManager.setValue('ih', 0.3 * value); } catch(_) {}
      break;
    case 'neutral':
    default: break;
  }
};
applyEmotion(currentEmotion.current, emotionValue);
```

---

## 3. VRM 口型/眨眼

### 3.1 口型（LipSync）

- **说明**：说话时对口部 BlendShape（如 `aa`、`oh`）做周期性 0/1 或 0~1 的切换，不依赖外部 TTS 音素。
- **触发**：`VRMModel` 的 useFrame 中 `isSpeaking === true` 时进入口型分支。
- **逻辑**：
  - `mouthTimer` 每帧累加 `delta`，每 0.15 秒切换 `mouthState`（0=闭，1=开）。
  - 用 `mouthProgress` 做线性插值得到 `mouthValue`。
  - 按 `Math.floor(time * mouthSpeed) % 2` 交替：一半时间 `setValue('aa', mouthValue*0.6)`、`setValue('oh', 0)`；另一半 `setValue('aa', 0)`、`setValue('oh', mouthValue*0.5)`。
- **参数**：`mouthSpeed = 8`，周期 0.15s。不读取「表情名列表」，仅写死 `aa`/`oh`。

### 3.2 眨眼（Blink）

- **说明**：定时将 `blink` 设为 1，约 100ms 后归 0，模拟眨眼。
- **两处实现**：
  1. **VRMModel 内 useFrame**（约 1579–1598 行）：`blinkTimer` 累加，达到 `nextBlinkTime` 时 `setValue('blink', 1)`，`setTimeout` 100ms 后 `setValue('blink', 0)`，并重置 `nextBlinkTime = 2~5` 秒随机。
  2. **IdleMotion**（`vrmMotions.js` 约 106–139 行）：同样逻辑，通过 `expressionManager` / `blendShapeProxy` 的 `setValue('blink', 1/0)` 或 `setExpression('blink', 1/0)`。
- **参数**：眨眼间隔 `nextBlinkTime` 在 2–5 秒随机；眨眼持续时间 100ms。

### 3.3 涉及组件与调用

| 组件/文件 | 作用 |
|-----------|------|
| `VRMViewer.jsx` → `VRMModel` | useFrame 内口型与眨眼（主逻辑） |
| `vrmMotions.js` → `IdleMotion.applyBlink` | 若使用 IdleMotion，则由其负责眨眼 |

### 3.4 代码与函数

**VRMModel 内眨眼（VRMViewer.jsx useFrame）：**

```javascript
blinkTimer.current += delta;
if (blinkTimer.current >= nextBlinkTime.current && !isBlinking.current) {
  isBlinking.current = true;
  blinkTimer.current = 0;
  const expressionManager = vrm.expressionManager;
  if (expressionManager?.setValue) {
    expressionManager.setValue('blink', 1);
    setTimeout(() => {
      if (vrmRef.current?.expressionManager?.setValue) {
        vrmRef.current.expressionManager.setValue('blink', 0);
      }
      isBlinking.current = false;
      nextBlinkTime.current = Math.random() * 3 + 2;  // 2~5 秒
    }, 100);
  }
}
```

**VRMModel 内口型（VRMViewer.jsx useFrame，isSpeaking 时）：**

```javascript
if (isSpeaking && expressionManager.setValue) {
  mouthTimer.current += delta;
  const mouthSpeed = 8;
  if (mouthTimer.current >= 0.15) {
    mouthState.current = mouthState.current === 0 ? 1 : 0;
    mouthTimer.current = 0;
  }
  const mouthProgress = mouthTimer.current / 0.15;
  const mouthValue = mouthState.current === 1 ? Math.min(1, mouthProgress) : Math.max(0, 1 - mouthProgress);
  const useAa = Math.floor(time * mouthSpeed) % 2 === 0;
  if (useAa) {
    expressionManager.setValue('aa', mouthValue * 0.6);
    expressionManager.setValue('oh', 0);
  } else {
    expressionManager.setValue('aa', 0);
    expressionManager.setValue('oh', mouthValue * 0.5);
  }
}
```

**IdleMotion.applyBlink（src/utils/vrmMotions.js）：**

```javascript
applyBlink(deltaTime) {
  this.blinkTimer += deltaTime;
  if (this.blinkTimer >= this.nextBlinkTime && !this.isBlinking) {
    this.isBlinking = true;
    this.blinkTimer = 0;
    const expressionManager = this.vrm.expressionManager || this.vrm.blendShapeProxy;
    if (expressionManager) {
      if (expressionManager.setValue) expressionManager.setValue('blink', 1);
      else if (expressionManager.setExpression) expressionManager.setExpression('blink', 1);
      setTimeout(() => {
        const em = this.vrm.expressionManager || this.vrm.blendShapeProxy;
        if (em) {
          if (em.setValue) em.setValue('blink', 0);
          else if (em.setExpression) em.setExpression('blink', 0);
        }
        this.isBlinking = false;
        this.nextBlinkTime = Math.random() * 3 + 2;
      }, 100);
    }
  }
}
```

---

## 4. VRM 发话表情（若启用）

### 4.1 说明

当前该功能在代码中**被注释掉**。若重新启用，流程为：在「当前要说的文本」确定时，用 VRM 的可用表情名列表调用 AI，得到「表情参数」，在说话时叠加到情感与口型之上。

### 4.2 数据流（若启用）

1. **收集可用表情名**：从 `vrm.expressionManager.expressionMap` 的键或 `expressionManager.expressions[].expressionName` 得到 `availableExpressions`（字符串数组）。
2. **请求 AI**：`aiService.generateExpressionParams(currentSpeechText, availableExpressions)`。
3. **AI 返回**：`generateExpressionParams` 返回的是**字符串数组**（如 `["happy", "ih"]`），不是 `{ name: weight }`。
4. **应用**：注释内代码用 `Object.entries(gptExpressionParams.current)` 遍历并 `setValue(expName, value)`，即期望的是**对象**。若直接使用当前 AI 返回值（数组），需要一层转换：将数组转成 `{ [morphName]: 1.0 }` 再应用。

### 4.3 涉及组件与参数

| 组件/文件 | 说明 |
|-----------|------|
| `VRMViewer.jsx` → `VRMModel` | 约 1675–1712（注释块）：收集 availableExpressions，调用 `aiService.generateExpressionParams`，写入 `gptExpressionParams.current` |
| 同上 | 约 1747–1760：`isSpeaking` 时用 `gptExpressionParams.current` 做 `Object.entries` 并 `setValue`；若启用需保证传入为对象或先做数组→对象转换 |
| `aiService.js` | `generateExpressionParams(context, availableExpressions)`，见第 8 节 |

### 4.4 代码（当前注释块）

**收集可用表情并请求 AI（VRMViewer.jsx，注释中）：**

```javascript
if (currentSpeechText && currentSpeechText !== lastSpeechText.current && expressionManager) {
  lastSpeechText.current = currentSpeechText;
  const availableExpressions = [];
  if (expressionManager.expressionMap) {
    availableExpressions.push(...Object.keys(expressionManager.expressionMap));
  } else if (expressionManager.expressions) {
    expressionManager.expressions.forEach(exp => {
      if (exp.expressionName) availableExpressions.push(exp.expressionName);
    });
  }
  if (availableExpressions.length > 0 && aiService.isReady) {
    (async () => {
      const params = await aiService.generateExpressionParams(currentSpeechText, availableExpressions);
      if (params && lastSpeechText.current === currentSpeechText) {
        gptExpressionParams.current = params;  // AI 返回的是 string[]，此处若用 Object.entries 需先转成对象
      }
    })();
  }
}
```

**应用 GPT 表情（isSpeaking 时，注释外仍存在）：**

```javascript
if (gptExpressionParams.current) {
  Object.entries(gptExpressionParams.current).forEach(([expName, value]) => {
    if (expName !== 'aa' && expName !== 'oh' && typeof value === 'number') {
      expressionManager.setValue(expName, value);
    }
  });
}
```

---

## 5. MMD 顶点表情名

### 5.1 说明

MMD（PMX）的顶点变形表情名来自 Three.js **MMDLoader** 解析 PMX 后生成的 mesh。每个子 mesh 可能有 `morphTargetDictionary`（名→索引）和 `morphTargetInfluences`（索引→权重）。

### 5.2 数据来源

| 项目 | 说明 |
|------|------|
| **来源** | PMX 文件 → MMDLoader → 返回的 SkinnedMesh（及其子 mesh） |
| **承载对象** | `mesh.traverse` 下每个 child 的 `child.morphTargetDictionary`、`child.morphTargetInfluences` |
| **表情名列表** | `Object.keys(child.morphTargetDictionary)` 合并所有子 mesh 的键 |

### 5.3 涉及组件与调用

| 组件/文件 | 函数/位置 | 说明 |
|-----------|-----------|------|
| `VRMViewer.jsx` → `MMDModel` | MMDLoader 的 `load(url, ...)` 回调 | 得到根 `mesh`，其下子节点有 `morphTargetDictionary` |
| 同上 | 约 2715–2738 行 | 遍历 mesh 打印/统计 `morphTargetDictionary` |
| 同上 | GPT 发话表情、口型初始化 | 通过 `Object.keys(child.morphTargetDictionary)` 收集名字，用于 `availableExpressions` 或母音查找 |

### 5.4 参数与数据结构

- `morphTargetDictionary`：`{ [morphName: string]: number }`，值为 morph 索引。
- `morphTargetInfluences`：`number[]`，索引对应上面数字，值为 0–1 权重。
- 应用：直接写 `mesh.morphTargetInfluences[morphIndex] = value`。

### 5.5 代码与函数

**MMD 加载后遍历 morph 名（VRMViewer.jsx MMDModel load 回调）：**

```javascript
mesh.traverse((child) => {
  if (child.isMesh && child.morphTargetDictionary) {
    const morphNames = Object.keys(child.morphTargetDictionary);
    // 用于 GPT 表情：availableExpressions.push(...morphNames)
  }
});
```

**收集 availableExpressions 用于 GPT（MMDModel useFrame）：**

```javascript
const availableExpressions = [];
mesh.traverse((child) => {
  if (child.morphTargetDictionary && child.morphTargetInfluences) {
    Object.keys(child.morphTargetDictionary).forEach(morphName => {
      if (!availableExpressions.includes(morphName)) availableExpressions.push(morphName);
    });
  }
});
```

**MMDModel 组件签名（VRMViewer.jsx）：**

```javascript
function MMDModel({
  url,
  onLoad,
  vmdUrls = [],
  fileMap,
  onAnimationDuration,
  onMeshReady,
  onInteraction,
  tapMotionUrls = [],
  petMotionUrls = [],
  onMmdInteractionMotion,
  helperRef: parentHelperRef,
  sceneRef: parentSceneRef,
  clonedMeshRef: parentClonedMeshRef,
  enableCameraFollow = false,
  onCameraChange,
  cameraConfig,
  targetLoopCount = 3,
  onLoopComplete,
  enablePhysicsRef,
  enablePhysics,
  enablePmxAnimation,
  enableSimplePhysics = false,
  onTapEffect,
  isSpeaking = false,
  currentSpeechText = '',
  mmdScale = 0.09,
  mmdShininess = 50,
  mmdBrightness = 1.0,
  enableInteraction = true,
  enableManualCamera = true
}) { ... }
```

---

## 6. MMD 材质表情

### 6.1 说明

PMX 中除顶点 morph 外还有**材质 morph**（type=8），用于改变材质颜色、高光、不透明度等。程序用**自定义 PMX 解析器**只读取这类 morph，不依赖 MMDLoader 的顶点 morph。

### 6.2 数据来源

| 项目 | 说明 |
|------|------|
| **来源** | PMX 文件 → `parsePMXMaterialMorphs(url)`（`src/utils/pmxParser.js`） |
| **存储** | 解析结果放入 `window._mmdMaterialMorphs`（数组） |
| **触发解析** | MMD 模型加载成功后，在 `MMDModel` 的 load 回调内异步调用 `parsePMXMaterialMorphs(url)` |

### 6.3 解析与数据结构（pmxParser）

- **入口**：`parsePMXMaterialMorphs(url)` → fetch(url) → `PMXParser(buffer).parse()`。
- **只保留**：`type === 8` 的 morph（材质 morph）。
- **单个 morph**：`{ name, nameEn, panel, type, elements }`。`elements` 为数组，每项含：
  - `index`：材质索引
  - `calcMode`
  - `diffuse`, `specular`, `specularPower`, `ambient`, `edgeColor`, `edgeSize`, `textureColor`, `sphereColor`, `toonColor` 等
- **应用**：在 MMD 发话表情应用时，若 GPT 返回的某个名字在 `window._mmdMaterialMorphs` 中找到，则对该 mesh 的 material 按 element 写 diffuse/specular/opacity 等（见 `applyMaterialMorph`）。

### 6.4 涉及组件与调用

| 组件/文件 | 函数/位置 | 说明 |
|-----------|-----------|------|
| `pmxParser.js` | `readMorph()` | 读取 name、type、offsetCount；仅 type===8 时解析 elements |
| `pmxParser.js` | `parsePMXMaterialMorphs(url)` | 入口，返回 `morphs.filter(m => m && m.type === 8)` |
| `VRMViewer.jsx` → `MMDModel` | load 回调内约 2703–2712 行 | `parsePMXMaterialMorphs(url)` 并赋给 `window._mmdMaterialMorphs` |
| `VRMViewer.jsx` → `MMDModel` | 发话表情应用约 5071–5134 行 | `applyMaterialMorph(mesh, materialMorph, 1.0)`，写 material 的 color/specular/opacity |

### 6.5 代码与函数

**parsePMXMaterialMorphs（src/utils/pmxParser.js）：**

```javascript
export async function parsePMXMaterialMorphs(url) {
  try {
    const response = await fetch(url);
    const buffer = await response.arrayBuffer();
    const parser = new PMXParser(buffer);
    const morphs = parser.parse();
    return morphs.filter(m => m && m.type === 8);
  } catch (error) {
    console.error('[PMX Parser] Error:', error);
    return [];
  }
}
```

**readMorph（pmxParser.js，仅 type===8 时返回 morph）：**

```javascript
readMorph() {
  const name = this.readText();
  const nameEn = this.readText();
  const panel = this.readByte();
  const type = this.readByte();
  const offsetCount = this.readInt();
  const morph = { name, nameEn, panel, type, elements: [] };
  if (type === 8) {
    for (let i = 0; i < offsetCount; i++) {
      const element = {
        index: this.readIndex(this.materialIndexSize),
        calcMode: this.readByte(),
        diffuse: [this.readFloat(), this.readFloat(), this.readFloat(), this.readFloat()],
        specular: [this.readFloat(), this.readFloat(), this.readFloat()],
        specularPower: this.readFloat(),
        ambient: [...],
        edgeColor: [...],
        edgeSize: this.readFloat(),
        textureColor: [...],
        sphereColor: [...]
      };
      morph.elements.push(element);
    }
    return morph;
  }
  this.skipMorphData(type, offsetCount);
  return null;
}
```

**MMDModel load 内解析材质 morph：**

```javascript
(async () => {
  const materialMorphs = await parsePMXMaterialMorphs(url);
  window._mmdMaterialMorphs = materialMorphs;
})();
```

**applyMaterialMorph（VRMViewer.jsx MMDModel useFrame 内）：**

```javascript
const applyMaterialMorph = (mesh, mmdMorph, weight) => {
  if (!mmdMorph?.elements) return false;
  mmdMorph.elements.forEach(element => {
    const materials = Array.isArray(mesh.material) ? mesh.material : [mesh.material];
    const targetMaterial = materials[element.index] || materials[0];
    if (!targetMaterial) return;
    const applyValue = (base, add, mul) => base * mul + add * weight;
    if (element.diffuse) {
      targetMaterial.color.setRGB(
        applyValue(targetMaterial.color.r, element.diffuse[0], element.diffuse[3]),
        applyValue(targetMaterial.color.g, element.diffuse[1], element.diffuse[3]),
        applyValue(targetMaterial.color.b, element.diffuse[2], element.diffuse[3])
      );
    }
    if (element.specular && targetMaterial.specular) {
      targetMaterial.specular.setRGB(...);
    }
    if (element.opacity !== undefined) {
      targetMaterial.opacity = applyValue(targetMaterial.opacity, element.opacity[0], element.opacity[1]);
      targetMaterial.transparent = targetMaterial.opacity < 1.0;
    }
    targetMaterial.needsUpdate = true;
  });
  return true;
};
```

---

## 7. MMD 口型

### 7.1 说明

MMD 口型通过「母音 morph」实现：在 `morphTargetDictionary` 中按预定候选名查找「あ/い/う/え/お」等对应的索引，说话时按节奏选一个母音并对其 `morphTargetInfluences[index]` 写入 0~1。

### 7.2 初始化：母音目标列表

- **函数**：`initializeMmdLipSyncTargets(root)`（`VRMViewer.jsx` 约 2145–2207 行）。
- **入参**：`root` 为 MMD 根 mesh（或克隆后的根）。
- **逻辑**：
  - 对每个子节点检查 `child.morphTargetDictionary` 与 `child.morphTargetInfluences`。
  - 对 a/i/u/e/o 各有一组候选名（如 `a: ['あ','あ１','A','a','aa','mouth_a','mouthA']` 等）。
  - 用 `findIndex(candidates)` 在 dictionary 中找第一个存在的名字，得到 `{ index, name }`。
  - 若该 mesh 至少找到一个母音，则加入 `targets`：`{ mesh, influences, vowels, vowelNames }`。
- **存储**：`mmdLipSyncRef.current = { targets }`。
- **调用时机**：
  - 模型/动画加载后：约 2404–2405、2891–2893 行。
  - 动画切换后：约 3184 行。
  - mesh 克隆后：约 5705–5707 行通过 `newMesh.userData.initializeMmdLipSyncTargets()`。

### 7.3 每帧口型应用（useFrame）

- **条件**：`isSpeaking && mesh`，且 `mmdLipSyncRef.current.targets.length > 0`。
- **节奏**：`mmdMouthStateRef.current` 记录开/闭与下一周期时间；每约 0.08–0.16 秒随机切换开/闭，开口时按权重随机选一个母音（a 30%, i 25%, u 20%, e 15%, o 10%）。
- **插值**：`mouthValue` 在 minOpenness 与各母音最大强度之间做 ease-in-out。
- **写入**：对每个 target，先把该 mesh 所有母音对应索引的 `influences[index]` 置 0，再对当前选中的母音写 `influences[morphInfo.index] = mouthValue`。

### 7.4 涉及组件与参数

| 组件/文件 | 说明 |
|-----------|------|
| `VRMViewer.jsx` → `MMDModel` | `initializeMmdLipSyncTargets`、useFrame 内口型逻辑（约 4902–5012） |
| `mmdLipSyncRef.current` | `{ targets: [{ mesh, influences, vowels, vowelNames }] }` |
| 母音候选 | `vowelCandidates` 在 `initializeMmdLipSyncTargets` 内写死（a/i/u/e/o 多语言名） |

### 7.5 代码与函数

**initializeMmdLipSyncTargets（VRMViewer.jsx MMDModel 内）：**

```javascript
const initializeMmdLipSyncTargets = (root) => {
  if (!root) {
    mmdLipSyncRef.current = { targets: [] };
    return;
  }
  const vowelCandidates = {
    a: ['あ', 'あ１', 'あ2', 'あ０', 'A', 'a', 'aa', 'mouth_a', 'mouthA'],
    i: ['い', 'い１', 'い2', 'い０', 'I', 'i', 'ii', 'mouth_i', 'mouthI'],
    u: ['う', 'う１', 'う2', 'う０', 'U', 'u', 'uu', 'mouth_u', 'mouthU'],
    e: ['え', 'え１', 'え2', 'え０', 'E', 'e', 'ee', 'mouth_e', 'mouthE'],
    o: ['お', 'お１', 'お2', 'お０', 'O', 'o', 'oo', 'mouth_o', 'mouthO', 'oh']
  };
  const targets = [];
  root.traverse?.((child) => {
    const dict = child?.morphTargetDictionary;
    const influences = child?.morphTargetInfluences;
    if (!dict || !influences) return;
    const findIndex = (names) => {
      for (const name of names) {
        if (dict[name] !== undefined) return { index: dict[name], name };
      }
      return null;
    };
    const vowels = {};
    for (const [vowel, candidates] of Object.entries(vowelCandidates)) {
      const info = findIndex(candidates);
      if (info) vowels[vowel] = info;
    }
    if (Object.keys(vowels).length > 0) {
      targets.push({ mesh: child, influences, vowels, vowelNames: Object.keys(vowels) });
    }
  });
  mmdLipSyncRef.current = { targets };
};
```

**调用时机示例：**

```javascript
mesh.userData.initializeMmdLipSyncTargets = () => initializeMmdLipSyncTargets(mesh);
initializeMmdLipSyncTargets(mesh);  // 模型/动画加载后
// 动画切换后：initializeMmdLipSyncTargets(mesh);
// 克隆后：newMesh.userData.initializeMmdLipSyncTargets();
```

**每帧口型写入（isSpeaking && lipSyncTargets.length > 0）：**

```javascript
lipSyncTargets.forEach((target) => {
  const { influences, vowels, vowelNames } = target;
  for (const vowel of vowelNames) {
    const morphInfo = vowels[vowel];
    if (morphInfo && typeof morphInfo.index === 'number') influences[morphInfo.index] = 0;
  }
  if (state.currentVowel && vowels[state.currentVowel]) {
    const morphInfo = vowels[state.currentVowel];
    influences[morphInfo.index] = mouthValue;  // mouthValue 由 ease-in-out 插值得到
  }
});
```

---

## 8. MMD 发话表情

### 8.1 说明

在 MMD 模型上，根据「当前要说的文本」用 AI 从「顶点 + 材质 morph 名」中选出 1–3 个表情名，在说话时应用：顶点 morph 写 `morphTargetInfluences`，材质 morph 写 mesh.material 的 diffuse/specular/opacity。

### 8.2 数据流

1. **文本变化**：`currentSpeechText` 变化且与 `mmdLastSpeechTextRef.current` 不同时触发一次。
2. **收集可用表情名**：`mesh.traverse` 下所有 `child.morphTargetDictionary` 的 `Object.keys` 合并去重 → `availableExpressions`。
3. **请求 AI**：`aiService.generateExpressionParams(textToGenerate, availableExpressions)`（异步）。
4. **AI 返回**：字符串数组，如 `["笑い","困る"]`，写入 `mmdGptExpressionParamsRef.current`。
5. **应用**：在 useFrame 中，当 `isSpeaking && lipSyncTargets.length > 0` 时，对 `mmdGptExpressionParamsRef.current` 中每个名字：
   - 若在 `window._mmdMaterialMorphs` 中找到同名，则 `applyMaterialMorph(mesh, materialMorph, 1.0)`；
   - 否则在 `mesh.morphTargetDictionary` 中找完全匹配或部分匹配，写 `mesh.morphTargetInfluences[index] = 1.0`。
6. **规则**：母音名直接跳过；单用规则（如「はぅ」「なごみ」「ウィンク」只保留一个）；「笑い」与部分目 morph 互斥等，见约 5021–5051 行。

### 8.3 AI 服务：generateExpressionParams

- **文件**：`src/services/aiService.js`。
- **签名**：`async generateExpressionParams(context, availableExpressions)`。
- **参数**：
  - `context`：当前发话文本（string）。
  - `availableExpressions`：表情名数组（VRM 来自 expressionManager，MMD 来自 morphTargetDictionary 键）。
- **过滤**：母音（あ/い/う/え/お 等）、瞳大小、口角相关名从候选中剔除。
- **返回**：`string[]`，1–3 个表情名，或 `null`。
- **内部**：调用 GPT（gpt-4.1-mini），system 与 user prompt 要求按发话内容选表情、返回 JSON 数组；解析 `[...]` 后 push 进 `expressionHistory`（最多保留 5 条）。

### 8.4 涉及组件与调用

| 组件/文件 | 说明 |
|-----------|------|
| `VRMViewer.jsx` → `MMDModel` | 约 4851–4897：文本变化时收集 availableExpressions，调用 `aiService.generateExpressionParams`，写 `mmdGptExpressionParamsRef.current` |
| 同上 | 约 5014–5184：每帧应用 GPT 返回的名字（材质 morph + 顶点 morph），以及单用/笑い互斥规则 |
| `aiService.js` | `generateExpressionParams` 实现与过滤、历史 |

### 8.5 代码与函数

**aiService.generateExpressionParams（src/services/aiService.js）：**

```javascript
// 函数签名
async generateExpressionParams(context, availableExpressions)

// 参数
// - context: string，当前发话文本
// - availableExpressions: string[]，从 expressionManager 或 morphTargetDictionary 收集的表情名列表

// 过滤：母音（あ/い/う/え/お 等）、瞳サイズ、口角相关从候选中剔除
const expressionMorphs = availableExpressions.filter(morphName => {
  const lower = morphName.toLowerCase();
  if (['あ','い','う','え','お','ワ','ω'].includes(morphName) || ['a','i','u','e','o'].includes(lower)) return false;
  if (morphName.includes('瞳小') || morphName.includes('瞳大') || morphName.includes('瞳増大') || ...) return false;
  if (morphName.includes('口角広げ') || morphName.includes('口角上げ')) return false;
  return true;
});

// 调用 GPT，要求返回 JSON 数组
const result = await this.simpleQuery(`発話内容: "${context}"\n利用可能なモーフ: ${morphList}...\nJSON:`, systemPrompt, { model: 'gpt-4.1-mini', maxTokens: 500 });
const jsonMatch = result.trim().match(/\[[\s\S]*?\]/);
const parsed = JSON.parse(jsonMatch[0]);  // string[]
this.expressionHistory.push(parsed);
if (this.expressionHistory.length > this.maxExpressionHistory) this.expressionHistory.shift();
return parsed;  // string[] | null
```

**AIService 内与表情相关的成员（aiService.js）：**

```javascript
this.expressionHistory = [];   // 直近 5 次选择的表情数组
this.maxExpressionHistory = 5;
```

**MMDModel 内文本变化时触发 GPT 表情（useFrame）：**

```javascript
if (currentSpeechText && currentSpeechText !== mmdLastSpeechTextRef.current && mesh) {
  mmdLastSpeechTextRef.current = currentSpeechText;
  mmdGptExpressionParamsRef.current = [];
  const availableExpressions = [];
  mesh.traverse((child) => {
    if (child.morphTargetDictionary && child.morphTargetInfluences) {
      Object.keys(child.morphTargetDictionary).forEach(morphName => {
        if (!availableExpressions.includes(morphName)) availableExpressions.push(morphName);
      });
    }
  });
  if (availableExpressions.length > 0 && aiService.isReady) {
    (async () => {
      const selectedMorphs = await aiService.generateExpressionParams(textToGenerate, availableExpressions);
      if (selectedMorphs && Array.isArray(selectedMorphs)) {
        mmdGptExpressionParamsRef.current = selectedMorphs;  // string[]
      }
    })();
  }
}
```

**MMD 发话表情应用（材质 morph 优先，否则顶点 morph）：**

```javascript
const vowelMorphs = ['あ', 'い', 'う', 'え', 'お', 'a', 'i', 'u', 'e', 'o', 'ワ', 'ω'];
let morphsToApply = [...mmdGptExpressionParamsRef.current];
// 单用规则、笑い与目 morph 互斥等过滤 ...

lipSyncTargets.forEach((target) => {
  const mesh = target.mesh;
  morphsToApply.forEach((selectedMorphName) => {
    if (vowelMorphs.includes(selectedMorphName)) return;
    const materialMorphs = window._mmdMaterialMorphs || [];
    const materialMorph = materialMorphs.find(m => m.name === selectedMorphName);
    if (materialMorph) {
      applyMaterialMorph(mesh, materialMorph, 1.0);
      return;
    }
    const exactMatch = Object.keys(mesh.morphTargetDictionary).find(n => n === selectedMorphName);
    if (exactMatch) {
      const morphIndex = mesh.morphTargetDictionary[exactMatch];
      mesh.morphTargetInfluences[morphIndex] = 1.0;
    } else {
      Object.keys(mesh.morphTargetDictionary).forEach(morphName => {
        if (morphName.includes(selectedMorphName) || selectedMorphName.includes(morphName)) {
          mesh.morphTargetInfluences[mesh.morphTargetDictionary[morphName]] = 1.0;
        }
      });
    }
  });
});
```

---

## 9. 总览与调用关系图

### 9.1 组件层级

```
App.jsx
  └─ state: currentEmotion, emotionIntensity, isSpeaking, currentSpeechText, ...
  └─ guessEmotionFromMotion, detectEmotionFromText, setCurrentEmotion, setEmotionIntensity
  └─ VRMViewer (props: emotion, emotionIntensity, isSpeaking, currentSpeechText, ...)
       ├─ VRMModel (url, emotion, emotionIntensity, isSpeaking, currentSpeechText, ...)
       │    └─ useFrame: 情感应用、口型、眨眼、GPT 表情（注释）
       └─ MMDModel (url, isSpeaking, currentSpeechText, ...)
            └─ useFrame: 口型、GPT 发话表情；initializeMmdLipSyncTargets
```

### 9.2 数据与调用汇总表

| 模块 | 数据从哪来 | 谁消费 | 主要 API/存储 |
|------|------------|--------|----------------|
| VRM 表情名/列表 | VRM 文件 → expressionManager | VRMModel 初始化、GPT 块（注释） | expressionManager.expressions / expressionMap |
| VRM 情感权重 | App：guessEmotionFromMotion / detectEmotionFromText | VRMModel useFrame | setCurrentEmotion, setEmotionIntensity → applyEmotion → setValue |
| VRM 口型 | 内部节奏（0.15s） | VRMModel useFrame | setValue('aa'/'oh', mouthValue) |
| VRM 眨眼 | 内部定时（2–5s + 100ms） | VRMModel useFrame、IdleMotion | setValue('blink', 1/0) |
| VRM 发话表情 | AI（若启用） | VRMModel useFrame（注释） | generateExpressionParams → gptExpressionParams.current → setValue |
| MMD 顶点表情名 | PMX → MMDLoader → morphTargetDictionary | MMDModel traverse | Object.keys(morphTargetDictionary) |
| MMD 材质表情 | PMX → parsePMXMaterialMorphs | MMDModel load、发话表情应用 | window._mmdMaterialMorphs, applyMaterialMorph |
| MMD 口型 | morphTargetDictionary 母音候选 | initializeMmdLipSyncTargets + useFrame | mmdLipSyncRef.current.targets, influences[index] |
| MMD 发话表情 | morphTargetDictionary + _mmdMaterialMorphs；AI 选名 | MMDModel useFrame | generateExpressionParams → mmdGptExpressionParamsRef.current → 顶点/材质应用 |

### 9.3 关键文件索引

- **VRM 加载与表情应用**：`src/components/VRMViewer.jsx`（VRMModel：约 324–1920，useFrame 约 1540–1800）。
- **VRM 表情常量与眨眼**：`src/utils/vrmMotions.js`（VRM_EXPRESSIONS、IdleMotion.applyBlink）。
- **情感推断**：`src/App.jsx`（guessEmotionFromMotion、detectEmotionFromText、setCurrentEmotion 多处）。
- **MMD 加载、口型、发话表情**：`src/components/VRMViewer.jsx`（MMDModel：约 1923–3200，initializeMmdLipSyncTargets 约 2145–2207，useFrame 口型与表情约 4850–5185）。
- **PMX 材质 morph**：`src/utils/pmxParser.js`（parsePMXMaterialMorphs、readMorph type===8）。
- **AI 表情参数**：`src/services/aiService.js`（generateExpressionParams 约 1136–1246）。

---

## 10. 详细调用链（按场景）

### 10.1 VRM 情感从「动作名」到「表情」的完整调用链

```
App.jsx
  playFallbackMotion() / updateIdleEmotion()
    → pickFallbackMotion(null) 得到 targetMotion（如 'happy_walk'）
    → guessEmotionFromMotion(targetMotion) 得到 resolvedEmotion（如 'happy'）
    → setCurrentEmotion(resolvedEmotion)
    → setEmotionIntensity(intensity)

VRMViewer 渲染
  → emotion={currentEmotion} emotionIntensity={emotionIntensity} 传给 VRMModel

VRMModel
  useEffect([emotion])
    → currentEmotion.current = emotion
    → targetEmotionValue.current = 1（或保持）
  useFrame(delta)
    → emotionValue = targetEmotionValue.current * amplifiedIntensity
    → applyEmotion(currentEmotion.current, emotionValue)
      → expressionManager.setValue('happy', value) 等
```

### 10.2 VRM 情感从「AI 回复文本」到「表情」的调用链

```
App.jsx（AI 回复处理处）
  → detectedEmotion = detectEmotionFromText(assistantMessage)
  → finalEmotion = detectedEmotion || fallbackEmotion
  → setCurrentEmotion(finalEmotion)
  → setEmotionIntensity(intensity)

（之后与 10.1 相同：VRMViewer → VRMModel → useFrame → applyEmotion → setValue）
```

### 10.3 MMD 发话表情从「文本」到「morph 应用」的完整调用链

```
App.jsx
  → currentSpeechText 更新（TTS 要说的内容）
  → VRMViewer 将 isSpeaking、currentSpeechText 传给 MMDModel

MMDModel useFrame
  条件：currentSpeechText !== mmdLastSpeechTextRef.current
    → availableExpressions = []；mesh.traverse 收集 Object.keys(child.morphTargetDictionary)
    → aiService.generateExpressionParams(textToGenerate, availableExpressions)
        → GPT 返回 JSON 数组，parse 得到 ['笑い','困る'] 等
    → mmdGptExpressionParamsRef.current = selectedMorphs

MMDModel useFrame（同一帧或后续帧，isSpeaking && lipSyncTargets.length > 0）
  → morphsToApply = 规则过滤后的 mmdGptExpressionParamsRef.current
  → lipSyncTargets.forEach(target => {
       morphsToApply.forEach(selectedMorphName => {
         若 window._mmdMaterialMorphs 中有同名 → applyMaterialMorph(mesh, materialMorph, 1.0)
         否则 mesh.morphTargetDictionary 完全/部分匹配 → mesh.morphTargetInfluences[index] = 1.0
       })
     })
```

### 10.4 MMD 口型从「初始化」到「每帧写入」的调用链

```
MMDModel 模型/动画加载完成
  → initializeMmdLipSyncTargets(mesh)
    → root.traverse(child => { 用 vowelCandidates 在 child.morphTargetDictionary 中找 a/i/u/e/o })
    → mmdLipSyncRef.current = { targets: [{ mesh, influences, vowels, vowelNames }] }

MMDModel useFrame（isSpeaking && mesh）
  → lipSyncTargets = mmdLipSyncRef.current.targets
  → 每 0.08–0.16s 切换开/闭，开口时按权重选 state.currentVowel（a/i/u/e/o）
  → mouthValue = ease-in-out 插值
  → lipSyncTargets.forEach(target => {
       influences[vowels[vowel].index] = 0（先清零所有母音）
       influences[vowels[state.currentVowel].index] = mouthValue
     })
```

---

## 11. 参数速查表

| 参数/变量 | 类型 | 所属 | 说明 |
|-----------|------|------|------|
| `emotion` | string | App → VRMViewer → VRMModel | 情感键：neutral \| happy \| sad \| angry \| surprised \| thinking \| sleeping |
| `emotionIntensity` | number | 同上 | 0–1，情感强度 |
| `currentSpeechText` | string | App → VRMViewer → VRMModel/MMDModel | 当前要说的文本（TTS） |
| `isSpeaking` | boolean | 同上 | 是否正在说话 |
| `expressionManager.setValue(name, value)` | (string, number) => void | VRM | 设置单个 BlendShape 权重 |
| `morphTargetDictionary` | Record<string, number> | MMD mesh | 表情名 → morph 索引 |
| `morphTargetInfluences` | number[] | MMD mesh | 索引 → 权重 0–1 |
| `availableExpressions` | string[] | 临时变量 | 从 expressionManager 或 morphTargetDictionary 收集的名字列表 |
| `gptExpressionParams.current` (VRM) | object \| null | VRMModel ref | 若启用：期望 { [expName]: number }，AI 实际返回数组需转换 |
| `mmdGptExpressionParamsRef.current` (MMD) | string[] \| null | MMDModel ref | GPT 返回的 1–3 个表情名 |
| `mmdLipSyncRef.current.targets` | Array<{ mesh, influences, vowels, vowelNames }> | MMDModel ref | 口型目标列表 |
| `window._mmdMaterialMorphs` | Array<{ name, type, elements }> | 全局 | PMX 材质 morph 列表 |

---

## 12. 关键函数签名与参数一览

| 函数/API | 文件 | 签名/参数 | 返回值 |
|----------|------|-----------|--------|
| `guessEmotionFromMotion` | App.jsx | `(motionName?: string) => string` | 'happy' \| 'sad' \| 'thinking' \| 'surprised' \| 'angry' \| 'sleeping' \| 'neutral' |
| `detectEmotionFromText` | App.jsx | `(text?: string) => string \| null` | 同上情感键或 null |
| `setCurrentEmotion` | App.jsx | `(emotion: string) => void` | — |
| `setEmotionIntensity` | App.jsx | `(intensity: number) => void` | — |
| `expressionManager.setValue` | @pixiv/three-vrm | `(name: string, value: number) => void` | — |
| `VRM_EXPRESSIONS` | vrmMotions.js | 常量对象 `Record<string, string>` | — |
| `IdleMotion.applyBlink` | vrmMotions.js | `(deltaTime: number) => void` | — |
| `generateExpressionParams` | aiService.js | `async (context: string, availableExpressions: string[]) => string[] \| null` | 1–3 个表情名或 null |
| `parsePMXMaterialMorphs` | pmxParser.js | `async (url: string) => Promise<Array<{ name, nameEn, panel, type, elements }>>` | 仅 type===8 的 morph |
| `PMXParser.readMorph` | pmxParser.js | `() => morph \| null` | type===8 时返回 { name, nameEn, panel, type, elements } |
| `initializeMmdLipSyncTargets` | VRMViewer.jsx | `(root: Object3D) => void` | 写入 mmdLipSyncRef.current |
| `applyMaterialMorph` | VRMViewer.jsx（MMDModel 内） | `(mesh, mmdMorph, weight: number) => boolean` | 是否应用成功 |
| `getBoneCategory` | VRMViewer.jsx | `(boneName?: string) => string` | 'intimate' \| 'head' \| 'shoulder' \| 'arm' \| 'leg' \| 'default' |
| `BODY_PART_REACTIONS` | VRMViewer.jsx | 常量：`Record<string, Array<{ name, params }>>` | 部位 → 表情选项，params 为 { blink, ih, ou, aa, oh 等 } |

**VRMViewer 组件签名（forwardRef）：**

```javascript
const VRMViewer = forwardRef(({
  modelUrl,
  modelType = 'auto',
  onMotionReady,
  enableMouseFollow = true,
  enableInteraction = true,
  emotion = 'neutral',
  emotionIntensity = 0.5,
  isTyping = false,
  gesture = null,
  isSpeaking = false,
  currentSpeechText = '',
  cameraConfig = { position: [0, 1.4, 2.5], fov: 50, lookAt: [0,1,0] },
  manualCamera = true,
  onCameraChange,
  mmdFileMap,
  mmdVmdUrls = [],
  mmdTapMotionUrls = [],
  mmdPetMotionUrls = [],
  onMmdAnimationDuration,
  onInteraction,
  onMmdInteractionMotion,
  enableCameraFollow = false,
  enableManualCamera = true,
  mmdTargetLoopCount = 3,
  onMmdLoopComplete,
  overlayBlendRatio = 1.0,
  enablePhysics = true,
  enablePmxAnimation = false,
  enableSimplePhysics = false,
  mmdScale = 0.09,
  vrmScale = 1.0,
  mmdShininess = 50,
  mmdBrightness = 1.0,
  parentClonedMeshRef,
  aiStatus = 'not-initialized'
}, ref) => { ... });
```

**BODY_PART_REACTIONS 结构（VRMViewer.jsx，用于点击部位反应用表情）：**

```javascript
const BODY_PART_REACTIONS = {
  intimate: [
    { name: 'shy', params: { blink: 0.7, ih: 0.2 } },
    { name: 'angry', params: { blink: 0.3, ou: 0.5 } },
    { name: 'surprised', params: { aa: 0.6, blink: 0.0 } }
  ],
  head: [
    { name: 'happy', params: { ih: 0.6, blink: 0.3 } },
    { name: 'shy', params: { blink: 0.9, ih: 0.3 } }
  ],
  shoulder: [ { name: 'neutral', params: { blink: 0.2 } } ],
  arm: [
    { name: 'happy', params: { ih: 0.5, blink: 0.2 } },
    { name: 'surprised', params: { aa: 0.4, oh: 0.2 } }
  ],
  leg: [
    { name: 'surprised', params: { aa: 0.5, oh: 0.3 } },
    { name: 'confused', params: { ou: 0.3, blink: 0.4 } }
  ],
  default: [ { name: 'neutral', params: { blink: 0.2 } } ]
};
```

---

*文档版本：基于当前代码库整理，若实现有变更请以源码为准。*
