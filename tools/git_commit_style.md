# Git commit 正确姿势

使用正确的 git commit 方式，不仅可以帮助出现 bug 时回溯到正确的版本，并且也可用 commit 日志自动生成 chang log 文档等。

## git commit 作用

### 快速浏览历史信息

通过 git log 命令，可以查看过去提交的所用信息，每个 commit 占据一行

```bash
$ git log <last tag> HEAD --pretty=format:%s
```

### 快速过滤特定 commit

使用 grep 参数，可以快速找出某些特定 commit

```bash
$ git log <last tag> HEAD --grep feature
```

### 生成 Change Log



## git commit 命令

在过去 git commit 使用时，总是添加 `-m` 参数来简单记录提交内容。

```bash
$ git commit -m "hello world"
```

为了更详细的说明提交，可以只执行 git commit，随后在跳出的文本编辑器中书写完整内容

```bash
$ git commit
```

## git commit 格式要求

每次提交，git message 应该包含三部分内容：header，body 和 fotter。具体格式为

```
<type>(<scope>): <subject>
// 空一行
<body>
// 空一行
<footer>
```

其中，Header 是必需的，Body 和 Footer 可以省略。

### Header

Header部分只有一行，包括三个字段：`type`（必需）、`scope`（可选）和`subject`（必需）。

**（1）type**

`type` 用于说明 commit 的类别，只允许使用下面7个标识。

- feat：新功能（feature）
- fix：修补bug
- docs：文档（documentation）
- style：格式（不影响代码运行的变动）
- refactor：重构（即不是新增功能，也不是修改 bug 的代码变动）
- test：增加测试
- chore：构建过程或辅助工具的变动

**（2）scope**

`scope` 用于说明 commit 影响的范围，比如数据层、控制层、视图层等等，视项目不同而不同。

**（3）subject**

`subject `是 commit 目的的简短描述，不超过50个字符。

### Body

Body 部分是对本次 commit 的详细描述，可以分成多行。下面是一个范例。

有两个注意点。

（1）使用第一人称现在时，比如使用 `change `而不是 `changed` 或 `changes`。

（2）应该说明代码变动的动机，以及与以前行为的对比。

### Footer

Footer 部分只用于两种情况。

**（1）不兼容变动**

如果当前代码与上一个版本不兼容，则 Footer 部分以 `BREAKING CHANGE` 开头，后面是对变动的描述、以及变动理由和迁移方法。

```
BREAKING CHANGE: isolate scope bindings definition has changed.

    To migrate the code follow the example below:

    Before:

    scope: {
      myAttr: 'attribute',
    }

    After:

    scope: {
      myAttr: '@',
    }

    The removed `inject` wasn't generaly useful for directives so there should be no code using it.
```

**（2）关闭 Issue**

如果当前 commit 针对某个issue，那么可以在 Footer 部分关闭这个 issue 。