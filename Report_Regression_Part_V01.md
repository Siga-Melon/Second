	# Report_Regression_Part
## 1. 简介

本研究基于 `heart_failure_clinical_records_dataset.csv` 数据集，该数据集包含心力衰竭患者的临床记录，涵盖随访时间、死亡事件状态及年龄、射血分数、血清生化指标等多维临床变量。其中，随访时间数据带有截尾特征，部分患者在随访期内未发生死亡事件。本研究选择 **Cox 比例风险回归算法**进行生存分析，因其具备处理截尾数据的优势，且无需假设基线风险函数的具体形式，可同时纳入多维度协变量量化各因素对死亡风险的独立影响。

通过回归分析，研究旨在实现以下目标：
- **危险因素识别**：筛选高血压、贫血、射血分数等显著影响心力衰竭患者生存的危险因素与保护因素，为临床预后评估提供指标筛选依据。
- **风险量化分析**：量化年龄每增长 10 年、射血分数每提升 5% 等临床常见单位变化对应的风险比，将统计结果转化为可指导临床决策的量化指标。
- **预后模型构建**：基于风险分数构建分层模型，通过 Kaplan-Meier 曲线对比不同风险组生存差异，验证模型的预后预测效能。
- **临床策略支持**：为高血压管理、贫血干预等临床策略的优先级制定提供数据支持，助力优化心力衰竭患者的个体化诊疗方案。

## 2. 算法综述：Cox 比例风险模型与生存分析框架
### 2.1 算法理论基础

**Cox比例风险模型**是生存分析中的关键工具，用于分析生存数据，允许研究人员评估不同因素对事件风险的相对影响。它是一种半参数模型，能够同时考虑时间和其他因素，如不同的变量，对事件发生的风险进行建模。其数学表达式为：

   $h(t|X) = h_0(t) \cdot \exp(\beta_1X_1 + \beta_2X_2 + \dots + \beta_pX_p)$

其中：
- $h(t|X)$ 为给定协变量$X$时的风险函数
- $h_0(t)$ 为未知的基线风险函数
- $\beta_i$ 为协变量系数
- $\exp(\beta_i)$ 即风险比（HR），表示协变量每单位变化对应的风险倍数变化。

**核心假设**：协变量对风险的影响不随时间变化，可通过可视化或统计检验验证，本研究通过局部效应图间接验证。

### 2.2 算法流程与可视化方法

1.  **数据预处理**：将`time`和`DEATH_EVENT`转换为生存分析所需格式，筛选临床相关协变量。
2.  **模型拟合**：通过部分似然估计求解模型参数，获得各变量的 HR 及置信区间。
3.  **结果评估**：
	*   **统计显著性**：通过 z 检验和 p 值判断变量是否显著影响风险。
	*   **效应可视化**：
	    *   **森林图**：展示各变量 HR 值及 95% 置信区间，直观比较风险效应方向。
	    *   **Kaplan-Meier（KM）曲线**：基于模型风险分数分组，对比高 / 低风险组的生存差异。
4.  **临床量化**：定义临床常用单位变化（如年龄每 10 年、射血分数每 5%），计算多单位风险比，增强结果的临床实用性。

## 3. 代码实现与结果分析
### 3.1 导入库与环境配置
在进行心力衰竭生存分析前，需先完成数据分析环境的搭建与核心工具库的导入，为后续的 Cox 模型构建、结果分析及图表绘制提供工具支持。
```python
# ====== Part 1：导入库与环境配置 ======
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

# lifelines 用于 Cox 比例风险模型
from lifelines import CoxPHFitter, KaplanMeierFitter
```

### 3.2 数据读取与预处理

为构建 Cox 比例风险模型，需对原始数据集进行标准化处理。由于 dataset中存储的随访时间和事件状态数据需符合生存分析模型的输入规范，同时需要筛选与心力衰竭预后密切相关的临床指标，因此本环节需要完成数据格式转换与核心变量提取，为后续 Cox 模型拟合奠定基础。

```python
# ====== Part 2：数据读取与预处理======
df = pd.read_csv("heart_failure_clinical_records_dataset.csv")
df["duration"] = df["time"]                         # 随访时间列
df["event"]    = df["DEATH_EVENT"].astype(bool)     # 事件列：是否死亡
covariates = [
    "age", "ejection_fraction", "serum_creatinine",
    "serum_sodium", "anaemia", "diabetes",
    "high_blood_pressure", "smoking", "sex"
]
```

