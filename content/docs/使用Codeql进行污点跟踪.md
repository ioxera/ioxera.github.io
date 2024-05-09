---
title: 使用Codeql进行污点跟踪
layout: post
tags:
  - codeql
  - 污点跟踪
date: 2022-10-26
toc: true
---

## Codeql 简介
CodeQL 是开发人员用来自动执行安全检查以及安全研究人员用来执行变体分析的分析引擎。

在 CodeQL 中，代码被视为数据。安全漏洞、错误和其他错误被建模为可以从代码中提取的数据库执行的查询。可以运行由 GitHub 研究人员和社区贡献者编写的标准 CodeQL 查询，或者编写自己的查询以用于自定义分析。发现潜在错误的查询直接在源文件中突出显示结果。

## QL语言简介
QL 是一种声明式的、面向对象的查询语言，经过优化可以有效分析分层数据结构，特别是表示软件组件的数据库。

QL 的语法类似于 SQL，但 QL 的语义基于 Datalog，一种经常用作查询语言的声明式逻辑编程语言。这使得 QL 主要是一种逻辑语言，QL 中的所有操作都是逻辑操作。此外，QL 继承了 Datalog 的递归谓词，并增加了对聚合的支持，即使是复杂的查询也变得简洁明。例如，考虑一个包含人的父子关系的数据库。如果我们想找到一个人的后代数量，通常我们会：

1. 查找给定人的后代，即孩子或孩子的后代。
2. 计算使用上一步找到的后代数。

当你用 QL 编写这个过程时，它非常类似于上面的结构。请注意，我们使用递归来查找给定人的所有后代，并使用聚合来计算后代的数量。由于语言的声明性质，在不添加任何过程细节的情况下将这些步骤转换为最终查询是可能的。QL 代码看起来像这样：

``` ql
Person getADescendant(Person p) {
  result = p.getAChild() or
  result = getADescendant(p.getAChild())
}

int getNumberOfDescendants(Person p) {
  result = count(getADescendant(p))
}
```

以下是通用编程语言和 QL 之间在概念和功能上的一些显著差异：

* QL 没有任何命令式功能，例如变量赋值或文件系统操作。
* QL 对元组集进行操作，查询可以被视为定义查询结果的一组复杂的操作序列。
* QL 基于集合的语义使得处理值集合变得非常自然，而不必担心如何有效地存储、索引和遍历它们。
* 在面向对象的编程语言中，实例化一个类涉及通过分配物理内存来保存该类实例的状态来创建一个对象。在 QL 中，类只是描述已存在值集的逻辑属性。

