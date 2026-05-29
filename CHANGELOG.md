# 坦克大战 修复日志

## Commit: 2029049 — fix: 修复移动速度、AI行为和添加开始界面

---

### 一、修复移动速度过快的 bug

**问题描述**: 玩家坦克移动速度极快（约 1.9 格/帧，每秒可穿越整个地图上百次），加上网格吸附逻辑，导致移动手感极差，难以精确控制。

**根因分析**:
- 原速度值 `0.12` 直接乘以 `dt`（约 16ms），即 `0.12 × 16 ≈ 1.92` 格/帧
- 坦克位置存储的是格子坐标（非像素），所以每帧移动近 2 格
- 网格吸附 `Math.round(x * 2) / 2` 将位置强制对齐到 0.5 格单位，造成抖动

#### 1.1 降低基础速度

**文件**: `tank-battle.html` 第 237 行 (原第 185 行)

```diff
// createTank() 函数内
- speed: isPlayer ? 0.12 : 0.025 + wave * 0.003,
+ speed: isPlayer ? 0.0055 : 0.002 + wave * 0.0005,
```

- 玩家速度: `0.12` → `0.0055`（约 0.09 格/帧，5.3 格/秒），降低约 22 倍
- 敌人速度: `0.025` → `0.002`（约 0.03 格/帧，2 格/秒），降低约 12 倍
- 敌速随波次增长也从 `0.003/波` → `0.0005/波`，避免后期敌人速度失控

#### 1.2 移除网格吸附，改为平滑移动 + 贴墙滑动

**文件**: `tank-battle.html` 第 387-399 行 (原第 334-350 行)

原代码:
```javascript
// 原：先尝试移动, 然后强制吸附到 0.5 格子单位
if (isWalkable(nx, ny, player)) {
    const clampedX = DX[player.dir] === 0
        ? Math.round(player.x * 2) / 2 : nx;  // 不在移动轴上的坐标强制吸附
    const clampedY = DY[player.dir] === 0
        ? Math.round(player.y * 2) / 2 : ny;
    if (isWalkable(clampedX, clampedY, player)) {
        player.x = clampedX;
        player.y = clampedY;
    }
    moved = true;
}
```

新代码:
```javascript
// 新：平滑移动, 碰到墙时沿墙滑动（只执行可行的轴向移动）
if (isWalkable(nx, ny, player)) {
    player.x = nx;
    player.y = ny;
} else if (DX[player.dir] !== 0 && isWalkable(nx, player.y, player)) {
    // 水平移动被墙挡住 → 尝试仅水平滑动
    player.x = nx;
} else if (DY[player.dir] !== 0 && isWalkable(player.x, ny, player)) {
    // 垂直移动被墙挡住 → 尝试仅垂直滑动
    player.y = ny;
}
```

**功能说明**: 新方案允许坦克贴墙滑行 — 当斜向顶住墙壁时，玩家仍然可以沿墙壁方向滑动而不是完全卡住。这在狭窄通道中尤其有用。

#### 1.3 更新道具系统中的速度上限

**文件**: `tank-battle.html` 第 556 行 (原第 512 行) 和 第 571 行 (原第 524 行)

```diff
// 拾取加速道具
- if (pu.type === 'speed') player.speed = Math.min(0.22, player.speed + 0.03);
+ if (pu.type === 'speed') player.speed = Math.min(0.009, player.speed + 0.002);

// 道具过期恢复
- player.speed = 0.12;
+ player.speed = 0.0055;
```

极速上限从 `0.22`（旧）降到 `0.009`（新），道具加速幅度从 `+0.03` 降到 `+0.002`，与新的基础速度相匹配。

---

### 二、修复 AI 坦克只在玩家周围移动的 bug

**问题描述**: 所有敌方坦克都围在玩家附近，不会在地图其他地方巡逻，导致游戏体验单一。

