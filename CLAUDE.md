## 项目目标

随着低版本 python 逐渐 end of life，许多 python 依赖逐渐提高了最低版本要求，`py38deps` 将他们反向移植到低版本 python 上，尽量支持到 python 3.8。

## 文件结构

`README.md` 表格中记录了已经移植完成的依赖

`repo/{DEP_NAME}` 是各个依赖的 git 仓库。remote origin 指向我们的二次开发仓库，remote upstream 指向官方仓库

`wheel/{DEP_NAME}/{DEP_VERSION}` 目录存放编译好的 wheel 文件

## 对话要求

- 在对话回复中使用中文
- 在代码注释中使用英文
- 在修改已有代码的时候，不要对无关的部分进行修改

##  测试环境

本地有 cp38 ~ cp314 的测试环境，python 运行时提取自 uv 打包好的 portable python

```
envs/cp38/python.exe
envs/cp39/python.exe
envs/cp310/python.exe
envs/cp311/python.exe
envs/cp312/python.exe
envs/cp313/python.exe
envs/cp313t/python.exe
envs/cp314/python.exe
envs/cp314t/python.exe
```

### 测试要求

在运行测试的时候，先在 cp38 下进行测试。测试通过了之后再进行其他版本的测试

### 优先使用codewhale内置工具

优先使用 codewhale 内置工具而不是调用外部命令行。如果遇到权限问题再去尝试使用命令行访问。如果在任务中多次需要访问一个外部路径，提示用户可以执行 `/trust add {PATH}` 添加信任，这样下一次可以使用内置工具进行访问。

内置工具是使用 rust 编写的，比命令行更快。内置工具会过滤类似 `node_modules`，`build` 等目录，避免搜索无关内容消耗大量时间。

- 在查找文件的时候优先使用 `file_search` 工具或者 `list_dir` 工具来查找文件，而不是使用 `find` 命令或者 `Get-ChildItem` 命令。

- 在查找文件内容优先使用 `grep_files` 工具来查找，而不是使用 `grep` 命令或者 `Select-String` 命令。
- 优先使用内置的 git 工具来查看仓库状态，而不是执行 git 命令。
- 在批量编辑文件的时候优先使用内置的 `edit` / `fim_edit` / `apply_patch` 工具进行编辑，而不是编写临时脚本去执行替换。因为内置工具有丰富的模糊匹配与约束，自行编写替换代码很容易遇到 文件编码问题、换行符匹配问题、行号/内容匹配问题、特殊符号转义问题。不要怕多次调用工具进行单次编辑很麻烦，遇到编码问题转义问题反复修改临时脚本更麻烦。

### 运行python程序

使用测试环境中的 python 解释器来执行脚本和运行测试，比如：

```bash
# run a specific file "module/config/gen.py"
"<path_to_python>" -m module.config.gen
# run test file "tests/base/test_servertime.py"
"<path_to_python>" -m pytest tests/base/test_servertime.py
```

不要直接使用 `python` 命令，因为项目有单独配好的虚拟环境，不使用全局 python 环境。

在运行项目内文件的时候不要直接执行 `python module/config/gen.py` 而是使用 `python -m module.config.gen` 作为模块运行，这样运行路径会在项目根目录。

### PowerShell兼容提示

Windows 运行环境下的 PowerShell 版本可能很低，执行这样的命令会报错，因为不支持 `&&`

```powershell
cd "E:\xxx" && "<path_to_python>" -m pytest ...
```

改用 `;` 分隔命令，并且增加 `&` 来表示调用被引号包裹的路径

```powershell
cd "E:\xxx"; & "<path_to_python>" -m pytest ...
```

因为 codewhale （也包括其他 AI agent）执行命令时会创建新的命令行环境，所以回退或者不回退路径都无所谓。

### 临时运行python代码提示

如果你希望临时运行一段简单的 python 测试代码，不要使用 `python -c` 去运行，因为编写转义非常容易出错，使用 stdin 去输入代码。

在 Windows 上这样运行：

注意，即便运行的代码只有一行也必须写成多行，因为 `@'` 和 `'@` 标记需要在行开头

```powershell
cd "E:\xxx";
@'
print("hello world")
import json
data = {"key": "value"}
print(json.dumps(data))
'@ | & "<path_to_python>"
```

在 Linux 上这样运行，通过 heredoc 传入：

注意，必须给 heredoc 定界符加引号（`<< 'EOF'`），否则 `$` 和反引号会被 shell 展开。

```bash
cd /path/to/dir
python << 'EOF'
print("hello world")
import json
data = {"key": "value"}
print(json.dumps(data))
EOF
```

如果需要临时执行的代码过于复杂，或者需要 stdin，那么在项目根目录编写临时文件来运行它。

## 创建移植库流程

在创建新的移植库之前，需要知道：

- 二次开发仓库的地址（`ORIGIN_URL`），例如：`git@github.com:LmeSzinc/python-zstandard.git` ，二次开发地址需要使用 git 协议，如果是 HTTP 地址需要转换为 git 协议地址。
- 官方仓库地址（`UPSTREAM_URL`），例如：`https://github.com/indygreg/python-zstandard`，使用 HTTP 地址
- 依赖名称（`DEP_NAME`），例如 `python-zstandard`。注意，这里使用的是库名称（Distribution Name），也就是在 Pypi 中注册的名字，用于 `pip install ...` ；而不是导入名称（Import Name）。对于 `python-zstandard` 而言，库名称叫 `python-zstandard`，而导入名称是 `zstandard`。

### 步骤1：添加submodule

```bash
git submodule add <ORIGIN_URL> repo/<DEP_NAME>
```

### 步骤 2：配置子模块的 Remote (Origin 与 Upstream)

进入子模块内部，配置远程仓库关联：

```bash
cd repo/<DEP_NAME>

# 确认 origin 指向二次开发仓库
git remote set-url origin <ORIGIN_URL>

# 添加 upstream 指向官方仓库
git remote add upstream <UPSTREAM_URL>

# 拉取 upstream 的所有分支与 tag 信息
git fetch upstream --tags
```

### 步骤 3：切换到二次开发分支 (防止游离态/Detached HEAD)

如果你直接进入子模块目录修改代码并 git commit，代码可能会提交到一个“游离分支”，导致丢代码。**必须**显式切换到二次开发分支，严禁在临时提交上工作。

`BACKPORT_BRANCH` 一般是 `main` 分支或者 `master` 分支

```bash
# 从 upstream 的默认分支（或指定 tag）切出新的 backport 分支
# 注意：如果 origin 远程已经存在该分支，则直接 checkout
git checkout <BACKPORT_BRANCH>
```

