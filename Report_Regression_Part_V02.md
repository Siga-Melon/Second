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
**结果分析**：
Cox模型的输出结果为我们提供了丰富的统计信息，每个参数都承载着重要的临床含义。下面从多个维度对结果进行深入解读。
1. **模型系数的病理生理学解读**
	从模型系数的符号和大小可以看出心力衰竭预后因素的内在逻辑：
	- **负系数变量的保护机制**：
		- **射血分数（coef = -0.048）**：负系数反映了心脏收缩功能与生存的正相关关系。每1%的射血分数提升对应4.8%的风险对数下降，这在生物学上意味着心脏泵血能力越强，循环系统的代偿能力越好，患者的生存机会自然更大。
		- **血清钠（coef = -0.042）**：虽然效应相对较小，但负系数提示了电解质平衡的重要性。血钠水平往往反映了神经内分泌系统的激活状态，较高的血钠可能意味着更好的体液调节能力。
	- **正系数变量的风险机制**：
		- **血清肌酐（coef = 0.313）**：系数最大，反映了心肾轴在心力衰竭进展中的核心作用。肾功能恶化不仅是心力衰竭的并发症，更是预后恶化的强预测因子，这种双向恶化循环在临床上极为常见。
		- **高血压（coef = 0.447）**：作为分类变量，其系数直接反映了高血压患者相对于正常血压患者的额外风险负担。
2. **统计显著性的临床价值判断**
	- **强显著性变量的临床优先级**：
		年龄、射血分数和血清肌酐的极小P值（均<0.001）不仅表明统计学意义，更重要的是反映了这些因素在心力衰竭病理进程中的核心地位。这种统计学的强显著性为临床决策提供了坚实的证据基础——在资源有限的情况下，优先关注这些指标无疑是最具成本效益的策略。
	- **边缘显著性的临床思考**：
		贫血（P = 0.067）和血清钠（P = 0.074）虽未达到传统的0.05显著性水平，但其P值仍提示了潜在的临床意义。在心力衰竭这样复杂的疾病中，多种因素相互作用，单纯依赖P值可能会遗漏重要的临床信息。这种"接近显著"的结果提醒我们需要结合更大样本或亚组分析来进一步验证。
3. **效应量的临床实际意义
	- **大效应量变量的治疗价值**：
		血清肌酐的HR值1.368意味着每1 mg/dL的升高对应36.8%的风险增加，这种大幅度的效应提示肾功能保护在心力衰竭管理中的关键地位。临床上，即使是轻微的肌酐升高也应引起足够重视，早期的肾功能保护措施可能带来显著的预后改善。
	- **小效应量变量的累积影响**：
		年龄的单年HR值仅为1.047，看似影响微弱，但考虑到年龄的不可逆性和累积效应，10年的年龄差异将带来58.4%的风险增加（1.047^10 = 1.584）。这种累积效应的概念在老年心力衰竭患者的管理中尤为重要。
4. **临床决策的量化支持**
	- **精准化风险分层**：
		通过组合多个变量的信息，我们能够实现比单一指标更精准的风险分层。例如，一个年轻但射血分数极低的患者与一个年老但心功能相对保留的患者可能面临不同类型但程度相当的风险。
	- **治疗策略的数据依据**：
		模型结果清晰地显示了各危险因素的相对重要性，为治疗资源的分配提供了科学依据。在临床实践中，优先控制血压、保护肾功能、改善心功能的策略选择获得了强有力的统计学支持，使临床医生能够根据患者的具体情况快速评估死亡风险并制定相应的治疗强度。

### 3.4 可视化各变量的 Hazard Ratios

