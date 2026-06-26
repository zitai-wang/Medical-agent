# 输出 JSON Schema（output_schema.md）

本文件定义结构化提取的完整 JSON 输出格式。技能须**先输出人类可读表格（见主文件输出模板），再输出本 JSON**。

> 设计要点：每个字段对象统一带 `value` / `source_text` / `source_doc`，实现"忠于原文、可追溯"——这是赛道2"准确率"的可验证保证。缺失字段 `value` 填 `"未提及"`。

---

## 顶层结构

```json
{
  "source": {
    "input_type": "text | image",
    "documents_detected": ["门诊病历", "首次病程记录", "MRI检查报告单", "病理检查报告", "检验结果摘录", "住院病案首页"],
    "privacy_note": "已脱敏字段：姓名/住院号/证件号等"
  },

  "patient_info": {
    "age":        {"value": "高龄患者", "source_text": "年龄：高龄患者", "source_doc": "门诊病历"},
    "sex":        {"value": "男", "source_text": "性别：男", "source_doc": "门诊病历"},
    "anthropometry": {"value": "未提及", "source_text": "", "source_doc": ""},
    "department": {"value": "肝胆胰外科", "source_text": "就诊科室：肝胆胰外科门诊", "source_doc": "门诊病历"},
    "visit_status": {"value": "初诊", "source_text": "就诊状态：●初诊", "source_doc": "门诊病历"},
    "admission_route": {"value": "门诊", "source_text": "入院途径：门诊", "source_doc": "住院病案首页"}
  },

  "clinical_presentation": {
    "chief_complaint": {"value": "发热5天", "source_text": "主诉：发热5天。", "source_doc": "门诊病历"},
    "hpi_summary": {"value": "...", "source_text": "...", "source_doc": "现病史"},
    "symptoms": [
      {"name": "发热", "status": "有", "source_text": "发热5天", "source_doc": "门诊病历"},
      {"name": "腹痛", "status": "轻微", "source_text": "伴轻微腹痛", "source_doc": "现病史"}
    ],
    "physical_signs": {"value": "上腹部轻压痛", "source_text": "上腹部轻压痛", "source_doc": "体格检查"}
  },

  "risk_background": {
    "hbv": {"value": "未提及", "detail": "", "antiviral": "", "source_text": "", "source_doc": ""},
    "hbv_dna": {"value": "未提及", "source_text": "", "source_doc": ""},
    "hbv_panel": {"value": "未提及", "source_text": "", "source_doc": ""},
    "cirrhosis": {"value": "未提及", "basis": "", "source_text": "", "source_doc": ""},
    "fibrosis": {"value": "未提及", "source_text": "", "source_doc": ""},
    "past_history": {"value": "冠心病行支架治疗15年余，肝血管瘤病史；否认糖尿病", "source_text": "...", "source_doc": "既往史"},
    "personal_history": {"value": "未提及", "source_text": "", "source_doc": ""},
    "family_history": {"value": "未提及", "source_text": "", "source_doc": ""}
  },

  "imaging": [
    {
      "imaging_modality": "肝MRI平扫+增强+MRCP+DWI",
      "lesion_site": "肝内（近门静脉左支）",
      "lesion_size": "59.7mm×63.1mm×63.4mm",
      "lesion_number": "单发",
      "signal": "T1WI/T2WI混杂信号，DWI不均匀稍高信号",
      "enhancement": "动脉期不均匀明显强化，延迟期强化减退（快进快出）",
      "imaging_signs": "可见假包膜",
      "vascular": "病灶与门静脉左支分支关系密切",
      "imaging_impression": "肝脏占位，考虑原发性肝细胞癌",
      "incidental": "胆囊小结石；左肾囊肿",
      "source_doc": "MRI检查报告单"
    }
  ],

  "pathology": {
    "pathology_dx": {"value": "高分化肝细胞性肝癌", "source_text": "考虑为高分化肝细胞性肝癌", "source_doc": "病理检查报告"},
    "specimen": {"value": "肝脏穿刺组织", "source_text": "（肝脏穿刺组织）", "source_doc": "病理检查报告"},
    "grade": {"value": "高分化", "source_text": "高分化", "source_doc": "病理检查报告"},
    "ihc": {
      "value": "Glypican-3(-), Hepatocyte(+), CD34(血管+), AFP(-), CK7(-), CK19(-), Ki-67(8%+), GS(+), β-catenin(+), HSP70(+), INSM1(-)",
      "parsed": [
        {"marker": "Glypican-3", "result": "-"},
        {"marker": "Hepatocyte", "result": "+"},
        {"marker": "Ki-67", "result": "8%+"},
        {"marker": "GS", "result": "+"},
        {"marker": "HSP70", "result": "+"}
      ],
      "source_doc": "病理检查报告"
    },
    "special_stain": {"value": "未提及", "source_text": "", "source_doc": ""},
    "path_note": {"value": "送检组织极少；经科室多位主任阅片", "source_text": "...", "source_doc": "病理检查报告"}
  },

  "tumor_markers": [
    {"name": "甲胎蛋白(AFP)", "value": "55.95", "unit": "ng/mL", "ref_range": "0-8.1", "abnormal": "升高", "source_doc": "检验结果摘录"},
    {"name": "癌胚抗原(CEA)", "value": "1.14", "unit": "ng/mL", "ref_range": "0-5", "abnormal": "正常", "source_doc": "检验结果摘录"}
  ],

  "diagnosis_staging": {
    "outpatient_dx": {"value": "肝占位性病变（血管瘤？）；胆囊结石", "source_doc": "门诊病历"},
    "admission_dx": {"value": "肝占位性病变（考虑HCC）", "source_doc": "首次病程记录"},
    "main_diagnosis": {"value": "肝细胞癌", "source_doc": "病理/病案首页"},
    "other_diagnoses": ["胆囊结石", "左肾囊肿"],
    "icd10": {"code": "C22.0", "match_level": "P0", "confidence": 0.95, "basis": "穿刺病理确诊肝细胞癌"},
    "icdo": {"code": "8170/3", "match_level": "P0", "confidence": 0.95},
    "benign_malignant": {"value": "恶性", "basis": "病理：高分化肝细胞性肝癌；免疫组化Hep(+)/GS(+)/HSP70(+)"},
    "cnlc": {"value": "Ⅰb（置信度中，缺转移评估完整信息）", "confidence": 0.7},
    "bclc": {"value": "A（置信度中）", "confidence": 0.7},
    "tnm": {"value": "未提供"}
  },

  "treatment": [
    {"type": "治疗计划", "detail": "肝脏MRI增强、肿瘤四项、生化等完善检查", "date": "未提及", "source_doc": "门诊病历"}
  ],

  "quality_summary": {
    "total_fields": 32,
    "filled_fields": 24,
    "missing_fields": 8,
    "coverage": "75%",
    "warnings": [
      "门诊初诊诊断为'血管瘤?'，后病理确诊HCC，存在诊断演变（非矛盾，记录供参考）"
    ],
    "warnings_count": 1,
    "icd_systems_used": ["ICD-10", "ICD-O-3", "CNLC", "BCLC"]
  }
}
```

---

## 字段对象通用规范

| 键 | 含义 | 必填 |
|----|------|------|
| `value` | 标准化后的字段值；缺失填 `"未提及"` | 是 |
| `source_text` | 原文片段（支撑该值） | 有值时必填 |
| `source_doc` | 来源文书类型 | 有值时必填 |
| `confidence` | 映射/判定置信度 0-1（编码与分期字段） | 编码类必填 |
| `match_level` | P0/P1/P2/P3（编码字段） | 编码类必填 |

## 输出顺序（强制）

1. 先输出主文件定义的**可读结构化表格**（中文，给临床医生看）。
2. 再输出本 **JSON**（给系统/数据集用）。
3. JSON 必须是合法可解析的 JSON，不得含注释、不得截断。
