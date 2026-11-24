# OpenMoHAA 官方中文汉化移植技术全景日志

**项目名称**: OpenMoHAA 中文本地化移植与渲染引擎修复  
**Fork 仓库**: [xiao-mantou/openmohaa](https://github.com/xiao-mantou/openmohaa)  
**上游基准 (Upstream Baseline)**: `openmoh/openmohaa:main` (`a2f3401`)  
**测试平台**: Windows 11 x64, MinGW GCC 16.1.0, 荣誉勋章：联合袭击 战友之歌 (Allied Assault War Chest 1.11)

---

## 1. 移植背景与体系结构

《荣誉勋章：联合袭击》（Medal of Honor: Allied Assault，下称 MOHAA）官方简体中文版由当年第三波 / EA 官方汉化发行。中文补丁的核心资源结构如下：
* **`pak6sc.pk3`**: 核心中文资源包，包含 `global/localization.txt`（字典映射表）、中文化 `.urc` 菜单布局文件、`.scr` 任务剧情脚本。
* **`pak7sc1.pk3`**: 中文字体资源包，包含点阵/矢量字库定义及其贴图：
  * `fonts/verdana-12.txt`：菜单与设置界面主要字库（包含 2 张页纹理 `verdana-12-0.tga`、`verdana-12-1.tga`，收录 3,878 个汉字与符号映射）。
  * `fonts/facfont-20.txt`：战役任务简报与大标题字库（包含 5 张页纹理 `facfont-20-0.tga` ~ `facfont-20-4.tga`）。
* **OpenMoHAA 现状**: 社区开源重构版基于 idTech 3 / Ritual 引擎分支现代重写，默认仅测试了英文与单页 ASCII 字体渲染；当载入官方中文资源包后，由于底层词法分析器 (Lexer)、多页 Ritual 字库解析器 (RitFontList) 以及渲染排版管线存在多处缺陷，导致游戏界面出现大面积“火星文 / 鬼字”以及严重排版右偏。

---

## 2. 故障现象与复现记录

将官方汉化补丁复制入 OpenMoHAA 后，直接运行出现以下异常：

| 界面区域 | 正确中文文本 | 游戏内错误呈现 | 异常特征 |
| :--- | :--- | :--- | :--- |
| **图形设置 - 显示模式** | 全开 | **盘家榴嘉** | 乱码且文字严重靠右偏移 |
| **图形设置 - 阴影** | 复杂 | **可镜绕确** | 乱码且文字严重靠右偏移 |
| **图形设置 - 地形/特效**| 最大 | **三头镜亡** | 乱码且文字严重靠右偏移 |
| **图形设置 - 模型精细度**| 最高 | **三头可视** | 乱码且文字严重靠右偏移 |
| **图形设置 - 曲线圆精细度**| 高 | **可视** | 乱码且文字严重靠右偏移 |
| **战役模式任务简报** | （中文任务说明） | 乱码字符流 | 完全无法辨认 |
| **主菜单主选项** | 新游戏 / 选项等 | 正常 | 静态单行主菜单文本正常显示 |

---

## 3. 根因剖析与乱码数学推导

经过对 OpenMoHAA 渲染引擎与 UI 子系统的反汇编与动态比对，定位到三个级联引发的致命缺陷：

### 3.1 致命缺陷一：多页字库加载器 EOF 未处理导致字库被清空 (`LoadFont: Bad Token`)

官方中文字库 `verdana-12.txt` 采用 `RitFontList` 格式：
```txt
RitFontList
CodePage 936
CharTableLength 3878
RitFontCount 2
RitFont0 fonts/verdana-12-0
RitFont1 fonts/verdana-12-1
CharTable
{
  32 32 0
  ...
  65374 135 1
}
```
* **原版代码逻辑 (`R_LoadFont`)**:
  ```cpp
  while (ref && !error) {
      token = COM_Parse(&ref);
      if (!Q_stricmp(token, "CodePage")) { ... }
      else if (!Q_stricmp(token, "CharTableLength")) { ... }
      else if (!Q_stricmp(token, "RitFontCount")) { ... }
      else if (!Q_stricmp(token, "CharTable")) { ... }
      else {
          ri.Printf(PRINT_WARNING, "LoadFont: Bad Token %s\n", token);
          error = qtrue; // 致命错误标记！
      }
  }
  ```
* **故障机制**: 当读取完 `CharTable` 后的最后一个字符时，文件已到达末尾 (EOF)。`COM_Parse(&ref)` 返回空字符串 `""`（`token[0] == '\0'`）。由于原代码**没有检查 EOF**，空字符串不匹配任何关键字，直接跌入 `else` 分支，触发 `LoadFont: Bad Token` 并将 `error` 置为 `qtrue`！
* **连锁破坏**:
  ```cpp
  if (error) {
      ri.Free(header->charTable);
      header->numPages = 0;
      header->charTableLength = 0;
      header->charTable = NULL;
  }
  ```
  这直接把刚刚成功解析的 3,878 个汉字映射表和页纹理全部释放，将 `numPages` 强制归零！`facfont-20` 也因同样原因被彻底损毁。

---

### 3.2 乱码“鬼字”生成的数学映射推导

当 `header->numPages == 0` 时，渲染器 `R_DrawString` 会直接回退到单页单字节模式：
```cpp
if (!font->numPages) {
    if (font->sgl[0]) {
        R_DrawString_sgl(font->sgl[0], text, x, y, maxlen, pvVirtualScreen);
    }
    return;
}
```
此时，GBK 编码的中文文本不再按“双字节编码”查找 `charTable`，而是将汉字的两个字节拆解为两个独立的单字节，每个单字节直接作为索引，到第一页贴图 `verdana-12-0` 对应的 256 个字形位置中检索！

#### 示例推导：
1. **“全开” 为什么变成 “盘家榴嘉”？**
   * 中文字符 `全` 的 GBK 编码为 `0xC8 0xAB`（十进制 200, 171）。
   * 中文字符 `开` 的 GBK 编码为 `0xBF 0xAA`（十进制 191, 170）。
   * 拆分出来的单字节序列为：`[0xC8, 0xAB, 0xBF, 0xAA]`（共 4 个字节）。
   * 在单页字库 `verdana-12-0` 中，这 4 个索引位置对应的字形恰好就是汉字 **`盘`**、**`家`**、**`榴`**、**`嘉`**！
   * 原本 2 个汉字，被误当成了 4 个独立的单字节字符绘制，呈现为 `盘家榴嘉`。

2. **其他选项乱码对照推导表**：

| 选项原意 | GBK 16 进制编码 | 单字节序列 (十进制) | 单页贴图字形映射结果 |
| :--- | :--- | :--- | :--- |
| **全开** | `C8 AB BF AA` | 200, 171, 191, 170 | **盘 家 榴 嘉** |
| **复杂** | `B8 B4 D4 D3` | 184, 180, 212, 211 | **可 镜 绕 确** |
| **最大** | `D7 EE B4 F3` | 215, 238, 180, 243 | **三 头 镜 亡** |
| **最高** | `D7 EE B8 DF` | 215, 238, 184, 223 | **三 头 可 视** |
| **高** | `B8 DF` | 184, 223 | **可 视** |

这在数学上完全证实了乱码是由**双字节字形查找表丢失**后，退化为**单字节贴图索引**所造成的必然结果。

---

### 3.3 致命缺陷二：文字宽度计算与居中对齐严重右偏

MOHAA 的选项框文字采用居中对齐，公式为：
$$\text{draw\_x} = \text{box\_x} + \frac{\text{box\_width} - \text{text\_width}}{2}$$

* **原版宽度计算缺陷 (`R_GetFontStringWidth`)**:
  1. 上游代码在遍历字符串时存在逻辑笔误：
     ```cpp
     for (i = 0; s[i]; i++) {
         char c = *s; // 错误！每次循环都固定读取 s[0]，而不是 s[i]！
     ```
  2. 当 `numPages == 0` 时，函数调用 `R_GetFontStringWidth_sgl`，单字节索引模式无法解析高位字节宽度，计算得到的 `text_width` 为 `0`。
* **右偏后果**:
  $$\text{draw\_x} = \text{box\_x} + \frac{\text{box\_width} - 0}{2} = \text{box\_x} + \frac{\text{box\_width}}{2}$$
  文字不是居中显示，而是直接从选项框的正中心作为起点向右绘制，导致所有选项文字出现严重的“明显右偏”！

---

### 3.4 致命缺陷三：词法解析器 (Lexer) 有符号字符吞字符缺陷

在 `q_shared.c`、`script.cpp` 和 `tiki_script.cpp` 中：
* 默认 `char` 在 x86/x64 C/C++ 中为 `signed char`，取值范围为 `-128 ~ 127`。
* GBK 汉字首字节和很多尾字节均大于等于 `0x80`（取值 `128 ~ 255`），在有符号表示下变为**负数**（如 `0xC8` 变为 `-56`）。
* 原版词法检查：
  ```c
  while ((c = *data) <= ' ') { ... } // ' ' 是 32
  ```
  因为负数恒小于等于 32，原版词法分析器会将所有中文双字节判定为“空白字符”，从而在分词时吃掉汉字或提前截断！
* 另外，GBK 编码中存在部分汉字的尾字节恰好为 `0x5C`（即 ASCII 反斜杠字符 `\`），在解析引申字符串时，若不进行 DBCS 双字节识别，会将汉字尾字节误认为是 C 语言转义字符（如 `\n`、`\"`），从而破坏文本。

---

## 4. 修复方案与分支架构（对比上游基准 a2f3401）

为方便向官方 `openmoh/openmohaa` 提交 PR，并利用官方 GitHub Actions 实现纯净标准构建，本仓库采用双分支维护架构：

* **`main` 分支（核心中文支持，推荐）**：
  仅包含**纯粹的 8 个中文字库与本地化渲染核心文件**，完全对齐上游官方构建体系。无需任何本地编译妥协，由上游 GitHub Actions 原生云端 MSVC 自动化构建并打包 Release。
* **`mingw-build` 分支（本地 MinGW 工具链适配备用）**：
  包含了本地使用 MinGW GCC 编译所需的 6 个平台适配修改（屏蔽含空格路径下的 `windres`、处理 POSIX `stat.h` 与 `alloca` 等）。若未来需要在本地离线调试，可切至该分支。

---

### 4.1 核心汉化修复代码审计（`main` 分支 8 个文件）

#### 1. 词法解析与 DBCS 支持
* **`code/qcommon/q_shared.c`**:
  - 在 `SkipWhitespace`、`COM_ParseExt`、`COM_ParseExtIgnoreQuotes`、`COM_GetToken` 中将 `*data` 转换为 `(unsigned char)*data`，确保高位字节（`0x80..0xFF`）被正确判定为大于 32 的有效字符，彻底解决汉字被视为空白吃掉的问题。
* **`code/corepp/str.cpp`**:
  - 在 `str::strip` 和 `strstrip` 中将 `isspace((int)*s)` 转换为 `*s && isspace((unsigned char)*s)`。GBK 汉字高位字节在 signed char 下为负数，传给 `isspace` 属于未定义行为并在 MSVC Debug 下触发断言崩溃，转为 `(unsigned char)` 杜绝崩溃隐患。
* **`code/corepp/script.cpp` & `code/corepp/tiki_script.cpp`**:
  - 统一使用 `(unsigned char)*script_p <= (unsigned char)TOKENSPACE && *script_p != '\0'`。
  - 在 `GetString` 中增加 DBCS 双字节识别：
    ```cpp
    if ((unsigned char)*script_p >= 0x81 && (unsigned char)*script_p <= 0xFE && (script_p + 1 < end_p) && *(script_p + 1) != '"') {
        *token_p++ = *script_p++;
        *token_p++ = *script_p++;
        continue;
    }
    ```
    彻底防止汉字尾字节 `0x5C` (`\`) 被误当做转义字符。

#### 2. 本地化与服务端/单机消息路由
* **`code/sys/win_localization.cpp`**:
  - `Sys_LV_ConvertString`: 恢复通过 `Sys_LV_CL_ConvertString` 进行字典查找。单人战役中大量 HUD 打印（例如“游戏已保存”、“获得 1 枚手榴弹”等）均通过 `gi.LV_ConvertString` 发出，转换以保证单人战役系统提示显示为中文。
  - `ConvertString` 与 `bjb_strnscmp` 中修正所有字符比较为 `(unsigned char)`。

#### 3. 字体度量与 UI 计算
* **`code/uilib/uifont.cpp`**:
  - 修复 `DBCSIsMaekin` / `DBCSIsAtokin` 比较：`p[1] == (uch & 0xFF)`。
  - 修复 `UI_FontCodeSearch` 二分查找越界错误（原版在未命中时 `l == charTableLength` 会导致内存越界访问）。
  - 增加防御性校验：`if (!font || !font->charTable || font->charTableLength <= 0) return -1;`。
  - 修复 `UI_FontStringWidth` 与 `UI_FontStringMaxWidth` 中的尾字节读取错误（原版错误地将前导字节与自身进行位或运算：`uch = (uch << 8) | pszString[i]`）。

#### 4. 渲染器与字库加载核心修复 (GL1 & GL2)
* **`code/renderergl1/tr_font.cpp` & `code/renderergl2/tr_font.cpp`**:
  - **EOF 截断防护**: 在 `R_LoadFont` 中增加：
    ```cpp
    token = COM_Parse(&ref);
    if (!token || !token[0]) {
        break;
    }
    ```
    杜绝文件末尾进入 `Bad Token` 分支，彻底保护 `charTable` 与 `numPages`。
  - **verdana-14 回退**: 官方中文补丁仅制作了 `verdana-12` 和 `facfont-20` 的中文字库。当 HUD 或控件请求 `verdana-14` 时，检测若 `verdana-12` 具备多页中文，则平滑回退至 `verdana-12`，保证中文能正常显示。
  - **二分查找越界与防空校验**: 修正 `CodeSearch` 二分查找区间，加入针对空指针和 0 长度的防护。
  - **字符串绘制与排版修复 (`R_DrawString`)**:
    - 修正 `(unsigned char)text[i]` 读取与双字节组合；
    - 修正缓冲区切片刷新逻辑，由使用全局 `maxlen` 改为当前切片长度 `buflen`；
    - 修正换行时绘制坐标由静态 `y` 改为累加后的 `curY`；
    - 宽度步进与缩放比率对齐：`curX += curWidth * s_fontGeneralScale`。
  - **字符串精确宽度测算 (`R_GetFontStringWidth`)**:
    - 修复 `s[0]` 死循环 bug，逐字符扫描 `s[i]`；
    - 正确合成 16-bit `unsigned short uch`；
    - 宽度返回时按 `width * s_fontGeneralScale * 256.0f` 进行缩放，与绘制完全 1:1 吻合，确保居中算法精确无误。

---

## 5. GitHub Actions 自动化编译使用指引

上游 OpenMoHAA 仓库原生接入了 GitHub Actions 持续集成。在 fork 仓库启用后，由云端微软官方镜像进行全自动构建并生成发行包：

1. **启用 Actions**：
   * 访问你的仓库页面：`https://github.com/xiao-mantou/openmohaa/actions`。
   * 点击绿色按钮 **"I understand my workflows, go ahead and enable them"**。
   * 在 `Settings` -> `Actions` -> `General` -> `Workflow permissions` 下，确保勾选 **"Read and write permissions"**。
2. **触发构建**：
   * 每次对 `main` 分支执行 `git push`，或者在 Actions 页面点击 `Builds` 工作流旁边的 `Run workflow`，云端虚拟机将自动启动。
3. **获取产物**：
   * 构建完成后，在对应运行记录的 **Artifacts** 区域即可直接下载：
     - `out-windows-x64`（64位便携绿色版，解压即用）
     - `out-windows-x64-installer`（官方标准 MSI 安装包 `Setup.msi`）
     - 另外同时生成了 x86、ARM64 以及 Linux、macOS 全套构建产物！

---

## 6. 编译验证与测试结论

* **编译方式**:
  ```powershell
  $env:PATH = "E:\Code\mingw64\bin;d:\Antigravity\Medal of Honor\openmohaa\thirdparties\bin;$env:PATH"
  cd "d:\Antigravity\Medal of Honor\openmohaa\build_win"
  ninja -j16
  ```
* **测试结果**:
  * `openmohaa.exe` 与 `renderer_opengl1.dll` 链接成功，编译过程 0 Error。
  * 最新二进制已同步部署至游戏目录 `e:\GOG Games\Medal of Honor - Allied Assault War Chest`。
  * 实机运行测试：“全开”、“复杂”、“最大”、“最高”、“高”等选项文字全部正常显示中文，文字精确居中，战役任务简报文字恢复正常，移植任务圆满达成！

---

## 7. 高分辨率与 UI / 字幕缩放 (`ui_scale`)

### 7.1 问题背景
在 2K (2560x1440) 或 4K 显示器上游玩时，引擎自带的高分辨率缩放因子仅为 `vidHeight / 1080`（2K 下为 1.33x），导致局内 HUD（生命、弹药、罗盘、准星）、字幕（Subtitle）、任务目标（Objectives）及聊天文字偏小，影响观感。

### 7.2 解决方案
新增 `ui_scale` 控制变量（`cvar`，带 `CVAR_ARCHIVE` 标志）：
* **默认值**: `1.0`（完全保持与上游默认表现 100% 向后兼容）。
* **自动持久化**: 会自动保存并读取自 `%APPDATA%\openmohaa\main\configs\omconfig.cfg`。
* **实时生效**: 在控制台输入 `ui_scale 1.5` 或通过 `autoexec.cfg` 配置后，局内 HUD、任务目标和字幕即时平滑缩放，无需重启游戏。
* **缩放范围说明**:
  - 底层代码对输入参数做了 `Q_clamp(val, 0.2f, 5.0f)` 安全钳制，因此缩放范围是 **0.2 倍至 5.0 倍**。
  - 之前提到的 2.0x 并不是硬上限，而是以 2K 分辨率默认基础缩放 1.33x 乘以用户参数 1.5 得到的结果（$1.333 \times 1.5 \approx 2.0$）。
  - 用户完全可以根据视力和显示器尺寸自由指定 `ui_scale 2.0`、`2.5`、`3.0` 等更高数值。
* **配置示例** (推荐 2K 分辨率设置 1.3 ~ 1.6，4K 设置 1.8 ~ 2.5)：
  ```cfg
  seta ui_scale "1.5"
  ```

---

## 8. 引擎性能、显示刷新率与可执行文件辨析

### 8.1 刷新率、全屏模式与垂直同步控制
* **显示器刷新率 (`r_displayRefresh`)**:
  - **默认值 `0` (已优化)**：在最新版本中，当设为 `0` 时，引擎会自动读取当前 Windows 桌面原生的高刷频率（如 240Hz、280Hz），杜绝回退到 60Hz。
  - **高刷上限解放**：移除了历史上将刷新率硬编码限制在 200Hz 的缺陷，上限放宽至 1000Hz，完美支持现代 240Hz/280Hz/360Hz/540Hz 电竞显示器。若需手动指定，可在配置中写入 `seta r_displayRefresh "240"`。
* **全屏显示模式 (`r_fullscreen`)**:
  - `0`：常规窗口化（Windowed）。
  - `1`：传统独占全屏（Exclusive Fullscreen），由显卡驱动切换硬件时序。若分辨率或刷新率与桌面不一致，进出游戏或 Alt-Tab 时会导致显示器重协商黑屏 2~4 秒。
  - **`2`：现代无边框全屏桌面模式（Borderless Fullscreen / Fullscreen Desktop，强烈推荐）**：
    - 采用 SDL2 的 `SDL_WINDOW_FULLSCREEN_DESKTOP`。
    - 屏幕时序严格锁定在 Windows 桌面物理时序（如 2560x1600 @ 240Hz），**Alt-Tab 切换桌面 0 毫秒无缝秒切，完全杜绝黑屏**。
    - 完美支持 Windows 11 / NVIDIA 显卡驱动的 **VRR / G-Sync 可变刷新率**，输入延迟走 DirectFlip 翻转链路，无任何性能折损。
* **垂直同步 (`r_swapInterval`)**:
  - `0`：关闭垂直同步，减少输入延迟（结合 VRR / G-Sync 体验最佳）。
  - `1`：开启垂直同步，避免画面撕裂并与显示器刷新率严格同步。
* **帧率限制 (`com_maxfps`)**:
  - 原版默认通常限制在 `85` FPS。
  - 对于 240Hz 高刷屏，强烈建议将其设置为与显示器最高刷新率一致（例如 `seta com_maxfps "240"`）或设为 `0`（解除上限），充分发挥 240Hz 的极速响应。

### 8.2 启动程序架构辨析
在构建和运行目录中，可能存在多个可执行文件：
* **`openmohaa.exe`（核心本体主程序）**:
  游戏真正的原生主进程，负责引擎启动、图形初始化、声音系统及游戏循环。所有实际的计算与渲染均在此进程中执行。
* **`launch_openmohaa_*.exe`（快捷参数封装器）**:
  辅助启动器，主要作用是预设启动参数并拉起 `openmohaa.exe`（例如添加 `+set fs_game`、指定单人/多人或特定控制台标志），方便通过不同快捷方式运行不同模式或资料片。

---

## 9. 敌人 AI 命中率与战斗难度调优

### 9.1 原版 AI 命中率机制分析
2002 年发行的原版《荣誉勋章：联合袭击》以高硬核、高拟真为特色，但其敌人 AI（尤其是中远距离的步枪手和狙击手）具有几乎即时发现即开枪的瞬时射线判定机制：
1. 敌人索敌射击的命中散布范围较小，在开阔地带玩家即便做左右摇晃或横向机动，敌人的射线仍可能紧跟玩家包围盒造成瞬间致命击中。
2. 缺乏人性化的索敌延迟时间，常常在玩家探头的一瞬间就被集火。

### 9.2 难度与精准度调节控制变量 (`cvar`)
玩家可通过在 `%APPDATA%\openmohaa\main\configs\omconfig.cfg` 或 `autoexec.cfg` 中加入以下控制变量，对敌人的射击散布和反应速度进行人性化微调：

| 控制变量 (cvar) | 默认值 | 作用与机制 | 推荐调优值 |
| :--- | :--- | :--- | :--- |
| `g_aiScatterWide` | 0 | AI 射击水平散布范围。增大此值会使敌人向玩家左右两旁脱靶的概率显著增加，大幅降低左右晃动时被瞬间锁定的几率。 | `10` ~ `25` |
| `g_aiScatterHeight` | 0 | AI 射击垂直散布范围。增大此值会使敌人射偏到脚下或头顶。 | `10` ~ `20` |
| `g_ai_noticescale` | 1.0 | AI 索敌察觉反应时间缩放。增大数值可增加敌人在发现玩家到举枪射击之间的延迟，给予玩家更多的反应与找掩体时间。 | `1.5` ~ `2.5` |
| `g_aiDamageMult` | 1.0 | AI 造成伤害的全局倍率。若感觉高难度下狙击手一枪秒杀挫败感强，可调低该数值。 | `0.7` ~ `0.85` |

* **推荐配置代码段（直接追加至 `main/autoexec.cfg` 即可生效）**:
  ```cfg
  // --- 敌人 AI 精准度与反应时间人性化平衡 ---
  seta g_aiScatterWide "15"
  seta g_aiScatterHeight "12"
  seta g_ai_noticescale "1.8"
  seta g_aiDamageMult "0.8"
  ```

---

## 10. 经典视觉渲染现象与已知上游缺陷深度剖析

针对玩家社区反馈较多的几个视觉与渲染问题，对 OpenMoHAA 源码及原版游戏机制进行了深度溯源排查：

### 10.1 现象一：阴影设置为最高/复杂时爆炸无火光特效（上游已知 Issue #538）
* **故障现象**:
  在视频设置中将阴影级别调至“复杂”或最高（对应配置 `seta cg_shadows "2"`），游戏中手榴弹、炮弹等各类爆炸特效只留下飞溅碎片，标志性的爆炸火焰与烟雾面片（`boom1.tga` ~ `boom7.tga`）完全不可见；而设置为 `1`（简单阴影）或 `0`（关闭）时爆炸火光完全正常。
* **底层根因剖析**:
  1. 引擎变量映射与清理不一致：在 `renderergl1` 中，`cg_shadows` 被映射到底层渲染器的 `r_shadows`。在 `tr_backend.c:628` 中，清空深度与模板缓冲区的逻辑为：
     ```c
     if ( r_measureOverdraw->integer || r_shadows->integer == 3 ) {
         clearBits |= GL_STENCIL_BUFFER_BIT;
     }
     ```
     当 `r_shadows` 为 `2` 时，**模板缓冲区（Stencil Buffer）完全没有执行清空操作**，残留着上一帧未初始化的脏数据。
  2. 阴影收尾渲染覆盖深度：在每一帧主场景渲染末尾调用 `RB_ShadowFinish()`：
     ```c
     void RB_ShadowFinish( void ) {
         if ( r_shadows->integer != 2 ) return;
         ...
         GL_State( GLS_DEPTHMASK_TRUE | GLS_SRCBLEND_DST_COLOR | GLS_DSTBLEND_ZERO );
         qglBegin( GL_QUADS );
         qglVertex3f( -100, 100, -10 ); ...
         qglEnd ();
     }
     ```
     `RB_ShadowFinish` 针对未清空的模板缓冲区，以写深度模式（`GLS_DEPTHMASK_TRUE`）在视锥体近平面（$z = -10$ 处）绘制了一个全屏四边形，**将脏区域的深度缓冲强行覆盖为最近距离**。
  3. 特效精灵在深度测试中全部被剔除：在主场景后，引擎通过 `RC_SPRITE_SURFS`（`RB_SpriteSurfs`）统一绘制爆炸粒子面片。由于爆炸面片在渲染时启用了常规深度测试，遇到刚才被 `RB_ShadowFinish` 错误覆盖的 $z = -10$ 深度值，全部未通过深度测试而被 OpenGL 硬件管线直接丢弃剔除，导致爆炸火光整体隐形。
* **代码修复方案 (已合入)**:
  在 `code/renderergl1/tr_shadows.c` 中，将无 stencil buffer 支持且强制写近平面深度的 `RB_ShadowFinish()` 通过 `#if 0` 禁用（与 `renderergl2` 完全保持一致）。禁用后，`cg_shadows 2` 下复杂的阴影渲染依然按引擎既定逻辑绘制，且爆炸粒子特效在 `RC_SPRITE_SURFS` 阶段深度测试不再被破坏，火光与烟雾完全正常显示，彻底解决了上游 Issue #538。
* **应对建议**:
  如果使用未修复的上游旧版本，推荐在图形设置中将“阴影”设为 **`简单` (`cg_shadows 1`)**。该模式使用基于地面的足底圆形贴花投影（Blob/Decal Shadow），不仅计算开销极低、帧率稳定，而且完全避开了模板缓冲区冲突。而在本项目最新编译版本中，`cg_shadows 2` 与 `cg_shadows 1` 均已完美支持爆炸特效。

### 10.2 现象二：打掉探照灯后地面仍然残留照射光斑
* **故障现象**:
  在夜间渗透关卡（如第一章第二关 `m1l2a` 机场破坏潜入等），打掉哨塔上的探照灯或击毙操作机枪手后，虽然探照灯停止了扫动，但有时地面上仍能看到一圈圆形的高亮照射光斑，仿佛探照灯依然在工作。
* **底层根因剖析**:
  1. **实体分离设计与状态不同步**: 2002 年原版 MOHAA 的关卡脚本（`global/spotlight.scr`）将探照灯设计为多个独立实体的组合：
     - 基座物理模型与可破坏灯头模型（`miscobj/searchlight.tik`）；
     - 伤害检测触发盒（`trigger_multiple` 命名的 `spotdamage`）；
     - 地面投射的独立动态光源实体（`local.light = spawn script_model "fx/dummy.tik"` 和 `local.org = spawn script_origin`，二者分别带有 `lightOn` 和 `lightRadius 300` 的动态光照）；
     - 负责沿曲线路径飞行的点路径驱动器（`spotpath` 线程与 `flypath` 运动）。
  2. **原版关卡脚本死循环缺陷**:
     在 `global/spotlight.scr` 中存在一段著名的原版逻辑缺陷：
     ```scr
     if !(isalive self.spotter)
     {
         while (self.checkdeath == 0)
             wait 0.1
     }
     ```
     当玩家先击毙了操作机枪的哨兵（`self.spotter`）时，主监视线程退出并进入了死等待；此时若玩家未再向灯体碰撞盒追加射击，`self.checkdeath` 保持为 0，程序永远无法执行到后方的 `self.org lightOff` 与 `local.light delete`，导致地面光源实体永久驻留。
  3. **路径飞行状态未完成**: 即使触发了灯体爆炸，由于 `flypath` 路径运动在独立线程中执行，动态光照实体的关闭如果未被及时同步到世界表面（`dlightMap` 缓存），也会造成短暂的光斑残留。此现象属于 2002 原版关卡脚本固有逻辑机制。
* **脚本修复方案：逻辑 B（人死灯停，照射部分一体彻底灭）(已合入)**:
  通过自定义优化脚本 `main/global/spotlight.scr`（并打包为 `main/pak8_searchlight_fix.pk3` 确保高优先级覆盖）：
  1. **受击判定完全契合原版**: 严格遵从原版关卡设计，探照灯金属底座（`searchlightbase.tik`）与哨塔支架不受子弹伤害；唯有玩家精准射击上方灯头（命中地图专属 `spotdamage` 伤害触发盒）时方可损毁探照灯，保留最原汁原味的二战狙击机制。
  2. **人死灯停、光斑同步定格**: 当操作机枪的哨兵死亡时，探照灯停止追踪转动，并立即终止 `flypath` 路径移动线程，地面光斑同步定格在当前射向位置，不再在地面盲目乱跑，同时死亡监听线程持续等待玩家射毁灯头。
  3. **照射系统一体化灭绝（灯灭光灭）**: 将灯头模型（`bulb`）、光束面片（`flare`）、地面投射动态光源（`org` / `light`）紧密绑定为一体化照明系统。灯头被击毁时，立即切换为碎裂模型 `miscobj/searchlightoff.tik`、播放爆炸音效与火花，并对地面动态光源及光束面片依次执行 `lightOff` 与 `delete`。光斑、光束与灯头瞬间同步消亡，彻底杜绝地面残留光斑现象。

### 10.3 现象三：直视太阳时整个屏幕大白（强光耀斑机制与门窗穿透 Issue #398）
* **故障现象**:
  在晴朗白昼关卡中，当视角朝向上方直接注视太阳时，画面迅速泛白直至整个屏幕完全变成纯白色死白（大白屏）；即使视角转回地面，也需要半秒到一秒左右才逐渐消退。有时在室内透过关闭的木门或薄墙，视角朝向太阳时也会莫名泛白。
* **底层根因剖析**:
  1. **双重强力全屏加色混合叠加**:
     在 `renderergl1/tr_sun_flare.cpp` 与 `scripts/sunflare.shader` 中定义了太阳耀斑管线：
     - 当视线向量与太阳方向的夹角余弦值 `dot > 0.55`（即视野内朝向太阳）时，引擎首先绘制尺寸为视口 **1.4 倍**（即超出全屏）的巨型耀斑面片 `textures/sprites/sunflare_main`，混合模式为 `GL_SRC_ALPHA GL_ONE`（加色高光）。
     - 紧接着，`ScreenBlend()` 再次在全屏幕上绘制全屏纯白矩形 `textures/jack/whitefade`（源注释明确标注为 `//SED: Nasty screen overlay`），混合模式同样为加色混合 `blendfunc GL_SRC_ALPHA GL_ONE`。
     - 两重全屏加色混合使屏幕各色彩通道值迅速远超 1.0（255），在现代无色调映射（Tonemapping）的高动态现代显示器上直接溢出饱和，呈现为彻底的刺眼“大白屏”。
  2. **射线遮挡判定无法识别动态门窗（上游已知 Issue #398）**:
     引擎在检测太阳是否被遮挡时，执行的是：
     ```cpp
     ri.CM_BoxTrace(&trace, viewOrigin, sunTraceOrigin, vec3_origin, vec3_origin, 0, CONTENTS_SOLID, qfalse);
     ```
     该射线追踪**只检测 BSP 静态世界画刷（`CONTENTS_SOLID`）**，而不检测动态门（`func_door`）、可破坏物体、植被或实体模型。因此即便关着门或身处室内，只要世界画刷判定穿透到了天空面（`SURF_SKY`），引擎就会误以为玩家直面烈日，从而触发全屏眩光。该问题上游已确认与原版完全一致（标为 `in-original-game`）。
* **代码修复方案：新增 `r_sunblind` 控制强光眩目 (已合入)**:
  为了在保留太阳真实镜头光晕（Lens Flare、日环、日冕粒子）的同时彻底解决现代高对比显示器上的刺眼大白屏，在底层渲染器 `renderergl1` 与 `renderergl2` 中新增归档控制变量 `r_sunblind`（默认值为 `0.2`）：
  - **声明与注册**: 在 `tr_local.h` 与 `tr_init.c` 中注册 `r_sunblind = ri.Cvar_Get("r_sunblind", "0.2", CVAR_ARCHIVE);`。
  - **优雅平滑缩放**: 在 `tr_sun_flare.cpp:ScreenBlend()` 中，将覆盖全屏的纯白混合面片 (`whitefade`) 亮度乘以 `r_sunblind->value`，并在内部添加上限保护 `lastbright = (lastbright > 0.4f ? 0.4f : lastbright)`；若玩家设置 `r_sunblind 0`，则完全关闭全屏白屏遮罩，仅保留太阳本体日轮与镜头衍射环。
  - **门窗遮挡保持原版**: 遵照原版设计，保留了光晕在门窗边缘的穿透感，不破坏原汁原味的二战阳光滤镜氛围。
  - **参数调节指南**:
    - `r_sunblind "0.2"`（默认推荐）：舒适微眩光，直视太阳时伴随柔和的白光弥散，不遮挡视线、不刺眼。
    - `r_sunblind "0"`：完全关闭全屏泛白，太阳仅作为天空盒中的高光日轮与镜头光斑。
    - `r_sunblind "1.0"`：恢复 2002 原版暴烈的大白屏眩光。

---

## 11. 中文字体技术溯源与现代高分辨率模糊根因剖析

### 11.1 中文字体的来源背景
当前游戏内加载的完整中文字体（3,878 个汉字与全角标点），**100% 忠实提取自 2002 年 EA / 第三波发行的《荣誉勋章：联合袭击》官方简体中文版核心资源包 `pak7sc1.pk3`**：
* 对应文件：`fonts/verdana-12.txt`（描述字库）以及两张纹理贴图 `gfx/fonts/verdana-12-0.tga` (512×512)、`gfx/fonts/verdana-12-1.tga` (256×256)。
* 大字标题与任务简报：`fonts/facfont-20.txt` 及其 5 张页纹理 `gfx/fonts/facfont-20-0.tga` ~ `facfont-20-4.tga`。

### 11.2 现代高清屏（2.5K / 4K）上文字“模糊发虚”的底层物理成因
在现代 2560×1600 或 2560×1440 等高分辨率显示器上，玩家常会直观感受到中文边缘有一圈柔和的半透明灰边或发虚感，其技术根因如下：
1. **2002 时代点阵位图字体的固有分辨率限制**：
   - 2002 年的官方汉化基于当年典型的 640×480 与 800×600 CRT 显示器制作。
   - 在一张 512×512 的贴图上塞入 3,000 多个汉字，意味着**每个汉字字形仅有约 12×12 像素**的物理位图空间。
   - 在 640×480 分辨率下，12 像素几乎相当于屏幕的物理点对点显示，笔画清晰锐利。
2. **现代虚拟屏幕与 UI 缩放带来的数倍拉伸**：
   - OpenMoHAA 渲染 UI 时使用 640×480 的标准虚拟坐标空间。
   - 当将画面呈现在 2560×1600 屏幕上时，横向放大了 **$4.0\times$**，纵向放大了 **$3.33\times$**；若再配合 `ui_scale 1.5` ~ `2.0` 放大排版，每个 12 像素的小汉字在屏幕上实际要拉伸到 40~50 像素的大小来绘制！
3. **OpenGL 双线性纹理过滤（`GL_LINEAR`）的插值效应**：
   - 引擎加载字体着色器时，默认使用 OpenGL 的线性插值过滤（`GL_LINEAR`）。
   - 将 12 像素的小位图放大 3~4 倍时，线性过滤会将原本黑白分明的黑/白色块笔画在相邻像素间进行半透明颜色渐变平滑，导致字形边缘产生一圈“毛边/半透明发虚晕影”。
4. **后续潜在的高清优化路径**：
   - **方案一（高倍清晰点阵）**：通过脚本重新生成 2048×2048 分辨率、字形为 48px 的高清中文 TGA 纹理贴图包，完全保持 Ritual 引擎格式兼容性同时提供 4 倍锐度。
   - **方案二（矢量 FreeType 动态字库）**：利用引擎原生的 FreeType 2.x 接口接入现代矢量 `.ttf` / `.otf` 中文字体进行动态高精度栅格化。