为了更直观地展示各临床变量对心力衰竭患者死亡风险的影响程度，我们从Cox模型结果中提取了风险比（HR）值及其95%置信区间，并绘制了森林图进行可视化分析。在图表设计上，用红色表示风险因素（HR>1），蓝色表示保护因素（HR<1），通过误差线和横线展示置信区间，反转 y 轴使高风险变量排列在上方。
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
**结果分析**：
森林图的分析结果呈现出明显的层次化特征，各变量对死亡风险的影响可以清晰地分为三个梯队。
1. **显著危险因素**
	位于图表上方的三个红色条带代表了最重要的危险因素：
	- **高血压（HR=1.564）**：表现最为突出，其置信区间（1.026-2.383）完全位于HR=1的参考线右侧，说明这一发现具有统计学意义且临床相关性强。这一结果与心力衰竭的病理生理机制高度吻合——持续的血压升高会加重心脏负荷，加速心功能恶化进程。
    - **贫血（HR=1.472）**：紧随其后，虽然其置信区间下限接近1.0，但仍显示出明显的风险增加趋势。从临床角度分析，贫血会降低血氧运输能力，迫使已经受损的心脏做更多功来维持组织灌注，这种恶性循环往往导致预后恶化。
    - **血清肌酐（HR=1.368）**：作为肾功能的重要指标，其升高反映了心肾综合征的存在，这在心力衰竭患者中极为常见且预后不良。
2. **中等程度影响因素**
	年龄、糖尿病和吸烟状况的HR值相对较小，但仍需要临床关注：
	- **年龄因素（HR=1.047）**：看似影响微弱，但考虑到年龄的累积效应，每增加10年带来的风险增幅实际上是相当可观的。
	- **糖尿病（HR=1.133）和吸烟（HR=1.128）**：置信区间跨越了HR=1的参考线，提示在当前样本中这些传统危险因素的独立作用可能被其他更强的预测因子所掩盖，或者存在与其他变量的交互作用。
3. **保护性因素**
	图表下方的蓝色区域展现了具有保护作用的因素：
	- **射血分数（HR=0.954）**：保护效应最为明确，其置信区间（0.934-0.973）完全位于HR=1左侧，这与心力衰竭的核心病理——心脏泵血功能受损——直接相关。每1%的射血分数改善都能带来约4.6%的死亡风险降低，这为临床上通过药物或器械治疗改善心功能提供了强有力的证据支持。
	- **血清钠（HR=0.959）**：虽然保护效应相对较弱，但其在维持体液平衡和细胞功能方面的重要性不容忽视。
	- **性别差异（女性vs男性，HR=0.837）**：显示女性患者可能具有一定的生存优势，这可能与激素保护、疾病严重程度差异或治疗依从性等多种因素相关。

**临床意义与决策支持**
这一可视化结果为临床决策提供了清晰的优先级指导：
1. **重点干预目标**：对于新诊断的心力衰竭患者，积极控制血压、监测和纠正贫血状态、维护肾功能应当成为治疗的重点
2. **功能改善策略**：通过药物治疗、心脏康复等手段改善射血分数，保持适当的血钠水平，都有助于改善患者预后
3. **资源配置优化**：这种基于数据的风险分层不仅有助于个体化治疗方案的制定，也为医疗资源的合理配置提供了科学依据
### 3.5 量化多单位变化的风险比

由于单单位变量变化的 HR 值临床参考价值有限，因此，我们需要定义临床常用的单位递进步长（如年龄每 10 年、射血分数每 5%）来计算多单位变化的风险比，使结果更贴近临床实践。通过定义各变量的临床相关单位变化及描述，利用模型系数计算多单位变化的 HR 值，并按 HR 值降序排序，突出高风险因素的影响。

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
**结果分析**
1. **年龄每增加10年（HR=1.584）**：年龄的累积效应使其成为最强的风险因子，58.4%的风险增幅反映了衰老过程中心血管系统不可逆的功能衰退。
2. **高血压存在vs不存在（HR=1.564）**：作为可控的最强风险因子，56.4%的风险增加强调了血压管理在心力衰竭患者中的核心地位和紧迫性。
3. **贫血存在vs不存在（HR=1.472）**：47.2%的风险增加揭示了血氧携带能力下降对已受损心脏的额外负担，提示贫血纠正应纳入常规治疗考量。
4. **血清肌酐每升高1 mg/dL（HR=1.368）**：36.8%的风险增加体现了心肾轴恶化的严重后果，即使轻微的肾功能下降也需积极干预。
5. **糖尿病存在vs不存在（HR=1.133）**：仅13.3%的风险增加显示糖尿病在本队列中的独立效应有限，可能与现代糖尿病管理水平提升相关。
6. **吸烟vs不吸烟（HR=1.128）**：12.8%的风险增加低于预期，提示吸烟的影响可能被其他更强的因素所掩盖，或反映戒烟干预的效果。
7. **女性vs男性（HR=0.837）**：女性死亡风险降低16.3%，可能与激素保护、疾病严重程度差异或治疗依从性差异相关。
8. **血清钠每增加5 mmol/L（HR=0.809）**：风险降低19.1%强调了电解质平衡的重要性，适当的血钠水平反映了更好的神经内分泌调节状态。
9. **射血分数每提高5%（HR=0.788）**：风险降低21.2%为心功能改善治疗提供了强有力的数据支撑，是所有保护因素中效应最大的。

