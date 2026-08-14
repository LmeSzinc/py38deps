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

### 编译工具

编译工具在 `mingw64/bin` 目录下，禁止自行在系统环境中安装编译工具。

### git commit 要求

在修改完依赖之后，在 submodule 中执行 `git commit`，但是禁止 `git push`。

在修改完依赖之后，不要自动在 py38deps 仓库中提交来更新 submodule 引用，让用户自己确认 ci 运行正常再让用户自己 commit。

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

### 编写计划文档

如果用户要求编写计划文档，那么将计划写入到 markdown 文件 `doc/{yyyy}-{mm}-{dd}_{title}.md`，比如 `doc/2026-08-13_somethong-matters.md`

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



## 反向移植流程

1. 回退 git 历史到最新版本的 tag，因为我们需要构建的是最新版本的反向移植，不应该引入未发布的内容

2. 查找 git 历史，看看是哪些 commit 移除了旧版本的支持，回退这些修改。

3. 找到废除旧版本支持后的变更，查看变更是否引入了旧版本不支持的python语法。
4. 在 cp38 下进行测试，确认基本行为正确，并且没有不支持的语法
5. 测试通过了之后再在其余 python 版本下运行完整测试。

> 对于纯python实现的库，可以直接使用测试环境的python运行时进行测试，不需要把库安装到环境中。
>
> 对于需要编译的库，才安装到环境中。

## 本地 tox 验证（如果项目使用 tox）

如果项目使用了 tox（pyproject.toml 中有 `[tool.tox]` 或存在 `tox.ini`），**必须在本地完整模拟 CI 的 tox 运行**，不能只直接跑 pytest。直接跑 pytest 会漏掉 tox 特有的问题（依赖组解析、sdist 构建安装、coverage 命令、`base_python` 解释器解析、gh-actions 映射过滤等），反复 push 会浪费 CI 资源。

实例教训（hpack 移植，2026-08）：上游 pyproject.toml 写的是 `[tool.pytest]`（非标准键，pytest 官方是 `[tool.pytest.ini_options]`），pytest 9.x 恰好能读、pytest 8.x 读不到，导致 cp38/cp39 下 `testpaths` 失效、`bench/` 被收集、缺 pytest-benchmark 报 setup error——直接跑 `pytest tests/` 完全发现不了，只有完整 tox 模拟才暴露。

模拟方法：tox-gh-actions 会按**运行 tox 的解释器版本**匹配 gh-actions 映射，得出该 job 要运行的 env 列表，因此每个 job 用对应版本的 python 运行 tox（3.8 job 用 cp38 的 python）。

```powershell
# 1) 在对应版本环境安装 tox + tox-gh-actions（cp38 下自动解析到最后一个支持 3.8 的 tox 4.23.2）
& "E:\ProgramData\Pycharm\py38deps\envs\cp38\python.exe" -m pip install "tox>=4.23.2" "tox-gh-actions"

# 2) tox 按 env 名找解释器（py38 -> python3.8），portable python 只有 python.exe，创建硬链接并加入 PATH
New-Item -ItemType HardLink -Path "envs\cp38\python3.8.exe" -Target "envs\cp38\python.exe"
$env:PATH = "E:\ProgramData\Pycharm\py38deps\envs\cp38;$env:PATH"

# 3) 设置 GitHub Actions 环境变量（必须与运行 tox 在同一条命令内，环境变量不跨命令保留）
$env:GITHUB_ACTIONS="true"; $env:GITHUB_WORKFLOW="CI"; $env:GITHUB_JOB="tox"
$env:GITHUB_RUN_ID="1"; $env:GITHUB_REF="refs/heads/master"; $env:GITHUB_EVENT_NAME="push"
$env:GITHUB_REPOSITORY="python-hyper/hpack"; $env:GITHUB_ACTOR="test"; $env:GITHUB_SHA="deadbeef"

# 4) 完整跑该 job 的 env（等价于 CI 的 Initialize + Test 两步）
cd "E:\ProgramData\Pycharm\py38deps\repo\<DEP_NAME>"
& "E:\ProgramData\Pycharm\py38deps\envs\cp38\python.exe" -m tox --parallel auto
```

注意事项：

