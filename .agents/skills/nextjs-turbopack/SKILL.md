---
name: nextjs-turbopack
description: Next.js 16+ 和 Turbopack — 增量打包、文件系统缓存、开发速度，以及何时使用 Turbopack vs webpack。
origin: ECC
---

# Next.js 和 Turbopack

Next.js 16+ 默认在本地开发中使用 Turbopack：一个用 Rust 编写的增量打包器，显著加快开发启动和热更新速度。

## 何时使用

- **Turbopack（默认开发）**：用于日常开发。更快的冷启动和 HMR，尤其在大型应用中。
- **Webpack（遗留开发）**：仅在遇到 Turbopack bug 或在开发中依赖仅限 webpack 的插件时使用。用 `--webpack` 禁用（或 `--no-turbopack`，取决于你的 Next.js 版本；查看你版本的文档）。
- **生产环境**：生产构建行为（`next build`）可能使用 Turbopack 或 webpack，取决于 Next.js 版本；查看你的版本的官方 Next.js 文档。

适用于：开发或调试 Next.js 16+ 应用、诊断慢开发启动或 HMR，或优化生产包。

## 工作原理

- **Turbopack**：Next.js 开发的增量打包器。使用文件系统缓存使重启快得多（如大型项目上 5-14 倍）。
- **开发中默认**：从 Next.js 16 起，`next dev` 默认使用 Turbopack 运行，除非禁用。
- **文件系统缓存**：重启复用之前的工作；缓存通常在 `.next` 下；基本使用无需额外配置。
- **Bundle Analyzer（Next.js 16.1+）**：实验性 Bundle Analyzer 用于检查输出和发现重依赖；通过配置或实验性标志启用（查看你版本的 Next.js 文档）。

## 示例

### 命令

```bash
next dev
next build
next start
```

### 使用

运行 `next dev` 进行本地开发。使用 Bundle Analyzer（查看 Next.js 文档）优化代码分割和修剪大型依赖。尽可能使用 App Router 和服务端组件。

## 最佳实践

- 保持使用最新的 Next.js 16.x 以获得稳定的 Turbopack 和缓存行为。
- 如果开发慢，确保你使用 Turbopack（默认）且缓存没有被不必要地清除。
- 对于生产包大小问题，使用你版本的官方 Next.js 包分析工具。
