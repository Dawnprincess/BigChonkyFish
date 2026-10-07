# 从 PSD2Live 导出 → BongoCat 可用模型：转换说明

> 适用：BongoCat 1.14.0（Windows，vladelaina 构建）+ PSD2Live 2.0.x
> 目标读者：需要把模型交付给别人的人，或需要执行这套流程的 AI。
> 本文只讲「导出之后怎么变成能用的模型」，不讲怎么建模/绑定。
> 文中的路径、命令、判定标准都是可复制执行的。

---

## 0. 三句话结论（先看这个）

1. **PSD2Live 导出的文件不能直接给 BongoCat 用**，必须包装成 **Mver 格式**：
   外面一层 `config.json`，模型文件放进 `img/standard/cat_model/`。
2. **`config.json` 必须声明键码**（`standard.hand` / `standard.keyboard`），否则敲键盘**完全没反应**。
3. 装到 `%LOCALAPPDATA%\BongoCat\models\<模型名>\` 后**必须完全重启 BongoCat**，
   判定成功的标准是日志里出现 `model-loaded`，且**没有** `Skipping installed model package`。

---

## 1. 为什么必须转换（原因 + 证据）

### 1.1 BongoCat 的模型库只自动识别 Mver 格式

**现象**：把 PSD2Live 导出的文件夹（含 `model.model3.json`）拖进 BongoCat 界面或直接放进模型库，
导入失败，提示「未找到支持的模型，或模型包文件不完整」。

**日志证据**（`%LOCALAPPDATA%\BongoCat\logs\BongoCat.log`）：

```
[WARN] Skipping installed model package: path=...\models\<模型名> error=Cannot inspect nearby model directory
```

**对照实验**（说明这不是"模型做得不对"）：

| 实验 | 结果 |
| --- | --- |
| 把 PSD2Live 导出的模型放进模型库 | ❌ `Cannot inspect nearby model directory` |
| 换英文文件夹名 / 换 Cubism 4.0 导出 / 只留最小文件 / 加表情 / 去物理 / 去动作 / 缩小贴图 | ❌ 全部同样失败 |
| 把**正版第三方 Live2D 模型**（VTube Studio 用）放进模型库 | ❌ 同样被拒 |
| 把一份**已有的 Mver 皮肤**复制进去 | ✅ 自动导入成功 |

**结论**：与该模型的具体内容无关；BongoCat 1.14 的模型库**只自动导入 Mver 格式**
（外有 `config.json`，内有 `img/standard/cat_model/…`）。

### 1.2 `config.json` 不声明键码 → 键盘毫无反应

**日志证据**：

```
[input] consumer model=<模型名> mode=standard drained_keys=3638 mapped_keys=0 unmapped_keys=3638
                                                        ↑ 0 = 一个键都没映射
