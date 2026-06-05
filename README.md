# NotePic OSS

> Upload images referenced in Markdown, MDX, HTML, and Obsidian notes to Aliyun OSS.

[![License](https://img.shields.io/github/license/Luhui-Dev/NotePic-OSS)](LICENSE)

**Made by [@LuhuiDev](https://luhuidev.com) · Part of [LuhuiDev Toolkit](https://luhuidev.com)**

NotePic OSS 是一组面向写作者和知识库维护者的图片上传工具：把文档里引用的本地图片压缩、去重、上传到阿里云 OSS，然后把原文链接重写成公网 URL。

这个仓库是 NotePic OSS 的产品入口和文档首页。拆分后的源码、发布包和安装说明分别维护在独立仓库中。

## 项目矩阵

| 项目                 | 仓库                                                                                | 适合场景                                                     | 入口          |
| -------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------ | ------------- |
| NotePic-OSS-CLI      | [Luhui-Dev/NotePic-OSS-CLI](https://github.com/Luhui-Dev/NotePic-OSS-CLI)           | 批量处理 Markdown / MDX / HTML，适合脚本、CI、站点构建流程   | `notepic-oss` |
| NotePic-OSS-Obsidian | [Luhui-Dev/NotePic-OSS-Obsidian](https://github.com/Luhui-Dev/NotePic-OSS-Obsidian) | 在 Obsidian 当前笔记中一键上传，或通过图片管理面板选择性上传 | Obsidian 插件 |

两个实现共享同一套对象命名约定：

```text
<prefix>/<sha256[:24]>.<ext>
```

这意味着同一张图片无论先被 CLI 还是 Obsidian 插件上传，另一端都能识别为已上传内容，避免重复写入 OSS。

## 能解决什么

- 把 `![](...)`、`<img src="...">`、引用式图片定义和 Obsidian `![[...]]` 图片统一迁移到 OSS。
- 上传前压缩 JPEG、PNG、WebP，GIF / SVG / 动画图保持原样。
- 用内容哈希作为对象名，重复图片天然去重。
- 跳过代码块、行内代码、HTML 的 `script/style/pre/code` 等不该改写的区域。
- 支持自定义 Bucket 前缀和 CDN / 自定义域名。

## 快速选择

如果你要处理一批文章或把图片上传接入发布流水线，用 CLI：

```bash
git clone git@github.com:Luhui-Dev/NotePic-OSS-CLI.git
cd NotePic-OSS-CLI
pip install -e .
cp .env.example .env
notepic-oss article.md --env-file .env
```

如果你主要在 Obsidian 里写笔记，用 Obsidian 插件：

1. 到 [NotePic-OSS-Obsidian Releases](https://github.com/Luhui-Dev/NotePic-OSS-Obsidian/releases) 下载 `main.js`、`manifest.json`、`styles.css`。
2. 放入 `<Vault>/.obsidian/plugins/notepic-oss/`。
3. 在 Obsidian 的第三方插件设置中启用 NotePic OSS。

## OSS 配置约定

CLI 从环境变量读取配置；Obsidian 插件从插件设置页读取配置。字段含义保持一致。

| 配置                   | 说明                                                       |
| ---------------------- | ---------------------------------------------------------- |
| Access Key ID / Secret | 阿里云 RAM 子账号凭据                                      |
| Endpoint               | 区域 Endpoint，例如 `https://oss-cn-hangzhou.aliyuncs.com` |
| Bucket                 | 目标 Bucket 名称                                           |
| Prefix                 | 可选，对象名前缀，例如 `markdown`                          |
| Custom Domain / CDN    | 可选，用于生成最终图片 URL                                 |

建议为 NotePic OSS 创建专属 RAM 子账号，并把权限限制到目标 Bucket 的对象读写范围。不要把 AccessKey 提交到 Git。

## 仓库职责

| 仓库                                                                      | 职责                                                        |
| ------------------------------------------------------------------------- | ----------------------------------------------------------- |
| [NotePic-OSS](https://github.com/Luhui-Dev/NotePic-OSS)                   | 产品首页、统一说明、演示素材                                |
| [NotePic-OSS-CLI](https://github.com/Luhui-Dev/NotePic-OSS-CLI)           | Python CLI 源码、Python 包发布、CLI issue / release         |
| [NotePic-OSS-Obsidian](https://github.com/Luhui-Dev/NotePic-OSS-Obsidian) | Obsidian 插件源码、插件 release、社区插件提交所需根目录文件 |

## License

MIT
