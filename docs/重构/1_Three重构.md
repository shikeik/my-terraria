## 迁移步骤清单（my-terraria 从 v0 开始）

目录：`/mnt/agents/output/my-terraria/`
文件：`my-terraria_v0.html`, `v1`, `v2`...

---

### 阶段一：基础骨架（先跑起来）

| 步骤 | 版本 | 目标 | 验证标准 |
|------|------|------|---------|
| **1** | **v0** | Three.js 基础环境 + OrthographicCamera + 左下坐标系证明 | 屏幕显示网格/坐标轴，y 向上为正，红色方块在 (5,5) 确实出现在左上方 |
| **2** | **v1** | 方块注册表 + InstancedMesh 静态地形 + 笛卡尔世界生成 | 世界生成完成，200×100 地形渲染，相机可拖动浏览，帧率稳定 |
| **3** | **v2** | 玩家 Group 绘制 + 相机跟随 + 方向键移动（无碰撞） | 玩家出现在重生点，相机跟随，方向键可左右移动+跳跃（无重力无碰撞） |

---

### 阶段二：物理与交互（核心玩法）

| 步骤 | 版本 | 目标 | 验证标准 |
|------|------|------|---------|
| **4** | **v3** | 物理引擎（y-up 重力、碰撞检测、自动台阶、平台、水中物理） | 玩家受重力下落，碰到地形停止，能跳跃，水中减速，平台可站/可下跳 |
| **5** | **v4** | 输入系统（Pointer Events + Raycaster）+ 长按连续挖掘/放置 | 点击地形能挖/放，长按连续操作，方块选择器切换类型生效 |
| **6** | **v5** | 水体 Shader 动画 + 平台 ShapeGeometry 圆角 + 星星背景 | 水有波浪动画，平台有圆角木板效果，夜空有星星视差 |

---

### 阶段三：UI 与 polish（收尾）

| 步骤 | 版本 | 目标 | 验证标准 |
|------|------|------|---------|
| **7** | **v6** | 高亮框 + 方块选择器 UI + 信息面板 + 动作按钮 | 鼠标悬停高亮，UI 按钮正常工作，显示坐标 |
| **8** | **v7** | 全屏切换 + 内置控制台嫁接 + 配置暴露 (`window.__CONFIG`) | 全屏/横屏锁定可用，控制台可执行 JS，可调重力/移速/跳跃力 |
| **9** | **v8** | 性能优化 + 移动端适配 + 代码清理 | 手机流畅运行，无发热，包体合理 |

---

## 现在讨论 v0：基础环境 + 坐标系证明

### v0 具体做什么？

1. **单文件 HTML**，通过 CDN 引入 Three.js (module)
2. 创建 `Scene` + `OrthographicCamera` + `WebGLRenderer`
3. 设置相机为**左下原点**：世界坐标 `(x, y)` 中 `y=0` 在底部，`y` 增大向上
4. 画一个**网格辅助线**（GridHelper）证明坐标方向
5. 在 `(5, 5)` 和 `(5, 10)` 放置**红色方块**，肉眼确认 y 向上
6. 支持**鼠标拖拽平移相机**（方便浏览验证）

### v0 不做的事：
- 不接入世界生成
- 不做玩家
- 不做物理
- 不做输入
- 不做 UI

### 关键技术点确认：

**Q1: Three.js 版本？**
- 推荐 `r160`（最新稳定版），CDN: `https://unpkg.com/three@0.160.0/build/three.module.js`

**Q2: 坐标系具体怎么设？**
- `PlaneGeometry` 默认中心在原点，需要 `geometry.translate(TS/2, TS/2, 0)` 让左下角对齐
- 相机 `OrthographicCamera(left, right, top, bottom, near, far)` —— Three.js 的 `top` 是上边界（y 大），`bottom` 是下边界（y 小），这和我们要的左下原点天然一致

**Q3: 背景天空？**
- v0 先简单用 `scene.background = new THREE.Color(0x1a1a3e)`，后续 v5 再换渐变/星星

**Q4: 画布尺寸？**
- `renderer.setSize(window.innerWidth, window.innerHeight)`
- 相机视口随窗口 resize 调整

---

## 确认问题

1. **步骤拆分是否合理？** 有没有想合并或拆更细的？
2. **v0 范围是否OK？** 只验证坐标系+基础环境，不做其他功能
3. **Three.js 版本 r160 是否接受？**
4. **v0 需要鼠标拖拽相机吗？** 还是固定相机即可？

