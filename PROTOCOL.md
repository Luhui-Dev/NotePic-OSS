# NotePic OSS 协议规范

本文档是 [NotePic-OSS-CLI](NotePic-OSS-CLI) 与 [NotePic-OSS-Obsidian](NotePic-OSS-Obsidian) 共享的“线上/盘上契约”的唯一权威来源。两端各自声明一个 `PROTOCOL_VERSION` 常量；本文件定义这个版本号意味着什么，以及每一端要声明它必须满足哪些保证。

- CLI：`notepic_oss/__init__.py` → `PROTOCOL_VERSION`
- Obsidian：`src/protocol.ts` → `PROTOCOL_VERSION`

**当前版本：`1.0`**

## 0. 范围

协议覆盖的是两端同时操作同一个 OSS bucket 和/或同一个 vault/文档时必须互通的部分：对象如何命名、公网 URL 如何构造、如何识别“这已经是我们自己上传过的”、文档语法如何解析/改写。本协议**特意不覆盖**内部图片编码策略（JPEG/PNG/WebP 的重编码参数）——那些只影响输出质量，不影响互操作性，单独在 §4 跟踪。

## 1. 版本规则

- **升大版本号**（`1.0` → `2.0`）：当某项改动会导致一端产出的对象/URL 在另一端无法识别或读取时——例如更换哈希算法、哈希长度、key 结构，或 §3 中的 URL 构造规则。
- **升小版本号**（`1.0` → `1.1`）：向后兼容的增量式契约变更——例如新增一种 Markdown 图片语法、新增一个跳过区域。
- 只有一端升版本号而另一端没跟上，视为发布阻断项：任何已打 tag 的发布中，两个 `PROTOCOL_VERSION` 常量必须相等。理想情况下 CI 应在二者不一致时拦截发布（见 §7）。
- 只涉及 §4（仅影响质量的行为）或 §5（明确允许两端不一致的行为）的改动，**不需要**升协议版本号。

## 2. v1.0 的兼容性声明

> 在协议 `1.0` 下，由任意一端上传的对象，必须能被另一端在协议 `1.0` 下正确寻址（URL 可访问）、正确识别为“已上传”（`is_own_url` / `isOwnUrl`），并能从相同字节重新推导出相同的 key。

## 3. 规范性契约（必须逐字节保持一致）

### 3.1 对象 key 推导

```text
key = "<prefix>/<sha256(bytes)[:24]><ext>"   当 prefix 非空时
key = "<sha256(bytes)[:24]><ext>"            否则
```

- `sha256(bytes)` 是对**最终待上传字节**（如果执行了压缩，则是压缩后的字节）计算的哈希，十六进制小写编码，截取前 24 个字符。
- `ext` 全部小写，必须以 `.` 开头，且 `.jpeg` 在用于拼接 key 之前要先归一化为 `.jpg`。
- `prefix` 任何时候都不带首尾斜杠（由调用方在加载配置时统一裁剪一次，见 §3.5 中的配置说明）。
- 幂等性要求：PUT 之前必须先检查 `key` 是否已存在（`HEAD` / `object_exists`），存在则跳过上传。

### 3.2 公网 URL 构造

```text
若设置了 custom_domain：
    url = "https://<custom_domain>/<key>"     # 仅当原值没有协议前缀时才补上 "https://"
否则：
    host = endpoint 去掉 "http://" / "https://" 前缀后的结果
    url = "https://<bucket>.<host>/<key>"
```

### 3.3 “是否为自家 URL”判定（`is_own_url` / `isOwnUrl`）

```text
true   若设置了 custom_domain 且 custom_domain 是该 url 的子串
true   若 "<bucket>." 是该 url 的子串，且 host（按 §3.2 计算）也是该 url 的子串
false  其他情况
```

这必须保持为纯字符串子串判断（不能发起网络请求）——它会在扫描到的每一条引用上运行，用来决定是否跳过已上传的图片。

### 3.4 Content-Type 赋值

两端都必须从一张**以扩展名为 key 的静态表**中解析 `Content-Type`，不能依赖平台相关的 API（Python 的 `mimetypes.guess_type` 读取操作系统自带的 mime 数据库，不能保证与 TypeScript 端的静态 `MIME` 表一致——见 §6 中的待办项）。

权威对照表（按字母顺序扩展，两端务必保持同步）：

| 扩展名      | Content-Type  |
| ----------- | ------------- |
| .jpg, .jpeg | image/jpeg    |
| .png        | image/png     |
| .webp       | image/webp    |
| .gif        | image/gif     |
| .svg        | image/svg+xml |
| .avif       | image/avif    |
| .bmp        | image/bmp     |
| .tif, .tiff | image/tiff    |
| .ico        | image/x-icon  |

如果扩展名不在表中，应省略 `Content-Type` 头，而不是去猜测。

### 3.5 识别的文档语法

两端必须以完全相同的方式解析：

1. Markdown 图片：`![alt](url)`、`![alt](url "title")`、`![alt](<带空格的 url>)`。
2. HTML/JSX `<img src="...">`（只处理带引号的值；`src={expr}` 这种 JSX 表达式忽略）。
3. 引用式定义：`[label]: url`——只有当 `url` 的扩展名属于图片扩展名集合（§3.6）时才当作图片引用保留。
4. Obsidian wikilink 嵌入：`![[target]]`、`![[target|alt]]`、`![[target|400]]` / `![[target|400x200]]`、`![[target|alt|400x200]]`（管道分段的顺序不影响解析；匹配 `^\d+(x\d+)?$` 的那个分段是尺寸，其余都是 alt 文本）。只有当 `target` 的扩展名属于图片扩展名集合时才会被识别——非图片的 wikilink（笔记转录）保持原样不动。

