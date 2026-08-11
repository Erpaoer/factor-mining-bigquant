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