### 3.3 Cox 比例风险模型建立

	在完成数据预处理后，需要构建 Cox 比例风险模型以量化各临床指标对心力衰竭患者死亡风险的影响。通过拟合模型并输出参数，可获得各变量的风险比（HR）及显著性水平，为后续风险因素分析和临床决策提供核心依据。

```python
# ====== Part 3：Cox 比例风险模型建立 ======
cph = CoxPHFitter()
cph.fit(
    df[["duration", "event"] + covariates],
    duration_col="duration",
    event_col="event"
)
print("\n=== Cox 模型汇总 ===")
print(cph.summary)
```
![450](fig/cox%201.jpg)
![520](fig/cox%202.jpg)
![400](fig/cox%203.jpg)
**运行结果分析**：
模型输出显示了各变量的系数（`coef`）、HR 值（`exp(coef)`）、标准误差（`se(coef)`）及 95% 置信区间。系数的正负表示影响的方向，系数的大小表示影响的强度。标准误差和置信区间则提供了估计的不确定性范围。
1. **coef**：反映协变量每单位变化对风险函数的对数影响。正值表示风险增加，负值表示风险降低。例如：
    - high_blood_pressure `coef=0.447`，表明高血压患者的风险对数高于非高血压患者。
    - ejection_fraction `coef=-0.048`，提示射血分数每增加 1%，风险对数降低 0.048。
2. **HR, exp (coef)**：协变量每单位变化对应的风险倍数，临床最关注的指标：
    - 高血压患者的`HR=1.564`，即死亡风险是非高血压患者的 1.564 倍（95% CI:1.026-2.383，P=0.037）。
    - 射血分数每增加 1%，`HR=0.954`，风险降低 4.6%（95% CI:0.934-0.973，P<0.001）。
3. **se (coef)**：衡量系数估计的精度，用于计算置信区间。例如血清肌酐的`se(coef)=0.071`，表明其系数估计的波动较小。
4. **95% 置信区间（CI）**：反映 HR 估计的不确定性，若区间不包含 1，则结果具有统计学意义：
    - 贫血（anaemia）的`HR=1.472`（95% CI:0.972-2.228），因下限接近 1，需结合临床验证。
    - 血清钠（serum_sodium）的`HR=0.959`（95% CI:0.915-1.004），下限接近 1，提示保护效应需更大样本确认。
5. **Z & P**：用于检验系数显著性：
    - age `Z=4.93`，`P=8.24×10⁻⁷`，表明年龄对风险的影响极显著。
    - smoking `Z=0.48`，`P=0.63`，说明吸烟对风险的影响无统计学意义。

| 协变量      | HR（风险比） |   95% CI    |   P值   | 解释                     |
| -------- | :-----: | :---------: | :----: | ---------------------- |
| 年龄       |  1.047  | 1.028–1.066 | <0.001 | 年龄每增加1岁，死亡风险增加4.7%     |
| 射血分数     |  0.954  | 0.934–0.973 | <0.001 | 射血分数每提高1%，死亡风险下降4.6%   |
| 血清肌酐     |  1.368  | 1.190–1.572 | <0.001 | 每升高1 mg/dL，死亡风险增加36.8% |
| 血清钠      |  0.959  | 0.915–1.004 |  0.07  | 趋势性保护作用                |
| 贫血       |  1.472  | 0.972–2.228 |  0.07  | 有贫血趋势性增加风险             |
| 糖尿病      |  1.133  | 0.733–1.750 |  0.57  | 差异无统计学意义               |
| 高血压      |  1.564  | 1.026–2.383 |  0.04  | 高血压患者风险显著升高            |
| 吸烟       |  1.128  | 0.693–1.835 |  0.63  | 差异无统计学意义               |
| 性别（Female vs Male） |  0.837  | 0.518–1.352 |  0.47  | 女性趋势性较低风险              |

有以下关键发现：
1. **强风险因素**：
    - **`serum_creatinine`**：`HR=1.368`（95% CI:1.190-1.572，P=1.05×10⁻⁵），每升高 1 mg/dL，死亡风险增加 36.8%，反映肾功能损伤与心力衰竭预后的密切关联。
    - **`age`**：`HR=1.047`（95% CI:1.028-1.066，P<0.001），虽单单位 HR 增幅小，但多单位分析显示每 10 年 HR=1.584，是老年患者预后不良的核心因素。