**根因分析**:
- AI 每 40-120ms 重新决策方向，**60% 概率**朝向玩家
- 敌人没有巡逻概念，只有"追向玩家"和"随机方向"两种状态
- 射击条件中 `canSee` 范围是 12 格（覆盖地图约一半），导致敌人频繁射击

#### 2.1 重写 AI 方向决策

**文件**: `tank-battle.html` 第 416-432 行 (原第 361-381 行)

原代码:
```javascript
// 原：无论距离，60% 朝向玩家
e.aiTimer -= dt;
if (e.aiTimer <= 0) {
    e.aiTimer = 40 + Math.random() * 80;
    const dx = player.x - e.x;
    const dy = player.y - e.y;
    if (Math.random() < 0.6) {
        if (Math.abs(dx) > Math.abs(dy)) {
            e.dir = dx > 0 ? DIR.RIGHT : DIR.LEFT;
        } else {
            e.dir = dy > 0 ? DIR.DOWN : DIR.UP;
        }
    } else {
        e.dir = Math.floor(Math.random() * 4);
    }
}
```

新代码:
```javascript
// 新：仅距离 < 8 格时 35% 追向玩家，其余时间随机巡逻
e.aiTimer -= dt;
if (e.aiTimer <= 0) {
    e.aiTimer = 50 + Math.random() * 120;
    const dx = player.x - e.x;
    const dy = player.y - e.y;
    const distToPlayer = Math.abs(dx) + Math.abs(dy);

    if (distToPlayer < 8 && Math.random() < 0.35) {
        // 近距离：35% 概率追向玩家
        if (Math.abs(dx) > Math.abs(dy)) {
            e.dir = dx > 0 ? DIR.RIGHT : DIR.LEFT;
        } else {
            e.dir = dy > 0 ? DIR.DOWN : DIR.UP;
        }
    } else if (Math.random() < 0.7) {
        // 远距离或不追击时：70% 随机方向，30% 保持当前方向
        e.dir = Math.floor(Math.random() * 4);
    }
}
```

**功能说明**:
- **距离判断** (`distToPlayer < 8`): 只有距离玩家 8 格曼哈顿距离内才可能追击
- **追击概率** (`35%`): 即使在范围内，也只有 35% 概率面向玩家，避免全部围拢
- **巡逻漫游**: 70% 概率随机转向，30% 保持当前方向继续前进，模拟真实巡逻行为
- **决策间隔** 从 40-120ms 延长到 50-170ms，减少频繁转向

#### 2.2 改进墙壁碰撞处理

**文件**: `tank-battle.html` 第 435-442 行 (原第 383-391 行)

```diff
  if (isWalkable(nx, ny, e)) {
      e.x = nx;
      e.y = ny;
  } else {
-     e.aiTimer = 0; // change direction next frame
+     e.dir = Math.floor(Math.random() * 4);  // 碰墙立即随机转向
  }
```

原逻辑将 `aiTimer` 置零，下一帧才决定新方向，导致坦克可能在墙前短暂停顿。新逻辑碰墙后立即随机转向，移动更流畅。

#### 2.3 调整射击行为

**文件**: `tank-battle.html` 第 445-462 行 (原第 393-413 行)

```diff
  // 面向玩家条件不变，但用 distToPlayer < 10 替代 canSee < 12
- const canSee = Math.abs(dx) < 12 && Math.abs(dy) < 12;
- if (facingPlayer && canSee && Math.random() < 0.6) {
+ if (facingPlayer && distToPlayer < 10 && Math.random() < 0.5) {
      fireBullet(e);
      e.shootCooldown = e.shootRate;
- } else if (Math.random() < 0.1) {   // 盲射 10%
+ } else if (Math.random() < 0.06) {  // 盲射 6%
      fireBullet(e);
      e.shootCooldown = e.shootRate;
  }
```

- 面向玩家射击概率: `60%` → `50%`
- 随机盲射概率: `10%` → `6%`
- 检测范围用曼哈顿距离 (`dx + dy`) 替代矩形范围，更合理