绝不能被改写的跳过区域：围栏代码块（` ``` ` / `~~~`）、行内代码（`` ` ``...`` ` ``），以及 HTML 中的 `<script>`、`<style>`、`<pre>`、`<code>` 和 `<!-- -->` 注释内部。

图片扩展名集合：`.png .jpg .jpeg .gif .webp .svg .avif .bmp .tif .tiff .ico`。

参考实现：`NotePic-OSS-CLI/notepic_oss/processor.py` 里的正则 ↔ `NotePic-OSS-Obsidian/src/core/regex.ts`（后者明确标注是前者的 1:1 移植——必须保持这个状态；任何正则改动都要在同一次提交里同步改两端）。

## 4. 仅影响质量的行为（不属于协议规范）

这些只影响输出的文件大小/质量，不影响互操作性——同一个对象哪怕两端编码方式不同，依然是 §3 意义下合法、可寻址、幂等的对象。它们**应该**趋同以保证用户体验一致，但这里的不一致属于 bug，不是协议违规，不会阻塞 `1.0` 的打标签。

- 4.1 各格式的压缩参数（JPEG 质量/渐进式、PNG 无损 vs 量化、WebP 编码方式）。

  PNG 的语义两端已对齐（修复时无需跟随协议版本号变化，但作为约定记录于此，避免再次出现分歧）：复用同一个 `quality`（1–100）参数，**不**为 PNG 引入独立的开关。
  - `quality == 100`：无损优化（仅做 deflate 级别的重新编码，不丢任何像素信息）。
  - `quality < 100`：量化为 256 色调色板（保留 alpha 通道），CLI 用 Pillow `Image.quantize(colors=256, method=Image.Quantize.FASTOCTREE)`，Obsidian 用 UPNG.js `cnum=256`。两者算法路线不同，产出字节不要求一致（仍属于 §4，不要求位级对齐），但“质量旋钮在哪个值切换有损/无损”这件事两端必须一致，否则同一份配置在两端会有完全不同的视觉结果，这才是真正会让用户困惑的分歧。
  - 任何一端要改变这个切换阈值（例如改成按百分比线性插值色彩数），必须同时改两端并在同一次提交里说明。

  “未知格式带 alpha 通道”的回退路径两端已对齐：解码后的像素只要带透明度，就回退到上面的 PNG 路径（同样按 `quality` 决定无损/量化）；否则拍平为 JPEG。

  唯一允许保留的差异（永久性平台限制，不是 bug）：CLI 用 PIL 的图像 mode（`RGBA`/`LA`/`P`）判断"是否有 alpha 通道"，这是一个结构性判断——哪怕通道里所有像素都不透明，只要 mode 带 alpha 就会走 PNG 路径；浏览器的 Canvas/`ImageBitmap` API 解码后只剩裸 RGBA 像素，无法还原"原始格式是否带 alpha 通道"这个结构信息，所以 Obsidian 端改成扫描解码后的像素，只要存在任意一个 alpha<255 的像素就判定为"有透明度"。两端因此只在"有 alpha 通道但全图不透明"这一冷僻场景上可能选择不同的输出格式（一个存 PNG，一个存 JPEG）——参照 §5 的精神，这属于平台能力差异，不需要也不应该试图消除。
- 4.2 远程图片的扩展名探测顺序（先看 Content-Type 头还是先看 URL 路径）。

  两端已对齐为：先精确匹配 Content-Type 头（忽略 `;charset=...` 等参数，大小写不敏感）；查不到再退回 URL 路径后缀；两者都没有则默认 `.jpg`。Content-Type → 扩展名的对照表是 §3.4 那张表的反向映射（CLI：`processor.py` 的 `_CONTENT_TYPE_EXT`；Obsidian：`pipeline.ts` 的 `CONTENT_TYPE_EXT`），必须保持同步。选择"Content-Type 优先"是因为很多图床/CDN 的 URL 不带扩展名或扩展名不可信，而 Content-Type 头通常更可靠；之前 Obsidian 用的是子串匹配（如 `ct.includes("png")`），会被 `image/apng` 之类的类型误判，现在改成对完整 mime 值做精确匹配。

## 5. 明确允许两端不一致的行为

- **Wikilink 目标 → 文件解析。** CLI 通过遍历文件系统自建文件名索引来解析 `![[name]]`（并检测重名歧义）；Obsidian 插件则直接调用 Obsidian 自身的 `metadataCache.getFirstLinkpathDest`，其行为会受用户配置的附件文件夹影响。当 vault 中存在重名文件时，两端完全可能合理地给出不同的解析结果。这被认定为永久性的平台差异，不是需要修复的 bug——在没有重新读过本节之前，不要试图“修”某一端去对齐另一端。

## 6. 距离 v1.0 完全质量对齐还存在的待办项

（不阻塞 `1.0` 打标签；记录于此是为了不被误认成“故意如此”的设计。）

- CLI 的 `Content-Type` 查找用的是 `mimetypes.guess_type`（依赖运行平台），而不是 §3.4 中定义的静态表。
- 两个包的发布 CI 目前都不会校验两个仓库的 `PROTOCOL_VERSION` 是否一致，这一步还需要人工跨仓库核对。要自动化的话，需要两个仓库共享一个 submodule，或者在 CI 时由一方去抓取另一方的常量——目前尚未实现。

## 7. 如何声明合规

- CLI：`notepic-oss --version` 必须同时打印包版本号和协议版本号，例如 `notepic-oss 1.0.0 (protocol 1.0)`。
- Obsidian：设置页底部必须显示协议版本号，并应在插件加载时通过 `console.debug` 打一行日志，方便支持/排障时确认。
- 两个包的发布 CI 理论上都应该在涉及 §3 范围内文件改动的发布前，校验两个仓库的 `PROTOCOL_VERSION` 是否一致。
