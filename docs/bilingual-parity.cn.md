中文 | [English](./bilingual-parity.md)

# 已发布快照双语一致性

## 范围

本记录覆盖已发布的 `1.0.0-alpha.18` 快照。它只修复翻译遗漏，不导入尚未
发布的 alpha.19 协议版本，也不改变 release metadata。

## 已对齐表面

- 统一错误码 registry、RFC-0005 与 RFC-0006。
- AaaS 一致性语义与 NDP code example。
- alpha.6 至 alpha.11 roadmap 历史。
- 套件 changelog 历史与技术标识符。

## 核验

NPS-Dev 的 alpha.19 双语门禁自动发现本仓库的每个 EN/CN 文档对，并比较
heading 结构、code-fence language、技术标识符、Version header 与 Unicode 完整性。

## 结果

40 个已发布快照文档对全部对齐。套件仍为 `1.0.0-alpha.18`；未执行 alpha.19
物化或发布。
