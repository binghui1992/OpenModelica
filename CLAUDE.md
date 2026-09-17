# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 本仓库是什么

OpenModelica 是一个基于 Modelica 的建模与仿真环境。本检出是**超项目(superproject)**,
位于标签 **v1.13.0**(分支 `li_v1.13.0`,约 2018 年代的版本)。它自身几乎不含代码:
真正的源码都在 git 子模块(submodule)里,每个子模块都是独立仓库,有独立的历史和 PR。

| 子模块 | 内容 |
|---|---|
| `OMCompiler` | 编译器(`omc`),用 MetaModelica 编写;仿真运行时;第三方依赖 |
| `testsuite` | 回归测试套件(`rtest`、`partest`、`difftool`、参考结果) |
| `libraries` | Modelica 标准库等 OMLibraries 的构建脚本与补丁 |
| `OMEdit` / `OMPlot` / `OMShell` / `OMNotebook` / `OMOptim` | Qt/C++ 图形客户端与工具 |
| `OMSimulator` | FMI/SSP 联合仿真工具 |
| `doc` | Sphinx 用户指南(`doc/UsersGuide/source/*.rst`)、tex 文档、man 手册 |

**在子模块里改文件,就意味着在子模块自己的仓库里提交**,然后更新超项目里的指针。
在超项目根目录执行 `git status` 只会显示子模块指针变化(` M OMCompiler`),
永远不会显示具体文件。不要指望单仓库的工作流。

## 环境与构建

本树已经完成配置和构建,实际使用的配置是:

```bash
./configure CC=gcc CXX=g++ CFLAGS=-g -O0 CXXFLAGS=-g -O0 QMAKE=/usr/bin/qmake-qt5 --prefix=/home/libinghui/om-install
```

- 本树的构建输出目录 / `OPENMODELICAHOME`:`/home/libinghui/code/OpenModelica/build`
- 安装前缀:`/home/libinghui/om-install`(此前已存在一次完整安装)
- 可用的编译器二进制:`build/bin/omc`(版本 `OMCompiler v1.13.0`)

顶层目标(在超项目根目录执行;`Makefile.in` 会分派进各子模块):

```bash
make omc                 # 构建 OMCompiler(编译器)
make omlibrary-core      # 把测试库安装到 build/lib/omlibrary
make omedit / omplot / omnotebook / omshell / omsimulator
make omc-diff            # 构建 testsuite/difftool -> build/bin/omc-diff
make ReferenceFiles      # 解压 testsuite/ReferenceFiles/*.mat.xz
make -j8                 # 构建 configure 启用的一切(见 configure.ac ALL_TARGETS)
```

端到端冒烟测试(编译一个模型并跑仿真,约 0.5s):

```bash
cd /tmp && printf 'loadString("model M Real x(start=1); equation der(x) = -x; end M;"); getErrorString();\nsimulate(M); getErrorString();\n' > t.mos
/home/libinghui/code/OpenModelica/build/bin/omc t.mos
```

Linux 下,`omc` 和图形客户端通过 RPATH 定位运行时库——从 `build/bin` 运行它们
**不需要**设置 `OPENMODELICAHOME`。测试套件是例外(见"测试"一节)。

## 编译器开发循环(自举/bootstrap)

`OMCompiler/Compiler/` 中的编译器用 **MetaModelica**(`.mo` 文件)编写,并且**自己编译自己**——
不存在独立的 MetaModelica 编译器。构建新编译器需要一个已有的 `omc`:

```bash
make -C OMCompiler/Compiler/boot          # 修改 .mo 文件后的增量重建
make -C OMCompiler/Compiler/boot interfaces   # 只重新生成包的接口
```

只有发生变化的包会被重新生成,所以这是常规的"改完就重编"循环。如果没有可用的 `omc`
(或交叉编译时),构建会退化为 `bootstrap-from-tarball`:从预生成的 C 代码
`OMCompiler/Compiler/boot/bootstrap-sources.tar.xz` 出发的三阶段自举
(阶段由 `-DOMC_BOOTSTRAPPING_STAGE_1/2` 加上 `Compiler/Stubs/` 里的桩实现来区分)。

