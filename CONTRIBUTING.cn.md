# 为 NPS 做贡献

> [English](CONTRIBUTING.md) | 中文版

感谢你对 Neural Protocol Suite 的关注。

## Issue 前缀

| 前缀 | 用途 |
|------|------|
| `spec:` | 规范问题与设计讨论 |
| `impl:` | 实现相关的 Bug 与修复 |
| `sdk:`  | SDK（Python / TypeScript 等）相关 |
| `docs:` | 文档改进 |

## 工作流程

1. 对任何非琐碎变更，先开一个 Issue 讨论
2. Fork 仓库并切出分支：`feature/你的功能` 或 `fix/你的修复`
3. 提交 Pull Request 并关联对应 Issue

## 规范变更

对 `spec/` 下文件的修改，需要先开讨论 Issue 再提 PR。
影响 wire format 或帧结构的规范变更必须同时升版本号。

## 代码风格

- **C# / .NET**：遵循标准 Microsoft C# 约定，启用 Nullable
- **Python**：PEP 8，必须带类型注解
- **TypeScript**：启用 strict 模式

## 许可证

提交贡献即表示你同意贡献内容按 Apache 2.0 授权。