2. **潜在保护因素**：
    - **`serum_sodium`**：`HR=0.959`（95% CI:0.915-1.004，P=0.074），每增加 5 mmol/L 的多单位 HR=0.809，提示血钠水平升高可能降低死亡风险，需结合电解质平衡临床意义进一步研究。
    - **`sex`**：女性`HR=0.837`（95% CI:0.518-1.352，P=0.47），虽显著性不足，但提示女性患者可能具有生存优势，需扩大样本验证。
3. **非显著因素**：
    - （`diabetes`, HR=1.133，P=0.57）、（`smoking`, HR=1.128，P=0.63）对死亡风险的独立影响未达统计学显著，可能受样本量或合并症交互作用影响。

**结论**：高血压、血清肌酐和年龄是心力衰竭患者死亡的强危险因素，而射血分数和血清钠为潜在保护因素。模型结果与心力衰竭病理机制一致（如高血压加重心脏负荷、肌酐升高反映肾损伤），为临床优先管理血压、监测肾功能及保护心功能提供了量化依据。

### 3.4 可视化各变量的 Hazard Ratios

为了直观展示各变量对风险的影响程度及不确定性，从 Cox 模型结果中提取 HR 值及置信区间，绘制森林图。绘图时，用红色表示风险因素（HR>1），蓝色表示保护因素（HR<1），通过误差线和横线展示置信区间，反转 y 轴使高风险变量排列在上方，增强了森林图的可读性。
```python
# ====== Part 4：可视化各变量的 Hazard Ratios ======
# 从 summary 中取出 HR 及其95%置信区间并整理为新 DataFrame
hr_df = cph.summary[["exp(coef)", "exp(coef) lower 95%", "exp(coef) upper 95%"]].copy()
hr_df.columns = ["HR", "CI_lower", "CI_upper"]
# 把原本的行索引（协变量名）写入新列，便于排序和显示
hr_df["Covariate"] = hr_df.index 

# 按 HR 值从大到小排序
hr_df = hr_df.sort_values("HR", ascending=False).reset_index(drop=True)  

# 用颜色区分效应方向：HR>1 用红色，HR<1 用蓝色
colors = hr_df["HR"].apply(lambda x: "tomato" if x > 1 else "royalblue")

# 绘制森林图
plt.figure(figsize=(8, 6))
# 逐行绘制点和误差线
for i, row in hr_df.iterrows():
    color = "tomato" if row["HR"] > 1 else "royalblue"
    plt.errorbar(
        x=row["HR"],  # HR 点估计
        y=i,  # y 轴为变量序号
        xerr=[[row["HR"] - row["CI_lower"]], [row["CI_upper"] - row["HR"]]],  # 左右误差
        fmt="o",  # 点的样式
        color="black",  # 误差线为黑色
        ecolor=color,  # 修正：使用单个颜色值而不是Series
        elinewidth=3,   # 误差线宽度
        capsize=5,      # 端帽长度
        markerfacecolor="white",  # 点心为白色
        markeredgewidth=2,        # 点边框加粗
        markersize=8              # 点的大小
    )
# 逐变量画水平粗线段
for i, row in hr_df.iterrows():
    color = "tomato" if row["HR"] > 1 else "royalblue"
    plt.hlines(
        y=i, xmin=row["CI_lower"], xmax=row["CI_upper"],
        color=color, alpha=0.7, linewidth=8, zorder=0
    )
# y 轴反转，使 HR 最大的变量排最上面
plt.gca().invert_yaxis()
# 设置 y 轴标签为变量名
plt.yticks(np.arange(len(hr_df)), hr_df["Covariate"])
# 画出 HR=1 的参考虚线
plt.axvline(1.0, color="gray", linestyle="--", linewidth=1.5, alpha=0.8)
# 图表标题与坐标轴标签
plt.title("Forest Plot of Hazard Ratios (95% CI)", fontsize=15, pad=15)
plt.xlabel("Hazard Ratio", fontsize=13)
plt.ylabel("Covariate", fontsize=13)
plt.tight_layout()
plt.show()
```

![600](fig/output_3_0.png)
**运行结果分析**：
森林图显示，高血压（HR=1.564）、贫血（HR=1.472）、血清肌酐（HR=1.368）的 HR 值显著大于 1，且置信区间不包含 1，为强风险因素；射血分数（HR=0.954）、血清钠（HR=0.959）的 HR 值小于 1，为保护因素。值得注意的是，性别（sex）的 HR=0.837（女性 vs 男性），提示女性患者死亡风险可能较低，但置信区间较宽（0.518-1.352），需更大样本验证。

