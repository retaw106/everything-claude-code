---
name: nextjs-turbopack
description: Next.js 16+ 和 Turbopack — 增量打包、文件系统缓存、开发速度，以及何时使用 Turbopack vs webpack。
origin: ECC
---

# Next.js 和 Turbopack

Next.js 16+ 默认在本地开发中使用 Turbopack：一个用 Rust 编写的增量打包器，可显著加快开发启动和热更新速度。

## 何时使用

- **Turbopack（默认开发）**：用于日常开发。更快的冷启动和 HMR，特别是在大型应用中。
- **Webpack（传统开发）**：仅当遇到 Turbopack bug 或在开发中依赖仅支持 webpack 的插件时使用。使用 `--webpack` 禁用（或根据你的 Next.js 版本使用 `--no-turbopack`；查看你版本的文档）。
- **生产环境**：生产构建行为（`next build`）根据 Next.js 版本可能使用 Turbopack 或 webpack；查看官方 Next.js 文档了解你的版本。

使用时机：开发或调试 Next.js 16+ 应用、诊断慢开发启动或 HMR，或优化生产包。

## 工作原理

- **Turbopack**：Next.js 开发的增量打包器。使用文件系统缓存，使重启更快（例如，大型项目上快 5-14 倍）。
- **开发默认**：从 Next.js 16 开始，`next dev` 使用 Turbopack 运行，除非禁用。
- **文件系统缓存**：重启重用之前的工作；缓存通常在 `.next` 下；基本使用无需额外配置。
- **Bundle Analyzer（Next.js 16.1+）**：实验性 Bundle Analyzer 用于检查输出并发现重度依赖；通过配置或实验性标志启用（查看你的版本的 Next.js 文档）。

## 示例

### 命令

```bash
next dev
next build
next start
```

### 用法

运行 `next dev` 进行使用 Turbopack 的本地开发。使用 Bundle Analyzer（查看 Next.js 文档）优化代码分割并修剪大型依赖。尽可能使用 App Router 和服务器组件。

## 最佳实践

- 保持使用最新的 Next.js 16.x 以获得稳定的 Turbopack 和缓存行为。
- 如果开发缓慢，确保你在使用 Turbopack（默认）并且缓存没有被不必要地清除。
- 对于生产包大小问题，使用你版本的官方 Next.js 包分析工具。
