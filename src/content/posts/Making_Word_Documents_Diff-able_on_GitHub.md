---
title: '让 Word 文档在 Git 中也能“看”得见差异'
pubDate: '2025-11-24'
category: 'other'
---


用 Git 管理 `.docx` 文件却有一个巨大的痛点：**Word 文档是压缩包（二进制文件）**。

这意味着，当你修改了文档中的一段话并提交后，Git 无法告诉你具体改了哪里，只会冷冰冰地显示 `Binary file modified`。GitHub 上的 diff 页面也是一片空白，无法进行版本对比。

本文介绍一种基于 Windows 环境的轻量级解决方案：**利用 Git Hook 和 Pandoc，为每一个 .docx 自动生成同名的 .md 纯文本“影子文件”，随版本提交自动更新**。这样，我们就能在 GitHub 上直观地看到文档的修改历史了。

## 准备工作

本文基于 **Windows** 环境，为了降低门槛，我们将使用 GUI 工具而非纯命令行。

### 1. 准备仓库
1.  登录 Github.com。
2.  点击右上角 **Create a new repository**。
3.  输入仓库名称，建议选择 **Private**（私有）。
4.  如果你不熟悉命令行初始化，请务必勾选 **Add a README file**。
5.  点击 Create repository。

### 2. 选择 Git GUI 工具
推荐使用 [Github Desktop](https://desktop.github.com/)，它与 GitHub 账号集成得最好，操作直观。当然，其他工具（如 SourceTree）也可以，前提是你已经配置好了 Git 环境。

### 3. 克隆仓库到本地
1.  打开 Github Desktop，登录你的账号。
2.  点击 **Clone a repository**，在列表中找到刚才创建的仓库。
3.  选择本地路径，点击 Clone。记住这个文件夹的位置。

### 4. 配置转换工具 Pandoc
我们需要 Pandoc 将 Word 转为 Markdown。为了方便迁移，这里使用便携版（Portable），无需安装。

1.  下载 Pandoc：访问 [Pandoc Releases](https://github.com/jgm/pandoc/releases)，下载 `windows-x86_64.zip` 版本。
2.  在刚才克隆的仓库根目录下，新建一个名为 `tools` 的文件夹。
3.  解压下载的压缩包，将其中的 `pandoc.exe` 复制到这个 `tools` 文件夹中。
4.  **配置忽略文件**：在仓库根目录新建一个名为 `.gitignore` 的文本文件，写入以下内容：
    ```text
    *.exe
    ```
    *这样做是为了防止 git 将 pandoc.exe 这个工具软件本身上传到仓库中。*

## 核心步骤：配置 Pre-commit Hook

我们将编写一个脚本，告诉 Git：“在每次提交之前，请帮我检查有没有 Word 文档被修改，如果有，把它转换成 Markdown 顺便一起提交了。”

1.  打开仓库文件夹，设置 Windows 显示“隐藏的项目”，找到隐藏文件夹 `.git`。
2.  进入 `.git` -> `hooks` 文件夹。
3.  新建一个文本文件，重命名为 `pre-commit` (**注意：一定要删除 .txt 后缀名，该文件没有后缀**)。
4.  用记事本或代码编辑器打开该文件，粘贴以下代码：

```bash
#!/bin/sh
# pre-commit hook: convert staged .docx files to .md before commit

# 遍历本次提交中已暂存(staged)的 docx 文件
# diff-filter=ACM 确保只处理 添加(Added)、复制(Copied)、修改(Modified) 的文件
for file in $(git diff --cached --name-only --diff-filter=ACM | grep '\.docx$'); do
    # 定义生成的 md 文件名，例如 test.docx 会生成 test.for_diff.md
    mdfile="${file%.docx}.for_diff.md"
    
    echo "Converting $file -> $mdfile"
    
    # 调用 tools 目录下的 pandoc 进行转换
    # 注意：这里假设你在 Windows Git Bash 环境下运行，路径处理已做适配
    tools\\pandoc "$file" -t markdown -o "$mdfile"
    
    # 将生成的 md 文件自动加入到本次 commit 中
    git add "$mdfile"
done
```

5.  保存文件。

**现在，一切就绪了！** 
你可以试着在仓库里放一个 `.docx` 文件，写点东西，然后在 GitHub Desktop 中提交。你会发现，除了 Word 文档外，Git 还自动帮你提交了一个 `.for_diff.md` 文件。当你在 GitHub 网页端查看提交记录时，点开这个 md 文件，就能清晰地看到所有的文字变动。

---

## Bonus：本地查看差异

上面的方法主要为了解决“在 GitHub 网页上无法看差异”的问题。如果你想在本地的 GitHub Desktop 软件里直接点击 `.docx` 就看到文字对比，可以进行以下额外配置。

1.  进入 `.git` 文件夹，找到 `config` 文件，用记事本打开。
2.  在文件末尾添加以下内容：

```ini
[diff]
    textconv=tools/pandoc.exe --to=markdown
```

**效果：**
保存后，在 GitHub Desktop 的 Changes 栏中点击 `.docx` 文件，原本的提示会消失，取而代之的是转换后的 Markdown 文本对比。

*注：这个 Bonus 配置仅对本地当前仓库生效，不会上传到 GitHub，仅供自己本地查看方便使用。*

---

### 总结

通过引入一个简单的脚本和 Pandoc，我们成功打破了二进制文件版本控制的黑盒。现在，你既保留了 Word 强大的排版功能，又享受了 Git 带来的清晰的版本回溯能力。