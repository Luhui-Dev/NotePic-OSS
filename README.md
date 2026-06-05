# NotePic OSS

> Upload images referenced in Markdown, MDX, HTML, and Obsidian notes to Aliyun OSS.

[![License](https://img.shields.io/github/license/Luhui-Dev/NotePic-OSS)](LICENSE)

**Made by [@LuhuiDev](https://luhuidev.com) · Part of [LuhuiDev Toolkit](https://luhuidev.com)**

This repository is the product home for NotePic OSS. Source code and releases live in the dedicated project repositories:

| Project | Repository | Purpose |
|---|---|---|
| NotePic-OSS-CLI | [Luhui-Dev/NotePic-OSS-CLI](https://github.com/Luhui-Dev/NotePic-OSS-CLI) | Batch-process Markdown / MDX / HTML files from the command line. |
| NotePic-OSS-Obsidian | [Luhui-Dev/NotePic-OSS-Obsidian](https://github.com/Luhui-Dev/NotePic-OSS-Obsidian) | Upload and rewrite images from inside Obsidian. |

Both projects use the same OSS object naming convention:

```text
<prefix>/<sha256[:24]>.<ext>
```

That means images uploaded by the CLI can be recognized by the Obsidian plugin, and images uploaded by the plugin will not be duplicated by the CLI.

## Install

### CLI

```bash
git clone git@github.com:Luhui-Dev/NotePic-OSS-CLI.git
cd NotePic-OSS-CLI
pip install -e .
cp .env.example .env
notepic-oss article.md --env-file .env
```

See [NotePic-OSS-CLI](https://github.com/Luhui-Dev/NotePic-OSS-CLI#readme) for full usage.

### Obsidian Plugin

Use [NotePic-OSS-Obsidian](https://github.com/Luhui-Dev/NotePic-OSS-Obsidian) for manual installation, GitHub releases, and Obsidian community plugin submission.

## Repository Layout

```text
NotePic-OSS/
├── docs/
├── LICENSE
└── README.md
```

## Release Ownership

| Deliverable | Repository | Tag pattern |
|---|---|---|
| CLI | [NotePic-OSS-CLI](https://github.com/Luhui-Dev/NotePic-OSS-CLI) | `v0.2.0` |
| Obsidian plugin | [NotePic-OSS-Obsidian](https://github.com/Luhui-Dev/NotePic-OSS-Obsidian) | `1.1.0` |

The Obsidian plugin repository keeps `manifest.json`, `README.md`, and `LICENSE` at the repository root so it can be submitted to the official Obsidian community plugin directory.

## License

MIT