---

### 三、添加游戏开始界面

**问题描述**: 游戏打开后直接开始战斗，没有准备时间，也没有操作说明，新玩家不知道如何操作。

#### 3.1 添加 CSS 样式

**文件**: `tank-battle.html` 第 39-110 行 (原第 39-73 行)

新增样式:
```css
#startScreen, #gameOver {
    /* 开始界面和结束界面共享基础样式 */
    position: absolute;
    inset: 0;
    display: none;          /* 默认隐藏 */
    flex-direction: column;
    justify-content: center;
    align-items: center;
    background: rgba(0,0,0,0.88);
}
#startScreen {
    display: flex;          /* 开始时显示 */
}
#startScreen h1 {
    color: #4af;            /* 蓝色标题 */
    font-size: 42px;
    text-shadow: 0 0 30px rgba(68,170,255,0.7);  /* 蓝色发光 */
    letter-spacing: 6px;    /* 字符间距 */
}
#startScreen .controls {
    border: 1px solid #333; /* 操作说明边框 */
    padding: 16px 24px;
    line-height: 2;         /* 双倍行距，易读 */
}
#startScreen button {
    animation: pulse 2s infinite;  /* 按钮呼吸动画 */
}
```

#### 3.2 添加 HTML 结构

**文件**: `tank-battle.html` 第 127-135 行

```html
<div id="startScreen">
    <h1>坦 克 大 战</h1>
    <div class="subtitle">TANK BATTLE</div>
    <div class="controls">
        <span>W A S D</span> 或 <span>↑ ↓ ← →</span> 移动<br>
        <span>空格</span> 或 <span>J</span> 射击
    </div>
    <button id="startBtn" onclick="startGame()">▶  开 始 游 戏</button>
</div>
```

#### 3.3 添加 `gameStarted` 状态和 `startGame()` 函数

**文件**: `tank-battle.html` 第 165 行

```diff
+ let gameStarted = false;   // 游戏是否已开始
```

**文件**: `tank-battle.html` 第 761-779 行 (新增函数)

```javascript
function startGame() {
    initAudio();                              // 初始化音频上下文
    gameStarted = true;
    gameOver = false;
    score = 0; kills = 0; lives = 3; wave = 1;
    bullets = []; particles = []; enemies = []; powerUps = [];
    document.getElementById('startScreen').style.display = 'none';  // 隐藏开始界面
    document.getElementById('gameOver').style.display = 'none';     // 隐藏结束界面
    generateMap();
    initPlayer();
    spawnWave();
    updateUI();
    lastTime = 0;
}
```

#### 3.4 修改 `update()` 入口判断

**文件**: `tank-battle.html` 第 374 行 (原第 321 行)

```diff
- if (gameOver || !player) return;
+ if (gameOver || !gameStarted || !player) return;
```

游戏循环始终运行（用于渲染），但 `update()` 在 `gameStarted === false` 时跳过所有逻辑，确保开始前没有任何游戏行为。

#### 3.5 修改初始化流程

**文件**: `tank-battle.html` 第 802-804 行 (原第 728-734 行)

```diff
- // Init
- generateMap();
- initPlayer();
- spawnWave();
- updateUI();
+ // Init: start the render loop but wait for start screen
  lastTime = 0;
  gameLoopId = requestAnimationFrame(gameLoop);
```

原来自动调用 `generateMap()` / `initPlayer()` / `spawnWave()` 直接开战，现在只启动渲染循环，等待用户点击"开始游戏"按钮。

---

### 修改文件清单

| 文件 | 修改类型 | 说明 |
|------|----------|------|
| `tank-battle.html` | 修改 | 上述所有改动集中在此文件 |
| `CHANGELOG.md` | 新增 | 本文档 |

### 统计数据

- **新增行数**: 103 行
- **删除行数**: 37 行
- **净增**: +66 行