```

对比一份正常工作的皮肤：`mapped_keys=532`。

**原因**：Mver 运行时靠 `standard.hand` / `standard.keyboard` 里的**键码数组**来决定"哪些键要响应"。
数组为空 = 不响应任何键。补上键码后同一模型的日志变成 `mapped_keys=1250  keyboard_overlays=625`。

### 1.3 参数必须"真正定义"过，否则导出会丢

**现象**：在 PSD2Live 里把网格绑到参数 `CatParamLeftHandDown`，参数面板里也能看到这条绑定，
但导出的 moc3 里**没有这个参数**，BongoCat 里当然不会动。

**原因**：只被网格"引用"、没有在参数面板**新建定义**的参数，导出时会被丢弃。

**验证方法**（任选）：

```powershell
# 方法 A：直接查 moc3 二进制里有没有这个参数名
python -c "b=open(r'<导出目录>\model.moc3','rb').read(); print('CatParamLeftHandDown' in b.decode('utf-8','ignore'))"
# 方法 B：查 cdi3（参数显示信息）
python -c "import json;d=json.load(open(r'<导出目录>\model.cdi3.json',encoding='utf-8-sig'));print([p['Id'] for p in d.get('Parameters',[])])"
```

---

## 2. 转换步骤

设：

- `$EXPORT` = PSD2Live 的导出目录，例如 `D:\BongoCat\成品\Test`
- `$PKG`    = 要生成的 Mver 包目录，例如 `D:\BongoCat\成品\Mver包_大肥鱼`
- `$NAME`   = 模型名（BongoCat 列表里显示的名字），例如 `大肥鱼`

### 步骤 A · 在 PSD2Live 里导出

`文件 → 导出`（`Ctrl+G`）→ **目标版本选 Cubism 5.0** → 导出到 `$EXPORT`。

导出会生成（✅ = 必须进包，❌ = 编辑用，可不进包）：

| 文件 | 说明 | 进包 |
| --- | --- | --- |
| `model.moc3` | 模型本体 | ✅ |
| `model.model3.json` | 模型清单（引用纹理/物理/动作） | ✅ |
| `model.cdi3.json` | 参数显示信息 | ✅ |
| `model.physics3.json` | 物理（头发摆动等） | ✅ |
| `model.*.motion3.json` | 动作（idle/blink/nod/shake…） | ✅ |
| `model.4096/texture_00.png` | 图集纹理 | ✅ |
| `model.cmo3` | Cubism 工程文件 | ❌ |
| `model.psd2live.json` | **导出报告**（含绘制顺序、每层 switchId） | ❌ |

### 步骤 B · 摆成 Mver 结构

```
$PKG\
├── config.json                       ← 步骤 C 手写
└── img\standard\cat_model\           ← 把上表 ✅ 的文件全部复制进来
    ├── model.moc3
    ├── model.model3.json
    ├── model.cdi3.json
    ├── model.physics3.json
    ├── model.idle.motion3.json  …（其他动作）
    ├── model.4096\texture_00.png
    └── resources\cover.png          ← 可选：模型列表缩略图（整画布渲染图）
```

### 步骤 C · 写 `config.json`

模板（键码见 §5.2）：

```json
{
  "decoration": {
    "window_size": [1024, 910],
    "l2d_offset": [0, 0],
    "l2d_correct": 2,
    "l2d_horizontal_flip": false,
    "l2d_auto_frame": true
  },
  "mode": 1,
  "standard": {
    "l2d": true,
    "hand":     [[65],[66],[67],[68],[69],[70],[71],[72],[73],[74],[75],[76],[77],[78],[79],[80],[81],[82],[83],[84],[85],[86],[87],[88],[89],[90],[48],[49],[50],[51],[52],[53],[54],[55],[56],[57],[32],[16],[17],[18],[13],[9],[8],[27],[20],[37],[38],[39],[40],[186],[187],[188],[189],[190],[191],[219],[220],[221],[222]],
    "mouse": false,
    "keyboard": [[65],[66],[67],[68],[69],[70],[71],[72],[73],[74],[75],[76],[77],[78],[79],[80],[81],[82],[83],[84],[85],[86],[87],[88],[89],[90],[48],[49],[50],[51],[52],[53],[54],[55],[56],[57],[32],[16],[17],[18],[13],[9],[8],[27],[20],[37],[38],[39],[40],[186],[187],[188],[189],[190],[191],[219],[220],[221],[222]],
    "face": [], "sounds": [], "l2d_expression": [], "l2d_motion": [],
    "l2d_motion_lockhand": [], "hand_offset": [0, 0]
  }
}
```

字段含义与取值规则：

| 字段 | 含义 | 取值规则 |
| --- | --- | --- |
| `decoration.window_size` | 模型画布尺寸 | **必须等于 moc3 的画布尺寸**（如 1024×910） |
| `decoration.l2d_correct` | 投影缩放修正 | 一般 2（参考已有皮肤的值） |
| `decoration.l2d_auto_frame` | 自动取景 | `true` |
| `standard.l2d` | 用 Live2D 模型 | `true` |
| `standard.hand` | **键码数组（左手/键盘）** | 每个元素是一个"和音"数组，如 `[65]`=A、`[16,65]`=Shift+A |
| `standard.keyboard` | 同上（键盘模式用） | 与 `hand` 保持一致即可 |
| `standard.mouse` | 是否用鼠标覆盖图 | 一般 `false` |
| `hand` 的第 i 项 | 对应 `img/standard/hand/i.png` 叠加图 | 没有该文件也能跑，只是第 i 个键没有叠加反馈 |

### 步骤 D · 安装

```powershell
# 1) 复制到模型库（目录名就是 BongoCat 里显示的名字）
Copy-Item '<$PKG>' "$env:LOCALAPPDATA\BongoCat\models\<$NAME>" -Recurse -Force

