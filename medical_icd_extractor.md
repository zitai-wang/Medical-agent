---
name: medical_icd_extractor
description: 本技能用于从**医学图片**（诊断报告、病历截图、检查单等）中自动完成医学实体与 ICD 编码的互转
---

# Medical ICD Extractor — 图片到ICD医学实体映射

## 概述

本技能用于从**纯文本医学图片**（诊断报告、病历截图、检查单等）中自动完成以下任务：

1. **OCR 文字提取** — 从图片中提取中文医学文本
2. **医学实体抽取** — 识别所有类别的医学实体标签（疾病名、解剖部位、症状、手术操作、病因、形态学等共 12 类）
3. **ICD 编码映射** — 将**每一类医学实体**反查为对应的 ICD 编码，输出 `医学实体 → ICD编码` 的完整映射

> 图片中通常**不含预印的 ICD 编码**，所有编码通过实体名称反查获得。

---

## 触发条件

当用户提供医学相关图片，并要求提取 ICD 编码或医学实体时触发。
关键词：`ICD`、`诊断编码`、`疾病编码`、`医学实体`、`病历提取`、`OCR`。

---

## 工作流程

### 阶段 1：图片预处理与 OCR

**目标**：从输入图片中提取完整的原始中文医学文本。

1. **接收图片输入** — 支持 PNG / JPG / PDF（含扫描件）格式
2. **OCR 文字识别** — 使用 OCR 工具提取图片中的所有文字
3. **文本清洗** — 去除水印、页码、医院Logo等无关文字
4. **输出原始文本** — 将清洗后的文本作为下一阶段的输入

> 详细 OCR 处理指南参见：[references/ocr_guidelines.md](references/ocr_guidelines.md)

---

### 阶段 2：全类别医学实体抽取

**目标**：从 OCR 文本中抽取**所有类别**的医学实体标签。

**实体类别**（详见 [references/medical_entity_types.md](references/medical_entity_types.md)）：

| 类别 | type | 示例 | 有 ICD 编码 |
|------|------|------|:----------:|
| 疾病/诊断名称 | `disease` | 肺炎、2型糖尿病、心肌梗死 | ✅ WHO ICD-10 |
| 解剖部位 | `anatomy` | 肺、冠状动脉、右肺上叶 | ✅ WHO ICD-10 (关联疾病码) |
| 病因/病原体 | `etiology` | 肺炎链球菌、高血压性、酒精 | ✅ WHO ICD-10 B95-B97 / V01-Y98 |
| 症状/体征 | `symptom` | 发热、咳嗽、呼吸困难 | ✅ WHO ICD-10（多为 R00-R99） |
| 手术/操作 | `procedure` | 冠状动脉搭桥术、腹腔镜胆囊切除术 | ✅ ICD-9-CM-3 |
| 药物/物质 | `medication` | 胰岛素、阿司匹林 | ✅ WHO ICD-10 T36-T65 |
| 形态学描述 | `morphology` | 恶性肿瘤、炎性改变、鳞状细胞癌 | ✅ ICD-O |
| 严重程度 | `severity` | 急性、慢性、3级、2型 | ⚠️ 修饰编码 |
| 并发症 | `complication` | 酮症酸中毒、腹水 | ✅ WHO ICD-10 (关联) |
| 合并症 | `comorbidity` | (同 disease) | ✅ WHO ICD-10 |
| 侧别 | `laterality` | 左、右、双侧 | ⚠️ 修饰编码 |
| 检验指标 | `lab_value` | 血糖升高 | ✅ WHO ICD-10（按指标类型） |

**抽取原则**：
- 从文本中抽取**所有可识别的实体**，不限于疾病名
- 症状、解剖部位、手术操作等均有独立的 ICD 编码体系
- 修饰类实体（severity/laterality）用于细化主编码，不独立输出编码

---

### 阶段 3：ICD 编码反查

**目标**：为每个已抽取的实体查找对应的 ICD 编码。

**查找策略**（详见 [references/entity_to_icd_mapping.md](references/entity_to_icd_mapping.md)）：

1. **名称标准化** — 去除侧别、急慢性等修饰成分，展开缩写，统一同义词
   - `右侧急性社区获得性肺炎` → 核心词 `肺炎`
   - `心梗` → `急性心肌梗死`
   - `COPD` → `慢性阻塞性肺疾病`

2. **按实体类型分表查找**：
   - `disease` → WHO ICD-10 疾病编码（第 1-22 章）
   - `symptom` → WHO ICD-10 症状体征编码（多为第 18 章 R00-R99，少数按 WHO 归入其他章节）
   - `procedure` → ICD-9-CM-3 手术操作编码
   - `etiology` → WHO ICD-10 病原体附加编码（B95-B97）或外因编码（V01-Y98）
   - `morphology` → ICD-O 肿瘤形态学编码
   - `anatomy` → 该部位常见疾病的相关 WHO ICD-10 编码
   - `medication` → WHO ICD-10 药物/物质中毒编码（T36-T65）

