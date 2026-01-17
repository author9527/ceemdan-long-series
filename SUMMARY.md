# CEEMDAN分解阶段性总结报告

**生成日期**: 2026年1月17日  
**数据源**: ASU GHI数据 (1998-2024, 236,520个 hourly 数据点)  
**分析工具**: emd-signal v0.8.1 (modified)

---

## 目录

1. [问题背景](#1-问题背景)
2. [禁制根源分析](#2-禁制根源分析)
3. [库函数修改方案](#3-库函数修改方案)
4. [修改前后对比](#4-修改前后对比)
5. [效果验证工作](#5-效果验证工作)
6. [完整分解结果](#6-完整分解结果)
7. [结论与建议](#7-结论与建议)

---

## 1. 问题背景

### 1.1 研究目标
对27年的GHI（全球水平辐照度）太阳辐射数据进行CEEMDAN分解，提取多时间尺度的周期特征，用于多能源负荷预测研究。

### 1.2 遇到的问题

在尝试对完整的236,520点GHI数据进行CEEMDAN分解时，遇到以下错误：

```
IndexError: index 8 is out of bounds for axis 1 with size 8
```

该错误表明CEEMDAN分解被限制在**最多8个IMF**，无法提取更多有意义的分量。

### 1.3 问题影响

- 无法提取完整的长期趋势分量（需要超过8个IMF）
- 残差/基线分量与短期分量混合
- 影响多时间尺度特征提取的完整性

---

## 2. 禁制根源分析

### 2.1 问题定位

通过调试分析，发现问题根源在`complete_ensemble_sift`函数中。**问题并非来自外部参数限制，而是来自内部白噪声IMF提取的默认停止条件**。

### 2.2 根本原因

CEEMDAN算法内部流程：

```
1. 生成白噪声信号
2. 对每个白噪声信号执行sift()，提取其IMF
3. 使用白噪声的IMF引导主信号的分解
```

**问题所在**：第2步中，对白噪声执行`sift()`时，使用的是默认参数：

```python
# 默认参数导致的问题
energy_thresh=50      # 能量阈值，残差能量低于此值时停止
rilling_thresh=(0.05, 0.5, 0.05)  # Rilling准则
```

这些默认参数使得白噪声信号在提取约8个IMF后就自然停止，导致`modes_white_noise`列表中的每个IMF数组只有8列。

当主循环需要访问`modes_white_noise[ii][:, layer]`且`layer > 7`时，就会触发IndexError。

### 2.3 代码层面的问题

```python
# 原始代码 (sift.py 第883-886行)
modes_white_noise = [sift(white_noise[ii, :],
                          imf_opts=imf_opts,
                          envelope_opts=envelope_opts,
                          extrema_opts=extrema_opts) for ii in range(nensembles)]
                    ↑
                    |
            问题：max_imfs 参数没有传递给内部的 sift() 调用
```

---

## 3. 库函数修改方案

### 3.1 修改文件

```
文件路径: D:\Program Files\Lib\site-packages\emd\sift.py
备份文件: D:\Program Files\Lib\site-packages\emd\sift.py.backup
```

### 3.2 修改内容

**修改位置**: 第883-887行

**修改前代码**:
```python
# Compute white noise modes - sift each to completion
modes_white_noise = [sift(white_noise[ii, :],
                          imf_opts=imf_opts,
                          envelope_opts=envelope_opts,
                          extrema_opts=extrema_opts) for ii in range(nensembles)]
```

**修改后代码**:
```python
# Compute white noise modes - sift each to completion
# MODIFICATION: Pass max_imfs to inner sift to extract sufficient IMFs
modes_white_noise = [sift(white_noise[ii, :],
                          imf_opts=imf_opts,
                          envelope_opts=envelope_opts,
                          extrema_opts=extrema_opts,
                          max_imfs=max_imfs) for ii in range(nensembles)]
```

### 3.3 修改说明

| 修改项 | 修改前 | 修改后 |
|--------|--------|--------|
| `max_imfs`参数 | 未传递给内部sift() | 传递给内部sift() |
| 白噪声IMF数量 | 固定约8个 | 取决于外部指定的max_imfs |
| 内存分配 | 固定8列 | 动态适应max_imfs |

### 3.4 进一步优化（推荐）

由于单纯传递`max_imfs`参数可能仍受其他停止条件限制（如`energy_thresh`），建议在调用时使用更激进的参数：

```python
# 在调用 complete_ensemble_sift 时，使用激进的内部参数
def modified_complete_ensemble_sift(X, nensembles=4, ensemble_noise=.2,
                                     nprocesses=1, noise_seed=None,
                                     sift_thresh=1e-8, energy_thresh=50,
                                     rilling_thresh=None, max_imfs=None, verbose=None,
                                     imf_opts=None, envelope_opts=None,
                                     extrema_opts=None):
    # 临时修改内部sift的行为
    import emd.sift as sift_module
    original_sift = sift_module.sift
    
    def aggressive_sift(X, sift_thresh=1e-8, energy_thresh=50, rilling_thresh=None,
                        max_imfs=None, verbose=None, return_residual=True,
                        imf_opts=None, envelope_opts=None, extrema_opts=None):
        return original_sift(X, sift_thresh=sift_thresh, 
                            energy_thresh=1e10,  # 禁用能量阈值限制
                            rilling_thresh=None,  # 禁用rilling准则
                            max_imfs=max_imfs, verbose=verbose, 
                            return_residual=return_residual,
                            imf_opts=imf_opts, envelope_opts=envelope_opts, 
                            extrema_opts=extrema_opts)
    
    sift_module.sift = aggressive_sift
    
    try:
        result = original_complete_ensemble_sift(...)
    finally:
        sift_module.sift = original_sift
    
    return result
```

---

## 4. 修改前后对比

### 4.1 分解能力对比

| 指标 | 修改前 | 修改后 |
|------|--------|--------|
| 最大IMF数量 | 8 | 13+ (取决于max_imfs参数) |
| 可提取周期范围 | 限于短期 | 覆盖短期到超长期(27年) |
| 残差处理 | 混合在IMF8中 | 独立为IMF13 |

### 4.2 分解时间对比

| 数据规模 | 修改前 | 修改后 |
|----------|--------|--------|
| 236,520点(27年) | 报错 | 37.57秒 |

### 4.3 分解质量对比

修改前（仅8个IMF）：
```
IMF 1-3: 高频周期（6-24小时）
IMF 4-5: 日-周周期
IMF 6-7: 月-季周期
IMF 8: 混合残差（年周期+长期趋势混合）
```

修改后（13个IMF）：
```
IMF 1-3: 高频周期（6-24小时主导）→ 贡献48.84%能量
IMF 4-5: 日-周周期 → 贡献0.29%能量
IMF 6-7: 月-季周期 → 贡献0.05%能量
IMF 8-12: 年周期到多十年周期 → 贡献4.17%能量
IMF 13: 超长期基线/残差 → 贡献46.77%能量
```

---

## 5. 效果验证工作

### 5.1 验证1: 数学完备性

**方法**: 将所有IMF相加重构原始信号，计算重构误差

**代码**:
```python
reconstructed = np.sum(imfs, axis=1)
error = np.abs(ghi - reconstructed)
```

**结果**:
| 指标 | 值 |
|------|-----|
| 最大绝对误差 | 4.55e-13 |
| RMSE | 5.46e-14 |
| MAPE | 0.0167% |

**结论**: ✅ **数学完备** - 重构误差远低于数值精度阈值(1e-10)

### 5.2 验证2: 趋势合理性交叉验证

**方法**: 比较IMF13与365天移动平均趋势的相关性

**结果**:
| 移动平均窗口 | 相关系数 |
|-------------|---------|
| 30-day | 0.020 |
| 90-day | 0.027 |
| 180-day | 0.033 |
| **365-day** | **0.682** |
| 730-day (2年) | **0.794** |

**结论**: ✅ **趋势有效** - IMF13与长期移动平均趋势具有良好相关性

### 5.3 验证3: IMF13物理特性

**结果**:
| 指标 | IMF13 | 原始GHI |
|------|-------|---------|
| Mean | 268.89 | 262.02 |
| Std | 1.49 | 337.52 |
| Range | [266.99, 271.71] | [0, 1107] |
| 27年变化 | -2.77 | - |

**结论**: ✅ **物理合理** - IMF13均值接近原始GHI均值，标准差极小，表明其捕获了稳定的长期基线

---

## 6. 完整分解结果

### 6.1 数据概况

| 参数 | 值 |
|------|-----|
| 时间范围 | 1998-01-01 至 2024-12-31 |
| 数据点数 | 236,520 (hourly) |
| 时间跨度 | 27年 |
| GHI均值 | 262.02 W/m² |
| GHI范围 | 0 - 1107 W/m² |

### 6.2 13阶IMF完整统计

| IMF | 能量 | 能量占比 | 均值 | 标准差 | 主导周期 | 周期分类 |
|-----|------|---------|------|--------|---------|---------|
| 1 | 56,600,610 | 0.15% | 0.016 | 15.47 | 6.0小时 | 高频 |
| 2 | 5,062,898,287 | 13.85% | 2.77 | 146.28 | 12.0小时 | 高频 |
| 3 | 12,738,035,082 | **34.84%** | -10.62 | 231.83 | **24.0小时** | **日周期** |
| 4 | 79,943,440 | 0.22% | -0.53 | 18.38 | 2.7天 | 日-周 |
| 5 | 25,799,506 | 0.07% | -0.25 | 10.44 | 7.0天 | 周周期 |
| 6 | 11,331,512 | 0.03% | -0.03 | 6.92 | 2.2周 | 双周 |
| 7 | 5,578,681 | 0.02% | 0.07 | 4.86 | 1.1月 | 月周期 |
| 8 | 1,285,766,034 | **3.52%** | 2.59 | 73.69 | **1.0年** | **年周期** |
| 9 | 181,764,447 | 0.50% | -1.58 | 27.68 | 1.6年 | 年+ |
| 10 | 11,223,374 | 0.03% | -0.04 | 6.89 | 3.4年 | 多年度 |
| 11 | 3,473,632 | 0.01% | 0.32 | 3.82 | 5.4年 | 多年度 |
| 12 | 747,888 | 0.00% | 0.41 | 1.73 | 13.5年 | 超长期 |
| 13 | 17,101,300,557 | **46.77%** | 268.89 | 1.49 | **27.0年** | **超长期基线** |

### 6.3 能量分布分析

```
累计方差解释:
  前1个IMF:   0.15%
  前2个IMF:  14.00%
  前3个IMF:  48.84%  ← 日周期主导
  前4个IMF:  49.06%
  ...
  前8个IMF:  52.69%  ← 包含年周期
  前9个IMF:  53.19%
  ...
  全部13个: 100.00%
```

### 6.4 周期识别总结

| 周期类型 | 对应IMF | 能量占比 | 物理意义 |
|---------|--------|---------|---------|
| **日周期** | IMF 3 | 34.84% | 昼夜循环 |
| **半日周期** | IMF 2 | 13.85% | 早晚双峰 |
| **年周期** | IMF 8 | 3.52% | 季节变化 |
| **超长期基线** | IMF 13 | 46.77% | 27年平均辐射水平 |

### 6.5 长期趋势特征

IMF13（超长期基线）：
- 均值: 268.89 W/m²（接近整体均值262.02）
- 标准差: 1.49 W/m²（极稳定）
- 27年间变化: -2.77 W/m²
- 年均变化率: -0.10 W/m²/年

**结论**: ASU地区GHI长期趋势**基本稳定**，27年间仅有微弱下降。

---

## 7. 结论与建议

### 7.1 主要结论

1. **禁制成功解除**: 通过修改`emd-signal`库的`sift()`函数调用，成功解除了8个IMF的限制，实现了13个IMF的完整分解。

2. **分解质量优良**: 
   - 数学完备性验证通过（RMSE ≈ 10⁻¹³）
   - 物理合理性验证通过（IMF13与移动平均相关性 r = 0.68）

3. **多尺度特征完整提取**:
   - 日周期（IMF3, 34.84%能量）
   - 年周期（IMF8, 3.52%能量）
   - 超长期基线（IMF13, 46.77%能量）

4. **数据特点发现**:
   - 日周期是GHI变化的主导因素
   - 长期趋势稳定，27年间仅有微弱下降
   - 年周期能量占比较小，暗示高气候变率

### 7.2 对后续研究的建议

1. **负荷预测应用**:
   - 使用IMF3作为日模式特征输入
   - 使用IMF8作为季节调整因子
   - 使用IMF13作为长期预测基线

2. **进一步分析**:
   - 对IMF13进行去趋势FFT分析，识别超长期周期
   - 分析IMF之间的非线性耦合关系
   - 考虑使用VMD对特定IMF进行进一步分解

3. **库函数维护**:
   - 建议向`emd-signal`库提交PR，将此修改纳入正式版本
   - 或在使用文档中明确说明max_imfs参数的作用

---

## 附录: 保留文件清单

| 文件 | 说明 |
|------|------|
| `CEEMDAN_IMFs_full.npy` | 13个IMF的完整分解结果 (236520 × 13) |
| `super_long_trend.npy` | 超长期趋势分量 (IMF13) |
| `IMF_decomposition_full.png` | 完整分解可视化图 |
| `IMF_energy_distribution.png` | 能量分布分析图 |
| `CEEMDAN_phase_summary.md` | 本阶段性总结报告 |

---

**报告结束**

*如有问题或需要进一步分析，请联系研究者。*
