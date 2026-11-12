# Profex-MCP 🧪🔬

> [Chinese (Simplified)](README-CN.md) · English

**AI-Agent-Friendly MCP Server for Profex / BGMN — XRD Analysis & Rietveld Refinement**

[![License](https://img.shields.io/badge/License-GPLv2-blue.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/Python-3.10+-blue.svg)]()
[![MCP Protocol](https://img.shields.io/badge/MCP-1.0+-green.svg)](https://modelcontextprotocol.io)
[![Search-Match DB](https://img.shields.io/badge/Fingerprints-2289-success.svg)]()

Profex-MCP wraps the classic powder XRD analysis suite [Profex](https://www.profex-xrd.org/) / [BGMN](https://www.bgmn.de/) as a standard **MCP (Model Context Protocol)** server, exposing two independent server groups: XRD analysis (Search-Match / COD query / data conversion) and BGMN refinement (refinement control / batch refinement / result analysis).

---

## Purpose

This repository combines the complete **Profex Qt5 sources** with an added **Python MCP layer**. The core C++ code is kept as upstream; the MCP capability is layered on top as an external Python package, `pyprofex`.

| Feature | Description |
|------|------|
| 🧩 **MCP protocol service** | Two server groups: XRD analysis (11 tools) + BGMN refinement (4 tools + 2 prompts) |
| 🔬 **Search-Match engine** | 2289-fingerprint library + iterative subtraction + independent scoring + adaptive tolerance |
| 🌐 **COD database integration** | Online COD search; match phases by element / mineral / d-spacing |
| 🧪 **Amorphous detection** | Amorphous phase identification via the dual-background method |
| 🤖 **AI workflows** | Refinement advisor, Search-Match guidance, RAG knowledge base |
| 📦 **Fast loading** | Precompiled pickle fingerprint library loads in ~0.02s (build artifact, see below) |

### Architecture

```
AI Agent (Claude/Copilot)
        │ MCP stdio
        ▼
┌─────────────────┐                    ┌────────────────────────┐
│  mcp_server.py  │  ← Search-Match →  │  search_match.py       │
│  (XRD Analysis)  │                    │  (2289 fingerprints)   │
│  11 tools       │  ← COD API ──────→ │  COD online database    │
└─────────────────┘                    └────────────────────────┘
        │ MCP stdio
        ▼
┌──────────────────────┐   subprocess   ┌────────────────────────┐
│  mcp_server_refine   │ ─────────────→ │  BGMN (Rietveld eng.)  │
│  (BGMN refine ctrl)   │                │  .sav → .par refine    │
│  4 tools + 2 prompts │                └────────────────────────┘
└──────────────────────┘
```

---

## Code Structure

```
Profex-MCP/
├── pyprofex/                               # 💚 Python MCP server + analysis engine (core)
│   ├── __init__.py                         #    Python package entry point
│   ├── mcp_server.py                       #    MCP Server: XRD analysis (11 tools / 1 prompt / 3 resources)
│   ├── mcp_server_refine.py                #    MCP Server: BGMN refinement (4 tools / 2 prompts / 1 resource)
│   ├── search_match.py                     #    🔍 Search-Match engine
│   │                                       #      iterative subtraction / independent scoring / adaptive tolerance / element filtering / preferred orientation
│   ├── peaks.py                            #    📈 Peak-finding algorithm (Gaussian fitting)
│   ├── fingerprints.py                     #    📚 Manually verified fingerprint library (27 minerals)
│   ├── fingerprints_auto.py                #    📚 CIF auto-generated fingerprint library (~2260 minerals)
│   ├── build_fingerprints.py               #    🏗️ COD db3 → fingerprint library build
│   ├── build_unified_db.py                 #    🏗️ Three-source merge → unified pickle
│   ├── build_powcod_fingerprints.py        #    🏗️ Qualx2 POWCOD conversion
│   ├── cif2fingerprint.py                  #    🔬 CIF → theoretical diffraction calculation engine
│   ├── profex_cli.py                       #    🖥️ CLI entry point (search / list-db / mcp, etc.)
│   ├── pyproject.toml  pyprofex.spec       #    Python package config / PyInstaller spec
│   └── tests/test_mcp_server.py            #    🧪 MCP protocol tests
├── knowledge-base/                         # 📚 XRD / Rietveld / BGMN RAG knowledge base (8 documents)
├── profex/                                 # 🟨 Profex C++ sources (Qt5, unmodified)
├── cmdtools/                               # 🟨 Profex command-line tools (pxanytoxy / pxreport …)
├── modules/                                # 🟨 Optional Profex modules
├── mcp-server/                             # 📖 MCP integration docs (README / AI_INTEGRATION)
├── scripts/                                # 🔧 build.sh / setup-dev.sh
├── schemas/                                # JSON Schema + YAML examples
├── .mcp.json                               # MCP host config
├── AI_ROADMAP.md                           # 📋 Full development roadmap
├── SEARCH_MATCH_ROADMAP.md                 # 📋 Search-Match optimization roadmap
└── Doxyfile                                # 📖 Doxygen config
```

> ⚠️ **About the fingerprint library pickle**: `fingerprints_unified.pkl` (2289 entries, loads in ~0.02s) is a **build artifact**, excluded by `.gitignore` (`*.pkl` / `*.json` / `*.db3` / `COD/`), and **not distributed with the repository**. Generate it yourself on first use by running `build_unified_db.py`; online COD queries and the built-in fingerprints in `fingerprints.py` / `fingerprints_auto.py` are unaffected.

### Module Dependencies

```
pyprofex (Python package)
    │
    ├── mcp_server.py          — MCP XRD analysis server (11 tools)
    │   ├── search_match.py    — Phase identification engine
    │   ├── fingerprints.py    — Manual fingerprint library (27)
    │   ├── fingerprints_auto.py — CIF auto fingerprints (~2260)
    │   ├── peaks.py           — Peak-finding algorithm
    │   └── profex_cli.py      — CLI interface
    │
    ├── mcp_server_refine.py   — MCP BGMN refinement server (4 tools)
    │   ├── Call BGMN binary    — Refinement execution
    │   └── .par parsing        — Result extraction
    │
    └── COD API (online)      — COD database query
```

---

## Search-Match Engine

The core engine lives in `search_match.py` and integrates three fingerprint sources:

### Fingerprint Database

| Source | Minerals | Description | Type |
|----|--------|------|------|
| `fingerprints.py` | **27** | Manually verified, high quality | manual |
| `fingerprints_auto.py` | **~2260** | COD CIF theoretical calculation (symmetry expansion + structure factors) | auto_cif |
| `fingerprints_powcod.json` | 0 (requires COD db3) | Qualx2 POWCOD (13944) | powcod |
| **Unified pickle** | **2289** | Precompiled, loads in ~0.02s (build artifact, not in repo) | merged |

### Algorithm Features

| Algorithm | Description |
|------|------|
| **Iterative subtraction** | Match phases one by one → subtract matched peaks → iterate until no peaks remain |
| **Independent scoring** | Score all candidates independently and recommend the best match |
| **Rarity weighting** | Phases matching "rare" peaks get a bonus |
| **Intensity-weighted FOM** | FOM = matches × intensity_coverage / (1 + unmatched) |
| **Adaptive tolerance** | Tolerance enlarged to 1.6× when d > 4Å |
| **Preferred orientation detection** | Flag PO when strong peaks are unmatched but all other peaks match |
| **Element filtering** | Use EDX/EDS element information to improve accuracy |

### Example

```python
from pyprofex.search_match import iterative_search_match

d_spacings = [3.34, 2.46, 1.82, 1.54, 1.38]   # quartz
result = iterative_search_match(d_spacings, ['Si', 'O'])

for p in result['phases']:
    print(f"{p['name']}: matches={p['matches']}, FOM={p['fom']}")
# → manual_Quartz: matches=5, FOM=2.5
```

---

## Installation

### Prerequisites

| Dependency | Purpose | How to install |
|------|------|----------|
| **Python 3.10+** | MCP server | `apt install python3.11` |
| Python packages | MCP / numpy / scipy | `pip install numpy scipy "mcp>=1.0"` |
| **Profex + BGMN** | GUI + Rietveld refinement (optional) | Download from the Profex website or build from source |
| **Qt 5.15+** | Profex GUI (optional) | System package manager |
| **C++ compiler** | Building Profex (optional) | `apt install build-essential` |

### 1️⃣ Get the code and Python dependencies

```bash
git clone https://github.com/FullPatt/Profex-MCP.git
cd Profex-MCP

pip install numpy scipy "mcp>=1.0"

# Verify the built-in fingerprint library
python3 -c "from pyprofex.search_match import get_db; db = get_db(); print(f'{len(db)} fingerprints')"
```

### 2️⃣ Build the unified fingerprint library (optional, for the 2289-entry pickle)

```bash
# Prepare COD data (db3) yourself; the build script converts it into a unified pickle
python3 pyprofex/build_unified_db.py
```

You can also use the built-in fingerprint library and online COD queries without this step.

### 3️⃣ Install Profex and BGMN (optional — for refinement)

```bash
# Method A: build scripts
bash scripts/setup-dev.sh
bash scripts/build.sh

# Method B: download a prebuilt package from the Profex website → https://www.profex-xrd.org/

# Set environment variables after installation
export PROFEX_HOME=/path/to/profex
export PATH=$PROFEX_HOME/profex:$PATH
```

### 4️⃣ Verify the installation

```bash
python3 -c "from pyprofex.mcp_server import server; print('✅ mcp_server OK')"
python3 -c "from pyprofex.mcp_server_refine import server_refine; print('✅ mcp_server_refine OK')"
python3 -c "from pyprofex.search_match import get_db; db = get_db(); print(f'✅ {len(db)} entries')"
```

---

## MCP Server Usage

### Launch

```bash
# XRD analysis server (Search-Match + COD + data conversion)
python3 pyprofex/mcp_server.py

# BGMN refinement server (refinement execution + result analysis)
python3 pyprofex/mcp_server_refine.py
```

### Claude Desktop Config

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

### XRD analysis tools (11)

| Tool | Function |
|------|------|
| `list_projects` | List Profex project files |
| `convert_file` | Convert XRD data formats |
| `parse_parameters` | Parse BGMN .par files |
| `list_scans` | List XRD scan data |
| `available_formats` | List supported import/export formats |
| `search_phase` | Search phase structure files |
| **`identify_phases`** | 🔥 Automatic Search-Match from a d-spacing list |
| `cod_search` | COD database search (by element / mineral / chemical formula / ID) |
| `cod_get_cif` | Download COD CIF files |
| `cod_search_by_d` | Search COD phases by d-spacing |
| **`search_match`** | 🔥 End-to-end pipeline: load XRD data → find peaks → Search-Match |

### BGMN refinement tools (4 + 2 prompts)

| Tool | Function |
|------|------|
| `run_refinement` | Run BGMN refinement via .sav control files |
| `get_results` | Parse .par results (Rwp, GoF, phase quantification) |
| `batch_refine` | Automatic batch refinement over multiple parameter combinations |
| `list_sav_files` | List .sav control files |

| Prompt | Function |
|--------|------|
| `refine_advisor` | AI refinement advisor — parameter selection and problem diagnosis |
| `search_match_workflow` | Guided Search-Match workflow |

### COD database queries

| Query type | Example |
|----------|------|
| By element | `cod_search("Fe,O")` → phases containing Fe and O |
| By mineral name | `cod_search("Quartz")` → quartz entries |
| By chemical formula | `cod_search("SiO2")` → silicon dioxide |
| By COD ID | `cod_search("1011097")` → the specified CIF |
| d-spacing match | `cod_search_by_d("3.34,2.46,1.82")` |

---

## Profex Capabilities

[Profex](https://www.profex-xrd.org/) is an open-source powder XRD analysis GUI built on [BGMN](https://www.bgmn.de/), developed by **Nico B.**

| Feature | Description |
|------|------|
| **Rietveld refinement** | Whole-pattern fitting via the BGMN engine |
| **Data format conversion** | Convert between 40+ XRD formats |
| **Peak finding + fitting** | Automatic/manual peak finding, Pseudo-Voigt fitting |
| **Search-Match** | Automatic phase identification via this repository's Python engine |
| **Batch processing** | Unified multi-file processing pipeline |
| **Structure parameter refinement** | Lattice constants, atomic positions, occupancies, temperature factors |
| **Microstructure analysis** | Crystallite size, microstrain (isotropic/anisotropic) |
| **Preferred orientation** | March-Dollase function correction |
| **Instrument calibration** | Calibration with standards (LaB6, NIST 660a/b/c) |

---

## Profex C++ Compilation

```bash
# 1. Install Qt5 + dependencies
sudo apt install qt5-qmake qtbase5-dev libqt5xmlpatterns5-dev \
  libqt5opengl5-dev libssh2-1-dev

# 2. Build Profex
cd Profex-MCP/profex
qmake profex.pro
make -j$(nproc)
./profex --version

# 3. Build the command-line tools
cd ../cmdtools
for tool in pxanytoxy pxapplypreset pxreport; do
    cd $tool && qmake && make -j$(nproc) && cd ..
done
```

---

## Changes vs Upstream

| Change | Path | Description |
|------|------|------|
| pyprofex Python package | `pyprofex/` | Complete MCP server package |
| MCP server (XRD) | `pyprofex/mcp_server.py` | 11 XRD analysis tools |
| MCP server (refinement) | `pyprofex/mcp_server_refine.py` | 4 refinement tools + 2 prompts |
| Search-Match engine | `pyprofex/search_match.py` | Iterative subtraction + independent scoring + adaptive tolerance |
| Fingerprint libraries | `pyprofex/fingerprints*.py` | 27 manually verified + ~2260 CIF auto-generated |
| Fingerprint build pipeline | `pyprofex/build_*.py` | Automatically generates the unified pickle (artifact not in repo) |
| COD API | `pyprofex/mcp_server.py` | cod_search / cod_get_cif / cod_search_by_d |
| CLI tool | `pyprofex/profex_cli.py` | search / list-db / db-info / suggest-elements |
| MCP integration docs | `mcp-server/` | README + AI_INTEGRATION |
| XRD knowledge base | `knowledge-base/` | 8 RAG documents |
| Development scripts | `scripts/` | build.sh, setup-dev.sh |
| JSON Schema | `schemas/` | refinement-config.example.yaml, etc. |
| `.mcp.json` | root directory | MCP host integration config |

> The Profex C++ sources (`profex/`, `cmdtools/`, `modules/`) are kept as upstream, unmodified.

---

## Development Roadmap

| Phase | Contents | Status |
|------|------|------|
| Phase 1 | Project foundation (LICENSE / README / Doxygen / Qt5 compatibility / MCP basics) | ✅ |
| Phase 2 | Build scripts and CI (build.sh / setup-dev.sh / GitHub Actions) | ✅ |
| Phase 3 | Headless CLI + Python bridge (profex_cli.py / JSON schema / YAML) | ✅ |
| Phase 4 | MCP Server (11 tools / resources / prompts / AI Editor integration) | ✅ |
| Phase 5 | Deep AI integration (refinement control / batch refinement / reports / XRD RAG knowledge base) | ✅ |
| Phase 6 | COD database API (search / CIF download / d-spacing matching) | ✅ |
| Phase S0–S3 | Search-Match optimization (iterative subtraction / adaptive tolerance / amorphous detection) | ✅ |
| — | Unified fingerprint library pickle build | ✅ |

### Next Steps

- [ ] Extend the fingerprint library to the full COD (~530,000 entries)
- [ ] Automatic PXRD data preprocessing pipeline
- [ ] Docker image (with Profex + BGMN preinstalled)
- [ ] CI/CD automated testing

---

## Testing

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

# MCP protocol tests
python3 -m pytest pyprofex/tests/test_mcp_server.py -v
```

---

## Related Projects

| Project | Description | Repository |
|------|------|------|
| **MAUD-MCP** | MAUD combined diffraction analysis MCP server | [FullPatt/MAUD-MCP](https://github.com/FullPatt/MAUD-MCP) |
| **FullProf-App-MCP** | FullProf Rietveld refinement MCP server | [FullPatt/FullProf-App-MCP](https://github.com/FullPatt/FullProf-App-MCP) |
| **GSAS2-MCP** | GSAS-II engine packaging (Rietveld refinement runtime foundation) | [FullPatt/GSAS2-MCP](https://github.com/FullPatt/GSAS2-MCP) |

---

## License

**GNU General Public License v2.0 (GPL-2.0)**, consistent with upstream Profex.

Profex is developed and maintained by **Nico B.**; this repository retains all copyright notices of upstream contributors.

---

## Acknowledgements

- **Nico B.** — Profex author and maintainer
- **Juan Rodríguez-Carvajal** — FullProf author, ILL
- **Luca Lutterotti** — MAUD author, University of Trento

---

*Profex homepage: https://www.profex-xrd.org/*
*BGMN homepage: https://www.bgmn.de/*
