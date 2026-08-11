# Private GitHub Publication Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish the curated BigQuant factor library and five representative notebooks to the private `Erpaoer/factor-mining-bigquant` GitHub repository with an accurate Chinese README and verifiable exclusion of bulky research output.

**Architecture:** Keep the complete research directory untouched on disk and use a default-deny Git ignore policy to expose only approved source, documentation, and notebooks. Build and validate the repository locally in small commits, then create the private GitHub repository from `main`, push once, and verify remote privacy, branch, contents, and commit identity.

**Tech Stack:** PowerShell 5.1 with explicit UTF-8 pipeline encoding, Git 2.49, GitHub CLI 2.97, Python 3.13 via the Windows `py` launcher, pandas, NumPy, Jupyter Notebook JSON, BigQuant `dai` runtime for platform-specific execution.

---

## File map

- Modify: `.gitignore` — whitelist the repository deliverables while leaving the full local research archive in place.
- Create: `README.md` — public-facing project overview, factor catalog, usage, examples, validation boundary, and disclaimer.
- Existing source to track: `factor_lib/**/*.py` — 48 Python modules containing factors, adapters, preprocessing, discovery, evaluation, and strategy helpers.
- Existing documentation to track: `factor_lib/README.md` — detailed internal architecture and factor authoring conventions.
- Existing notebooks to track:
  - `我的文档/8.因子分析.ipynb`
  - `研究稿/华泰因子复现/华泰估值类因子/BP因子/BP因子.ipynb`
  - `研究稿/华泰因子复现/华泰财务质量因子/qfa_roe因子/qfa_roe.ipynb`
  - `研究稿/华泰因子复现/华泰成长类因子/Profit_G_q因子/Profit_G_q.ipynb`
  - `模拟策略/exp_wgt_return_6m防御性+收益补偿.ipynb`
- Existing decision records to retain:
  - `docs/superpowers/specs/2026-08-11-factor-mining-github-publishing-design.md`
  - `docs/superpowers/plans/2026-08-11-factor-mining-github-publishing.md`

No factor implementation file is edited by this plan.

### Task 1: Enforce the curated tracking boundary

**Files:**
- Modify: `.gitignore`
- Test: Git ignore behavior against explicit included and excluded paths

- [ ] **Step 1: Confirm the current ignore policy exposes a bulky notebook**

Run:

~~~powershell
$heavyNotebook = '研究稿\多因子策略--怪盗KIDV1研究稿\多因子策略--怪盗KIDV1.ipynb'
git check-ignore -q --no-index -- $heavyNotebook
if ($LASTEXITCODE -eq 0) {
    throw 'Baseline unexpectedly ignores the bulky notebook.'
}
Write-Output 'Baseline confirmed: the existing policy exposes the bulky notebook.'
~~~

Expected: `Baseline confirmed: the existing policy exposes the bulky notebook.`

- [ ] **Step 2: Configure readable non-ASCII Git paths for all later scope checks**

Run:

~~~powershell
git config --local core.quotepath false
$configured = git config --local --get core.quotepath
if ($configured -ne 'false') {
    throw "Unexpected core.quotepath value: $configured"
}
Write-Output 'Git path display configured: core.quotepath=false.'
~~~

Expected: `Git path display configured: core.quotepath=false.`

- [ ] **Step 3: Replace `.gitignore` with the exact default-deny policy**

Use `apply_patch` so the complete file becomes:

~~~gitignore
# Default-deny: the local folder is a complete research archive, while Git tracks
# only the curated private repository contents listed below.
*
!*/

# Repository entry points.
!.gitignore
!README.md

# Core factor library.
!factor_lib/README.md
!factor_lib/**/*.py