# 2) 完全退出 BongoCat（托盘右键退出，确认进程结束）再启动

# 3) 打开 模型管理，选中该模型
```

### 步骤 E · 验收（判定标准）

```powershell
# 判定 1：适配器缓存里有这个模型，且 adapterReady = true
Get-ChildItem "$env:LOCALAPPDATA\BongoCat\cache\model-adapters" -Directory |
  ForEach-Object { $c = Join-Path $_.FullName '.bongo-cat-cache.json'
    if (Test-Path $c) { "$($_.Name): " + ((Get-Content -Raw $c) -replace '\s+',' ') } }

# 判定 2：导入报告显示 format=bongo-cat-mver、status=imported
Get-ChildItem "$env:LOCALAPPDATA\BongoCat\cache\model-adapters" -Recurse -Filter '.bongo-cat-import-report.json' |
  ForEach-Object { (Get-Content -Raw $_.FullName) -replace '\s+',' ' }

# 判定 3：日志里没有 Skipping，且有 model-loaded
Select-String -Path "$env:LOCALAPPDATA\BongoCat\logs\BongoCat.log" -Pattern 'Skipping installed model package|model-loaded' |
  Select-Object -Last 5
```

期望看到：

```
.bongo-cat-cache.json        → "adapterReady": true, "source": "...\\models\\<$NAME>"
.bongo-cat-import-report.json → "format": "bongo-cat-mver", "status": "imported"
BongoCat.log                 → [INFO] model-loaded ... model=<$NAME> ... source=3
                               （并且没有 Skipping installed model package）
