```
程序整体流程如下：
1. 遍历输入目录中的 JSON 文件，逐个进行处理：图片本地化处理器
2. 读取并解析 JSON 数据
3. 根据文件名生成文档标题
4. 加载统一的 CSS 样式
5. 初始化图片本地化处理器: localizer = ImageLocalizer(OUTPUT_DIR, title)
6. 渲染文档元信息（Meta 信息）: render_document_meta()
7. 构建正文的层级结构: meta.get("law_regulation_article_jsons") → build_tree()
8. 将正文内容渲染为 HTML
9. 输出生成的 HTML 文件到指定目录
```

```
图片本地化处理器：把 HTML 里的图片 <img src="https://..."> 下载到本地，并把 src 改成本地路径
ImageLocalizer.localize_fragment()
输入 html_str
├── 若包含 <img
    ├── 用 lxml 包装成 <div> 解析 DOM
        遍历所有 <img>
        取 src → 调用 download(src) 获得 new_src
        把 img.src 替换为 new_src
        输出更新后的 innerHTML
```

```
渲染文档元信息
[
    ("文号：", meta.get("document_number", "")),
    ("发布机关：", meta.get("dispatch_authority", "")),
    ("生效日期：", meta.get("effective_date_str", meta.get("effective_date", ""))),
    ("适用范围：", meta.get("effective_range", "")),
    ("文件类型：", meta.get("eff_level", "")),
    ("业务标签：", labels_text),
]
```



```
FORWARD
maybe_cooldown() ← 如果之前 403 / html-block，先冷却
  ↓
fetch_page(page)
├─ 失败（403 / 502 / 503 / 504 / 其他异常）
    ↓
    记录 failed_pages += page
    更新 state.next_page = page + 1
    跳过这一页，继续向前
└─ 成功
    ↓
    判断 page_set 是否有问答
    ├─ 有 → 遍历 page_set 中的每条问答 msg
              ↓
            判断 msg_id 是否已存在
            fetch_detail_html(msg_id)
            parse_detail(html)
            ├─ 失败
                ↓
                记录 failed_ids += msg_id
                跳过这一条问答，继续向前
            └─ 成功
                ↓
                下载附件 download_one_attachment()
                获取问答 record
                upsert_record(db, record)
                更新 count += 1 & max_question_length & max_question_id
    └─ 无 → state.next_page = page + 1 → page += 1
  ↓
page > END_PAGE → FORWARD 完成，开始 BACKFIll
```

