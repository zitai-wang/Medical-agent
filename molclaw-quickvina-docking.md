---
name: molclaw-quickvina-docking
description: 使用 QuickVina2-GPU 在目标蛋白结构与小分子之间进行分子对接。
license: MIT license
metadata:
    skill-author: PJLab
---

# QuickVina2 分子对接

> **注意**：本 skill 涉及的所有工具均已在 agent 工具集中通过标准 MCP 协议预配置，直接调用即可，无需编写连接脚本。

## 前置准备

- 若输入为本地文件，请先使用 `DrugSDA_Tool_upload_file` 上传至服务器
- 若输入为 PDB 文件，建议先用 `fix_pdb` 进行预处理

---

## Step 1：获取目标蛋白结构

调用 `retrieve_protein_structure_by_gene_name` / `retrieve_protein_structure_by_uniprot_id` / `retrieve_protein_structure_by_pdb_id`（根据用户提供的标识符类型选择）获取目标蛋白结构文件。若用户已提供蛋白结构文件，跳过此步骤。

## Step 2：提取目标链（可选）

若用户指定了目标链，或 agent 自主识别出需要提取的单链/多链结构，调用 `extract_and_save_chains` 生成并保存对应的新 PDB 文件。否则跳过此步骤。

### 工具参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `pdb_file_path` | str | 原始 PDB 文件路径 |
| `chain_ids` | List[str] | 待提取的链 ID 列表，如 `["A", "C"]` |

返回 `out_file` — 提取链后生成的新 PDB 文件路径。

---

## Step 3：修复蛋白结构

调用 `fix_pdb` 修复蛋白结构，推荐设置如下：

| 参数 | 建议值 |
|------|-------|
| `input_path` | Step 1 或 Step 2 得到的 PDB 路径 |
| `add_hydrogens` | `true` |
| `ph` | `7.0` |
| `remove_heterogens` | `true` |
| `remove_water` | `true` |
| `replace_nonstandard` | `true` |

返回 `output_file` — 修复后的 PDB 文件路径。

---

## Step 4：检测结合口袋

调用 `fpocket_toolkit`（优先）或 `pred_pocket_prank` 检测蛋白结构的结合位点，返回最佳口袋信息（center_x/y/z、size_x/y/z）。若口袋中心坐标和盒子尺寸已知（如来自共晶配体），跳过此步骤，直接使用已知值。

---

## Step 5：执行分子对接

调用 `molecule_docking_quickvina_fullprocess` 进行分子对接。此工具接受 PDB 文件和 SMILES 字符串，**自动处理所有格式转换**（PDB→PDBQT, SMILES→PDBQT），**无需手动预处理**。

### 工具参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `pdb_file_path` | str | 蛋白受体 PDB 文件路径 |
| `smiles` | str | 配体分子 SMILES 字符串 |
| `pocket_center_x` | float | 对接口袋中心 X 坐标 |
| `pocket_center_y` | float | 对接口袋中心 Y 坐标 |
| `pocket_center_z` | float | 对接口袋中心 Z 坐标 |
| `pocket_size_x` | float | 对接口袋 X 轴尺寸（默认 25.0） |
| `pocket_size_y` | float | 对接口袋 Y 轴尺寸（默认 25.0） |
| `pocket_size_z` | float | 对接口袋 Z 轴尺寸（默认 25.0） |

### 返回结果

| 字段 | 说明 |
|------|------|
| `status` | success / error |
| `msg` | 状态消息 |
| `docking_affinity_value` | 对接亲和力值（kcal/mol），越负表示结合越强 |
| `docking_file` | 含对接构象的 PDBQT 文件路径 |

### 关键规则

- **盒子最小尺寸**：`pocket_size_x/y/z` 不得低于 **25.0 Å**。若口袋检测工具返回的尺寸小于 25 Å，用 25.0 Å 覆盖该维度。
- **分数验证**：若 `docking_affinity_value` 为正值，表示对接失败，不要接受。可尝试逐步增大盒子尺寸（25→30→40→50 Å）后重试。
- **筛选阈值参考**（药物样小分子 MW 300–500）：
  - ≤ -7 kcal/mol：潜在结合活性起点
  - ≤ -9 kcal/mol：强结合
  - 实践中建议对同一靶点的所有化合物排序后选取 **top n** 进一步验证

---

## 多靶点对接快捷方式

当对**同一分子**对接多个**已准备的靶点**（各靶点口袋参数已锁定，无需重新检测）时，跳过 Steps 1–4，直接为每个靶点调用 `molecule_docking_quickvina_fullprocess`，使用各靶点锁定的口袋参数。