```

功能自测：**静止**（眨眼/呼吸/头发摆动）→ **移动鼠标**（头/身体跟随）→ **敲键盘**
→ **点鼠标左右键**。

---

## 3. 一键脚本（推荐，避免手工复制出错）

脚本：`D:\BongoCat\工具\pack_test_export.py`
它做四件事：复制导出文件（排除 `model.cmo3` / `model.psd2live.json`）、渲染封面、写 `config.json`、打印包内容。

```powershell
$py = 'C:\Users\13113\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\python\python.exe'
# MCP 地址/令牌：PSD2Live 里「工具 → MCP → MCP 连接与安装」面板可查
$tok = ((Get-ItemProperty 'HKCU:\Software\JavaSoft\Prefs\io\github\psd2live\agent').agent_mcp_bearer_token -replace '/','').Trim()
$env:PSD2LIVE_MCP_ENDPOINT = 'http://127.0.0.1:23871/mcp'
$env:PSD2LIVE_MCP_TOKEN    = $tok
& $py 'D:\BongoCat\工具\pack_test_export.py'
```

脚本顶部两个常量控制输入输出，改它们就能打包别的工程：

```python
SRC = r"D:\BongoCat\成品\Test"          # PSD2Live 导出目录
DST = r"D:\BongoCat\成品\Mver包_大肥鱼"   # 要生成的 Mver 包
```

> 封面渲染需要 PSD2Live 正在运行（走它的 MCP）；渲染失败不影响包可用，只是列表里没有缩略图。

---

## 4. 排查速查表

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 导入失败 / `Cannot inspect nearby model directory` | 不是 Mver 格式 | 按 §2 步骤 B/C 组包 |
| `Selected directory contains no valid BongoCat model package` | `model3.json` 里引用的文件缺失 | 把被引用的文件全部复制进包 |
| 模型显示正常但**敲键盘没反应** | `config.json` 没声明键码 | 补 `standard.hand` / `standard.keyboard` |
| 模型列表里没有它 | 没重启 BongoCat（它按内容签名缓存） | 完全重启；必要时删 `cache\model-adapters\installed-*` 再重启 |
| 改了包但表现没变 | 同上（缓存） | 删对应 `installed-<hash>-1` 目录后重启 |
| 某个部件不显示 / 被别的部件挡住 | **绘制顺序**问题 | 见 §5.3 |
| 参数在面板里有、模型里没有 | 参数只被"引用"没被"定义" | 在参数面板新建该参数后重新导出 |
| 参数建不上（无报错但不生效） | 参数**名字用了中文** | 参数 ID 照抄，**名字用 ASCII**（如 `LeftHandDown`） |
| 图层切换（显隐）不生效 | 静态预览不显示 switch，且 switch 必须写进导出 | 见 §5.4，并用导出清单验证 |

---

## 5. 附录

### 5.1 BongoCat 需要的参数（标准模式）

| 参数 ID | 范围 | 绑到 | 作用 |
| --- | --- | --- | --- |
| `CatParamLeftHandDown` | 0~1 | 按键盘的那只手 | 键盘按下 |
| `CatParamRightHandDown` | 0~1 | 另一只手 | 键盘/手柄模式 |
| `ParamMouseX` / `ParamMouseY` | -30~30 | 握鼠标的那只手 | 鼠标移动 |
| `ParamMouseLeftDown` / `ParamMouseRightDown` | 0~1 | 同上 | 左右键按下 |

由 PSD2Live 自动生成、无需手动新建：`ParamAngleX/Y/Z`、`ParamBodyAngleX/Y/Z`、`ParamBodyLean`、
`ParamBreath`、`ParamEyeLOpen/ROpen`、`ParamEyeBallX/Y`、`ParamBrow*`、`ParamMouth*`、`ParamHair*`。

**实测（本机 BongoCat 1.14）**：

- 确认被驱动：`ParamAngleX/Y`、`ParamBodyAngle*`、`ParamBreath`、`ParamEyeLOpen/ROpen` —— 所以头/身体跟随、眨眼、呼吸都正常。
- 鼠标手绑 `ParamMouseX/Y` 后**能跟随光标** ✅（可另外绑 `ParamAngleX/Y` 作为补充）。
- 键盘手绑 `CatParamLeftHandDown` 时**未观察到动作**；实际可用的键盘反馈方案是 **Mver 按键叠加图**：
  `config.json` 声明键码 + 提供 `img/standard/hand/0..N.png`（第 i 张对应第 i 个键码，全画布透明 PNG，画布尺寸 = `window_size`）。

### 5.2 键码（Windows 虚拟键码）

| 键 | 码 | 键 | 码 |
| --- | --- | --- | --- |
| A–Z | 65–90 | 0–9 | 48–57 |
| 空格 | 32 | Shift / Ctrl / Alt | 16 / 17 / 18 |
| 回车 / Tab / 退格 / Esc | 13 / 9 / 8 / 27 | 方向键 | 37–40 |
| CapsLock | 20 | 小键盘 0–9 | 96–105 |
| `;` `=` `,` `-` `.` `/` | 186–191 | `[` `\` `]` `'` | 219–222 |

### 5.3 绘制顺序（为什么预览和模型可能不一致）

`model.psd2live.json` 的 `layers` 数组**就是导出模型里的绘制顺序**，**数组越靠后 = 画面越靠前**。

- 如果某个部件（例如"桌子"）排在手的**后面**，它就会盖住手 → 看起来"手在桌子下面"。
- **PSD2Live 的静态预览可能与该顺序不一致**（预览按自己的图层树渲染），所以：
  **以导出清单为准**，别只信预览。
- 修法：在 PSD2Live 的图层面板里调整顺序，重新导出后再次读 `layers` 确认。

### 5.4 图层显隐切换（switch）