3. **修饰成分细化** — 使用 severity/laterality/complication 选择更精确的子编码
   - `2型糖尿病` + `酮症酸中毒` → E11.1（非 E11.9）

4. **匹配优先级**：

| 优先级 | 条件 | 置信度 |
|:---:|------|:---:|
| P0 | 精准匹配 — 标准化名称完全对应 | 0.95+ |
| P1 | 上位匹配 — 匹配类别上位编码 | 0.85-0.94 |
| P2 | 模糊匹配 — 关键词部分命中 | 0.70-0.84 |
| P3 | 无法匹配 — 返回章节级未特指编码 | <0.50 |

---

### 阶段 4：构建输出映射

**目标**：将每个实体与其 ICD 编码关联，输出结构化映射。

**输出格式**（详见 [references/output_schema.md](references/output_schema.md)）：

```json
{
  "source": {
    "type": "image",
    "filename": "病历.png",
    "ocr_text": "患者因发热、咳嗽3天，诊断为右侧社区获得性肺炎。既往有2型糖尿病史。"
  },
  "mappings": [
    {
      "entity_type": "disease",
      "entity_text": "社区获得性肺炎",
      "standardized_name": "肺炎",
      "icd_code": "J18.9",
      "icd_version": "ICD-10",
      "icd_system": "WHO ICD-10 疾病编码",
      "match_level": "P0",
      "context": "右侧社区获得性肺炎",
      "confidence": 0.95
    },
    {
      "entity_type": "symptom",
      "entity_text": "发热",
      "standardized_name": "发热",
      "icd_code": "R50.9",
      "icd_version": "ICD-10",
      "icd_system": "WHO ICD-10 症状体征编码",
      "match_level": "P0",
      "context": "因发热、咳嗽3天",
      "confidence": 0.95
    },
    {
      "entity_type": "symptom",
      "entity_text": "咳嗽",
      "standardized_name": "咳嗽",
      "icd_code": "R05",
      "icd_version": "ICD-10",
      "icd_system": "WHO ICD-10 症状体征编码",
      "match_level": "P0",
      "context": "因发热、咳嗽3天",
      "confidence": 0.95
    },
    {
      "entity_type": "disease",
      "entity_text": "2型糖尿病",
      "standardized_name": "2型糖尿病",
      "icd_code": "E11.9",
      "icd_version": "ICD-10",
      "icd_system": "WHO ICD-10 疾病编码",
      "match_level": "P0",
      "context": "既往有2型糖尿病史",
      "confidence": 0.95
    },
    {
      "entity_type": "anatomy",
      "entity_text": "肺",
      "standardized_name": "肺",
      "icd_code": "J98.4",
      "icd_version": "ICD-10",
      "icd_system": "WHO ICD-10（肺部疾患）",
      "match_level": "P1",
      "context": "右侧社区获得性肺炎",
      "confidence": 0.85
    }
  ],
  "metadata": {
    "total_entities_found": 5,
    "entities_by_type": {
      "disease": 2,
      "symptom": 2,
      "anatomy": 1
    },
    "extraction_time": "2026-05-27T10:00:00+08:00",
    "icd_systems_detected": ["WHO ICD-10"],
    "warnings": []
  }
}
```

---

## 质量约束

1. **全类别覆盖** — 疾病、症状、解剖、手术、病因、形态学等所有实体类型均需查找 ICD 编码
2. **一实体一编码** — 每个独立实体返回一个最匹配的 ICD 编码（不合并）
3. **上下文关联** — 每个映射保留原始上下文，用于人工复核
4. **置信度标注** — 对无法完全确定的映射标注较低置信度
5. **编码体系区分** — 默认使用 WHO ICD-10，并明确标注 ICD-9-CM-3 / ICD-O 等非 WHO ICD-10 编码体系
6. **修饰保留** — severity 和 laterality 不独立输出编码，但参与主编码的细化选择

---

## 参考文件索引

| 文件 | 内容 |
|------|------|
| [references/ocr_guidelines.md](references/ocr_guidelines.md) | OCR 处理流程、中文医学文本清洗规则 |
| [references/medical_entity_types.md](references/medical_entity_types.md) | 医学实体分类体系、12 类实体定义与示例 |
| [references/entity_to_icd_mapping.md](references/entity_to_icd_mapping.md) | 全类别医学实体 → ICD 编码对照表与匹配策略 |
| [references/output_schema.md](references/output_schema.md) | 输出 JSON Schema 完整定义与字段说明 |
