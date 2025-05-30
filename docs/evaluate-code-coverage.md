---
title: "评估代码覆盖率"
description: "本文档解释了如果你在V8上进行更改并希望评估其代码覆盖率时应该怎么做。"
---
你正在进行一个更改。你希望评估新代码的代码覆盖率。

V8提供了两种工具来实现这一目标：在本地运行和使用构建基础设施支持。

## 本地

相对于v8仓库的根目录，使用`./tools/gcov.sh`（在Linux上经过测试）。该工具使用Gnu的代码覆盖工具和一些脚本生成一个HTML报告，你可以深入目录、文件，甚至是代码行，查看覆盖率信息。

脚本会使用`gcov`设置在一个单独的`out`目录下构建V8。我们使用单独的目录以避免覆盖正常的构建设置。这个单独的目录被称为`cov`，它会直接在仓库根目录下创建。`gcov.sh`脚本随后运行测试套件，并生成报告。脚本运行完成后会提供报告的路径。

如果你的更改包含与架构相关的组件，你可以累积收集来自特定架构运行的覆盖率。

```bash
./tools/gcov.sh x64 arm
```

这会为每个架构就地重新构建，会覆盖上次运行的二进制文件，但会保留并累计覆盖率结果。

默认情况下，脚本从`Release`版本的运行中收集数据。如果你需要`Debug`版本的数据，可以指定：

```bash
BUILD_TYPE=Debug ./tools/gcov.sh x64 arm arm64
```

运行脚本而不带任何选项，也会提供选项的摘要说明。

## 代码覆盖率机器人

对于每次提交到主分支的更改，我们都会运行x64覆盖率分析——请参阅[覆盖率机器人](https://ci.chromium.org/p/v8/builders/luci.v8.ci/V8%20Linux64%20-%20gcov%20coverage)。我们不会为其他架构运行覆盖率机器人。

要获取特定运行的报告，你需要列出构建步骤，找到其中的“gsutil覆盖率报告”步骤（靠后的位置），然后打开其下的“报告”。
