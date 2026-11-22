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