生成的 C 代码落在 `OMCompiler/Compiler/boot/build/`(约 4400 个文件)。
**绝不要手工编辑 `boot/build/` 下的任何东西**——它们会被重新生成,包括 `_main.c`
(由 `Compiler/boot/GenerateEntryPoint.mos` 生成)。

### 编译器架构(按流水线顺序)

`OMCompiler/Compiler/` 按编译阶段组织,包注解 `__OpenModelica_Interface` 标记包所属的阶段:

- `Main/Main.mo` — 入口(`Main.main`),分发交互模式与文件翻译。
- `FrontEnd/` — 前端:解析(`Parser.mo`)、AST(`Absyn.mo`)、实例化(`Inst*.mo`、`Static.mo`、
  `Lookup.mo`)、DAE、`Ceval*.mo` 常量求值。`NFFrontEnd/` 是较新的替代前端;
  `FFrontEnd/` 是早期 WIP。在这个版本里三者并存。
- `BackEnd/` — 后端:方程-变量匹配、指标约简、撕裂(tearing)、符号雅可比
  (`BackendDAE*.mo`、`Tearing.mo`、`Matching.mo`)。
- `SimCode/` — 把后端 DAE 降级为 C 代码生成器的输入。
- `Template/` — **Susan** 代码生成器。`.tpl` 模板(`CodegenC.tpl`、`CodegenFMU.tpl` 等)
  先由 Susan 编译成 `.mo` 文件,再编译进 `omc` 本身。修改 `.tpl` 需要重新生成模板源码,
  不只是重编译。
- `Script/` — 脚本 API(`CevalScript.mo`:`simulate`、`loadFile` 等)。
- `Util/Error.mo` — **所有**错误/警告消息。新增消息:添加 `ErrorID`、在 `errorTable`
  里添加条目(严重级别 + 带 `%s` 的消息模板),再用 `addMessage`/`addSourceMessage` 发出。
- `Util/Flags.mo` — 编译器标志/调试标志。
- `Compiler/runtime/` — `omc` 自身的 C/C++ 支撑代码(错误渲染、CORBA/socket 服务器)。
- `SimulationRuntime/c/` — 生成仿真可执行文件所链接的 C 运行时
  (`libSimulationRuntimeC.so`、`libOpenModelicaRuntimeC.so`、`libOpenModelicaFMIRuntimeC.so`)。
  `SimulationRuntime/cpp/` 是可选的 C++ 运行时(`./configure --with-cppruntime`)。

## 测试

测试套件是独立子模块,有自己的约定。**前置条件**(默认不构建;缺了它们每个测试都会报错):

```bash
make omc-diff          # 构建 testsuite/difftool -> build/bin/omc-diff
make ReferenceFiles    # 解压 .mat 参考结果
```

直接运行单个测试(最快的反馈循环):

```bash
testsuite/partest/runtest.pl testsuite/openmodelica/parser/DotName.mos
```

输出是一个彩色标记:**绿色 = 通过,红色 = 失败**。注意 `runtest.pl` 即使测试失败也退出码为 0——
看颜色,别看退出码。

用并行运行器跑一组测试(路径相对于 `testsuite/`):

```bash
cd testsuite/partest
printf './openmodelica/parser/DotName.mos\n' > /tmp/one.txt
./runtests.pl -file=/tmp/one.txt -nocolour
```

`runtests.pl` 其他标志:`-jN`(线程数)、`-f`(快速子集——跳过 libraries、bootstrapping、
metamodelica、cppruntime、tearing)、`-failing`(跑已知失败的测试)、`-nocpp`、
`-cppruntime`、`-with-xml`(写 `result.xml`,CI 用它)、`-b`(重设基线)。
这个版本**没有 `--match` 标志**——用 `-file=` 划分子集。退出码:**0 = 全部通过,
7 = 有失败**(CI 两者都接受)。

就地重写测试的期望输出,直接使用底层 `rtest`:

```bash
cd testsuite/flattening/modelica/algorithms-functions
../../../rtest -v Algorithm1.mo      # 详细模式,跑单个测试
../../../rtest -b Algorithm1.mo      # 把实际输出写回测试的 // Result: 块
```

### 测试的定义方式

- 测试目录有一个 `Makefile`(模板:`testsuite/Makefile_sample.txt`),列出
  `TESTFILES` 和 `FAILINGTESTFILES`。`rtest`/`partest` 靠解析它们发现测试。