确认后我直接写 `my-terraria_v0.html`。

---
---
---
------
---
------
---
---
------
---
---
------
---
------
---
---
---
------
---
---
---
---
---
---
---
---
---
好的，老师。这是一个**系统性重构**，涉及坐标系翻转 + 渲染引擎替换，必须零遗漏。我先完整列出计划，你确认后我再动手。

---

## 一、核心策略确认

**目标**：内部逻辑与渲染全部改为 **左下坐标系**（y=0 在底部，y 向上为正），使用 **Three.js OrthographicCamera 2D** 渲染，保持所有玩法功能不变。

**关键决策**：
1. **世界数组语义翻转**：`world[x][y]` 中 `y=0` 改为地底，`y=HEIGHT-1` 为天空（与左下坐标系一致）。
2. **玩家 `p.y` 定义翻转**：从"头顶 Y"改为**脚底 Y**（左下坐标系的自然定义）。
3. **重力/跳跃符号翻转**：重力向下拉（`vy -= G*dt`），跳跃向上推（`vy = +JUMP`）。
4. **Three.js 渲染器透明**：天空渐变保留 CSS 背景，Three.js 只渲染游戏实体（地形、水、玩家、高亮），降低复杂度。
5. **地形用 InstancedMesh**：200×100=2 万个方块，6 种固体类型各一个 InstancedMesh，性能最优。
6. **平台简化**：当前世界生成中无原生平台（仅玩家放置），平台用独立 `Mesh` + `BoxGeometry(32,8,1)`，放弃圆角（或后期用纹理贴图还原），避免自定义几何体拖慢进度。

---

## 二、逐模块改动清单（保证无遗漏）

### Module: Config
- `RESPAWN.y`：当前 `13`（从顶部向下），改为 `WORLD_HEIGHT - 1 - 13 = 86`（从底部向上）。
- `GRAVITY` / `JUMP_FORCE` / `MOVE_SPEED`：数值大小不变，**物理模块内部处理符号**。
- `WATER.BUOYANCY`：方向翻转（向上为正）。

### Module: World
- `init()`：生成逻辑保持，但**存储时翻转 y 索引**：
  ```js
  const flippedY = CONFIG.WORLD_HEIGHT - 1 - y;
  State.world[x][flippedY] = blockType;
  ```
- `placeStructure()`：矩阵行索引翻转（第 0 行对应世界最底部）。
- `getBlock/setBlock/isSolid`：直接操作已翻转后的数组，无需额外转换。

### Module: Physics（核心，最易出错）
- `checkCollision(px, py, pw, ph)`：
  - `py` 现为脚底 Y。
  - AABB 检测改为统一数学形式（与坐标方向无关）：
    - 玩家：`[px, px+pw] × [py, py+ph]`
    - 方块：`[tx*ts, (tx+1)*ts] × [ty*ts, (ty+1)*ts]`
    - 重叠条件：`maxX1 < minX2 && maxY1 < minY2`（标准 AABB）。
- `update(dt)`：
  - 重力：`p.vy -= CONFIG.GRAVITY * dt;`
  - 跳跃：`p.vy = CONFIG.JUMP_FORCE;`（`JUMP_FORCE` 改为正值，如 `660`）
  - 水平碰撞：台阶上升 `stepY = p.y + TILE_SIZE`（向上为正）。
  - 垂直碰撞：落地判断 `if (p.vy < 0) p.onGround = true;`（向下速度为负）。
  - 平台 Snap：玩家从上方（更大 Y）下落穿过平台顶部时吸附：
    - 平台顶部 Y = `(footY + 1) * TILE_SIZE`（格子顶部，即最大 Y）。
    - 上一帧脚底 `prevBottom = (p.y - p.vy * dt)`。
    - 若 `prevBottom >= platTop && p.y < platTop`，则 `p.y = platTop`。
  - 水中浮力：方向全部翻转（上正下负）。
  - 下跳逻辑：保持不变（按键触发穿透平台）。

### Module: Camera
- `update()`：`c.x, c.y` 表示视口**左下角**世界坐标。
  ```js
  c.x = p.x + p.w / 2 - cw / 2;
  c.y = p.y + p.h / 2 - ch / 2;
  ```
- `screenToWorld(sx, sy)`：
  ```js
  worldX = c.x + sx;
  worldY = c.y + (ch - sy); // 屏幕 Y=0 是顶部，对应更大世界 Y
  ```

