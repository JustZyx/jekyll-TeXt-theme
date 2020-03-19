---
title: AngularJs Git 提交规范
key: 10023
tags: 翻译
layout: article
category: blog
comment: true
date: 2020-03-19 23:48:00 +08:00
modify_date: 2020-03-19 23:48:00 +08:00
---

在给Github上开源项目做出Contribution的时候我们要遵循其所规定提交规范。如Redis、AngularJs等项目均有自己的Contribution约束。也有一些学习资料类的高星项目缺乏提交约束，此时我一般会遵循AngularJs的提交规范来PR，个人觉得这份规范具有很强的借鉴性，比较适合作为团队合作的规范，鉴于尚无中文版本，所以我打算翻译一下以供大家参考：

## 格式

```c++
<type>(<scope>): <subject> //消息头
<BLANK LINE> //空行
<body> //正文
<BLANK LINE> //空行
<footer> //消息尾
```

1. 为了兼容多种工具方便阅读git message，禁止每一行超过100个字符！
2. 一条commit message包含了header、body和footer，它们之间用空行分隔。

### 回滚

如果当前提交回滚了上一次的提交，它的头部应该以`revert: `开头，紧接着是被回滚的提交记录的头部。body应该描述成`This reverts commit <hash>.`，hash是被回滚的提交记录的SHA值。

### 消息头

消息头是单独的一行，对改动作出了简洁的描述，包含了type，scope(可选)以及标题

#### type(类型)

- 添加功能特性：feat (feature)
- 修复BUG：fix (bug fix)
- 添加文档：docs (documentation)
- 代码风格：style (formatting, missing semi colons, …)
- 重构：refactor
- 添加测试用例：test (when adding missing tests)
- chore (maintain)

#### scope(改动涉及区域)

scope要能够表明改动的区域。举个例子, $location, $browser, $compile, $rootScope, ngHref, ngClick, ngView, etc...这种按照模块来划分，如果是后端的话，通常会分层架构，可以按照接口粒度(user/login), 也可以按照service(UserService)、model(UserModel)的粒度

如果没有合适的区域可以用通配符*代替

#### text(标题)

对改动的简短描述
- 使用命令式、现在式： “change” 而不是 “changed” 或者 “changes”
- 第一个字母不要大写
- 不要以句号.或者。结尾

### 消息体

- 使用命令式、现在式：如 “change” 而不是 “changed” 或者 “changes”
- 包含改动的动机，并与以前的行为形成对比

### 消息尾

#### 重大变化

所有的重大更改应以单词`BREAKING CHANGE：`跟上空格或两个换行符开头，在消息尾的开头以区块的方式声明。剩下的提交信息是对改动的描述，理由和迁移说明。示例如下：

```
BREAKING CHANGE: isolate scope bindings definition has changed and
    the inject option for the directive controller injection was removed.
    
    To migrate the code follow the example below:
    
    Before:
    
    scope: {
      myAttr: 'attribute',
      myBind: 'bind',
      myExpression: 'expression',
      myEval: 'evaluate',
      myAccessor: 'accessor'
    }
    
    After:
    
    scope: {
      myAttr: '@',
      myBind: '@',
      myExpression: '&',
      // myEval - usually not useful, but in cases where the expression is assignable, you can use '='
      myAccessor: '=' // in directive's template change myAccessor() to myAccessor
    }
    
    The removed `inject` wasn't generaly useful for directives so there should be no code using it.
```

#### 参考的issues(tapd bugs)

已关闭的错误应在页脚的单独一行中列出，并以“ Closes”关键字作为前缀，如下所示：

```
Closes #234
```

或者关闭多个问题：

```
Closes #123, #245, #992
```



## 示例

----
feat($browser): onUrlChange event (popstate/hashchange/polling)

Added new event to $browser:
- forward popstate event if available
- forward hashchange event if popstate not available
- do polling when neither popstate nor hashchange available

Breaks $browser.onHashChange, which was removed (use onUrlChange instead)

------
fix($compile): couple of unit tests for IE9

Older IEs serialize html uppercased, but IE9 does not...
Would be better to expect case insensitive, unfortunately jasmine does
not allow to user regexps for throw expectations.

Closes #392
Breaks foo.bar api, foo.baz should be used instead

------
feat(directive): ng:disabled, ng:checked, ng:multiple, ng:readonly, ng:selected

New directives for proper binding these attributes in older browsers (IE).
Added coresponding description, live examples and e2e tests.

Closes #351

---
style($location): add couple of missing semi colons

docs(guide): updated fixed docs from Google Docs

Couple of typos fixed:
- indentation
- batchLogbatchLog -> batchLog
- start periodic checking
- missing brace

----
feat($compile): simplify isolate scope bindings

Changed the isolate scope binding options to:
  - @attr - attribute binding (including interpolation)
  - =model - by-directional model binding
  - &expr - expression execution binding

This change simplifies the terminology as well as
number of choices available to the developer. It
also supports local name aliasing from the parent.

BREAKING CHANGE: isolate scope bindings definition has changed and
the inject option for the directive controller injection was removed.

To migrate the code follow the example below:

Before:

scope: {
  myAttr: 'attribute',
  myBind: 'bind',
  myExpression: 'expression',
  myEval: 'evaluate',
  myAccessor: 'accessor'
}

After:

scope: {
  myAttr: '@',
  myBind: '@',
  myExpression: '&',
  // myEval - usually not useful, but in cases where the expression is assignable, you can use '='
  myAccessor: '=' // in directive's template change myAccessor() to myAccessor
}

The removed `inject` wasn't generaly useful for directives so there should be no code using it.

------

## 参考

- [Git Commit Message Conventions
](https://docs.google.com/document/d/1QrDFcIiPjSLDn3EL15IJygNPiHORgU1_OOAqWjiDU5Y/edit#)