QL语言的具体定义请参阅[官方文档](https://codeql.github.com/docs/ql-language-reference/)。

## 基本的查询结构
使用 CodeQL 编写的查询具有文件扩展名 ".ql"，并包含一个 select 子句。许多现有查询包括额外的可选信息，并具有以下结构：

``` ql
/**
 *
 * Query metadata
 *
 */

import /* ... CodeQL libraries or modules ... */

/* ... Optional, define CodeQL classes and predicates ... */

from /* ... variable declarations ... */
where /* ... logical formula ... */
select /* ... expressions ... */
```

## 污点跟踪的基本概念
以下以 python 代码示例，介绍污点跟踪的基本概念。

### Source
可以将 “source” 理解为我们想要跟踪其处理流程的数据在代码中首次出现的位置。例如，source 可以是用户输入或硬编码字符串（匹配特定字符串的形式），有时将 source 称为“污染”数据（例如 `TaintTracking::Configuration` 允许我们指定和自定义 source、sink 和流配置的其他几个部分）。

#### 正则表达式注入查询中的 source
给定以下代码段：

``` ql
@app.route("/direct")
def direct():

    unsafe_pattern = request.args["pattern"]
    re.search(unsafe_pattern, "foo")
```

由于我们正在寻找的漏洞发生在用户输入流入正则表达式操作（[正则表达式注入](https://blog.deesee.xyz/regex/security/2020/12/27/regular-expression-injection.html)）的第一个参数时，因此此处的 source 是 `request.args["pattern"]`。即使有其他方法来模拟此漏洞，数据流的 source 也保持不变，因为 `request.args["pattern"]` 是用户输入在代码中第一次出现的位置。

#### RemoteFlowSource
由于大多数与安全相关的查询的重点是，检查用户输入是否流入代码的特定部分（例如，函数的参数），因此 CodeQL 引入了一种结构为开发人员编译每个用户输入，开发者可以直接引入 `RemoteFlowSource`:

``` ql
import python
import semmle.python.dataflow.new.RemoteFlowSources

from RemoteFlowSource rfs // create a 'rfs' variable of type RemoteFlowSource
select rfs // return all of its appearances
```

### Sink
与 source 相反，“sink” 是 source 必须到达的能够导致漏洞的最后一个代码位置。

如下这段代码：

``` ql
@app.route("/demo")
def demo():

    cmd = request.args["pattern"]
    result = os.popen(cmd).read() # [1]
    return f"{cmd} has returned {result}" # [2]
```

这段代码中 source `request.args["pattern"]` 最后一次出现的位置是 [2]，但是根据以上定义，此查询中的实际 sink 位置是 [1]，[1]是 source 到达的会导致漏洞的最后位置。

#### 正则表达式注入查询中的 sink
给定以下代码段：

``` ql
@app.route("/direct")
def direct():

    unsafe_pattern = request.args["pattern"]
    re.search(unsafe_pattern, "foo")
```

本例中的 sink 是 `re.search` 的第一个参数 `unsafe_pattern`。

### 污点跟踪配置谓词
这个谓词是 “额外的（非必需）”，允许为污点跟踪配置指定一些细节。

#### isAdditionalTaintStep
`isAdditionalTaintStep` 让我们指定额外的“跳转”来“绕过”已知函数后，污染流可能会继续。如果指定此谓词，一旦 source 流动结束，CodeQL 应用指定的步骤继续寻找污染流。例如：

``` ql
@app.route("/lxml.etree.parse")
def lxml_parse():

    xml_content = request.args['xml_content']
    xml_content = StringIO(xml_content)

    return lxml.etree.parse(xml_content).text
```

CodeQL 污点跟踪将看到 `request.args['xml_content']` 流向 `StringIO` 并且会停止，因为下一步将是 `lxml.etree.parse(here)`，但这里的 `here` 将是`StringIO(request.args['xml_content'])` ，而不是 `request.args['xml_content']`。换句话说，`lxml.etree.parse` 的第一个参数是 `StringIO`（即使代码是易受攻击的）的结果。发生这种情况是因为，如果受污染的数据流入更改其内容的函数，CodeQL 可能会停止污染流分析。在这种情况下，`StringIO` 返回一个文件的文件名，其内容是提供的参数。

要使 Codeql 在此种情况下继续工作，需要指定一个附加的污染步骤：`StringIO` 第一个参数是 `nodeFrom` ，`StringIO`整个调用是 `nodeTo`。

`isAdditionalTaintStep` 污点跟踪配置中的谓词覆盖:

``` ql
override predicate isAdditionalTaintStep(DataFlow::Node nodeFrom, DataFlow::Node nodeTo) {
  exists(DataFlow::CallCfgNode ioCalls |
    ioCalls = API::moduleImport("io").getMember(["StringIO", "BytesIO"]).getACall() and
    nodeFrom = ioCalls.getArg(0) and
    nodeTo = ioCalls
  )
}
```



### Sanitizer
Sanitizers 与 AdditionalTaintStep 相反，让我们指定不希望 CodeQL 污染流通过的函数或行为。如果指定，每次污染流前进一步，它将检查此特定步骤/行为是否指定 Sanitizers（如果是，污染流将停止）。

给定以下片段：

``` ql
@app.route("/direct")
def direct():

    unsafe_pattern = request.args['pattern']
    safe_pattern = re.escape(unsafe_pattern)
    re.search(safe_pattern, "")
```

此示例中 CodeQL 将 `re.escape` 视为不会消除污染的函数（它一直被污染，因此流不会停止），我们应该将其指定为具有净化行为。

指定 `re.escape` 的第一个参数为 `isSanitizer` 的 `node` 参数，如果 CodeQL 的污染流在此位置，它将停止。

``` ql
override predicate isSanitizer(DataFlow::Node sanitizer) {
    sanitizer = API::moduleImport("re").getMember("escape").getACall().getArg(0)
}
```

此外，我们可以使用 `isSanitizerGuard` 来指定希望污染流停止的另一种情况。例如，`StringConstCompare`：
（根据 qldoc： 通过与常量字符串进行比较，进行未知 node 的验证：）

``` ql
override predicate isSanitizerGuard(DataFlow::BarrierGuard guard) {
    guard instanceof StringConstCompare
}
```

## 污点跟踪配置的基本示例

``` ql
/**
 * A taint-tracking configuration for detecting code injections.
 */
class CodeInjectionFlowConfig extends TaintTracking::Configuration {
  CodeInjectionFlowConfig() { this = "CodeInjectionFlowConfig" }

  override predicate isSource(DataFlow::Node source) { 
    source instanceof RemoteFlowSource 
  }

  override predicate isSink(DataFlow::Node sink) {
    sink = API::builtin("eval").getACall().getArg(0)
  }
}
```

此污点跟踪配置将检测流向任何 `eval` 调用的第一个参数的所有 `RemoteFlowSource` 。

让我们针对以下代码段尝试一下：

``` python
from flask import Flask, request

app = Flask(__name__)

@app.route("/flow1")
def flow1():
    code = request.args["code"]
    eval(code)


@app.route("/flow2")
def flow2():
    email = request.args["email"]
    eval("./send_email {email}".format(email=email))


def flow3_extra(text):
    return text.split("\n")

@app.route("/flow3")
def flow3():
    text = request.args["text"]
    eval(flow3_extra(text))


@app.route("/flow4")
def flow4():
    text = request.args["text"]
    tixt = text
    toxt = flow3_extra(tixt)
    tuxt = toxt
    eval(tuxt)


@app.route("/flow1_good")
def flow1_good():
    code = request.args["code"]
    if code == "print('Hello, Wo... CodeQL!')":
        eval(code)
```

在这个片段中，我们会测试：

* flow1 中的一个简单的污染流，其中 GET 参数 `code` 被赋值给一个变量，然后该变量被用作 `eval` 调用的第一个参数。
* flow2 中的一个污染流，其中 GET 参数 `email` 被赋值给一个变量，然后该变量被用作字符串格式的参数，格式化字符串作为 `eval` 调用的第一个参数。
* flow3 中的一个棘手的污染流，其中 GET 参数 `text` 被赋值给一个变量，然后将该变量用作 `flow3_extra` 的第一个参数，后者返回由 `\n`(LF) 拆分的文本并用作eval调用的第一个参数。
* flow4 中的一个较长的污染流，其中 GET 参数 `text` 被赋值给一个变量，然后将其赋值给另一个变量，然后用作 `floe3_extra` 的第一个参数，通过 `\n` 拆分并返回参数，然后将其赋值给另一个变量，然后用作eval调用的第一个参数。

我们的查询将如下所示：

``` ql
/*
 * @kind path-problem
 */
import python
import semmle.python.dataflow.new.TaintTracking
import semmle.python.dataflow.new.RemoteFlowSources
import semmle.python.ApiGraphs
import DataFlow::PathGraph

/**
 * A taint-tracking configuration for detecting code injections.
 */
class CodeInjectionFlowConfig extends TaintTracking::Configuration {
  CodeInjectionFlowConfig() { this = "CodeInjectionFlowConfig" }

  override predicate isSource(DataFlow::Node source) { 
    source instanceof RemoteFlowSource 
  }

  override predicate isSink(DataFlow::Node sink) {
    sink= API::builtin("eval").getACall().getArg(0)
  }
}

from CodeInjectionFlowConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
where
    config.hasFlowPath(source, sink)
select sink.getNode(), source, sink, "$@ eval argument comes from a $@",
    sink.getNode(), "This", source.getNode(), "user-provided value"
```

基本上，我们告诉 Codeql 当污点跟踪配置检查出 source 是 `RemoteFlowSource` 且 sink 是 `eval` 调用的第一个参数时，给出每个 source 和 sink 。由于使用了 `DataFlow::PathNode` 和 `@kind path-problem` ，结果将以污染流可以被轻松跟踪的方式显示。

正如看到的，除了 `flow1_good` 函数之外的所有函数都是漏洞，即使 `flow1_good` 也被标记了出来，但这是一个假阳性的结果。正如 [Sanitizer](#Sanitizer) 所述，可以添加一个像 ` StringConstCompare` 这样的 Sanitizers Guard 来避免 Codeql 污染流流过 “==" 比较运算。

``` ql
/*
 * @kind path-problem
 */
import python
import semmle.python.dataflow.new.TaintTracking
import semmle.python.dataflow.new.RemoteFlowSources
import semmle.python.ApiGraphs
import semmle.python.dataflow.new.BarrierGuards
import DataFlow::PathGraph

/**
 * A taint-tracking configuration for detecting code injections.
 */
class CodeInjectionFlowConfig extends TaintTracking::Configuration {
  CodeInjectionFlowConfig() { this = "CodeInjectionFlowConfig" }

  override predicate isSource(DataFlow::Node source) { 
    source instanceof RemoteFlowSource 
  }

  override predicate isSink(DataFlow::Node sink) {
    sink= API::builtin("eval").getACall().getArg(0)
  }

  override predicate isSanitizerGuard(DataFlow::BarrierGuard guard) {
    guard instanceof StringConstCompare
  }
}

from CodeInjectionFlowConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
where
    config.hasFlowPath(source, sink)
select sink.getNode(), source, sink, "$@ eval argument comes from a $@",
    sink.getNode(), "This", source.getNode(), "user-provided value"
```

以上仅为使用 Codeql 进行污点跟踪挖掘漏洞的简单示例，实际中的漏洞查询要比以上示例更复杂。
