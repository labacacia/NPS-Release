中文 | [English](./bilingual-parity.md)

# 发行候选双语一致性

## 范围

本记录覆盖从 NPS-Dev 物化的、经审查 alpha.19 规范候选。在最后的协调版本
升级之前，release metadata 仍保持在已发布的 `1.0.0-alpha.18` 基线。

## 已对齐表面

- 统一错误码 registry、RFC-0005 与 RFC-0006。
- AaaS 一致性语义与 NDP code example。
- alpha.6 至 alpha.11 roadmap 历史。
- 套件 changelog 历史与技术标识符。

## 核验

NPS-Dev 的 alpha.19 双语门禁自动发现本仓库的每个 EN/CN 文档对，并比较
heading 结构、code-fence language、技术标识符、Version header 与 Unicode 完整性。

## 结果

46 个发行仓文档对全部对齐。alpha.19 规范已物化供审查；套件仍为
`1.0.0-alpha.18`，且未执行 alpha.19 tag、package、image、registry upload 或 release publication。
