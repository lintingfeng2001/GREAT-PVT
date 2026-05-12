# GREAT-PVT RTK XML 配置卡解读与实操指南

本文用于解读 GREAT-PVT 的 RTK 配置文件（以 `doc/GREAT_RTK.xml` 为例），重点回答：

- 每个配置块负责什么；
- 常见参数怎么改；
- 新建一个可运行 RTK 配置时的最小步骤；
- 参数改坏时会出现什么现象。

---

## 1. 一张图看懂 RTK XML 结构

`config` 下主要分为 8 组：

1. `gen`：时间、系统、站点、采样间隔（任务边界）
2. `receiver`：接收机先验信息（尤其是基站坐标）
3. `inputs`：观测、星历、模型文件入口
4. `outputs`：日志和结果输出路径
5. `process`：解算策略（是否用相位、对流层、截止高度角等）
6. `filter`：滤波器与过程噪声
7. `ambiguity`：模糊度固定策略（LAMBDA 门限等）
8. `gps/gal/bds/glo/qzs`：各系统观测噪声、频点与频段选择

---

## 2. 分块逐项解读（按配置顺序）

## 2.1 `gen`：任务时间窗与对象

示例（节选）：

- `beg` / `end`：处理起止时间。
- `sys`：参与解算的 GNSS 系统（如 `GPS GAL BDS`）。
- `rec`：参与处理站名列表。
- `base` / `rover`：RTK 基站与流动站。
- `int`：采样间隔（秒）。

配置建议：

- `beg-end` 必须落在观测文件覆盖时段内，否则会看到“无历元/有效观测不足”。
- `base/rover` 必须是 `rec` 的子集，且站名与 RINEX 头一致（一般 4 字符）。
- `int` 建议与 RINEX 采样一致（1s/5s/30s 等）。

---

## 2.2 `receiver`：基站坐标入口

示例：

- `<rec id="WUDA" X="..." Y="..." Z="..."/>`

含义：

- 当 `process/basepos = CFILE` 时，程序将读取这里的基站坐标作为已知量。

配置建议：

- 坐标需使用 ECEF（米）。
- 基站坐标偏差过大，会直接拖累基线与固定率。

---

## 2.3 `inputs`：输入数据与模型文件

常见键：

- `rinexo`：RINEX 观测文件（可多站并列）。
- `rinexn`：导航电文（广播星历）。
- `atx`：天线相位中心改正文件。
- `blq`：海潮负荷模型。
- `de`：行星历（DE405 等）。
- `eop`：地球自转参数文件。

配置建议：

- 路径尽量使用相对路径并以“配置文件所在目录”为参考维护。
- 先保证 `rinexo/rinexn/atx` 可用，再逐步补 `blq/de/eop`。
- 多系统解算时，导航文件应覆盖对应系统。

---

## 2.4 `outputs`：结果落盘位置

常见键：

- `<log type="BASIC" level="INFO" />`：日志等级。
- `<ppp> ... </ppp>`：中间/轨迹输出前缀。
- `<flt> ... </flt>`：滤波结果文件。

注意：

- 虽节点名是 `ppp`，在 RTK 配置中仍用于输出命名，不影响 RTK 模式本身。
- `$(rec)` 会按站名展开，便于多站批处理。

---

## 2.5 `process`：核心策略开关

关键参数（RTK 常用）：

- `phase=true`：启用载波相位（RTK 基本必开）。
- `tropo/iono`：是否估计对流层/电离层参数。
- `minimum_elev`：截止高度角（度）。
- `obs_combination`：观测组合（示例为 `RAW_MIX`）。
- `pos_kin=true`：动态模式；静态场景可考虑 false（视策略）。
- `min_sat`：最小卫星数。
- `obs_weight`：观测权模型（示例 `SINEL`）。
- `basepos=CFILE`：基站坐标来源于配置文件。
- `frequency=2`：使用双频。

调参经验：

- 城市峡谷可把 `minimum_elev` 适度提高（如 10–15°）抑制低仰角多路径。
- 若固定率低，可先放宽 `max_res_norm`、检查周跳，再看 `ambiguity/ratio`。
- 多系统初期调试建议先 `GPS+BDS`，稳定后再扩系统。

---

## 2.6 `filter`：滤波器与过程噪声

示例属性：

- `method_flt="kalman"`：卡尔曼滤波。
- `noise_crd/noise_vel/noise_dclk`：坐标、速度、钟差过程噪声。
- `rndwk_ztd`：对流层随机游走。
- `reset_amb`：模糊度重置间隔控制。

调参经验：

- 动态场景噪声可略大，静态场景噪声可更小。
- 噪声过小会导致滤波“僵硬”，残差异常后不易收敛。
- 噪声过大则会导致解抖动、固定保持性变差。

---

## 2.7 `ambiguity`：模糊度固定策略

关键参数：

- `fix_mode`：`NO` 或 `SEARCH`。
- `part_fix` / `part_fix_num`：部分固定策略及门限。
- `ratio`：LAMBDA 比率检验阈值。
- `*_decision`：宽巷/窄巷判定参数（`maxdev/maxsig/alpha`）。

调参经验：

- `ratio` 提高：误固定风险降低，但固定成功率可能下降。
- `ratio` 降低：更易固定，但要关注误固定。
- 建议先保持默认，再结合残差和轨迹连续性微调。

---

## 2.8 `gps/gal/bds/glo/qzs`：系统级观测模型

每个系统块通常包含：

- `sigma_C`：伪距标准差；`sigma_L`：相位标准差。
- `freq`：参与滤波的频点编号。
- `band`：对应信号频段映射。

建议：

- 新数据源先使用官方样例的 `sigma_C/sigma_L` 量级。
- 若某系统观测质量差，可单独调大其 sigma（相当于降权）。

---

## 3. 从 0 到 1 配一个可跑 RTK XML（最小清单）

1. 复制 `doc/GREAT_RTK.xml` 作为模板。  
2. 改 `gen`：时间窗、`base/rover`、`sys`。  
3. 改 `receiver`：填基站 ECEF 坐标。  
4. 改 `inputs`：替换为你的 `rinexo/rinexn/atx` 路径。  
5. 改 `outputs`：输出到 `results/你的任务名/`。  
6. `process` 先保持默认，只调 `minimum_elev` 和 `frequency`。  
7. 运行后先看日志，再看 `.flt` 与轨迹图。  

---

## 4. 常见故障与定位

- **现象：程序直接结束，几乎无解算结果**  
  优先检查：`beg/end` 是否超出观测时段、`rinexo` 路径是否正确。

- **现象：浮点解有但固定率很低**  
  优先检查：基站坐标精度、周跳情况、`minimum_elev` 是否过低、`ratio` 是否过高。

- **现象：结果抖动大**  
  优先检查：`noise_crd/noise_vel` 是否过大、低仰角多路径、系统混合权重是否失衡。

- **现象：某系统几乎不起作用**  
  优先检查：该系统导航电文是否覆盖、`sys` 是否包含、该系统 `freq/band` 是否匹配观测类型。

---

## 5. 推荐学习路径（结合源码）

1. 先看 XML：明确“我让程序做什么”。
2. 再看 `src/app/GREAT_PVT/GREAT_PVT.cpp`：确认“程序如何读配置并调度流程”。
3. 最后看 `src/LibGREAT/gproc/gpvtflt.cpp` 与 `src/LibGREAT/gset/gsetamb.cpp`：聚焦滤波与模糊度固定实现。

这样能把“配置语义 -> 算法行为 -> 结果表现”串起来，学习效率最高。