- 测试是 `TestName.mo`(展平)或 `TestName.mos`(脚本)。**期望输出内嵌在测试文件里**,
  位于 `// Result:` 与 `// endResult` 之间,每行以 `// ` 开头。
- 头部指令:`// status: correct|incorrect`、`// cflags:`、`// setup_command:`、
  `// teardown_command:`、`// env:`、`// stack_size:`、`// partest-link:`。
- 测试目录里新增的非 `.mo`/`.mos` 文件必须列进 Makefile 的 `DEPENDENCIES`。
- 库仿真测试则对比 `testsuite/ReferenceFiles/<lib>/<Fully.Qualified.Model.Name>.mat`
  里的 `.mat` 文件(以 `.mat.xz` 压缩存储)。

### 坑

- `testsuite/rtest` 从当前目录推导环境:必须在 `testsuite/` 的子目录里运行,
  并且硬编码 **`OPENMODELICAHOME=<repo>/build`**。所以测试套件需要 `build/lib/omlibrary`
  存在——见下一节。
- 失败日志写到 `/tmp/omc-rtest-$USER/<testdir>/`(沙箱目录会被清理)。
- 编译器自测在 `testsuite/openmodelica/`——尤其是 `bootstrapping/`(编译器数据结构的
  单元测试,加载整个 `OMCompiler/Compiler/**`)和 `parser/`。
- 老式串行运行器仍然存在:`make -C testsuite test`(`fast`、`simulation` 等目标)。

## 代码风格 / 检查门禁

CI 跑下面这些,任何一项命中都会让构建失败(`Jenkinsfile` 第 56 行):

```bash
make -f Makefile.in -j$(nproc) --output-sync bom-error utf8-error thumbsdb-error spellcheck
```

也可用(目前在 CI 里被注释掉,但评审会执行):`trailing-whitespace-error`、
`tab-error`。`make fix-whitespace` 修复全树的行尾空格和制表符。

`./configure` 会安装 pre-commit 钩子(`common/pre-commit.sh`),自动去掉行尾空格。
提交消息有 50 字符摘要 / 72 字符正文行长的检查——见 `CONTRIBUTING.md`,
它还禁止二进制文件、生成代码和调试代码的反复增删。

## 本工作树的已知问题

`build/lib/omlibrary` 不存在,所以**所有依赖 MSL 的测试都会失败**,而纯编译器测试通过。
`testsuite/rtest` 强制 `OPENMODELICAHOME=<repo>/build`,而该路径下没有库:

```
Error: Failed to load package Modelica (3.2.1) using MODELICAPATH .../build/lib/omlibrary:...
```

编译器本身是好的——运行 `build/bin/omc` 时**不要**覆盖 `OPENMODELICAHOME`,
它会回退到配置的前缀 `/home/libinghui/om-install`,那里装了 MSL,加载正常。
修复方式:要么 `make omlibrary-core`(需要联网:把 MSL 克隆到 `libraries/git`,
该目录尚不存在),要么把构建目录指向现有安装:

```bash
ln -s /home/libinghui/om-install/lib/omlibrary /home/libinghui/code/OpenModelica/build/lib/omlibrary
```

## 调试 `omc`

`omc` 自带约 200 个调试标志(`omc --help=debug` 列出全部)。最常用的:

| 标志 | 作用 |
|---|---|
| `-d=failtrace` | `matchcontinue` 分支失败时打印回溯 |
| `-d=bltdump` | 输出指标约简 / BLT 信息 |
| `-d=dumpdaelow` | 输出进入后端时的方程系统 |
| `-d=optdaedump` | 输出优化模块的结果 |
| `-d=tearingdump` | 输出撕裂信息 |
| `-d=backenddaeinfo` | 后端前后的方程/变量数量 |
| `-d=execstat` | 各模块执行统计(性能分析) |
| `-d=initialization` | 初始化过程细节(也能消除那个常见警告) |
| `--showErrorMessages` | 错误即时输出,而不是攒到最后 |

`omc` 也可以作为脚本 shell 运行:`omc script.mos`,或交互模式 `omc -d=interactive`。
调用 API 后必须调用 `getErrorString()` 才能拿到消息。