- `--notest` 只创建 env 并安装依赖，用于快速验证解释器查找、依赖解析、sdist 构建；完整验证必须跑 `tox --parallel auto`（等价 CI 的 `tox --parallel auto --notest` + `tox --parallel 0`）
- tox-gh-actions 只保留 gh-actions 映射中、且存在于 `env_list` 的 env（未定义的残留 env 名如 h2spec 会被静默过滤，不会报错）
- 映射里 `base_python` 指定了特定版本解释器的 env（如 packaging 的 `python3.14`），本地必须提供对应解释器（硬链接 + PATH），否则报 `could not find python interpreter matching any of the specs`——CI runner 的系统 python 未必有该版本，本地验证能提前发现这类问题
- 模拟完清理：删除硬链接与 `.tox/`、`dist/` 目录

## 版本 Cheatsheet（构建与测试工具速查）

以下版本限制基于 cp38 ~ cp314 全环境实测（hyperframe 移植时验证，2026-08），后续适配其他库时直接套用，无需再纠结版本选择。

### 核心原则

- 反向移植通常只改元数据（pyproject.toml / CI / CHANGELOG），**先检查所有源文件是否都以 `from __future__ import annotations` 开头**，是则源码一般无需回退（见下方语法速查）
- cp38 分支用旧版工具，cp39+ 分支保持上游版本要求，通过 PEP 508 环境标记（`; python_version < '3.9'`）区分
- 如果上游声明的工具版本要求高于 3.8 可解析的上限，cp38 下 `pip wheel .` / `pip install` 会直接报 `No matching distribution found`，这就是需要加环境标记的信号

### build-system：构建隔离依赖（cp38 构建 wheel 的硬约束）

| 包 | 最后支持 3.8 的版本 | 3.9+ 可用 | 说明 |
| --- | --- | --- | --- |
| setuptools | 75.3.2（75.4.0 起要求 >=3.9，83.0.0 起要求 >=3.10） | 75.4.0+ | cp38 分支必须 `<75.4` |
| wheel | 0.45.1（0.46.0 起要求 >=3.9） | 0.46.0+ | 上游若声明 wheel 依赖，cp38 分支必须 `<0.46` |

推荐写法（cp38 分支加环境标记；cp39+ 分支保留上游的版本要求即可）：

```toml
[build-system]
requires = [
  "setuptools>=68,<75.4 ; python_version < '3.9'",
  "setuptools>=82 ; python_version >= '3.9'",  # keep upstream requirement
  "wheel>=0.45.1,<0.46 ; python_version < '3.9'",
  "wheel>=0.46.3 ; python_version >= '3.9'",   # omit if upstream has no wheel dep
]
build-backend = "setuptools.build_meta"
```

### 测试依赖：pytest 生态（cp38 下安装的硬约束）

| 包 | 最后支持 3.8 的版本 | 3.9+ 可用 | 说明 |
| --- | --- | --- | --- |
| pytest | 8.3.5（8.4.0 起要求 >=3.9，9.0.0 起要求 >=3.10） | 8.4.0+ | 约束 `<9` 时 cp38 自动解析到 8.3.5，无需环境标记 |
| pytest-cov | 5.0.0（6.0.0 起要求 >=3.9） | 6.0.0+ | cp38 分支必须 `<6` |
| pytest-xdist | 3.6.x（3.7.0 起要求 >=3.9） | 3.7.0+ | 约束 `<4` 时 cp38 自动解析到 3.6.x，无需环境标记 |
| coverage | 7.6.1（7.7.0 起要求 >=3.9，作为 pytest-cov 间接依赖） | 7.7.0+ | pip 自动解析，无需单独处理 |

推荐写法：

```toml
testing = [
  "pytest>=8.3.3,<9",
  "pytest-cov>=6.0.0,<7 ; python_version >= '3.9'",
  "pytest-cov>=5.0.0,<6 ; python_version < '3.9'",
  "pytest-xdist>=3.6.1,<4",
]
```

### 源码语法速查：哪些写法在 3.8 下安全

只要源文件都以 `from __future__ import annotations` 开头：

- **安全**：注解中的 `int | None`、`list[X]`、`dict[K, V]`、`tuple[A, B]`、`type[X]` 等（注解被字符串化，不参与运行时求值）
- **安全**：函数体内的局部变量注解（如 `x: set[str] = set()`）——CPython 中局部变量注解从不求值（cp38 ~ cp314 行为一致）
- **需要回退**：运行时真正求值的写法，例如赋值表达式右侧的 `set[str]`、`isinstance(x, int | str)`、模块级类型别名 `X = int | str` 等（3.8 下报 `TypeError: 'type' object is not subscriptable`）

