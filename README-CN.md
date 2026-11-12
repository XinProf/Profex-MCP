# Profex-MCP 🧪🔬

**AI-Agent-Friendly MCP Server for Profex / BGMN — XRD Analysis & Rietveld Refinement**

[![License](https://img.shields.io/badge/License-GPLv2-blue.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/Python-3.10+-blue.svg)]()
[![MCP Protocol](https://img.shields.io/badge/MCP-1.0+-green.svg)](https://modelcontextprotocol.io)
[![Search-Match DB](https://img.shields.io/badge/Fingerprints-2289-success.svg)]()

Profex-MCP 将经典粉末 XRD 分析软件 [Profex](https://www.profex-xrd.org/) / [BGMN](https://www.bgmn.de/) 封装为标准的 **MCP (Model Context Protocol)** 服务器，提供两组独立的 MCP 服务器：XRD 分析（物相鉴定 / COD 查询 / 数据转换）与 BGMN 精修（精修控制 / 批量精修 / 结果分析）。AI 智能体可直接调用全部功能。

Profex-MCP wraps the classic powder XRD analysis suite [Profex](https://www.profex-xrd.org/) / [BGMN](https://www.bgmn.de/) as a standard **MCP (Model Context Protocol)** server, exposing two independent server groups: XRD analysis (Search-Match / COD query / data conversion) and BGMN refinement (refinement control / batch refinement / result analysis).

---

## 项目目的 / Purpose

本仓库将完整的 **Profex Qt5 源码**与新增的 **Python MCP 支持层**打包在一起。核心 C++ 源码保持上游原样（未修改），MCP 能力以外部 Python 包 `pyprofex` 的形式叠加。

This repository combines the complete **Profex Qt5 sources** with an added **Python MCP layer**. The core C++ code is kept as upstream; the MCP capability is layered on top as an external Python package, `pyprofex`.

| 特性 | 说明 |
|------|------|
| 🧩 **MCP 协议服务** | 两组服务器：XRD 分析（11 工具）+ BGMN 精修（4 工具 + 2 提示词） |
| 🔬 **Search-Match 引擎** | 2289 指纹库 + 迭代扣除 + 独立评分 + 自适应容差 |
| 🌐 **COD 数据库集成** | 在线搜索 COD，按元素 / 矿物 / d-spacing 匹配物相 |
| 🧪 **非晶检测** | 双背景法非晶相识别 |
| 🤖 **AI 工作流** | 精修顾问、Search-Match 引导、RAG 知识库 |
| 📦 **快速加载** | 预编译 pickle 指纹库加载约 0.02s（构建产物，见下文） |

### 架构 / Architecture

```
AI Agent (Claude/Copilot)
        │ MCP stdio
        ▼
┌─────────────────┐                    ┌────────────────────────┐
│  mcp_server.py  │  ← Search-Match →  │  search_match.py       │
│  (XRD 分析)      │                    │  (2289 fingerprints)   │
│  11 tools       │  ← COD API ──────→ │  COD 在线数据库         │
└─────────────────┘                    └────────────────────────┘
        │ MCP stdio
        ▼
┌──────────────────────┐   subprocess   ┌────────────────────────┐
│  mcp_server_refine   │ ─────────────→ │  BGMN (Rietveld 引擎)  │
│  (BGMN 精修控制)      │                │  .sav → .par 精修      │
│  4 tools + 2 prompts │                └────────────────────────┘
└──────────────────────┘
```

---

## 代码结构 / Code Structure

```
Profex-MCP/
├── pyprofex/                               # 💚 Python MCP 服务器 + 分析引擎（核心）
│   ├── __init__.py                         #    Python 包入口
│   ├── mcp_server.py                       #    MCP Server: XRD 分析（11 tools / 1 prompt / 3 resources）
│   ├── mcp_server_refine.py                #    MCP Server: BGMN 精修（4 tools / 2 prompts / 1 resource）
│   ├── search_match.py                     #    🔍 Search-Match 引擎
│   │                                       #      迭代扣除 / 独立评分 / 自适应容差 / 元素过滤 / 择优取向
│   ├── peaks.py                            #    📈 寻峰算法（高斯拟合）
│   ├── fingerprints.py                     #    📚 手动验证指纹库（27 矿物）
│   ├── fingerprints_auto.py                #    📚 CIF 自动生成指纹库（~2260 矿物）
│   ├── build_fingerprints.py               #    🏗️ COD db3 → 指纹库构建
│   ├── build_unified_db.py                 #    🏗️ 三源合并 → 统一 pickle
│   ├── build_powcod_fingerprints.py        #    🏗️ Qualx2 POWCOD 转换
│   ├── cif2fingerprint.py                  #    🔬 CIF → 理论衍射计算引擎
│   ├── profex_cli.py                       #    🖥️ CLI 入口（search / list-db / mcp 等）
│   ├── pyproject.toml  pyprofex.spec       #    Python 包配置 / PyInstaller spec
│   └── tests/test_mcp_server.py            #    🧪 MCP 协议测试
├── knowledge-base/                         # 📚 XRD / Rietveld / BGMN RAG 知识库（8 篇）
├── profex/                                 # 🟨 Profex C++ 源码（Qt5，未修改）
├── cmdtools/                               # 🟨 Profex 命令行工具（pxanytoxy / pxreport …）
├── modules/                                # 🟨 可选 Profex 模块
├── mcp-server/                             # 📖 MCP 集成文档（README / AI_INTEGRATION）
├── scripts/                                # 🔧 build.sh / setup-dev.sh
├── schemas/                                # JSON Schema + YAML 示例
├── .mcp.json                               # MCP 宿主配置
├── AI_ROADMAP.md                           # 📋 完整开发路线图
├── SEARCH_MATCH_ROADMAP.md                 # 📋 Search-Match 优化路线图
└── Doxyfile                                # 📖 Doxygen 配置
```

> ⚠️ **关于指纹库 pickle**：`fingerprints_unified.pkl`（2289 条目、加载约 0.02s）是**构建产物**，由 `.gitignore` 排除（`*.pkl` / `*.json` / `*.db3` / `COD/`），**不随仓库分发**。首次使用需运行 `build_unified_db.py` 自行生成；在线 COD 查询与 `fingerprints.py` / `fingerprints_auto.py` 中的内置指纹不受影响。

### 模块依赖关系 / Module Dependencies

```
pyprofex (Python 包)
    │
    ├── mcp_server.py          — MCP XRD 分析服务器（11 tools）
    │   ├── search_match.py    — 物相鉴定引擎
    │   ├── fingerprints.py    — 手动指纹库（27）
    │   ├── fingerprints_auto.py — CIF 自动指纹（~2260）
    │   ├── peaks.py           — 寻峰算法
    │   └── profex_cli.py      — CLI 接口
    │
    ├── mcp_server_refine.py   — MCP BGMN 精修服务器（4 tools）
    │   ├── 调用 BGMN 二进制    — 精修执行
    │   └── .par 解析           — 结果提取
    │
    └── COD API（在线）         — COD 数据库查询
```

---

## Search-Match 引擎 / Search-Match Engine

核心引擎位于 `search_match.py`，集成三类指纹源：

### 指纹库 / Fingerprint Database

| 源 | 矿物数 | 说明 | 类型 |
|----|--------|------|------|
| `fingerprints.py` | **27** | 手动验证，高质量 | manual |
| `fingerprints_auto.py` | **~2260** | COD CIF 理论计算（对称扩展 + 结构因子） | auto_cif |
| `fingerprints_powcod.json` | 0（需 COD db3） | Qualx2 POWCOD（13944） | powcod |
| **统一 pickle** | **2289** | 预编译，加载约 0.02s（构建产物，不入库） | merged |

### 算法特性 / Algorithm Features

| 算法 | 说明 |
|------|------|
| **迭代扣除** | 逐相匹配 → 扣除匹配峰 → 迭代至无剩余峰 |
| **独立评分** | 对所有候选独立评分，推荐最优匹配 |
| **稀有度加权** | 匹配到"稀有"峰的物相加分 |
| **强度加权 FOM** | FOM = matches × intensity_coverage / (1 + unmatched) |
| **自适应容差** | d > 4Å 时容差放大至 1.6 倍 |
| **择优取向检测** | 强峰未匹配但其他峰均匹配时标记 PO |
| **元素过滤** | 结合 EDX/EDS 元素信息提升准确度 |

### 使用示例

```python
from pyprofex.search_match import iterative_search_match

d_spacings = [3.34, 2.46, 1.82, 1.54, 1.38]   # 石英
result = iterative_search_match(d_spacings, ['Si', 'O'])

for p in result['phases']:
    print(f"{p['name']}: matches={p['matches']}, FOM={p['fom']}")
# → manual_Quartz: matches=5, FOM=2.5
```

---

## 编译与安装 / Installation

### 前置条件 / Prerequisites

| 依赖 | 用途 | 安装方式 |
|------|------|----------|
| **Python 3.10+** | MCP 服务器 | `apt install python3.11` |
| Python 包 | MCP / numpy / scipy | `pip install numpy scipy "mcp>=1.0"` |
| **Profex + BGMN** | GUI + Rietveld 精修（可选） | 从 Profex 官网下载或自行编译 |
| **Qt 5.15+** | Profex GUI（可选） | 系统包管理器 |
| **C++ 编译器** | Profex 编译（可选） | `apt install build-essential` |

### 1️⃣ 获取代码与 Python 依赖

```bash
git clone https://github.com/FullPatt/Profex-MCP.git
cd Profex-MCP

pip install numpy scipy "mcp>=1.0"

# 验证内置指纹库
python3 -c "from pyprofex.search_match import get_db; db = get_db(); print(f'{len(db)} fingerprints')"
```

### 2️⃣ 构建统一指纹库（可选，用于 2289 条目 pickle）

```bash
# 需自行准备 COD 数据（db3），构建脚本会将其转换为统一 pickle
python3 pyprofex/build_unified_db.py
```

不做此步也可使用内置指纹库与在线 COD 查询。

### 3️⃣ 安装 Profex 和 BGMN（可选 — 用于精修）

```bash
# 方式 A：构建脚本
bash scripts/setup-dev.sh
bash scripts/build.sh

# 方式 B：从 Profex 官网下载预编译包 → https://www.profex-xrd.org/

# 安装后设置环境变量
export PROFEX_HOME=/path/to/profex
export PATH=$PROFEX_HOME/profex:$PATH
```

### 4️⃣ 验证安装

```bash
python3 -c "from pyprofex.mcp_server import server; print('✅ mcp_server OK')"
python3 -c "from pyprofex.mcp_server_refine import server_refine; print('✅ mcp_server_refine OK')"
python3 -c "from pyprofex.search_match import get_db; db = get_db(); print(f'✅ {len(db)} entries')"
```

---

## MCP 服务器使用 / MCP Server Usage

### 启动

```bash
# XRD 分析服务器（Search-Match + COD + 数据转换）
python3 pyprofex/mcp_server.py

# BGMN 精修服务器（精修执行 + 结果分析）
python3 pyprofex/mcp_server_refine.py
```

### MCP 宿主配置 / Claude Desktop Config

```json
{
  "mcpServers": {
    "profex-xrd": {
      "command": "python3",
      "args": ["pyprofex/mcp_server.py"],
      "cwd": "/path/to/Profex-MCP"
    },
    "profex-refine": {
      "command": "python3",
      "args": ["pyprofex/mcp_server_refine.py"],
      "cwd": "/path/to/Profex-MCP"
    }
  }
}
```

### XRD 分析工具（11 个）

| 工具 | 功能 |
|------|------|
| `list_projects` | 列示 Profex 项目文件 |
| `convert_file` | XRD 数据格式转换 |
| `parse_parameters` | 解析 BGMN .par 文件 |
| `list_scans` | 列示 XRD 扫描数据 |
| `available_formats` | 列出支持的导入/导出格式 |
| `search_phase` | 搜索物相结构文件 |
| **`identify_phases`** | 🔥 从 d-spacing 列表自动 Search-Match |
| `cod_search` | COD 数据库搜索（按元素 / 矿物 / 化学式 / ID） |
| `cod_get_cif` | 下载 COD CIF 文件 |
| `cod_search_by_d` | 按 d-spacing 搜索 COD 物相 |
| **`search_match`** | 🔥 端到端管线：加载 XRD 数据 → 寻峰 → Search-Match |

### BGMN 精修工具（4 个 + 2 个提示词）

| 工具 | 功能 |
|------|------|
| `run_refinement` | 通过 .sav 控制文件执行 BGMN 精修 |
| `get_results` | 解析 .par 结果（Rwp、GoF、物相定量） |
| `batch_refine` | 多参数组合自动批量精修 |
| `list_sav_files` | 列示 .sav 控制文件 |

| 提示词 | 功能 |
|--------|------|
| `refine_advisor` | AI 精修顾问 — 参数选择与问题诊断 |
| `search_match_workflow` | 引导式 Search-Match 工作流 |

### COD 数据库查询

| 查询类型 | 示例 |
|----------|------|
| 按元素 | `cod_search("Fe,O")` → 含 Fe 和 O 的物相 |
| 按矿物名 | `cod_search("Quartz")` → 石英条目 |
| 按化学式 | `cod_search("SiO2")` → 二氧化硅 |
| 按 COD ID | `cod_search("1011097")` → 指定 CIF |
| d-spacing 匹配 | `cod_search_by_d("3.34,2.46,1.82")` |

---

## Profex 能力概述 / Profex Capabilities

[Profex](https://www.profex-xrd.org/) 是基于 [BGMN](https://www.bgmn.de/) 的开源粉末 XRD 分析 GUI，由 **Nico B.** 开发。

| 功能 | 说明 |
|------|------|
| **Rietveld 精修** | 通过 BGMN 引擎执行全谱拟合 |
| **数据格式转换** | 支持 40+ XRD 格式互相转换 |
| **寻峰 + 拟合** | 自动/手动寻峰，Pseudo-Voigt 拟合 |
| **Search-Match** | 通过本仓库的 Python 引擎实现自动物相鉴定 |
| **批量处理** | 多文件统一处理管线 |
| **结构参数精修** | 晶格常数、原子位置、占有率、温度因子 |
| **微结构分析** | 晶粒尺寸、微应变（各向同性/各向异性） |
| **择优取向** | March-Dollase 函数修正 |
| **仪器校准** | 标准样品（LaB6、NIST 660a/b/c）校准 |

---

## Profex C++ 编译指南 / Profex C++ Compilation

```bash
# 1. 安装 Qt5 + 依赖
sudo apt install qt5-qmake qtbase5-dev libqt5xmlpatterns5-dev \
  libqt5opengl5-dev libssh2-1-dev

# 2. 编译 Profex
cd Profex-MCP/profex
qmake profex.pro
make -j$(nproc)
./profex --version

# 3. 编译命令行工具
cd ../cmdtools
for tool in pxanytoxy pxapplypreset pxreport; do
    cd $tool && qmake && make -j$(nproc) && cd ..
done
```

---

## 本仓库的变更 / Changes vs Upstream

| 变更 | 路径 | 说明 |
|------|------|------|
| pyprofex Python 包 | `pyprofex/` | 完整 MCP 服务器包 |
| MCP 服务器（XRD） | `pyprofex/mcp_server.py` | 11 个 XRD 分析工具 |
| MCP 服务器（精修） | `pyprofex/mcp_server_refine.py` | 4 个精修工具 + 2 个提示词 |
| Search-Match 引擎 | `pyprofex/search_match.py` | 迭代扣除 + 独立评分 + 自适应容差 |
| 指纹库 | `pyprofex/fingerprints*.py` | 手动验证 27 + CIF 自动 ~2260 |
| 指纹库构建管线 | `pyprofex/build_*.py` | 自动生成统一 pickle（产物不入库） |
| COD API | `pyprofex/mcp_server.py` | cod_search / cod_get_cif / cod_search_by_d |
| CLI 工具 | `pyprofex/profex_cli.py` | search / list-db / db-info / suggest-elements |
| MCP 集成文档 | `mcp-server/` | README + AI_INTEGRATION |
| XRD 知识库 | `knowledge-base/` | 8 篇 RAG 文档 |
| 开发脚本 | `scripts/` | build.sh、setup-dev.sh |
| JSON Schema | `schemas/` | refinement-config.example.yaml 等 |
| `.mcp.json` | 根目录 | MCP 宿主集成配置 |

> Profex C++ 源码（`profex/`、`cmdtools/`、`modules/`）保持上游原样，未作修改。

---

## 开发路线图 / Development Roadmap

| 阶段 | 内容 | 状态 |
|------|------|------|
| Phase 1 | 项目基础建设（LICENSE / README / Doxygen / Qt5 兼容 / MCP 基础） | ✅ |
| Phase 2 | 构建脚本与 CI（build.sh / setup-dev.sh / GitHub Actions） | ✅ |
| Phase 3 | Headless CLI + Python 桥接（profex_cli.py / JSON schema / YAML） | ✅ |
| Phase 4 | MCP Server（11 工具 / 资源 / 提示词 / AI Editor 集成） | ✅ |
| Phase 5 | AI 深层集成（精修控制 / 批量精修 / 报告 / XRD RAG 知识库） | ✅ |
| Phase 6 | COD 数据库 API（搜索 / CIF 下载 / d-spacing 匹配） | ✅ |
| Phase S0–S3 | Search-Match 优化（迭代扣除 / 自适应容差 / 非晶检测） | ✅ |
| — | 统一指纹库 pickle 构建 | ✅ |

### 下一步 / Next Steps

- [ ] 扩展指纹库至全部 COD（约 53 万条目）
- [ ] PXRD 数据自动预处理管线
- [ ] Docker 镜像（含 Profex + BGMN 预装）
- [ ] CI/CD 自动化测试

---

## 测试 / Testing

```bash
cd /path/to/Profex-MCP

python3 -c "
from pyprofex.search_match import get_db, iterative_search_match
db = get_db()
result = iterative_search_match([3.34, 2.46, 1.82, 1.54, 1.38], ['Si', 'O'])
print(f'✅ {len(db)} entries, found {result[\"total_phases_found\"]} phases')
"

python3 -c "from pyprofex.mcp_server import server; print('✅ mcp_server')"
python3 -c "from pyprofex.mcp_server_refine import server_refine; print('✅ mcp_server_refine')"

# MCP 协议测试
python3 -m pytest pyprofex/tests/test_mcp_server.py -v
```

---

## 相关项目 / Related Projects

| 项目 | 说明 | 仓库 |
|------|------|------|
| **MAUD-MCP** | MAUD 组合衍射分析 MCP 服务器 | [FullPatt/MAUD-MCP](https://github.com/FullPatt/MAUD-MCP) |
| **FullProf-App-MCP** | FullProf Rietveld 精修 MCP 服务器 | [FullPatt/FullProf-App-MCP](https://github.com/FullPatt/FullProf-App-MCP) |
| **GSAS2-MCP** | GSAS-II 引擎打包（Rietveld 精修运行时底座） | [FullPatt/GSAS2-MCP](https://github.com/FullPatt/GSAS2-MCP) |

---

## 许可证 / License

**GNU General Public License v2.0（GPL-2.0）**，与 Profex 上游一致。

Profex 由 **Nico B.** 开发维护；本仓库保留了上游贡献者的全部版权声明。

---

## 致谢 / Acknowledgements

- **Nico B.** — Profex 作者与维护者
- **Juan Rodríguez-Carvajal** — FullProf 作者，ILL
- **Luca Lutterotti** — MAUD 作者，University of Trento

---

*Profex 主页: https://www.profex-xrd.org/*
*BGMN 主页: https://www.bgmn.de/*