- **临床干预优先级的数据支撑**
	- **高优先级干预目标**：
		排名前四的因素中，除年龄外，高血压、贫血和肾功能均为可干预因素。高血压和贫血的风险比相近（1.564 vs 1.472），但高血压的干预手段更成熟、效果更确切，应作为首要干预目标。血清肌酐的显著影响提醒我们，即使在心力衰竭的治疗中也要时刻关注肾功能保护。
	- **中等优先级的策略调整**：
		糖尿病和吸烟的相对较低风险比并不意味着可以忽视，而是提示需要个体化评估。在糖尿病控制良好的患者中，可能需要将注意力更多转向其他危险因素；对于吸烟患者，戒烟仍然是重要的长期获益策略。
- **保护因素的治疗价值量化**
	射血分数改善的巨大保护效应（HR=0.788）为心功能优化治疗提供了明确的目标导向。每5%的射血分数提升带来超过20%的死亡风险下降，这种效应远超大多数单一药物干预。血清钠水平的保护作用虽然相对较小，但提醒我们在利尿剂使用和液体管理中需要精细化调节。
- **性别差异的临床意义**
	女性相对男性16.3%的生存优势值得深入思考。这种差异可能反映了生物学性别在心血管疾病中的保护作用，也可能与女性患者在疾病管理、药物依从性和医疗寻求行为方面的差异相关。这一发现提示我们在制定治疗方案时可能需要考虑性别特异性的策略。
- **风险分层的精细化应用**
	这种多单位风险比的量化方法使我们能够构建更精确的风险评估工具。例如，一个70岁合并高血压和轻度肾功能不全（肌酐1.5 mg/dL）的男性患者，其累积风险将是多个独立因素效应的乘积，这种量化评估为个体化治疗强度的确定提供了科学依据。再例如，将高血压患者的血压控制在正常范围可能比治疗贫血更能显著降低死亡风险，为临床资源分配提供数据支持。

**结论**：多单位风险比量化了临床干预的潜在效益，

### 3.6 局部效应可视化 

为了更深入地理解关键变量对患者生存概率的具体影响模式，我们采用局部效应分析方法，选择年龄这一核心危险因素进行可视化展示。局部效应图能够直观地呈现在其他变量保持不变的情况下，单一变量的不同取值如何影响患者的生存曲线走势。
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
**结果分析**：
局部效应图显示，年龄 50 岁、70 岁、90 岁的患者生存曲线随时间分化明显：高龄患者生存概率下降更快，90 岁患者的生存曲线显著低于 50 岁患者，与量化多单位风险比中 “年龄每 10 年 HR=1.584” 的结果一致，进一步验证了年龄对生存的显著影响。

### 3.7 Kaplan-Meier 曲线分组对比

为了验证Cox模型的风险分层能力，我们基于模型计算的风险分数将患者分为高低风险两组，并绘制Kaplan-Meier生存曲线来直观展示两组间的生存差异。使用这种分析方法能够从时间维度验证模型预测的准确性，同时为临床风险分层提供可视化的证据支持。

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
- **风险分层效果验证**
	KM 曲线显示，高风险组的生存概率显著低于低风险组。随访 200 天时，高风险组生存概率约为 0.3，而低风险组约为 0.6，差异幅度达 30%，验证了 Cox 模型的风险分层有效性。这表明基于临床指标构建的风险模型可有效预测患者预后，为临床决策（如强化监测或干预）提供依据。
- **时间依赖性特征**
	曲线分离模式反映了心力衰竭的进展性特征：早期两组差异相对较小，中期（100-200天）高风险组生存概率急剧下降，体现了危险因素累积效应的时间依赖性。这种模式为制定差异化的监测和干预策略提供了时间参考。
## 4. Results of Regression Part(待完成)


## 5. Discussion(待投喂组员的研究部分后进行修改)

