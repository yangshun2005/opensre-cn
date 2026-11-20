# OpenSRE 文档

本目录包含 OpenSRE 文档的源文件，由 [Mintlify](https://mintlify.com) 提供支持。

## 本地开发

要在本地预览文档，需要安装 Mintlify CLI。

### 前置条件

- [Node.js](https://nodejs.org/)（版本 18 或更高）
- [npm](https://www.npmjs.com/)

### 安装

1. 全局安装 Mintlify CLI：

   ```bash
   npm i -g mint
   ```

2. 在本目录下启动开发服务器：

   ```bash
   mint dev
   ```

   或者，在项目根目录下运行：

   ```bash
   make docs-dev
   ```

文档将在 `http://localhost:3000` 上提供。

## 目录结构

- `docs.json`：文档配置文件，包含导航和主题设置。
- `*.mdx`：使用 MDX（带 JSX 的 Markdown）编写的文档页面。
- `images/`：静态资源，如截图和图表。
- `snippets/`：可复用的内容块。

## 贡献

1. 为你的更改创建一个新分支。
2. 编辑相关的 `.mdx` 文件，或在添加新页面时更新 `docs.json`。
3. 使用 `mint dev` 在本地预览更改。
4. 提交 Pull Request，并清晰描述你的更改。

## 资源

- [Mintlify 文档](https://mintlify.com/docs)
- [MDX 语法指南](https://mdxjs.com/docs/what-is-mdx/)