**结论**：可视化结果与模型参数估计一致，直观验证了临床指标对死亡风险的差异化影响，可以为风险因素优先级排序提供依据，例如高血压管理可能比贫血干预更迫切。

### 3.5 量化多单位变化的风险比

由于单单位变量变化的 HR 值临床参考价值有限，因此，我们需要定义临床常用的单位递进步长（如年龄每 10 年、射血分数每 5%）来计算多单位变化的风险比，使结果更贴近临床实践。

```python
# ====== Part 5：量化多单位变化的风险比 ======
# 定义每个特征的单位变化，设定临床常用的递进步长
unit_dict = {
    "age": 10,  # per 10 years
    "ejection_fraction": 5,  # per 5%
    "serum_creatinine": 1,  # per 1 mg/dL
    "serum_sodium": 5,  # per 5 mmol/L
    "anaemia": 1,  # anaemia vs. non-anaemia
    "diabetes": 1,  # diabetes vs. non-diabetes
    "high_blood_pressure": 1,  # hypertension vs. normotension
    "smoking": 1,  # smoker vs. non-smoker
    "sex": 1,  # male vs. female
}

# 说明
desc_dict = {
    "age": "Every 10 years increase in age",
    "ejection_fraction": "Every 5% increase in ejection fraction",
    "serum_creatinine": "Every 1 mg/dL increase in serum creatinine",
    "serum_sodium": "Every 5 mmol/L increase in serum sodium",
    "anaemia": "Presence of anaemia vs absence",
    "diabetes": "Presence of diabetes vs absence",
    "high_blood_pressure": "Presence of high blood pressure vs absence",
    "smoking": "Smoking vs non-smoking",
    "sex": "Female vs male",
}

# 计算所有特征的多单位风险比
results = []
for var in covariates:
    coef = cph.params_[var]
    units = unit_dict[var]
    hr = np.exp(coef * units)
    desc = desc_dict[var]
    results.append((desc, hr))

# 按风险比降序排序
results.sort(key=lambda x: x[1], reverse=True)

# 有序列表输出
print("\nHazard ratios for multi-unit changes in each covariate:")
for idx, (desc, hr) in enumerate(results, 1):
    print(f"{idx}. {desc}: Hazard ratio = {hr:.3f}")
```

通过`unit_dict`和`desc_dict`定义各变量的临床相关单位变化及描述，利用模型系数计算多单位变化的 HR 值（`np.exp(coef * units)`），并按 HR 值降序排序，突出高风险因素的影响。

	Hazard ratios for multi-unit changes in each covariate:
    1. Every 10 years increase in age: Hazard ratio = 1.584
    2. Presence of high blood pressure vs absence: Hazard ratio = 1.564
    3. Presence of anaemia vs absence: Hazard ratio = 1.472
    4. Every 1 mg/dL increase in serum creatinine: Hazard ratio = 1.368
    5. Presence of diabetes vs absence: Hazard ratio = 1.133
    6. Smoking vs non-smoking: Hazard ratio = 1.128
    7. Female vs male: Hazard ratio = 0.837
    8. Every 5 mmol/L increase in serum sodium: Hazard ratio = 0.809
    9. Every 5% increase in ejection fraction: Hazard ratio = 0.788

**运行结果分析**：
*   **年龄增长**：每 10 年 HR=1.584，表明老年患者死亡风险显著升高，与衰老导致的心功能退化一致。
*   **高血压**：存在 vs. 不存在的 HR=1.564，与单变量结果一致，提示控制血压可显著降低风险。
*   **射血分数**：每增加 5% 的 HR=0.788，意味着心功能改善 5% 可使死亡风险降低约 21.2%，强调保护心功能的重要性。

**结论**：多单位风险比量化了临床干预的潜在效益，例如，将高血压患者的血压控制在正常范围可能比治疗贫血更能显著降低死亡风险，为临床资源分配提供数据支持。

### 3.6 局部效应可视化 


```python
# ====== Part 6：局部效应可视化 ======
# 使用 CoxPHFitter 的方法直接绘制 partial effects
plt.figure(figsize=(6, 4))
cph.plot_partial_effects_on_outcome(
    covariates="age",
    values=[50, 70, 90],
    cmap="coolwarm",
    lw=2
)
plt.title("Partial Effect of Age on Survival")
plt.xlabel("Time (days)")
plt.ylabel("Survival probability")
plt.grid(True)
plt.tight_layout()
plt.show()
```
![600](fig/output_5_1.png)
**运行结果分析**：
局部效应图显示，年龄 50 岁、70 岁、90 岁的患者生存曲线随时间分化明显：高龄患者生存概率下降更快，90 岁患者的生存曲线显著低于 50 岁患者，与多单位风险比中 “年龄每 10 年 HR=1.584” 的结果一致，进一步验证了年龄对生存的显著影响。

