# REA 接入与避坑说明

> 记录日期：2026-11-17　｜　当前版本：rea-agents **6.9.0**
> 用途：备份 REA 到个人库、或换机重装时，照此文档可一次性接回。

---

## 一、现状清单（本机实测路径）

| 部件 | 版本 | 路径 |
|---|---|---|
| REA 本体 | 6.9.0 | `D:/External/GitHubRepos/rea-runtime/`（node_modules/rea-agents） |
| Node 运行时 | 24.22.3 | `D:/WorkBuddyData/binaries/node/versions/node-v24.22.3-win-x64/node.exe` |
| Ghidra | 12.1.4 | `D:/External/REAtools/ghidra_12.1.4_PUBLIC` |
| JDK | 21.0.12.1 | `D:/External/REAtools/jdk-21.0.12.1+1` |
| 配套技能 | 6.9.0（255 工具） | `C:/Users/Administrator/.workbuddy/skills/reverse-engineer-anything/` |
| MCP 配置（生效） | — | `C:/Users/Administrator/.workbuddy/mcp.json` |

自检命令（应输出 `6.9.0`）：
```bash
cd "D:/External/GitHubRepos/rea-runtime" && \
export GHIDRA_INSTALL_DIR='D:\External\REAtools\ghidra_12.1.4_PUBLIC' \
       JAVA_HOME='D:\External\REAtools\jdk-21.0.12.1+1' && \
"D:/WorkBuddyData/binaries/node/versions/node-v24.22.3-win-x64/node.exe" \
  node_modules/rea-agents/scripts/rea.mjs --version
```

---

## 二、MCP 配置（完整可复制，接回就用这段）

放在 `C:/Users/Administrator/.workbuddy/mcp.json` 的 `mcpServers` 下：

```json
{
  "mcpServers": {
    "rea": {
      "type": "stdio",
      "command": "D:\\WorkBuddyData\\binaries\\node\\versions\\node-v24.22.3-win-x64\\node.exe",
      "args": [
        "D:\\External\\GitHubRepos\\rea-runtime\\node_modules\\rea-agents\\scripts\\rea.mjs",
        "mcp"
      ],
      "env": {
        "GHIDRA_INSTALL_DIR": "D:/External/REAtools/ghidra_12.1.4_PUBLIC",
        "JAVA_HOME": "D:/External/REAtools/jdk-21.0.12.1+1"
      }
    }
  }
}
```

要点：
- `type: "stdio"` 必须有——缺这个字段，部分版本不认；
- `args` 第二个参数是 **`mcp`**（新版写法；旧版 6.3.0 是 `--mcp`，别混用）；
- 路径按本机实际，换机后逐项替换。

---

## 三、接回步骤（关键在最后一步）

1. 本体、Node、Ghidra、JDK 按「一」的路径就位；
2. 把「二」的配置写进 `~/.workbuddy/mcp.json`；
3. **光写文件不够，重启也不生效** —— 必须打开 WorkBuddy 界面：
   - 左下角 **「+」/ 连接器管理** → **自定义连接器 / Custom MCP**；
   - 把上面那段 JSON 粘进去，**点保存**（或在列表里把 rea 的开关/信任打开）；
4. 保存后重启一次 WorkBuddy，rea 才出现在连接器列表并能调工具。

> 这次就是卡在第 3 步：磁盘文件早写好了，但界面没保存，列表里始终没有 rea。
> 粘过去点保存后立刻就有了。

---

## 四、今天踩的坑（逐条，别再踩）