### 验证命令（cp38）

```powershell
# 1) 验证 build-system 在 cp38 下可解析（构建隔离）
cd repo/<DEP_NAME>
& "E:\ProgramData\Pycharm\py38deps\envs\cp38\python.exe" -m pip wheel . --no-deps -w $env:TEMP\hwtest

# 2) 验证测试依赖在 cp38 下可安装
& "E:\ProgramData\Pycharm\py38deps\envs\cp38\python.exe" -m pip install "pytest>=8.3.3,<9" "pytest-cov>=5.0.0,<6" "pytest-xdist>=3.6.1,<4"

# 3) 运行测试（纯 Python 库；src layout 时需要 PYTHONPATH）
$env:PYTHONPATH="src"
& "E:\ProgramData\Pycharm\py38deps\envs\cp38\python.exe" -m pytest tests/
```

### 其他

- ruff 的 `target-version` 改为 `"py38"`（ruff 是二进制工具，与 Python 版本无关，无需环境标记）
- tox 的 `env_list` / `gh-actions` 映射增加 `py38`，CI 矩阵增加 `"3.8"`
- twine 5.x 不支持 Metadata-Version 2.4（新版 setuptools 77+ 构建产物默认为 2.4，`twine check` 会报 `InvalidDistribution: Metadata is missing required fields: Name, Version`）。packaging 依赖建议：`"twine>=6.1.0,<6.2 ; python_version < '3.9'"` + `"twine>=6.2.0,<7 ; python_version >= '3.9'"`（twine 6.2 起要求 >=3.9）
- ruff 新版会把 preview 规则转正，导致 `lint.select = ["ALL"]` 下旧版源码的 lint 失败（实例：PLC0415 `import` should be at the top-level，v6.1.0 时代不报、ruff 0.16 起报）。应对：跟随上游的 `# noqa` 修复，而不是限制 ruff 版本
- 以最新发布版 tag 为基线，不要引入未发布 commit（见"反向移植流程"第 1 条）

## CI 配置：手动触发与产物上传

### fail-fast

矩阵 job 中某个版本失败时，默认（`fail-fast: true`）会取消其余还在运行的 job，一次只能看到第一个失败。移植后应设置 `fail-fast: false`，让所有版本跑完并一次性暴露所有问题，避免反复触发 CI：

```yaml
    strategy:
      fail-fast: false
      matrix:
        python-version:
        - "3.8"
        - "3.9"
        ...
```

### 手动触发与产物上传

官方仓库通常配置了自动发布流程（打 tag 自动构建并发布到 PyPI / GitHub Release），而我们的二次开发仓库无法自动发布，因此移植后的 CI 需要额外做两件事：

1. **手动触发按钮**：CI 增加 `workflow_dispatch` 触发事件，GitHub Actions 页面出现 "Run workflow" 按钮，可随时手动运行 CI
2. **产物上传**：构建产物（wheel / sdist）默认不会暴露为下载，需要 `upload-artifact` 上传后才会出现在 Actions 运行页面的 Artifacts 区域（保留 90 天）

```yaml
on:
  push:
    branches: ["master"]
  pull_request:
    branches: ["master"]
  workflow_dispatch:   # manual trigger button on GitHub Actions page
```

上传产物时注意：矩阵 job 中 tox-gh-actions 会把 packaging env 绑定到特定 python 版本（如 hyperframe 的 `3.9: py39, h2spec, lint, docs, packaging`），因此 upload step 需要加对应的 `if` 条件，避免其他 job 上传空产物：

```yaml
    - name: Upload dist
      if: matrix.python-version == '3.9'   # packaging env runs in this job only
      uses: actions/upload-artifact@v4
      with:
        name: dist
        path: dist/
```

注意事项：

- `workflow_dispatch` 需要 push 到 GitHub 后按钮才会出现
- 版本参考：msgspec 用 `upload-artifact@v5`，python-zstandard 用 `@v4.6.2`（pin SHA），hyperframe 用 `@v4`
- 纯 Python 库的 wheel 为 `py3-none-any`，一个产物即可覆盖 cp38 ~ cp314，无需按平台分别构建



