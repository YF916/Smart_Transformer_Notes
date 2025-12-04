
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

```
Phase 4: 清洗编注区
annotation_cleaner.py
│
├── 配置文件 transform_rules.yaml
    ├── 编注前缀 annotation_prefixes
    └── 从编注规则 annotation_rules 提取 Priority 34 清理规则
└── clean_annotation()
    （只处理 BlockType.ANNOTATION）
    ├── 编注CSS类添加 (Priority 31) _add_annotation_css_class
        ├── 获取原 CSS class
        ├── 添加基础 annotation 类
        ├── 合并 CSS class
        └── 外层 <div> 保证
    ├── 编注结构规范化 (Priority 32) _normalize_annotation_structure
        └── 统一成 <div><p> ... </p></div> 结构
            ├── 原来是 p，在外层创建 div
            ├── 原来是 div 内无 p，创建 p
            └── 原来是其他标签 → div + p
    └── 编注内容清洗 (Priority 34) _clean_annotation_content
        ├── 移除 hr，清理 br
        ├── 递归遍历所有 .text 和 .tail
            └── 空格和换行清理
        ├── 整段 HTML 换行清理
        └── <br> → <br/>
```

```
Phase 5: 结构识别
structure_recognizer.py
│
├── 配置文件 transform_rules.yaml
├── _compile_patterns()
    ├── 遍历所有结构识别规则 pattern、pattern_flags
    ├── 编译正则表达式
    ├── 构建规则列表 
    └── 按 priority 排序
└── recognize()
    ├── 判断子标题满足条件
        requires_keywords: "点击下载"
        requires_keywords_mode: "any"
    ├── 依次尝试识别规则 recognizer(block, prev_block)
        self.recognition_rules = [
            (51, "subtitle",  self._recognize_subtitle),
            (52, "chapter",   self._recognize_chapter),
            (53, "section",   self._recognize_section),
            (53.5, "section_enum",   self._recognize_section_enum),
            (54, "article",   self._recognize_article),
            (55, "clause",    self._recognize_clause),
            (56, "item",      self._recognize_item),
        ]
        ├── Priority 51: 识别子标题 _recognize_subtitle
            ├── self._subtitle_keywords_ok = True
            ├── prev_block 必须是 hr
            ├── 获取 pattern
            ├── 获取 pattern_type
                ├── 如果 pattern_type: "content"（默认）→ 用 block.content（纯文本）做正则匹配
                └── 如果 pattern_type: "html" → 用 block.original_html（原始 HTML 片段）做匹配
            └── _apply_recognition()
                ├── block type & level & css_class & id_prefix
                └── _wrap_{rule_name}_html()
                    └── _generate_html_from_template()
                        ├── html_strategy & html_template & css_class & clean_content
                        └── 根据 html_strategy 选择不同策略
                            ├── 完全替换 full_replace 用新标签包纯文本
                            ├── 保留内部标签换外层标签 wrap_keep_inner
                                ├── 取原始 HTML 第一个元素
                                ├── 提取 inner HTML _inner_html() 获取 DOM 里面的字符串
                                └── 生成模板并填充
                            └── 提取编号并用标签包围编号 wrap_number_only
                                ├── 根据 pattern 匹配编号
                                ├── DOM 里查找以编号开头的文本节点
                                └── 把编号取出，包 <span class="xxx">
        ├── Priority 52: 识别章 _recognize_chapter
            └── _generic_recognize()
                ├── 获取配置 & pattern & pattern_type
                └── _apply_recognition()
                    └── ... ...
        ├── Priority 53: 识别节 _recognize_section
            └── _generic_recognize()
                └── ... ...
        ├── Priority 53.5: 识别中文枚举节 _recognize_section_enum
            └── _generic_recognize()
                └── ... ...
        ├── Priority 54: 识别条 _recognize_article
            └── _generic_recognize()
                └── ... ...
        ├── Priority 55: 识别款 _recognize_clause
            └── _generic_recognize()
                └── ... ...
        └── Priority 56: 识别项 _recognize_item
            └── _generic_recognize()
                └── ... ...
    └── 未识别的保持TEXT类型
```