| # | 坑 | 结论 / 处理 |
|---|---|---|
| 1 | 找不到连接器 | 本体一直在，是 `mcp.json` 里的 rea 段被清空了（`{"mcpServers":{}}`）——**不是软件丢了，是配置丢了** |
| 2 | 6.9.0 起不来 | 新版硬要求 **Node ≥22.23.4**，本机原 22.22.2 直接拒启 → 装 24.22.3 独立目录，`mcp.json` 写绝对路径 |
| 3 | npm audit 报 2 中危 | `minimatch` glob DoS，经 frida 带入；REA 是本地工具、glob 来自自己的配置，**不可利用**；且建议的"修复版"frida 16.1.4 反比现装 16.1.5 旧，属数据库倒挂误报，**不用管** |
| 4 | Hopper 引擎装不上 | 6.9.0 新增的分析引擎，**仅 macOS**（安装脚本全是 `hdiutil`/`codesign`/`.dmg`）→ Windows 跳过，**不影响 Ghidra 原生分析** |
| 5 | 配置写文件不生效 | 见「三」第 3 步：**必须在界面自定义连接器粘贴 JSON + 点保存**，光落盘/光重启都不行 |
| 6 | 技能 stale | 旧技能钉死 `@6.3.0`（139 工具）；已用 `rea setup --skill` 刷成 6.9.0 官方版（255 工具），旧版备份在 `reverse-engineer-anything.bak6.3.0` |
| 7 | 大件不进个人库 | JDK `lib/modules` 单文件 **135M > GitHub 100MB 上限**（直传必被拒），三样合计 1.5GB 超推荐值 → **本体/Node/Ghidra/JDK 一律不进库**，换机按「五」官方地址另下；库里只放本文档+配置 |

### 附：去旧留新的教训（底层文件别乱删）

- 清旧版本时把 `node/versions/22.22.2-6` 删了 568M，结果**重启后平台自动重装回来**（`CODEBUDDY_NODE_BIN` 环境变量写死指向该路径），白删还留下悬空风险 →
  **底层运行时由平台自管，删了会复活，不要动**；
- C 盘 `C:/Users/...` 属个人目录，批量删被系统 `bulk-delete` 守卫拦截，需换方式；
- 分寸：**自研项目内的旧源码/安装包/配置备份照删，node 这类底层运行时收着别碰**。

---

## 五、换机恢复（个人库不含大件，按下表另下）

个人库 `chenyu8886/rea` **只存本文档 + MCP 配置**（几 KB）。下列大件是公开官方工具，**不进库**，换机时按下表地址重新下载——全部 2026-11-17 实测可达。

### 5.1 需另下的四个大件

| 部件 | 版本 | 官方下载地址（实测） | 落地路径 |
|---|---|---|---|
| Node | 24.22.3 | `https://nodejs.org/dist/v24.22.3/node-v24.22.3-win-x64.zip` | `D:/WorkBuddyData/binaries/node/versions/node-v24.22.3-win-x64/` |
| Ghidra | 12.1.4 | `https://github.com/NationalSecurityAgency/ghidra/releases/download/Ghidra_12.1.4_build/ghidra_12.1.4_PUBLIC_20260921.zip` | 解压到 `D:/External/REAtools/`，目录名 `ghidra_12.1.4_PUBLIC` |
| JDK (Temurin) | 21.0.12.1+1 | `https://api.adoptium.net/v3/binary/version/jdk-21.0.12.1+1/windows/x64/jdk/hotspot/normal/eclipse`（自动重定向到最新构建） | 解压到 `D:/External/REAtools/`，目录名 `jdk-21.0.12.1+1` |
| rea 本体 | 6.9.0 | `npm install rea-agents@6.9.0`（npm registry，实测可达） | `D:/External/GitHubRepos/rea-runtime/node_modules/rea-agents` |

> JDK 发行版核对：解压后 `release` 文件应含 `IMPLEMENTOR="Eclipse Adoptium"`、`JAVA_VERSION="21.0.12.1"`。
> 若上面精确版本链接失效，改用 `latest/21/ga/windows/x64/jdk/hotspot/normal/eclipse` 拉 21 的最新版。

### 5.2 换机顺序

1. 按 5.1 下载四个大件 → 落到「一」的对应路径；
2. 刷新配套技能：`rea setup --skill --client workbuddy --skill-client workbuddy`；
3. 按「二」写 `mcp.json` → 按「三」在**界面粘贴保存** → 重启 WorkBuddy；
4. 跑「一」的自检命令应输出 `6.9.0`，再随便问个反编译任务确认工具可调。

### 5.3 为什么大件不进库

- GitHub **单文件 ≤100MB**：JDK `lib/modules` 135M，直传必失败；
- 三样合计约 1.5GB，远超仓库 1GB 推荐值，慢且易中断；
- 四样全是**公开官方工具**，随时可按 5.1 重下，进库无意义。
- 本机未装 Git LFS，不走大文件通道。
