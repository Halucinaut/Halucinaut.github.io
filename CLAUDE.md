# Halucinaut's Lab - Claude Code 工作指南

## 项目概述

这是一个基于 Hugo + PaperMod 主题的个人技术博客/作品集网站，部署在 GitHub Pages 上。

- **网站地址**: https://halucinaut.github.io/
- **技术栈**: Hugo 静态网站生成器 + PaperMod 主题
- **部署方式**: GitHub Actions 自动部署

## Git 工作流程

### 提交前的检查

```bash
# 查看当前状态
git status

# 查看修改了哪些文件
git diff --stat
```

### 标准提交流程

```bash
# 1. 添加源文件（不包括 public/，因为那是生成的）
git add content/ layouts/ themes/ assets/ hugo.toml

# 2. 提交
git commit -m "描述修改内容"

# 3. 推送到远程
git push origin main
```

### 处理远程冲突（当别人也有提交时）

```bash
# 方法1: 使用 rebase（推荐，保持历史整洁）
git fetch origin main
git pull --rebase origin main
git push origin main

# 方法2: 如果 rebase 失败，先丢弃 public/ 的更改
git checkout -- public/
git pull --rebase origin main
git push origin main
```

### 重要注意事项

- **不要提交 `public/` 目录**: 这是 Hugo 生成的静态文件，GitHub Actions 会自动生成
- **不要提交 `.DS_Store`**: macOS 系统文件
- **不要提交 `.claude/`**: Claude Code 的工作目录

## GitHub Actions 部署流程

### 触发条件

1. **自动触发**: 每次 push 到 `main` 分支
2. **手动触发**: 在 GitHub 仓库页面的 Actions 标签页点击 "Run workflow"

### 部署步骤

1. 检出代码（包含子模块）
2. 设置 Hugo 环境（版本 0.146.0，extended）
3. 执行 `hugo --gc --minify --baseURL "https://halucinaut.github.io/"`
4. 上传 `public/` 目录作为 artifact
5. 部署到 GitHub Pages

### 查看部署状态

- 在 GitHub 仓库页面 → Actions 标签页
- 部署成功后，访问 https://halucinaut.github.io/

## 本地开发

```bash
# 启动开发服务器
hugo server -D

# 访问 http://localhost:1313/
```

## Claude Code 授权

**授权级别**: 半自动

**授权范围**: 
- ✅ 修改后**询问是否提交**，确认后自动执行 commit 和 push
- ✅ 如果遇到远程冲突，自动尝试 rebase 解决
- ✅ 提交信息会清晰描述修改内容

**操作原则**:
- 只提交源文件（content/, layouts/, assets/ 等），不提交生成的 `public/` 目录
- 不提交系统文件（.DS_Store）和临时文件（.claude/）
- 每次操作后会告知用户结果

**需要手动处理的情况**:
- 复杂的合并冲突需要人工解决
- 涉及敏感信息或破坏性变更

## 内容结构

```
content/
├── about.md              # 关于页面
├── search.md             # 搜索页面
├── posts/                # 博客文章（目前为空）
│   └── _index.md
├── projects/             # 项目展示
│   ├── _index.md
│   ├── halucinaut-skills.md
│   └── paper-reproduction-assistant.md
└── demos/                # 演示（目前为空）
    └── _index.md

layouts/                  # 自定义布局模板
├── index.html           # 主页（时间流布局）
└── projects/
    ├── list.html        # 项目列表页
    └── single.html      # 项目详情页

assets/css/extended/      # 自定义样式
└── portfolio.css
```

## 常用修改

### 添加新项目

1. 在 `content/projects/` 创建新的 `.md` 文件
2. 参考现有项目文件的 frontmatter 格式
3. 可选添加 `parent_project` 字段将其设为子项目

### 修改主页时间线

编辑 `layouts/index.html`，修改时间线筛选逻辑或样式。

### 修改样式

编辑 `assets/css/extended/portfolio.css`，自定义项目卡片、按钮等样式。