### Module: Renderer（完全重写为 Three.js）
- **初始化**：
  - `Scene`, `OrthographicCamera(0, w, h, 0, 1, 1000)`, `WebGLRenderer({ alpha: true })`
  - 相机 `position.z = 10`
- **地形 InstancedMesh**：
  - 为 `dirt, stone, grass, sand, wood` 各创建 `InstancedMesh(BoxGeometry(32,32,1), MeshBasicMaterial, maxCount)`
  - 初始化遍历世界，为每个非空气/非水方块设置 `setMatrixAt`
  - 挖掘时：`setMatrixAt(index, zeroMatrix)` + `instanceMatrix.needsUpdate = true`
  - 放置时：设置对应类型 InstancedMesh 的矩阵
- **水 InstancedMesh**：
  - `ShaderMaterial`，vertex shader 做 `sin(time + x)` 波动，fragment shader 做透明蓝 + 高光。
- **平台 Mesh**：
  - 玩家放置时动态创建 `Mesh(BoxGeometry(32, 8, 1))`，y 位置为 `(y+1)*TILE_SIZE - 4`（格子顶部 8px）。
- **玩家 Group**：
  - `Group` 包含：
    - 身体：`BoxGeometry(24, 32, 1)`，相对 Y = 16（脚底向上 16+16？精确计算见实施）
    - 头：`BoxGeometry(20, 16, 1)`，相对 Y = 40
    - 脚×2：`BoxGeometry(8, 8, 1)`，相对 Y = 4
    - 工具 Group：根据 `facingRight` rotateZ。
  - 每帧更新 Group.position.set(p.x, p.y, 0)。
- **高亮框**：
  - `LineSegments(EdgesGeometry(BoxGeometry(32,32,1)))`，位置 `(wx*ts+16, wy*ts+16, 0)`。
- **天空/星星**：
  - 保留 CSS 渐变背景，Three.js 透明渲染。
  - 星星可后期用 `THREE.Points` 补充，暂不阻塞核心重构。

### Module: Input
- 画布 `pointerdown/move/up`：坐标转换改为左下公式（或直接用 Raycaster 投射到 z=0 平面）。
- `doRepeatAction()`：放置时的玩家 AABB 碰撞检测使用新坐标系。
- 方向键/跳跃/动作按钮：无改动。

### Module: Fullscreen / Console / UI
- **零改动**，完全保持现有 DOM 和事件逻辑。

---

## 三、关键风险与我的建议

| 风险点 | 影响 | 对策 |
|--------|------|------|
| **平台 Snap 逻辑翻转错误** | 玩家穿模或无法站立 | 我会单独加生命周期 log 验证 `prevBottom` 与 `platTop` 的数值关系 |
| **水 Shader 复杂度** | 可能拖慢移动端 | 先实现基础透明蓝水，波动作为可选后加 |
| **InstancedMesh 动态更新** | 挖掘/放置时实例矩阵更新遗漏 | 维护 `world[x][y]` → `instanceId` 的映射表，确保一一对应 |
| **玩家绘制部件 Y 偏移算错** | 玩家身体与碰撞盒错位 | 我会用红色线框绘制碰撞盒辅助验证 |

---

## 四、实施步骤（等你确认后执行）

1. **创建 `my-terraria_v0.html`**（新目录，干净上下文）。
2. **Phase 1**：引入 Three.js CDN + 初始化 Scene/Camera/Renderer。
3. **Phase 2**：世界生成 + 数组翻转 + 地形 InstancedMesh 初始化。
4. **Phase 3**：物理引擎坐标系改造（重力、碰撞、台阶、平台、水中）。
5. **Phase 4**：玩家 Group + 相机跟随 + 高亮框。
6. **Phase 5**：输入坐标转换 + 挖掘放置。
7. **Phase 6**：水 Shader + 平台渲染。
8. **Phase 7**：联调测试，输出下载链接。

---

**请确认以下两点我再开工**：

1. **平台圆角是否简化？** 当前 Canvas 2D 画了圆角平台，Three.js 中做圆角 InstancedMesh 很繁琐。我建议先改为矩形平台（视觉上仍是顶部 8px 横条，只是直角），功能完全一致，后期可贴图美化。你是否接受？
2. **水波动实现方式？** 方案 A：Shader（效果好，代码稍多）；方案 B：纯色透明 InstancedMesh 不做波动（最稳，先保功能）。你倾向哪个？

确认后我立即动手，每一步都有日志验证。