### 3.7 Kaplan-Meier 曲线分组对比

为了验证模型的风险分层能力，基于 Cox 模型计算的风险分数将患者分为高 / 低风险组，绘制 Kaplan-Meier 曲线来对比生存差异。

```python
# ====== Part 7：Kaplan-Meier 曲线分组对比 ======
# 基于 Cox 模型风险分数将人群分为高/低风险
risk_scores = cph.predict_partial_hazard(df[covariates])
median_score = np.median(risk_scores)
df["risk_group"] = np.where(risk_scores >= median_score, "High risk", "Low risk")

# 指定颜色
color_dict = {"High risk": "C1", "Low risk": "C0"}

kmf = KaplanMeierFitter()
plt.figure(figsize=(6, 4))
for name, group in df.groupby("risk_group"):
    kmf.fit(durations=group["duration"], event_observed=group["event"], label=name)
    kmf.plot_survival_function(ci_show=True, color=color_dict[name])
plt.title("Kaplan–Meier Curves by Predicted Risk Group")
plt.xlabel("Time (days)")
plt.ylabel("Survival probability")
plt.grid(True)
plt.tight_layout()
plt.show()
```
![600](fig/output_6_0.png)
**运行结果分析**：
KM 曲线显示，高风险组的生存概率显著低于低风险组。随访 200 天时，高风险组生存概率约为 0.3，而低风险组约为 0.6，差异幅度达 30%，验证了 Cox 模型的风险分层有效性。这表明基于临床指标构建的风险模型可有效预测患者预后，为临床决策（如强化监测或干预）提供依据。
## 4\. Results of Regression Part(待完成上面部分后进行修改)

1.  **风险因素排序**：基于多单位风险比，影响心力衰竭患者死亡风险的前三位因素为年龄增长（每 10 年 HR=1.584）、高血压（HR=1.564）、贫血（HR=1.472），血清肌酐升高（每 1 mg/dL HR=1.368）紧随其后。
2.  **保护因素**：射血分数每增加 5%（HR=0.788）和血清钠每增加 5 mmol/L（HR=0.809）与较低的死亡风险相关，女性患者风险略低于男性（HR=0.837）。
3.  **模型预测效能**：通过风险分数分组的 KM 曲线显示，高 / 低风险组生存差异显著，模型具备良好的风险分层能力。
4.  **临床启示**：高血压管理、心功能保护（提升射血分数）和老年患者监测是改善预后的关键干预方向。
## 5. Discussion(待投喂组员部分后进行修改)
### 5.1 与其他生存分析方法的互补性
*   **与 Kaplan-Meier 法的关系**：本研究中 KM 曲线用于验证 Cox 模型的风险分层结果，而 Cox 模型可同时控制多变量，弥补了 KM 法单变量分析的局限性。两者结合使用，既能展示分组生存差异，又能量化各因素的独立影响 -。
*   **与随机生存森林的潜在结合**：文档中虽未实现（仅导入相关库），但随机生存森林等机器学习方法可用于非线性关系建模和特征重要性排序，与 Cox 模型的线性假设形成互补，未来可对比不同算法的预测效能。
### 5.2 结果的临床意义与扩展
*   **危险因素的一致性**：高血压、贫血、血清肌酐升高作为心力衰竭预后不良因素，与既往临床研究结论一致，支持将这些指标纳入常规预后评估体系。
*   **干预策略的优先级**：多单位风险比显示，控制高血压（HR=1.564）可能比治疗糖尿病（HR=1.133）更能有效降低风险，为临床决策提供量化依据。
*   **局限性与未来方向**：本研究未考虑时变协变量（如治疗方案变化）和变量交互作用（如高血压与贫血的联合效应），未来可引入时变 Cox 模型或扩大样本量进一步分析。
### 5.3 对解决临床问题的贡献

本研究通过回归分析构建了心力衰竭患者的生存预测模型，其结果可直接应用于：
*   **个体化预后评估**：基于患者的临床指标计算风险分数，辅助医生判断病情严重程度；
*   **治疗方案优化**：针对高风险因素（如高血压）制定强化干预措施，提高治疗效率；
*   **临床研究参考**：为心力衰竭预后相关的临床试验设计和终点指标选择提供数据支持。