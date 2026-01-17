# CEEMDAN-Unlimited-IMFs

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.7+](https://img.shields.io/badge/python-3.7+-blue.svg)](https://www.python.org/downloads/)

**一个针对长时间序列分析的增强版CEEMDAN算法实现，解除了IMF数量的固有限制，可完整提取超长期趋势分量。**

---

## 🚀 项目简介

本项目是基于 `emd-signal` 库的 **CEEMDAN（自适应噪声完备集合经验模态分解）算法的增强版**。

原 `emd-signal` 库在CEEMDAN实现中存在一个**硬编码限制**：最多只能分解出 **8个** 本征模态函数（IMF）。这在处理**长时间序列数据**（如气候、金融、地质数据）时成为一个严重瓶颈，因为它无法将超长期的趋势或低频振荡从残差中完全分离出来，导致信息丢失。

**本项目的核心贡献是修复了这一问题**，通过修改内部实现，允许算法根据数据特性自适应地提取任意数量的IMF，直至满足自然的停止准则。这使得从多年甚至数十年的数据中，清晰分离出日周期、年周期、年代际振荡乃至长期气候基线成为可能。

## 🔧 核心原理与修改

### 原库的问题
在原 `emd-signal` 库的 `complete_ensemble_sift` 函数中，用于生成辅助白噪声IMF的过程**没有正确接收**用户指定的 `max_imfs` 参数，而是使用了内部默认的停止条件（约8个IMF）。这导致后续主分解过程无法访问更多阶的模态，从而触发 `IndexError`。

### 我们的解决方案
我们修复了关键函数调用，确保了 `max_imfs` 参数在整个分解流程中得以传递。这使得：
1.  **算法可以继续分解**，超越原有的8阶限制。
2.  **分解得以自然终止**，直到残余信号满足CEEMDAN的停止准则（如成为单调趋势），从而提取出有物理意义的**最终趋势项**。

此项修改可被视为对原库的一个 **"解锁"补丁**，并未改变CEEMDAN的核心数学原理，而是使其真正的"自适应"能力得以在长时间序列上完整发挥。

### 代码修改详情
**Modified code (sift.py, line 883-887):**
```python
# Before:
modes_white_noise = [sift(white_noise[ii, :],
                      imf_opts=imf_opts,
                      envelope_opts=envelope_opts,
                      extrema_opts=extrema_opts) for ii in range(nensembles)]

# After:
modes_white_noise = [sift(white_noise[ii, :],
                      imf_opts=imf_opts,
                      envelope_opts=envelope_opts,
                      extrema_opts=extrema_opts,
                      max_imfs=max_imfs) for ii in range(nensembles)]
```

## 📦 安装与使用

### 安装
1.  克隆本项目：
    ```bash
    git clone https://github.com/author9527/CEEMDAN-Unlimited-IMFs.git
    cd CEEMDAN-Unlimited-IMFs
    ```
2.  确保安装依赖库 `emd-signal` 和 `numpy`：
    ```bash
    pip install emd-signal numpy
    ```

### 快速开始
将本项目中的 `sift.py` 复制到您的项目目录并替换原库中的 `sift.py`，然后按如下方式使用：

```python
import numpy as np
import emd

# 1. 准备您的长时间序列数据
# 例如：一个包含多年每小时数据的数组
your_long_time_series = np.random.randn(10000)  # 请替换为您的真实数据

# 2. 执行增强版CEEMDAN分解
# 关键：现在可以指定大于8的 max_imfs 参数
imfs = emd.sift.complete_ensemble_sift(your_long_time_series, max_imfs=15, ensemble_noise=0.2)
# imfs 是一个 (n_samples, n_imfs) 的数组，每一列是一个IMF

print(f"成功提取了 {imfs.shape[1]} 个IMF")
```

## 📈 应用实例：27年太阳辐射数据分解

我们已成功将此工具应用于 **27年（1998-2024）逐小时全球水平辐照度（GHI）** 数据分析中。

### 分解结果亮点：
| 分解分量 | 主导周期 | 能量占比 | 物理意义 |
| :--- | :--- | :--- | :--- |
| **IMF 3** | 24小时 | 34.84% | **日周期（主导）** |
| **IMF 8** | 1年 | 3.52% | **年周期** |
| **IMF 13 (最终残差)** | 27年（趋势）| **46.77%** | **超长期气候基线** |

### 关键发现：
*   **完整分离**：成功提取了包含27年长期趋势在内的**13个IMF**，而原库在8阶后报错。
*   **趋势清晰**：最终残差（IMF13）揭示了GHI在27年间**极其稳定**，仅呈现微弱线性下降（年均-0.10 W/m²/年），标准差仅为1.49。
*   **多尺度特征**：清晰地分离了从数小时（天气）、数日（天气系统）、季节性到年代际的多尺度气候信号。

此案例证明了本工具在**气候时间序列分析、长期趋势检测、多尺度特征工程**方面的强大实用价值。

## ⚠️ 重要说明

**`max_imfs` 参数是一个上限，而非目标值。** 实际提取的IMF数量取决于：

1.  您指定的 `max_imfs` 值
2.  数据的内在特性
3.  库的停止条件（能量阈值、rilling测试等）

分解会在信号无法进一步有效分解时自然停止，无论 `max_imfs` 设置为多少。

## 📁 文件说明

-   `sift.py` - 修改后的库文件（替换原库中的同名文件）
-   `LICENSE` - MIT 许可证
-   `README.md` - 项目说明文档

## 🤝 致谢与许可证

*   本项目基于 [**emd-signal**](https://github.com/laszukdawid/emd) 库构建。由衷感谢原作者的杰出工作。
*   原始库采用 **MIT 许可证**，本修改版本同样遵循 **MIT 许可证** 开源。详见 [LICENSE](LICENSE) 文件。

## 📮 反馈与贡献

如果您在使用中发现任何问题，或有改进建议，欢迎通过 [GitHub Issues](https://github.com/author9527/CEEMDAN-Unlimited-IMFs/issues) 提交。

我们相信，解除IMF数量限制是许多长时间序列分析工作者的共同需求。希望这个工具能助力您的研究！
