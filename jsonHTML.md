程序整体流程如下：
1. 遍历输入目录中的 JSON 文件，逐个进行处理
2. 读取并解析 JSON 数据
3. 根据文件名生成文档标题
4. 加载统一的 CSS 样式
5. 初始化图片本地化处理器
6. 渲染文档元信息（Meta 信息）
7. 构建正文的层级结构
8. 将正文内容渲染为 HTML
9. 输出生成的 HTML 文件到指定目录

图片本地化处理器：把 HTML 里的图片 <img src="https://..."> 下载到本地，并把 src 改成本地路径
localize_fragment()
输入 html_str
|
  若包含 <img ? 
|
     是
      |
      v
[用 lxml 包装成 <div> 解析 DOM]
      |
      v
[遍历所有 <img>]
      |
      v
[取 src] -> 调用 download(src) -> [得到 new_src]
      |
      v
[把 img.src 替换为 new_src]
      |
      v
[输出更新后的 innerHTML]