```
Phase 6: 层级构建与编注归属
hierarchy_builder.py
annotation_context_assigner.py
│
├── Step 1: 为所有结构块生成ID （Priority 73）
    @staticmethod 这个函数不需要用到类自己的属性（self），只是放在类里面当一个工具函数。
    HierarchyBuilder.generate_ids()
    │
    ├── 初始化
        self.stack 容器栈
        self.output HTML输出缓冲区
        self.id_counters ID计数器
        self.path_stack 路径栈
        self.current_annotation_group_id 当前打开的编注组ID
        self.annotation_group_info 编注组信息：{group_id: {"first_idx": int, "last_idx": int}}
    └── generate_ids() 为每个结构块生成语义化 ID
        ├── 初始化
            id_counters：结构分别计数
            parent_stack：生成层级 ID
        └── 遍历每个 block 除了 ANNOTATION, TEXT, HR
            ├── 如果当前 block 的 level <= parent_stack 栈顶 → pop() 推出
            ├── 取父 ID
                ├── 如果栈不为空 → 父 ID 就是 parent_stack 顶节点的 ID
                └── 如果为空 → 当前是顶层结构
            ├── prefix 计数 + 1
            ├── 生成 ID
                ├── 顶层结构（章/节/条） → f"{prefix}-{id_counters[counter_key]}"
                └── 层级结构（款/项） → f"{parent_id}-{prefix}-{id_counters[counter_key]}"
            └── 把当前 block 推进 parent_stack
│
├── Step 2: 分配编注归属
    AnnotationContextAssigner.assign_context()
    │    
    ├── Step 1: 识别连续编注组（Priority 76）
        _identify_annotation_groups()
        └── 找到一个编注块后扫描连续出现的编注（跳过 TEXT/HR）
            至少两个编注形成组
            组内所有编注共享同一个 group_id
    └── Step 2: 为每条编注分配归属（Priority 77 - 79）
        遍历所有 ANNOTATION block
        _assign_annotation_context() （Priority 79）
            related_to: 它挂靠的结构类型
            related_id: 对应结构块的 id
            related_level: 对应结构块的层级 level
        ├── 找前置结构块（Priority 77）
            _find_preceding_structure()
            从当前编注往前找到最近的一个结构块（章/节/条/款/项）
        ├── 如果没有前置结构 → 文档级编注 "document"
        ├── 取前置结构块的 index 和 block
        ├── 判断当前前置结构块是不是它父节点下的最后一个子节点 （Priority 78）
            _is_last_child() 前置结构块和当前编注 index
            ├── 取前置结构块
            ├── 从当前编注开始向后扫描
            └── 直到遇到下一个结构节点
                ├── 同级元素 → 前置元素不是最后子元素(有兄弟节点)
                ├── 更深层级 → 前置元素不是最后子元素(有子节点)
                └── 更高层级(level更小) → 前置元素是最后子元素
        └── 根据判断是否是最后一个子节点的结果决定挂靠到谁
            ├── 前置结构块是最后子元素 → 挂父元素
                _find_parent_structure()
                从当前结构块（child）往前扫描，第一个 level 更小的结构块，就是它的 parent
            └── 不是最后子元素 → 挂前置结构块
│
└── Step 3: 构建层级HTML
    HierarchyBuilder.build()
    ├── 统计每个编注组的起止 index
    ├── 构建层级结构（Priority 71 – 72）
        _process_block()
        ├── 非结构块（ANNOTATION / HR / TEXT / TABLE）
            ├── 如果是 ANNOTATION 关闭栈直到栈顶level <= related_level
                pop 栈顶并 _close_container()
            └── _output_non_hierarchy_block()
                ├── ANNOTATION: 按编注组的起止位置开关 <div class="annotation-group"> 并加入 data-* 属性
                ├── HR: 跳过 annotation-group 中的 HR，其他直接输出
                └── TEXT/TABLE: 直接输出
        └── 结构块（章/节/条/款/项）
            ├── 关闭 level >= 当前 block 的 container
            ├── pop 栈顶并 _close_container()
            └── 输出当前 block
                ├── 是 container → _output_container_open() 并 push 入栈
                    ├── 结构块 id append 到 path_stack 并构造 data-path
                    └── 输出 <div ...> 作为结构容器并设置 id/CSS class/data-path
                        ├── CHAPTER: original_html / <h4>
                        ├── SECTION / SECTION_ENUM: original_html / <h5>
                        ├── SUBTITLE: original_html / <h3>
                        ├── ARTICLE: original_html / <p><strong>
                        ├── CLAUSE: 加 clause-title class
                        └── ITEM: 加 item-title
                └── 不是 container → _output_non_container() 不入栈
                    ├── 构建 data-path
                    └── 用 <span> 包 id/CSS class/data-path 写入内容并闭合
    ├── 关闭结构容器
        _close_container()
        ├── 输出 </div>
        └── 从 path_stack 中 pop 该 block
    └── 关闭编注组容器
    
```

```
拼回原文并保留头部/尾部
修复可能重复的"）"
```

```
Phase 7:
│
└── 公式检查与括号修正
    formula_checker.py
    ├── 仅检查不修正
        ├── find_formulas()
            ├── 遍历文本节点
            ├── is_formula(): 包含符号&数字&长度合理
            ├── check_brackets(): 检查括号多余/匹配/闭合
            └── 返回公式信息
        └── report(): 生成检查报告
    └── 修正
        fix_brackets_in_html()
        ├── 遍历文本节点
        ├── is_formula()
        ├── check_brackets()
        └── 替换修正文本并保持与原文相同的缩进和换行
            elem.replace_with(original_text.replace(text, corrected))
│
└── 最终验证与清理（Priority 91-1000）
```