# Publishing decision records.
!docs/superpowers/specs/*.md
!docs/superpowers/plans/*.md

# Curated lightweight notebooks.
!我的文档/8.因子分析.ipynb
!研究稿/华泰因子复现/华泰估值类因子/BP因子/BP因子.ipynb
!研究稿/华泰因子复现/华泰财务质量因子/qfa_roe因子/qfa_roe.ipynb
!研究稿/华泰因子复现/华泰成长类因子/Profit_G_q因子/Profit_G_q.ipynb
!模拟策略/exp_wgt_return_6m防御性+收益补偿.ipynb

# Runtime and notebook caches remain excluded even inside allowed directories.
__pycache__/
*.py[cod]
.ipynb_checkpoints/

# Local credentials and private key material.
.env
.env.*
*.pem
*.key
~~~

- [ ] **Step 4: Verify representative allow and deny decisions**

Run:

~~~powershell
function Assert-Ignored([string]$Path) {
    git check-ignore -q --no-index -- $Path
    if ($LASTEXITCODE -ne 0) {
        throw "Expected ignored path: $Path"
    }
}

function Assert-Visible([string]$Path) {
    git check-ignore -q --no-index -- $Path
    if ($LASTEXITCODE -eq 0) {
        throw "Expected visible path: $Path"
    }
}

Assert-Visible '.gitignore'
Assert-Visible 'factor_lib\Factor Repository\base_factors\return_nd.py'
Assert-Visible 'factor_lib\README.md'
Assert-Visible '我的文档\8.因子分析.ipynb'
Assert-Visible '模拟策略\exp_wgt_return_6m防御性+收益补偿.ipynb'

Assert-Ignored 'factor_lib\LLM_CONTEXT.md'
Assert-Ignored 'factor_lib\Factor Repository\base_factors\__pycache__\return_nd.cpython-312.pyc'
Assert-Ignored '研究稿\多因子策略--怪盗KIDV1研究稿\多因子策略--怪盗KIDV1.ipynb'
Assert-Ignored '研究稿\华泰因子复现\华泰换手率类因子\bias_std_turn_5d因子\bias_std_turn_5d.ipynb'
Assert-Ignored '研究稿\华泰因子复现\华泰估值类因子\BP因子\BP因子.png'

Write-Output 'Tracking policy checks passed.'
~~~

Expected: `Tracking policy checks passed.`

- [ ] **Step 5: Confirm exactly 54 approved project files remain untracked**

Run:

~~~powershell
$visible = @(
    git ls-files --others --exclude-standard |
        Where-Object { $_ -ne '.gitignore' }
)
$expectedVisible = 54
if ($visible.Count -ne $expectedVisible) {
    $visible
    throw "Expected $expectedVisible visible untracked project files, found $($visible.Count)."
}
Write-Output "Visible untracked files: $($visible.Count)"
~~~

Expected: `Visible untracked files: 54`. The command deliberately removes the new `.gitignore` from the project-payload count.

- [ ] **Step 6: Commit only the tracking policy**

Run:

~~~powershell
git add -- .gitignore
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne '.gitignore') {
    throw "Unexpected staged paths: $($staged -join ', ')"
}
git diff --cached --check
git commit -m 'chore: curate repository contents'
~~~

Expected: one commit containing only `.gitignore`.

### Task 2: Create and validate the root README

**Files:**
- Create: `README.md`
- Reference: `factor_lib/README.md`
- Test: required sections, relative links, factor discovery, and minimal local calculation

- [ ] **Step 1: Verify the repository currently has no root README**

Run:

~~~powershell
if (Test-Path -LiteralPath 'README.md') {
    throw 'README.md already exists; inspect it before continuing.'
}
Write-Output 'Expected failure state confirmed: root README.md is absent.'
~~~

Expected: `Expected failure state confirmed: root README.md is absent.`

- [ ] **Step 2: Create `README.md` with the exact content below**

Use `apply_patch` to add:

~~~~markdown
# Factor Mining Based on BigQuant

一个面向 BigQuant 研究环境的量化因子库与多因子研究项目。仓库将分散的因子公式、数据适配、预处理、因子评价和策略模板整理为可发现、可复用的 Python 组件，并保留少量代表性 Notebook 作为研究入口。

> 当前仓库是从完整研究目录中整理出的精简私有版本。大型 Notebook、批量结果图片和运行缓存仍保留在原本地目录中，没有上传到 GitHub。

## 项目能做什么

- 通过统一的 <code>FACTOR</code> 元数据登记和发现因子；
- 使用 <code>get_factor</code> 按名称调用因子，避免手工维护导入表；
- 将 BigQuant <code>dai</code> 数据映射为稳定的语义字段；
- 复用 MAD 去极值、Z-Score 标准化、OLS 与行业市值中性化；
- 计算基础指标、因子相关性并管理候选因子聚类；
- 使用市值分组策略模板衔接因子研究与 BigQuant 回测；
- 支持 27 类普通因子和 1 类 GAN-GRU 机器学习因子。

## 精简仓库结构

~~~text
.
├── factor_lib/
│   ├── Factor Repository/
│   │   ├── base_factors/                 # 27 类参数化普通因子
│   │   └── machine_learning_factors/     # GAN-GRU 截面评分因子
│   ├── common/
│   │   ├── data_adapters/                # BigQuant 数据适配
│   │   └── preprocess/                   # 公共预处理
│   ├── factor_hub/                       # 因子发现、查询与调用
│   ├── function/bigquant_function/
│   │   ├── factor_evaluation/            # 指标、相关性、验证与聚类
│   │   └── strategies/                   # 市值分组回测模板
│   └── README.md                         # 详细架构与开发约定
├── 我的文档/8.因子分析.ipynb
├── 研究稿/华泰因子复现/                  # 3 个代表性因子 Notebook
├── 模拟策略/                             # 1 个轻量策略 Notebook
└── docs/superpowers/                     # 本次发布的设计与实施记录
~~~

完整组件设计和因子脚本规范见 [factor_lib 技术文档](factor_lib/README.md)。

## 因子目录

当前因子中心可发现 28 类因子：

| 类别 | 因子 |
|---|---|
| 收益、动量与价格形态 | <code>return_nd</code>、<code>return_nm</code>、<code>exp_wgt_return_nm</code>、<code>wgt_return_nm</code>、<code>intraday_return</code>、<code>open_gap</code>、<code>high_relative_return</code>、<code>low_relative_return</code>、<code>price_ma_distance_nd</code>、<code>macd_hist_relative</code>、<code>rsi_nd</code>、<code>williams_r_nd</code> |
| 波动率与残差波动 | <code>return_std_nd</code>、<code>hml_r_std_nm</code>、<code>id1_std_nm</code>、<code>id2_std_nm</code> |
| 量价、换手与资金流 | <code>volume_ratio_nd</code>、<code>volume_ma_ratio_nd</code>、<code>volume_relative_ma_nd</code>、<code>amount_relative_ma_nd</code>、<code>bias_std_turn_nd</code>、<code>mfd_buyamt_nd</code>、<code>mfd_sellamt_nd</code> |
| 估值、质量与成长 | <code>book_to_price</code>、<code>qfa_roe</code>、<code>profit_g_q</code>、<code>sales_g_q</code> |
| 机器学习 | <code>gan_gru_score</code> |

这里的类别用于阅读导航；运行时以各模块中的 <code>FACTOR</code> 元数据为准。

## 运行环境

项目主要面向 BigQuant Python 环境：

- 普通因子和因子中心主要依赖 <code>pandas</code>、<code>numpy</code>；
- BigQuant 取数和平台评价入口需要平台提供的 <code>dai</code>；
- 绘图与候选因子聚类按需使用 <code>matplotlib</code>、<code>scipy</code>；
- <code>gan_gru_score</code> 的训练或推理需要 <code>torch</code>。

仓库没有提供猜测性的固定版本 <code>requirements.txt</code>。BigQuant 平台镜像与普通本地 Python 环境的依赖版本可能不同，应在目标环境中记录实际可用版本后再锁定。

## 快速开始

在仓库根目录运行 Python，查看可用因子：

~~~python
from factor_lib.factor_hub.list_factors import list_factors

catalog = list_factors()
print(catalog.to_string(index=False))
~~~

使用最小日频数据计算 2 日收益因子：

~~~python
import pandas as pd

from factor_lib.factor_hub.get_factor import get_factor

daily_data = pd.DataFrame(
    {
        "date": pd.to_datetime(["2024-01-02", "2024-01-03", "2024-01-04"]),
        "instrument": ["000001.SZ", "000001.SZ", "000001.SZ"],
        "close": [100.0, 102.0, 105.0],
    }
)

result = get_factor(
    "return_nd",
    data=daily_data,
    target_dates=["2024-01-04"],
    window=2,
)
print(result)
~~~

预期 <code>return_nd</code> 为 <code>105 / 100 - 1 = 0.05</code>。真实研究中，输入数据应由 BigQuant 适配器准备，并显式区分预热区间、目标截面日期和信息截止日期。

## 示例 Notebook

- [因子分析流程](我的文档/8.因子分析.ipynb)
- [BP 估值因子复现](研究稿/华泰因子复现/华泰估值类因子/BP因子/BP因子.ipynb)
- [qfa_roe 财务质量因子复现](研究稿/华泰因子复现/华泰财务质量因子/qfa_roe因子/qfa_roe.ipynb)
- [Profit_G_q 成长因子复现](研究稿/华泰因子复现/华泰成长类因子/Profit_G_q因子/Profit_G_q.ipynb)
- [exp_wgt_return_6m 防御性与收益补偿策略](<模拟策略/exp_wgt_return_6m防御性+收益补偿.ipynb>)

这 5 个文件均小于 0.4 MB，能够代表分析、复现和策略三个使用层次。其输出仅用于研究过程说明，不构成可复制的未来收益承诺。

## 已完成的本地检查

- 48 个 Python 文件均通过 AST 语法解析；
- 因子中心可发现 28 个名称唯一的因子；
- 27 个普通因子已使用本地合成面板数据完成冒烟测试；
- 5 个示例 Notebook 均为有效 JSON，且不包含已保存的 <code>error</code> 输出；
- 发布文件经过常见私钥、GitHub Token、云访问密钥、Bearer Token、凭据赋值和带密码 URL 模式扫描。

合成数据冒烟测试只验证接口和基本计算路径，不等价于 BigQuant 真实数据回测，也不能验证因子的经济有效性。

## 已知限制

- BigQuant 数据字段、交易日历和回测执行行为必须在平台内复核；
- 个别历史文档中的导入路径或接口描述可能落后于当前目录结构，应以源码和本 README 为准；
- 因子方向约定、财务数据点时性以及 T/T+1 信号执行语义仍需结合具体策略验证；
- 机器学习因子需要额外模型状态、训练配置与 PyTorch 环境；
- 当前没有自动化 CI，也没有在普通本地环境模拟 BigQuant 的全部能力。

## 研究声明

本仓库仅用于量化研究、代码整理和方法复现，不构成投资建议、交易邀约或收益保证。历史结果、示例输出和因子统计可能受到数据口径、幸存者偏差、交易成本、滑点及前视偏差等影响。

本仓库为私有研究仓库，暂未附加开源许可证。
~~~~

- [ ] **Step 3: Check required README sections and all local links**

Run:

~~~powershell
$readme = [IO.File]::ReadAllText(
    (Resolve-Path -LiteralPath 'README.md'),
    [Text.Encoding]::UTF8
)

$requiredHeadings = @(
    '## 项目能做什么',
    '## 精简仓库结构',
    '## 因子目录',
    '## 运行环境',
    '## 快速开始',
    '## 示例 Notebook',
    '## 已完成的本地检查',
    '## 已知限制',
    '## 研究声明'
)

foreach ($heading in $requiredHeadings) {
    if (-not $readme.Contains($heading)) {
        throw "Missing README heading: $heading"
    }
}

$linkedFiles = @(
    'factor_lib\README.md',
    '我的文档\8.因子分析.ipynb',
    '研究稿\华泰因子复现\华泰估值类因子\BP因子\BP因子.ipynb',
    '研究稿\华泰因子复现\华泰财务质量因子\qfa_roe因子\qfa_roe.ipynb',
    '研究稿\华泰因子复现\华泰成长类因子\Profit_G_q因子\Profit_G_q.ipynb',
    '模拟策略\exp_wgt_return_6m防御性+收益补偿.ipynb'
)

foreach ($path in $linkedFiles) {
    if (-not (Test-Path -LiteralPath $path -PathType Leaf)) {
        throw "Broken README target: $path"
    }
}

Write-Output 'README structure and local links passed.'
~~~

Expected: `README structure and local links passed.`

- [ ] **Step 4: Execute the README minimal example without writing bytecode**

Run:

~~~powershell
$env:PYTHONDONTWRITEBYTECODE = '1'
$OutputEncoding = New-Object Text.UTF8Encoding($false)
@'
import math
import pandas as pd

from factor_lib.factor_hub.get_factor import get_factor

daily_data = pd.DataFrame(
    {
        "date": pd.to_datetime(["2024-01-02", "2024-01-03", "2024-01-04"]),
        "instrument": ["000001.SZ", "000001.SZ", "000001.SZ"],
        "close": [100.0, 102.0, 105.0],
    }
)
result = get_factor(
    "return_nd",
    data=daily_data,
    target_dates=["2024-01-04"],
    window=2,
)
assert list(result.columns) == ["date", "instrument", "return_nd"]
assert len(result) == 1
assert math.isclose(float(result.loc[0, "return_nd"]), 0.05, rel_tol=0, abs_tol=1e-12)
print("README quick-start passed: return_nd=0.05")
'@ | py -3.13 -X utf8 -
if ($LASTEXITCODE -ne 0) {
    throw 'README quick-start failed.'
}
~~~

Expected: `README quick-start passed: return_nd=0.05`.

- [ ] **Step 5: Commit only the root README**

Run:

~~~powershell
git add -- README.md
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne 'README.md') {
    throw "Unexpected staged paths: $($staged -join ', ')"
}
git diff --cached --check
git commit -m 'docs: add project README'
~~~

Expected: one commit containing only `README.md`.

### Task 3: Validate and commit the source and five notebooks

**Files:**
- Track: `factor_lib/README.md`
- Track: `factor_lib/**/*.py`
- Track: the five approved Notebook paths from the file map
- Test: credential patterns, Python syntax, factor discovery, Notebook JSON, staged scope, and file size

- [ ] **Step 1: Scan every visible repository file for credential-like patterns**

Run:

~~~powershell
$ErrorActionPreference = 'Stop'
$paths = @(git ls-files --cached --others --exclude-standard)
if ($paths.Count -ne 58) {
    throw "Expected 58 publishable files before staging source, found $($paths.Count)."
}

$patterns = [ordered]@{
    private_key = '-----BEGIN (?:RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----'
    github_token = '\bgh[pousr]_[A-Za-z0-9_]{20,}\b'
    aws_access_key = '\b(?:AKIA|ASIA)[A-Z0-9]{16}\b'
    bearer_token = '(?i)\bBearer\s+[A-Za-z0-9._~+/=-]{20,}'
    credential_assignment = '(?i)\b(?:api[_-]?key|access[_-]?token|auth[_-]?token|password|passwd|secret)\b\s*[:=]\s*["''][^"''\r\n]{8,}["'']'
    url_credentials = '(?i)\b(?:https?|postgres(?:ql)?|mysql)://[^/\s:@]+:[^@\s/]+@'
}

$findings = @()
foreach ($path in $paths) {
    $resolved = Resolve-Path -LiteralPath $path
    $content = [IO.File]::ReadAllText($resolved, [Text.Encoding]::UTF8)
    foreach ($entry in $patterns.GetEnumerator()) {
        if ([regex]::IsMatch($content, $entry.Value)) {
            $findings += [PSCustomObject]@{
                Path = $path
                Pattern = $entry.Key
            }
        }
    }
}

if ($findings.Count -gt 0) {
    $findings | Format-Table -AutoSize
    throw "Credential-like patterns found in $($findings.Count) file-pattern pairs."
}

Write-Output "Credential scan passed: $($paths.Count) files checked."
~~~

Expected: `Credential scan passed: 58 files checked.` No matched value is printed.

- [ ] **Step 2: Parse all Python modules and verify the factor registry**

Run:

~~~powershell
$env:PYTHONDONTWRITEBYTECODE = '1'
$OutputEncoding = New-Object Text.UTF8Encoding($false)
@'
import ast
from pathlib import Path

python_files = sorted(Path("factor_lib").rglob("*.py"))
assert len(python_files) == 48, len(python_files)
for path in python_files:
    ast.parse(path.read_text(encoding="utf-8"), filename=str(path))

from factor_lib.factor_hub.discover_factors import discover_factors

factors = discover_factors()
assert len(factors) == 28, len(factors)
assert len(set(factors)) == 28
base = [name for name, meta in factors.items() if meta.get("factor_type") == "base"]
machine_learning = [
    name
    for name, meta in factors.items()
    if meta.get("factor_type") == "machine_learning"
]
assert len(base) == 27, len(base)
assert machine_learning == ["gan_gru_score"], machine_learning
print("Python and registry checks passed: 48 modules, 27 base factors, 1 ML factor.")
'@ | py -3.13 -X utf8 -
if ($LASTEXITCODE -ne 0) {
    throw 'Python or factor registry check failed.'
}
~~~

Expected: `Python and registry checks passed: 48 modules, 27 base factors, 1 ML factor.`

- [ ] **Step 3: Validate the five Notebook files and their saved outputs**

Run:

~~~powershell
$env:PYTHONDONTWRITEBYTECODE = '1'
$OutputEncoding = New-Object Text.UTF8Encoding($false)
@'
import json
from pathlib import Path

notebooks = [
    Path("我的文档/8.因子分析.ipynb"),
    Path("研究稿/华泰因子复现/华泰估值类因子/BP因子/BP因子.ipynb"),
    Path("研究稿/华泰因子复现/华泰财务质量因子/qfa_roe因子/qfa_roe.ipynb"),
    Path("研究稿/华泰因子复现/华泰成长类因子/Profit_G_q因子/Profit_G_q.ipynb"),
    Path("模拟策略/exp_wgt_return_6m防御性+收益补偿.ipynb"),
]

assert len(notebooks) == 5
for path in notebooks:
    notebook = json.loads(path.read_text(encoding="utf-8"))
    assert notebook.get("nbformat") == 4, path
    errors = [
        output
        for cell in notebook.get("cells", [])
        for output in cell.get("outputs", [])
        if output.get("output_type") == "error"
    ]
    assert not errors, f"{path}: {len(errors)} saved error output(s)"

print("Notebook checks passed: 5 valid files, 0 saved error outputs.")
'@ | py -3.13 -X utf8 -
if ($LASTEXITCODE -ne 0) {
    throw 'Notebook validation failed.'
}
~~~

Expected: `Notebook checks passed: 5 valid files, 0 saved error outputs.`

- [ ] **Step 4: Stage only the library and five notebooks**

Run:

~~~powershell
git add -- factor_lib
git add -- '我的文档/8.因子分析.ipynb'
git add -- '研究稿/华泰因子复现/华泰估值类因子/BP因子/BP因子.ipynb'
git add -- '研究稿/华泰因子复现/华泰财务质量因子/qfa_roe因子/qfa_roe.ipynb'
git add -- '研究稿/华泰因子复现/华泰成长类因子/Profit_G_q因子/Profit_G_q.ipynb'
git add -- '模拟策略/exp_wgt_return_6m防御性+收益补偿.ipynb'

$staged = @(git diff --cached --name-only)
$pythonFiles = @($staged | Where-Object { $_.EndsWith('.py') })
$notebookFiles = @($staged | Where-Object { $_.EndsWith('.ipynb') })
$markdownFiles = @($staged | Where-Object { $_ -eq 'factor_lib/README.md' })

if ($staged.Count -ne 54) {
    $staged
    throw "Expected 54 staged source/example files, found $($staged.Count)."
}
if ($pythonFiles.Count -ne 48) {
    throw "Expected 48 staged Python files, found $($pythonFiles.Count)."
}
if ($notebookFiles.Count -ne 5) {
    throw "Expected 5 staged notebooks, found $($notebookFiles.Count)."
}
if ($markdownFiles.Count -ne 1) {
    throw 'factor_lib/README.md is missing from the staged set.'
}

Write-Output 'Staged scope passed: 48 Python + 5 Notebook + 1 technical README.'
~~~

Expected: `Staged scope passed: 48 Python + 5 Notebook + 1 technical README.`

- [ ] **Step 5: Check staged file sizes and whitespace**

Run:

~~~powershell
$staged = @(git diff --cached --name-only)
$items = foreach ($path in $staged) {
    Get-Item -LiteralPath $path
}
$largest = $items | Sort-Object Length -Descending | Select-Object -First 1
$totalBytes = ($items | Measure-Object -Property Length -Sum).Sum

if ($largest.Length -ge 10MB) {
    throw "Unexpected large file: $($largest.FullName) ($($largest.Length) bytes)"
}
if ($totalBytes -ge 5MB) {
    throw "Curated source/example payload is unexpectedly large: $totalBytes bytes"
}

git diff --cached --check
if ($LASTEXITCODE -ne 0) {
    throw 'Staged whitespace validation failed.'
}

Write-Output ("Largest staged file: {0:N2} MB" -f ($largest.Length / 1MB))
Write-Output ("Total staged payload: {0:N2} MB" -f ($totalBytes / 1MB))
~~~

Expected: largest file below 1 MB and total staged payload below 5 MB.

- [ ] **Step 6: Commit the source and examples**

Run:

~~~powershell
git commit -m 'feat: publish factor library and curated examples'
~~~

Expected: one commit with 54 files.

- [ ] **Step 7: Confirm the curated local repository is clean and has 58 files**

Run:

~~~powershell
$status = @(git status --porcelain)
if ($status.Count -ne 0) {
    $status
    throw 'Working tree has visible uncommitted changes.'
}

$tracked = @(git ls-files)
if ($tracked.Count -ne 58) {
    throw "Expected 58 tracked files, found $($tracked.Count)."
}

Write-Output "Local repository ready: $($tracked.Count) tracked files."
git log --oneline --decorate -5
~~~

Expected: clean working tree, 58 tracked files, and at least six local commits covering design, plan and environment corrections, ignore policy, README, and source/examples.

### Task 4: Create and push the private GitHub repository

**Files:**
- Create remote: `Erpaoer/factor-mining-bigquant`
- Configure local remote: `origin`
- Push branch: `main`

- [ ] **Step 1: Verify GitHub authentication through the local proxy**

Run:

~~~powershell
$gh = 'C:\Program Files\GitHub CLI\gh.exe'
$env:HTTPS_PROXY = 'http://127.0.0.1:7897'
$env:HTTP_PROXY = 'http://127.0.0.1:7897'

& $gh auth status --hostname github.com
if ($LASTEXITCODE -ne 0) {
    throw 'GitHub CLI authentication is unavailable.'
}

$login = & $gh api user --jq '.login'
if ($LASTEXITCODE -ne 0 -or $login.Trim() -ne 'Erpaoer') {
    throw "Unexpected GitHub login: $login"
}

Write-Output 'GitHub authentication passed for Erpaoer.'
~~~

Expected: authenticated account `Erpaoer` with repository scope; no full token is displayed.

- [ ] **Step 2: Confirm the target repository does not already exist**

Run:

~~~powershell
$null = & $gh repo view 'Erpaoer/factor-mining-bigquant' --json nameWithOwner 2>$null
if ($LASTEXITCODE -eq 0) {
    throw 'Target repository already exists; stop rather than overwrite or repoint it.'
}
Write-Output 'Target repository name is available.'
~~~

Expected: `Target repository name is available.`

- [ ] **Step 3: Configure Git credentials and create the private repository**

Run:

~~~powershell
& $gh auth setup-git
if ($LASTEXITCODE -ne 0) {
    throw 'Git credential setup failed.'
}

$createArgs = @(
    'repo',
    'create',
    'Erpaoer/factor-mining-bigquant',
    '--private',
    '--description',
    'BigQuant-based quantitative factor library and curated research examples.',
    '--source',
    '.',
    '--remote',
    'origin',
    '--push'
)
& $gh @createArgs

if ($LASTEXITCODE -ne 0) {
    throw 'Private repository creation or initial push failed.'
}
~~~

Expected: GitHub prints the new repository URL and pushes local `main` to `origin/main`.

### Task 5: Verify remote privacy, branch, contents, and commit

**Files:**
- Inspect remote repository only; no file changes

- [ ] **Step 1: Verify repository metadata**

Run:

~~~powershell
$metadataJson = & $gh repo view 'Erpaoer/factor-mining-bigquant' --json nameWithOwner,visibility,defaultBranchRef,url
$metadata = $metadataJson | ConvertFrom-Json

if ($metadata.nameWithOwner -ne 'Erpaoer/factor-mining-bigquant') {
    throw "Unexpected repository: $($metadata.nameWithOwner)"
}
if ($metadata.visibility -ne 'PRIVATE') {
    throw "Repository is not private: $($metadata.visibility)"
}
if ($metadata.defaultBranchRef.name -ne 'main') {
    throw "Unexpected default branch: $($metadata.defaultBranchRef.name)"
}

Write-Output "Repository: $($metadata.nameWithOwner)"
Write-Output "Visibility: $($metadata.visibility)"
Write-Output "Default branch: $($metadata.defaultBranchRef.name)"
Write-Output "URL: $($metadata.url)"
~~~

Expected: repository `Erpaoer/factor-mining-bigquant`, visibility `PRIVATE`, default branch `main`.

- [ ] **Step 2: Verify local and remote commits are identical**

Run:

~~~powershell
$localHead = (git rev-parse HEAD).Trim()
$remoteLine = (git ls-remote origin refs/heads/main).Trim()
$remoteHead = ($remoteLine -split '\s+')[0]

if (-not $remoteHead) {
    throw 'Remote main branch was not found.'
}
if ($localHead -ne $remoteHead) {
    throw "Commit mismatch: local=$localHead remote=$remoteHead"
}

Write-Output "Commit match: $localHead"
~~~

Expected: one identical 40-character commit hash.

- [ ] **Step 3: Recheck the tracked composition**

Run:

~~~powershell
$tracked = @(git ls-files)
$pythonFiles = @($tracked | Where-Object { $_.EndsWith('.py') })
$notebookFiles = @($tracked | Where-Object { $_.EndsWith('.ipynb') })
$imageFiles = @(
    $tracked | Where-Object {
        $_ -match '(?i)\.(png|jpg|jpeg|svg|webp)$'
    }
)
$cacheFiles = @(
    $tracked | Where-Object {
        $_ -match '(^|/)__pycache__/|(?i)\.pyc$'
    }
)

if ($tracked.Count -ne 58) {
    throw "Expected 58 tracked files, found $($tracked.Count)."
}
if ($pythonFiles.Count -ne 48) {
    throw "Expected 48 tracked Python files, found $($pythonFiles.Count)."
}
if ($notebookFiles.Count -ne 5) {
    throw "Expected 5 tracked notebooks, found $($notebookFiles.Count)."
}
if ($imageFiles.Count -ne 0) {
    throw "Tracked image files found: $($imageFiles -join ', ')"
}
if ($cacheFiles.Count -ne 0) {
    throw "Tracked cache files found: $($cacheFiles -join ', ')"
}

Write-Output 'Tracked composition passed: 58 total, 48 Python, 5 Notebook, 0 images, 0 caches.'
~~~

Expected: `Tracked composition passed: 58 total, 48 Python, 5 Notebook, 0 images, 0 caches.`

- [ ] **Step 4: Confirm branch tracking and clean local state**

Run:

~~~powershell
$status = @(git status --porcelain)
if ($status.Count -ne 0) {
    $status
    throw 'Working tree is not clean after publication.'
}

$branchStatus = git status --short --branch
if ($branchStatus -notmatch '^## main\.\.\.origin/main$') {
    throw "Unexpected branch tracking state: $branchStatus"
}

Write-Output $branchStatus
~~~

Expected: `## main...origin/main`.

- [ ] **Step 5: Report the completed private publication**

Report all of the following without exposing credentials:

- repository URL returned by GitHub;
- visibility `PRIVATE`;
- default branch `main`;
- latest commit hash and subject;
- 58 tracked files: 48 Python modules, 5 notebooks, 0 images, 0 cache files;
- README and design/plan file links;
- reminder that excluded files remain intact locally and that BigQuant real-data validation is still required.

## Spec coverage map

| Design requirement | Implemented by |
|---|---|
| Private `Erpaoer/factor-mining-bigquant` repository on `main` | Tasks 4–5 |
| Root Chinese README with purpose, structure, factor catalog, usage, validation, limitations, and disclaimer | Task 2 |
| Track 48 Python modules and `factor_lib/README.md` | Task 3 |
| Track exactly five approved lightweight notebooks | Tasks 1 and 3 |
| Exclude bulky notebooks, 102 PNG files, and caches without deleting them | Task 1 and Task 5 |
| Do not guess dependency versions or add a public license | README content in Task 2 |
| Credential scan that does not print matched values | Task 3, Step 1 |
| Syntax, registry, Notebook, link, size, and scope checks | Tasks 2–3 |
| Verify remote privacy, branch, contents, and commit identity | Task 5 |