PSD2Live 支持"图层随参数显隐"：图层属性 = **类型 `switch` + 参数 + 切换 ID**，
含义是「该参数等于该切换 ID 时显示此图层」。导出清单里表现为每层的 `type` / `parameter` / `switchId`。

MCP 调用（等价于界面操作）：

```json
{"state":"<state>","layer_id":"<图层id>","type":"switch","parameter":"ParamMouseLeftDown","switch_id":1}
```

注意：

- **静态姿态预览不会显示 switch 效果**（渲染结果完全相同），只有实际运行时/BongoCat 里才看得到。
  → 验证方式是**导出后读 `model.psd2live.json`**，确认 `type`/`parameter`/`switchId` 写对了。
- 不要给"需要常显"的图层随便绑 `switch`：实测把某图层改成 `switch` 后，它在清单里的
  `drawable` 字段可能变成空字符串（= 该层不绘制 → 部件消失）。需要常显就保持 `preset`。

### 5.5 PSD2Live MCP 的两个前提

1. **工程必须先保存过（`Ctrl+S` 选过保存位置）**。
   否则所有外部写入会被**静默丢弃**：`parameter create` 之类调用会返回"成功"，
   但返回的 `state` 与传入**完全相同**（没有推进）。`revision save` 会提示
   `Choose a project save location in the application first`。
   → **判定写入是否真的生效：看返回的 `state` 是否变化。**
2. `inspect scope=parameters` 是**分页**的（响应里带 `next`），而且可能存在**列表之外的参数**
   （`create` 会回答 `Parameter already exists: xxx`）。不要用"列表里没有"来判断参数不存在。

### 5.6 术语

| 词 | 含义 |
| --- | --- |
| Mver / BongoCatMver | BongoCat 的皮肤格式：`config.json` + `img/...` 资源 |
| `*.model3.json` | Live2D 模型清单（引用 moc3/纹理/物理/动作） |
| `*.moc3` | Live2D 编译后的模型本体（Core 运行时加载） |
| `*.cdi3.json` | 参数显示信息（编辑用元数据） |
| `*.physics3.json` | 物理设置（头发/饰物摆动） |
| `*.motion3.json` | 动作数据 |
| `drawable` | 导出清单里"该图层对应的绘制网格"；为空 = 该层不绘制 |
| `switchId` | 图层显隐的触发值（配合 `parameter`） |
| `adapterReady` / `bongo-cat-mver` | BongoCat 适配器缓存里的成功标志 |

---

## 6. 给 AI 的执行清单（可直接照做）

1. 确认输入：`$EXPORT` 目录存在，且包含 `model.moc3`、`model.model3.json`、`model.*.4096/texture_00.png`。
2. 建立 `$PKG\img\standard\cat_model\`，把 `$EXPORT` 下除 `model.cmo3`、`model.psd2live.json`
   以外的文件**全部**复制进去。
3. 生成 `$PKG\config.json`：`window_size` = moc3 画布尺寸；`hand`/`keyboard` = §5.2 的键码列表
   （**不得为空**）；其余字段照 §2 步骤 C 模板。
4. （可选）把一张整画布渲染图存为 `$PKG\img\standard\cat_model\resources\cover.png`。
5. 校验（缺一不可）：
   - `config.json` 能被 `json.load` 解析，且 `standard.hand` 长度 > 0；
   - `model.moc3` 的前 4 字节是 `MOC3`；
   - `model.model3.json` 里 `FileReferences` 引用的每个文件都存在于包内。
6. 复制到 `%LOCALAPPDATA%\BongoCat\models\<$NAME>`。
7. 让用户**完全重启 BongoCat**，然后按 §2 步骤 E 的三条判定逐一核对。
8. 若第 7 步失败：
   - 日志有 `Cannot inspect nearby model directory` → 包结构不对（§1.1）；
   - 日志有 `Selected directory contains no valid BongoCat model package` → 清单引用的文件缺失；
   - 有 `adapterReady: true` 但敲键盘无反应 → 键码没声明（§1.2）；
   - 表现与上次完全相同 → 适配器缓存未失效，删 `cache\model-adapters\installed-<hash>-1` 后重启。
