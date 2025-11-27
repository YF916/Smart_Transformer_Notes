
```
原始HTML文件
    ↓
┌─────────────────────────────────────┐
│ Phase 1: 预处理 (preprocessor.py)    │
│ - 标签转换 (<b> → <strong>)           │
│ - 删除标签 (<font>保留内容)            │
│ - 编码规范化 (UTF-8)                  │
│ - 孤立标签保护 (占位符方案)             │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ Phase 2: 分区 (partitioner.py)       │
│ - 编注区识别 (正则表达式)              │
│ - 正文区识别                         │
│ - 自定义区识别                       │
│ - 生成有序内容块列表                  │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ Phase 3: 清洗 (cleaner.py)          │
│ - 规则21: td内p标签转换               │
│ - 规则22: hr独立化                   │
│ - 规则23: 移除空白p标签               │
│ - 孤立标签保护 (完整标签方案)           │
└─────────────────────────────────────┘
    ↓
转换后的HTML文件
```

```
Phase 2：内容标注 + 结构分区 （先语义后结构）
│
├── 一、process() —— “语义层”处理（文本级）
│   │
│   ├── 加载 YAML 中的 annotation 正则规则
│   ├── 在整篇 HTML（或 scope）中查找所有编注区间
│   ├── 对内容进行语义分类：
│   │     ├── 编注片段 → <div class="xxx-annotation">
│   │     └── 正文片段 → <div class="main-content">
│   ├── 输出带语义标签的 HTML
│   └── （目的：先把“内容类型”标清楚，为结构化做准备）
│
└── 二、partition() —— “结构层”处理（DOM级）
    │
    ├── 使用 lxml 解析 process 输出的 HTML
    ├── 按 DOM 顺序遍历节点
    ├── 依据标签和 class 判断结构类型：
    │     ├── annotation div → ANNOTATION block
    │     ├── main-content / p → TEXT block
    │     ├── hr → HR block
    │     ├── table → TABLE block
    │     └── center → TEXT block（标题／居中文字）
    ├── 跳过 annotation/table 等已整体处理的子节点
    └── 输出 ContentBlock 列表
```
```
Phase 3: 清洗 main-content
main_content_cleaner.py
│
├── 配置 transform_rules.yaml
（加载和正文清洗相关的所有规则和配置）
    ├── 规则列表
        ├── 取 main_content_rules
        ├── 过滤 enabled: true 的规则
        └── 按 priority 排序
    └── 清洗器配置
        ├── 取 main_content_cleaner_settings
        ├── CSS 相关
        ├── XPath 选择器
        ├── 文档包装标签（html/body/head）
        ├── 语义标签
        ├── iterations 次数限制
        ├── 空白字符规范化
        ├── 行内段落中断修复
        ├── 表格尾部多余 </p> 清理
        └── 附件标题 wrapper
└── clean()
  （遍历 ContentBlock，清洗 TEXT 和 TABLE，返回 cleaned_block 列表）
    ├── 提取正文区 TEXT （CSS class 判断）和 TABLE block
    ├── 根据优先级顺序执行规则
        ├── Priority 21：_transform_td_p_tags （防止 lxml 自动补标签，导致 table 结构扭曲）
            ├── <p>...</p> → ... + <span class="para_break">
            ├── <br> → <span class="para_break">
            └── 修复/结构性修补
                ├── inline_para_break_re 行内段落中断修复
                ├── trailing_td_re 清理 </p></td>
                └── 附件标题 wrapper
        ├── Priority 22：_ensure_hr_independence
            ├── 找到 p/div 里包着的 hr
            ├── 把 parent 分为前半段、hr、后半段
            └── 再插回 parent_parent
        ├── Priority 23：_remove_empty_p_tags
            ├── 删除没有文本，没有子元素，或子元素为空的 <p>
            └── _serialize_tree 提取实际内容
        ├── Priority 24：_remove_redundant_nesting
            ├── 去除多余嵌套
            └── _serialize_tree 提取实际内容
        ├── Priority 25：_normalize_whitespace
            └── 移除多余空格和换行
        └──  Priority 26：_clean_br_tags
            └── 清理正文区br标签
    ├── _extract_text 更新 block
    └── 放回 cleaned_block 返回

Notes:
priority 的逻辑顺序是：
1. 最会破坏结构的要最早处理 (td/p/tag 修复)
2. 会影响 DOM 结构的放中间，清除结构冗余 (hr 独立化 / 去空 p / 去嵌套容器)
3. 视觉层面的清理放最后 (空白规范化 / br 清理)
```

```
✅ 01_requirements_and_design.md
```
