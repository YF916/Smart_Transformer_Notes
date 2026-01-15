```
程序整体流程如下：
1. 遍历输入目录中的 JSON 文件，逐个进行处理：图片本地化处理器
2. 读取并解析 JSON 数据
3. 根据文件名生成文档标题
4. 加载统一的 CSS 样式
5. 初始化图片本地化处理器: localizer = ImageLocalizer(OUTPUT_DIR, title)
6. 渲染文档元信息（Meta 信息）: render_document_meta()
7. 构建正文的层级结构: meta.get("law_regulation_article_jsons") → build_tree()
8. 将正文内容渲染为 HTML: render_tree()
9. 输出生成的 HTML 文件到指定目录: build_html() 组装成完整 HTML 文档 → write_text()
```

```
图片本地化处理器：把 HTML 里的图片 <img src="https://..."> 下载到本地，并把 src 改成本地路径。
ImageLocalizer.localize_fragment()
输入 html_str
├─ 若包含 <img
    ├─ 用 lxml 包装成 <div> 解析 DOM
        遍历所有 <img>
        取 src → 调用 download(src) 获得 new_src
        把 img.src 替换为 new_src
        输出更新后的 innerHTML
```

```
渲染文档元信息：将 JSON 中的文档元信息统一渲染为 HTML 元信息区块，展示在文档顶部。
render_document_meta()
文号：meta.get("document_number", "")
发布机关：meta.get("dispatch_authority", "")
生效日期：meta.get("effective_date_str", meta.get("effective_date", ""))
适用范围：meta.get("effective_range", "")
文件类型：meta.get("eff_level", "")
业务标签：meta.get("labels")
↓
渲染每一项元信息（保持结构稳定）
组合成完整的 Meta 区块
```

```
构建正文的层级结构：根据 level，使用栈结构将节点列表构建为层级化的树形结构 (stack-based tree building)。
build_tree(): 仅将 level = 1/2/3/4 视为结构性节点
```

```
将正文内容渲染为 HTML：遍历结构树，根据不同 level，选择不同的渲染策略，生成最终的正文 HTML 字符串。
render_tree()
├─ level == 0
    ↓
    unescape()
    strip_links_keep_text(): 把 <a> 超链接标签去掉，但保留显示文字
    localize_fragment(): 图片本地化
    ├─ 块级 HTML（r"^\s*<(?:center|div|p|table|ul|ol|h[1-6]|hr)\b"）: 直接输出
    └─ 纯文本：统一包成 <p>
├─ level in (1,2,3)
    ↓
    render_heading(): 渲染标题为 <h3>/<h4>/<h5>
    render_tree(): 递归渲染下面所有子节点（正文内容）
└─ level == 4: 条（article）节点
    ↓
    render_article_div()
        ↓
        读取条信息与正文文本 idx/fullName/text
        unescape()
        localize_fragment(): 图片本地化
        split_to_paras(): 按 <br/> 和 \n 文本拆段
        strip_links_keep_text(): 去除 <a> 标签
        classify_para(): clause/item 识别
        ├─ r"^\s*(（[一二三四五六七八九十百千万]+）)\s*(.*)$"
        └─ r"^\s*((?:[1-9][0-9]*\.|[①②③④⑤⑥⑦⑧⑨⑩⑪⑫⑬⑭⑮⑯⑰⑱⑲⑳]))\s*(.*)$")
        找到第一个 para 作为标题段（可为空）
        输出条标题行：第X条 + 标题段(可选)
        遍历剩余段落：
        ├─ clause → clause div
        ├─ item   → item div
        └─ para   → <p>
```
