Python 很棒，但它并不适合所有情况。有时您可能会发现使用不同的语言可以更轻松地解决特定问题。由于某些技术领域（如控制工程、图像处理或系统编程）的表达能力更强，或者提供优于 Python 的自然性能提升，这些不同的语言可能会更好。 Python（连同默认的 CPython 实现）有一些不一定对性能有益的特性：

由于 CPython 中存在全局解释器锁 (GIL) 并且依赖于所选的 Python 实现，因此 CPU 密集型任务的线程可用性大大降低
Python 不是编译语言（就像 C 和 Go 一样），因此它缺乏许多编译时优化
Python 不提供静态类型和可能的优化
但是，某些语言更适合特定任务的事实并不意味着您在遇到此类问题时必须完全放弃 Python。通过适当的技术，可以编写利用许多技术的应用程序。

一种这样的技术是将应用程序构建为独立的组件，这些组件通过明确定义的通信通道相互通信。这通常以面向服务或微服务架构的形式出现。这在分布式系统中极为常见，其中系统架构的每个组件（服务）都可以在不同的主机上独立运行。用多种语言编写的系统通常被称为多语言系统。

使用多语言面向服务或微服务架构的缺点是，您通常必须为每种此类语言重新创建大量应用程序脚手架。这包括应用程序配置、日志记录、监控和通信层等内容，以及不同的框架、库、构建工具、通用约定和设计模式。引入这些工具和约定将花费时间和未来的维护工作，这通常会超过在架构中添加另一种语言的收益。

幸运的是，还有另一种方法可以克服这个问题。通常，我们从不同语言中真正需要的东西可以打包成一个独立的库，它只做一件事并且做得很好。我们需要做的是在 Python 和其他语言之间找到一座桥梁，让我们能够在 Python 应用程序中使用它们的库。这可以通过自定义 CPython 扩展或所谓的外部函数接口 (FFI) 来完成。

在这两种情况下，C 和 C++ 编程语言都充当通往以不同语言编写的库和代码的门户。 CPython 解释器本身是用 C 编写的，并提供 Python/C API（在 Python.h 头文件中定义），允许您创建可由解释器加载的共享 C 库。 C（和 C++，由于其与 C 语言的本机互操作性）可用于创建此类扩展。另一方面，FFI 可用于与任何兼容的编译共享库进行交互，而不管它是用什么语言编写的。这些库仍将依赖于 C 调用约定和基本类型。

本章将讨论用其他语言编写自己的扩展的主要原因，并向您介绍有助于创建它们的流行工具。我们将在本章中学习以下主题：

- C 和 C++ 作为 Python 可扩展性的核心
- 编译和加载 Python C 扩展
- 编写扩展
- 使用扩展的缺点
- 与没有扩展的编译动态库接口

为了将 Python 与不同的语言连接起来，我们需要一些额外的工具和库，让我们来看看本章的技术要求。

## 技术要求

为了编译本章中提到的 Python 扩展，您将需要 C 和 C++ 编译器。以下是合适的编译器，您可以在选定的操作系统上免费下载：

- Visual Studio 2019 (Windows)：https://visualstudio.microsoft.com
- GCC（Linux 和大多数 POSIX 系统）：https://gcc.gnu.org
- Clang（Linux 和大多数 POSIX 系统）：https://clang.llvm.org

在 Linux 上，GCC 和 Clang 编译器通常可通过特定于给定系统发行版的包管理系统获得。在 macOS 上，编译器是 Xcode IDE（可通过 App Store 获得）的一部分。

以下是本章提到的 Python 包，您可以从 PyPI 下载：

- Cython
- Cffi

关于如何安装包的信息包含在第 2 章，现代 Python 开发环境中。

> 本章的代码文件可以在 https://github.com/PacktPublishing/Expert-Python-Programming-Fourth-Edition/tree/main/Chapter%209 找到。

## C 和 C++ 作为 Python 可扩展性的核心

Python 的参考实现——CPython 解释器——是用 C 编写的。因此，Python 与其他语言的互操作性围绕 C 和 C++ 展开，C 和 C++ 具有与 C 的本机互操作性。甚至还有一个名为 Cython 的 Python 语言的完整超集，它使用源到源编译器使用扩展的 Python 语法为 CPython 创建 C 扩展。

事实上，如果语言支持以动态/共享库的形式编译，您可以使用任何语言编写的动态/共享库。因此，跨语言集成的可能性远远超出了 C 和 C++。这是因为共享库本质上是通用的。它们可以在任何支持加载的语言中使用。因此，即使您使用完全不同的语言（比如 Delphi 或 Prolog）编写这样的库，您也可以在 Python 中使用它。尽管如此，如果不使用 Python/C API，就很难将这样的库称为 Python 扩展。

不幸的是，使用裸 Python/C API 仅在 C 或 C++ 中编写自己的扩展是非常苛刻的。不仅因为它需要对相对难以掌握的两种语言之一有很好的理解，还因为它需要大量的样板。您将不得不编写大量重复代码，这些代码仅用于提供将核心 C 或 C++ 代码与 Python 解释器及其数据类型粘合在一起的接口。

无论如何，由于以下原因，了解纯 C 扩展是如何构建的很好：

您将更好地了解 Python 的一般工作原理
有一天，您可能需要调试或维护本机 C/C++ 扩展
它有助于理解用于构建扩展的高级工具如何工作
这就是为什么在本章中我们将首先学习如何从头开始构建一个简单的 Python C 扩展。我们稍后将使用不需要使用低级 Python/C API 的不同技术重新实现它。

但在我们深入了解编写扩展的细节之前，让我们看看如何编译和加载一个。

## 编译和加载 Python C 扩展

如果 Python 解释器使用 Python/C API 提供适用的接口，则 Python 解释器能够从动态/共享库（例如 Python 模块）加载扩展。构成 Python/C API 的所有函数、类型和宏的定义都包含在 Python.h C 头文件中，该文件与 Python 源代码一起分发。在 Linux 的许多发行版中，这个头文件包含在一个单独的包中（例如，Debian/Ubuntu 中的 python-dev），但在 Windows 下，它默认与解释器一起分发。在 POSIX 和 POSIX 兼容系统（例如，Linux 和 macOS）上，它可以在 Python 安装的 include/ 目录中找到。在 Windows 上，它可以在 Python 安装的 Include/ 目录中找到。

Python/C API 传统上随着 Python 的每个版本而变化。在大多数情况下，这些只是 API 新功能的添加，因此通常与源代码兼容。无论如何，在大多数情况下，由于应用程序二进制接口 (ABI) 的变化，它们不是二进制兼容的。这意味着必须为 Python 的每个主要版本单独编译扩展。此外，不同的操作系统具有不兼容的 ABI，因此实际上不可能为每个可能的环境创建单个二进制分发版。这就是大多数 Python 扩展以源代码形式分发的原因。

自 Python 3.2 起，Python/C API 的一个子集已被定义为具有稳定的 ABI。多亏了这一点，可以使用这个有限的 API（具有稳定的 ABI）构建扩展，因此对于给定的操作系统，扩展只能编译一次，并且它可以与高于或等于 3.2 的任何 Python 版本一起使用，而无需需要重新编译。无论如何，这限制了 API 功能的数量，并不能解决旧 Python 版本的问题。它还不允许您创建可在多个操作系统上运行的单个二进制分发版。这是一种权衡，稳定 ABI 的价格有时对于非常低的收益来说可能有点太高了。

重要的是要知道 Python/C API 是一项仅限于 CPython 实现的功能。已经做出了一些努力来为 PyPI、Jython 或 IronPython 等替代实现提供扩展支持，但目前似乎没有稳定和完整的解决方案。唯一可以轻松处理扩展的替代 Python 实现是 Stackless Python，因为它实际上只是 CPython 的修改版本。

Python 的 C 扩展需要先编译成共享/动态库，然后才能导入。没有直接从源代码导入 Python 中的 C/C++ 代码的本机方法。幸运的是，setuptools 包提供了帮助程序将编译后的扩展定义为模块，因此可以使用 setup.py 脚本处理编译和分发，就像它们是普通的 Python 包一样。

我们将在第 11 章，打包和分发 Python 代码中了解有关创建 Python 包的更多详细信息。

以下是官方文档中 setup.py 脚本的示例，该脚本处理具有用 C 编写的扩展的简单包分发的准备工作：

```python
from setuptools import setup, Extension 
 
module1 = Extension( 
    'demo', 
    sources=['demo.c'] 
) 
 
 
setup( 
    name='PackageName', 
    version='1.0', 
    description='This is a demo package', 
    ext_modules=[module1] 
)
```

我们将在第 11 章，打包和分发 Python 代码中了解有关分发 Python 包和 setup.py 脚本的更多信息。

以这种方式准备好后，您的分发流程中需要执行以下附加步骤：

```sh
python3 setup.py build
```

此步骤将根据 Extension() 构造函数提供的所有其他编译器设置编译定义为 ext_modules 参数的所有扩展。将使用的编译器是您的环境的默认编译器。如果包将作为源代码分发来分发，则不需要此编译步骤。在这种情况下，您需要确保目标环境具有所有编译先决条件，例如编译器、头文件和将链接到二进制文件的其他库（如果您的扩展需要的话）。打包 Python 扩展的更多细节将在稍后使用扩展的缺点部分进行解释。

在下一节中，我们将讨论您可能需要使用扩展的原因。

## 使用扩展的必要性

很难说何时用 C/C++ 编写扩展是一个合理的决定。一般的经验法则可能是“除非你别无选择，否则永远不要”。但这是一个非常主观的陈述，它为解释 Python 中不可行的内容留下了很多空间。事实上，很难找到使用纯 Python 代码做不到的事情。

尽管如此，仍然存在一些问题，通过添加以下好处，扩展可能特别有用：

- 在 CPython 线程模型中绕过 GIL
- 提高关键代码部分的性能
- 集成用不同语言编写的源代码
- 集成第三方动态库
- 创建高效的自定义数据类型

当然，对于每个这样的问题，通常都有一个可行的原生 Python 解决方案。例如，核心 CPython 解释器约束（例如 GIL）可以通过不同的并发方法轻松克服，例如协程或多处理，而不是线程模型（我们在第 6 章并发中讨论了这些选项）。为了解决第三方动态库和自定义数据类型，第三方库可以与 ctypes 模块集成，并且每个数据类型都可以在 Python 中实现。

尽管如此，原生 Python 方法可能并不总是最佳的。外部库的仅 Python 集成可能很笨拙且难以维护。如果无法访问低级内存管理，自定义数据类型的实现可能是次优的。因此，必须始终非常谨慎地做出最终决定，并考虑许多因素。一个好的方法是首先从纯 Python 实现开始，只有在原生方法证明不够好时才考虑扩展。

下一节将解释如何使用扩展来提高关键代码部分的性能。

### 提高关键代码部分的性能

说实话，Python 并不是因为它的性能而被开发人员选择的。它的执行速度不快，但可以让您快速开发。尽管如此，无论我们作为程序员的性能如何，多亏了这种语言，我们有时可能会发现使用纯 Python 可能无法有效解决的问题。

在大多数情况下，解决性能问题实际上主要是选择合适的算法和数据结构，而不是限制语言开销的常数因素。通常，如果代码已经编写得很差或没有使用有效的算法，那么依靠扩展来削减一些 CPU 周期并不是一个好方法。

通常可以将性能提高到可接受的水平，而无需通过向技术堆栈添加另一种语言来增加项目的复杂性。如果可以只使用一种编程语言，那么首先应该这样做。

无论如何，即使使用最先进的算法方法和最适合的数据结构，您也很可能无法仅使用 Python 来适应某些任意的技术约束。

对应用程序的性能设置一些明确定义的限制的示例字段是实时出价 (RTB) 业务。简而言之，整个 RTB 是关于以类似于真实拍卖或证券交易所的工作方式的方式买卖广告库存（在线广告的位置）。整个交易通常通过一些广告交换服务进行，该服务将有关可用库存的信息发送到有兴趣为其广告购买区域的需求方平台 (DSP)。这就是事情变得令人兴奋的地方。大多数广告交易平台使用 OpenRTB 协议（基于 HTTP）与潜在投标人进行通信。 DSP 是负责响应其 OpenRTB HTTP 请求的站点。广告交易平台总是对整个过程需要多长时间进行非常严格的时间限制。从接收到的第一个 TCP 数据包到 DSP 服务器写入的最后一个字节，它可以短至 50 毫秒。更有趣的是，DSP 平台每秒处理数万个请求的情况并不少见。能够将响应时间缩短几毫秒通常决定了服务的盈利能力。这意味着在这种情况下将即使是微不足道的代码移植到 C 可能是合理的，但前提是它是某些性能瓶颈的一部分并且无法在算法上进一步改进。正如圭多曾经说过的：

> 如果您觉得需要速度，(...) – 您无法击败用 C 编写的循环。

自定义扩展的一个完全不同的用例是集成用不同语言编写的代码，这将在下一节中解释。

### 集成用不同语言编写的现有代码

尽管与其他技术研究领域相比，计算机科学还很年轻，但我们已经站在巨人的肩膀上。许多伟大的程序员使用多种编程语言编写了许多有用的库来解决常见问题。每次出现新的编程语言时忘记所有这些遗产将是一个巨大的损失，但也不可能可靠地移植任何曾经用每种新语言编写的软件。

C 和 C++ 语言似乎是最重要的语言，它们提供了许多您希望集成到应用程序代码中的库和实现，而无需将它们完全移植到 Python。幸运的是，CPython 已经是用 C 语言编写的，因此集成这些代码的最自然的方式正是通过自定义扩展。

下一节解释了一个非常相似的用例：集成第三方动态库。

### 集成第三方动态库

集成使用不同技术编写的代码并不会以 C/C++ 结束。许多库，尤其是具有封闭源代码的第三方软件，都是作为编译后的二进制文件分发的。在 C 中，加载这样的共享/动态库并调用它们的函数真的很容易。这意味着您可以使用任何 C 库，只要您使用 Python/C API 将其包装为 Python 扩展即可。

当然，这不是唯一的解决方案，还有一些工具，例如 ctypes 和 CFFI，允许您使用纯 Python 代码直接与动态库交互，而无需用 C 编写扩展。很多时候，Python/C API 可能仍然是更好的选择，因为它在集成层（用 C 编写）和应用程序的其余部分之间提供了更好的分离。

最后但并非最不重要的一点是，扩展可用于通过新颖且高性能的数据结构来增强 Python。

### 创建高效的自定义数据类型

Python 提供了非常通用的内置数据类型选择。其中一些确实使用了专为在 Python 语言中使用而量身定制的最先进的内部实现（至少在 CPython 中）。对于新手来说，开箱即用的基本类型和集合的数量可能令人印象深刻，但很明显，它并不能满足程序员的所有需求。

当然，您可以在 Python 中创建许多自定义数据结构，方法是对内置类型进行子类化，或者将它们从头开始构建为全新的类。不幸的是，有时这种数据结构的性能可能不是最佳的。诸如 dict 或 set 之类的复杂集合的全部功能来自它们底层的 C 实现。为什么不做同样的事情并在 C 中实现一些自定义数据结构呢？

由于我们已经知道创建自定义 Python 扩展的可能原因，让我们看看如何实际构建一个。

## 编写扩展

如前所述，编写扩展不是一项简单的任务，但作为对您辛勤工作的回报，它可以为您带来很多好处。创建扩展的最简单方法是使用 Cython 等工具。 Cython 允许您使用非常类似于 Python 的语言编写 C 扩展，而无需 Python/C API 的所有复杂性。它将提高您的工作效率并使代码更易于开发、阅读和维护。

无论如何，如果您是这个主题的新手，最好通过仅使用裸 C 语言和 Python/C API 编写扩展来开始您的扩展冒险。这将提高您对扩展如何工作的理解，还将帮助您了解替代解决方案的优势。为简单起见，我们以一个简单的算法问题为例，并尝试使用以下两种不同的方法来实现它：

- 编写纯 C 扩展
- 使用 Cython

我们的问题将是找到斐波那契数列的第 n 个数字。这是一个数字序列，其中每个元素都是前面两个元素的总和。序列从0和1开始，序列的前10个数字如下：

```
0, 1, 1, 2, 3, 5, 8, 13, 21, 34
```

如您所见，该序列很容易解释，也很容易实现。您不太可能需要仅为解决此问题而创建编译扩展。但它非常简单，因此它将作为将任何 C 函数连接到 Python/C API 的一个很好的例子。我们的目标只是清晰和简单，因此我们不会尝试提供最有效的解决方案。

在我们创建第一个扩展之前，让我们定义一个参考实现，让我们可以比较不同的解决方案。我们在纯 Python 中实现的斐波那契函数的参考实现如下所示：

```python
"""Python module that provides fibonacci sequence function""" 
 
def fibonacci(n): 
    """Return nth Fibonacci sequence number computed recursively."""
    if n == 0:
        return 0
    if n == 1: 
        return 1 
    else: 
        return fibonacci(n - 1) + fibonacci(n - 2)
```

请注意，这是 fibonnaci() 函数最简单的实现之一。许多改进都可以应用于它。我们不优化我们的实现（例如使用记忆模式），因为这不是我们示例的目的。同样，在讨论 C 或 Cython 的实现时，我们不会在稍后优化我们的代码，即使编译后的代码为我们提供了更多的可能性。

> Memoization 是一种流行的技术，可以保存函数调用的过去结果以供以后参考以优化应用程序性能。我们在第 13 章代码优化中详细解释。

让我们在下一节中研究纯 C 扩展。

### 纯 C 扩展

如果您已经决定需要为 Python 编写 C 扩展，我假设您已经对 C 语言有所了解，可以让您完全理解所提供的示例。这本书是关于 Python 的，因此除了 Python/C API 的细节之外，这里不会解释任何其他内容。尽管这个 API 是精心制作的，但绝对不是 C 的好介绍，所以如果你根本不了解 C，你应该避免尝试用 C 编写 Python 扩展，直到你获得了这种语言的经验。把它留给其他人并坚持使用 Cython 或 Pyrex，从初学者的角度来看它们更安全。

如前所述，我们将尝试将 fibonacci() 函数移植到 C 并将其作为扩展公开给 Python 代码。让我们从类似于前面的 Python 示例的基本实现开始。没有任何 Python/C API 使用的裸函数大致如下：

```c
long long fibonacci(unsigned int n) { 
    if (n == 0) {
        return 0;
    } else if (n == 1) { 
        return 1; 
    } else { 
        return fibonacci(n - 2) + fibonacci(n - 1); 
    } 
}
```

这是一个完整的、功能齐全的扩展的例子，它在一个编译模块中公开了这个单一的功能：

```c
#define PY_SSIZE_T_CLEAN
#include <Python.h> 
 
long long fibonacci(unsigned int n) { 
    if (n == 0) {
        return 0;
    } else if (n == 1) { 
        return 1; 
    } else { 
        return fibonacci(n - 2) + fibonacci(n - 1); 
    } 
}  
 
static PyObject* fibonacci_py(PyObject* self, PyObject* args) { 
    PyObject *result = NULL; 
    long n; 
 
    if (PyArg_ParseTuple(args, "l", &n)) { 
        result = Py_BuildValue("L", fibonacci((unsigned int)n)); 
    } 
 
    return result; 
}
 
static char fibonacci_docs[] = 
    "fibonacci(n): Return nth Fibonacci sequence number " 
    "computed recursively\n"; 
 
 
static PyMethodDef fibonacci_module_methods[] = { 
    {"fibonacci", (PyCFunction)fibonacci_py, 
     METH_VARARGS, fibonacci_docs}, 
    {NULL, NULL, 0, NULL} 
}; 
 
 
static struct PyModuleDef fibonacci_module_definition = { 
    PyModuleDef_HEAD_INIT, 
    "fibonacci", 
    "Extension module that provides fibonacci sequence function", 
    -1, 
    fibonacci_module_methods 
}; 
 
 
PyMODINIT_FUNC PyInit_fibonacci(void) { 
    Py_Initialize(); 
 
    return PyModule_Create(&fibonacci_module_definition); 
}
```

我知道你的想法。乍一看，前面的示例可能有点让人不知所措。我们不得不添加四倍多的代码才能使 fibonacci() C 函数可从 Python 访问。稍后我们将逐步讨论该代码的每一部分，所以不要担心。但在我们这样做之前，让我们看看它是如何在 Python 中打包和执行的。

我们模块的以下最小 setuptools 配置需要使用 setuptools.Extension 类来指示解释器如何编译我们的扩展：

```python
from setuptools import setup, Extension 
 
setup( 
    name='fibonacci', 
    ext_modules=[ 
        Extension('fibonacci', ['fibonacci.c']), 
    ] 
)
```

扩展的构建过程可以使用 setup.py build 命令初始化，但它也会在包安装时自动执行。以下脚本显示了在可编辑模式下的安装结果（使用带有 -e 标志的 pip）：

```python
$ python3 -m pip install -e .
Obtaining file:///Users/.../Expert-Python-Programming-Fourth-Edition/Chapter%209/02%20-%20Pure%20C%20extensions
Installing collected packages: fibonacci
  Running setup.py develop for fibonacci
Successfully installed fibonacci
```

使用 pip 的可编辑模式允许我们查看在构建步骤中创建的文件。以下是在安装期间可以在您的工作目录中创建的文件示例：

```python
$ ls -1ap
./
../
build/
fibonacci.c
fibonacci.cpython-39-darwin.so
fibonacci.egg-info/
setup.py
```

fibonacci.c 和 setup.py 文件是我们的源文件。 fibonacci.egg-info/ 是存放包元数据的特殊目录，我们暂时不用关心。真正重要的是 fibonacci.cpython-39-darwin.so 文件。这是我们与 CPython 解释器兼容的二进制共享库。这是当我们尝试导入我们的 fibonacci 模块时 Python 解释器将加载的库。让我们尝试导入它并在交互式会话中查看它：

```python
$ python3
Python 3.9.1 (v3.9.1:1e5d33e9b9, Dec  7 2020, 12:10:52)
[Clang 6.0 (clang-600.0.57)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import fibonacci
>>> help(fibonacci)
Help on module fibonacci:
NAME
    fibonacci - Extension module that provides fibonacci sequence function
FUNCTIONS
    fibonacci(...)
        fibonacci(n): Return nth Fibonacci sequence number computed recursively
FILE
    /(...)/fibonacci.cpython-39-darwin.so
>>> [fibonacci.fibonacci(n) for n in range(10)]
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

现在让我们仔细看看我们的扩展的解剖结构。

#### 深入了解 Python/C API

由于我们知道如何正确打包、编译和安装自定义 C 扩展，并且我们确信它按预期工作，现在是详细讨论我们的代码的合适时机。

扩展模块从以下单个 C 预处理器指令开始，其中包括 Python.h 头文件：

```c
#include <Python.h>
```

这会拉取整个 Python/C API，并且是您编写扩展所需包含的一切。在更实际的情况下，您的代码将需要更多预处理器指令才能从 C 标准库函数中受益或集成其他源文件。我们的例子很简单，所以不需要更多的指令。

接下来，我们的模块的核心如下：

```c
long long fibonacci(unsigned int n) {
    if (n == 0) {
        return 0;
    } else if (n == 1) {
        return 1;
    } else {
        return fibonacci(n - 2) + fibonacci(n - 1);
    }
}
```

前面的 fibonacci() 函数是我们代码中唯一可以做一些有用事情的部分。它是 Python 默认无法理解的纯 C 实现。我们示例的其余部分将创建将通过 Python/C API 公开它的接口层。

将此代码暴露给 Python 的第一步是创建与 CPython 解释器兼容的 C 函数。在 Python 中，一切都是对象。这意味着在 Python 中调用的 C 函数也需要返回真正的 Python 对象。 Python/C API 提供 PyObject 类型，每个可调用对象都必须返回指向它的指针。我们函数的签名如下：

```c
static PyObject* fibonacci_py(PyObject* self, PyObject* args)
```

请注意，前面的签名没有指定确切的参数列表，但 PyObject* args 将保存指向包含所提供值的元组的结构的指针。

参数列表的实际验证必须在函数体内执行，这正是 fibonacci_py() 所做的。它解析 args 参数列表，假设它是单个 unsigned int 类型，并使用该值作为 fibonacci() 函数的参数来检索 Fibonacci 序列元素，如以下代码所示：

```c
static PyObject* fibonacci_py(PyObject* self, PyObject* args) { 
    PyObject *result = NULL; 
    long n; 
 
    if (PyArg_ParseTuple(args, "l", &n)) { 
        result = Py_BuildValue("L", fibonacci((unsigned int)n)); 
    } 
 
    return result; 
}
```

> 前面的示例函数有一个严重的错误，有经验的开发人员的眼睛应该很容易发现。尝试将其作为使用 C 扩展的练习。现在，为了简洁起见，我们将保持原样。稍后在异常处理部分讨论处理错误和异常的细节时，我们将尝试修复它。

PyArg_ParseTuple(args, "l", &n) 调用中的“l”（小写 L）字符串意味着我们期望 args 只包含一个 long 值。在失败的情况下，它将返回 NULL 并在每线程解释器状态中存储有关异常的信息。

解析函数的实际签名是 int PyArg_ParseTuple(PyObject *args, const char *format, ...) 格式字符串之后是一个可变长度的参数列表，表示解析值输出（作为指针）。这类似于 C 标准库中 scanf() 函数的工作方式。如果我们的假设失败并且用户提供了不兼容的参数列表，那么 PyArg_ParseTuple() 将引发适当的异常。一旦您习惯了，这是一种非常方便的函数签名编码方式，但与纯 Python 代码相比，它有一个巨大的缺点。这种由 PyArg_ParseTuple() 调用隐式定义的 Python 调用签名无法在 Python 解释器中轻松检查。在使用作为扩展提供的代码时，您需要记住这一事实。

如前所述，Python 期望从可调用对象返回对象。这意味着我们不能返回作为 fibonacci_py() 的结果从 fibonacci() 函数获得的原始 long 值。这样的尝试甚至不会编译，并且没有将基本 C 类型自动转换为 Python 对象。

必须改用 Py_BuildValue(*format, ...) 函数。它是 PyArg_ParseTuple() 的对应物，并接受一组类似的格式字符串。主要区别在于参数列表不是函数输出而是输入，因此必须提供实际值而不是指针。

定义 fibonacci_py() 之后，大部分繁重的工作就完成了。最后一步是执行模块初始化并向我们的函数添加元数据，这将使用户的使用更简单。这是我们扩展代码的样板部分。对于简单的例子，比如这个，它可能比我们想要公开的实际函数占用更多的空间。在大多数情况下，它只是由一些静态结构和一个初始化函数组成，这些函数将由解释器在模块导入时执行。

首先，我们创建一个静态字符串，它将作为 fibonacci_py() 函数的 Python 文档字符串的内容，如下所示：

```c
static char fibonacci_docs[] = 
    "fibonacci(n): Return nth Fibonacci sequence number " 
    "computed recursively\n";
```

请注意，这可能会在 fibonacci_module_methods 的后面某处内联，但将文档字符串分开并存储在它们所引用的实际函数定义附近是一个很好的做法。

我们定义的下一部分是 PyMethodDef 结构的数组，这些结构定义了将在我们的模块中可用的方法（函数）。 PyMethodDef 结构正好包含四个字段：

- char* ml_name：这是方法的名称。
- PyCFunction ml_meth：这是指向函数的 C 实现的指针。
- int ml_flags：这包括指示调用约定或绑定约定的标志。后者仅适用于类方法的定义。
- char* ml_doc：这是指向方法/函数文档字符串内容的指针。

这样的数组必须始终以 {NULL, NULL, 0, NULL} 的标记值结束。该标记值仅指示结构的结束。在我们的简单案例中，我们创建了静态 PyMethodDef fibonacci_module_methods[] 数组，该数组仅包含两个元素（包括标记值）：

```c
static PyMethodDef fibonacci_module_methods[] = { 
    {"fibonacci", (PyCFunction)fibonacci_py, 
     METH_VARARGS, fibonacci_docs}, 
    {NULL, NULL, 0, NULL} 
};
```

这是第一个条目映射到 PyMethodDef 结构的方式：

- ml_name = "fibonacci"：在这里，fibinacci_py() C 函数将作为斐波那契名称下的 Python 函数公开。
- ml_meth = (PyCFunction)fibonacci_py：这里，Python/C API 只需要转换到 PyCFunction，并由稍后在 ml_flags 中定义的调用约定决定。
- ml_flags = METH_VARARGS：这里，METH_VARARGS 标志表示我们函数的调用约定接受一个可变的参数列表，没有关键字参数。
- ml_doc = fibonacci_docs：这里，Python 函数将记录 fibonacci_docs 字符串的内容。

当一个函数定义数组完成后，我们可以创建另一个包含整个模块定义的结构。它使用 PyModuleDef 类型描述并包含多个字段。其中一些仅适用于需要对模块初始化过程进行细粒度控制的更复杂的场景。在这里，我们只对其中的前五个感兴趣：

- PyModuleDef_Base m_base：这应该总是用 PyModuleDef_HEAD_INIT 初始化。
- char* m_name：这是新创建的模块的名称。在我们的例子中，它是斐波那契。
- char* m_doc：这是指向模块文档字符串内容的指针。我们通常在一个 C 源文件中只定义了一个模块，因此可以在整个结构中内联我们的文档字符串。
- Py_ssize_t m_size：这是为保持模块状态而分配的内存大小。这仅在需要支持多个子解释器或多阶段初始化时使用。在大多数情况下，您不需要它，它的值为 -1。
- PyMethodDef* m_methods：这是一个指向包含由 PyMethodDef 值描述的模块级函数的数组的指针。如果模块不公开任何函数，则它可能为 NULL。在我们的例子中，它是 fibonacci_module_methods。

其他字段在官方 Python 文档（参考 https://docs.python.org/3/c-api/module.html）中有详细解释，但在我们的示例扩展中不需要。如果不需要，它们应设置为 NULL，并且在未指定时将使用该值隐式初始化。这就是为什么我们包含在 fibonacci_module_definition 变量中的模块描述可以采用以下简单形式的原因：

```c
static struct PyModuleDef fibonacci_module_definition = { 
    PyModuleDef_HEAD_INIT, 
    "fibonacci", 
    "Extension module that provides fibonacci sequence function", 
    -1, 
    fibonacci_module_methods 
};
```

为我们的工作加冕的最后一段代码是模块初始化函数。这必须遵循非常具体的命名约定，以便 Python 解释器可以在加载动态/共享库时轻松找到它。它应该命名为 PyInit_\<name\>，其中 \<name\> 是您的模块的名称。因此，它与在 PyModuleDef 定义中用作 m_base 字段以及作为 setuptools.Extension() 调用的第一个参数的字符串完全相同。如果您不需要对模块进行复杂的初始化过程，则它采用非常简单的形式，与我们的示例完全一样：

```c
PyMODINIT_FUNC PyInit_fibonacci(void) { 
    return PyModule_Create(&fibonacci_module_definition); 
}
```

PyMODINIT_FUNC 宏是一个预处理器宏，它将将此初始化函数的返回类型声明为 PyObject*，并在平台需要时添加任何特殊的链接声明。

Python 和 C 函数之间的一个非常重要的区别是调用和绑定约定。这是一个相当冗长的话题，所以让我们在单独的部分中讨论它。

#### 调用和绑定约定

Python 是一种面向对象的语言，具有使用位置和关键字参数的灵活调用约定。考虑以下 print() 函数调用：

```c
print("hello", "world", sep=" ", end="!\n")
```

提供给调用的前两个表达式（“hello”和“world”表达式）是定位的，将与 print() 函数的定位参数匹配。顺序很重要，如果我们修改它，函数调用将给出不同的结果。另一方面，以下 " " 和 "!\n" 表达式将与关键字参数匹配。只要名称不变，它们的顺序就无关紧要。

C 是一种只有位置参数的过程语言。在编写 Python 扩展时，需要支持 Python 的参数灵活性和面向对象的数据模型。这主要是通过支持的调用和绑定约定的显式声明来完成的。

如深入了解 Python/C API 部分所述，PyMethodDef 结构的 ml_flags 位字段包含用于调用和绑定约定的标志。调用约定标志如下：

- METH_VARARGS：这是 Python 函数或方法的典型约定，仅接受参数作为其参数。作为此类函数的 ml_meth 字段提供的类型应为 PyCFunction。该函数将提供两个 PyObject* 类型的参数。第一个是 self 对象（对于方法）或模块对象（对于模块函数）。具有该调用约定的 C 函数的典型签名是 PyObject* function(PyObject* self, PyObject* args)。
- METH_KEYWORDS：这是 Python 函数在调用时接受关键字参数的约定。其关联的 C 类型是 PyCFunctionWithKeywords。 C 函数必须接受 PyObject* 类型的三个参数 — self、args 和关键字参数字典。如果与 METH_VARARGS 结合使用，前两个参数的含义与前面的调用约定相同，否则 args 将为 NULL。典型的 C 函数签名是 PyObject* function(PyObject* self, PyObject* args, PyObject* keywds)。
- METH_NOARGS：这是不接受任何其他参数的 Python 函数的约定。 C 函数应该是 PyCFunction 类型，因此签名与 METH_VARARGS 约定的签名相同（带有 self 和 args 参数）。唯一的区别是 args 将始终为 NULL，因此无需调用 PyArg_ParseTuple()。这不能与任何其他调用约定标志结合使用。
- METH_O：这是接受单个对象参数的函数和方法的简写。 C 函数的类型还是 PyCFunction，因此它接受两个 PyObject* 参数：self 和 args。它与 METH_VARARGS 的不同之处在于不需要调用 PyArg_ParseTuple() 因为作为 args 提供的 PyObject* 已经表示在对该函数的 Python 调用中提供的单个参数。这也不能与任何其他调用约定标志结合使用。

接受关键字的函数用 METH_KEYWORDS 或 METH_VARARGS 形式的调用约定标志的按位组合来描述。 METH_KEYWORDS。如果是这样，它应该使用 PyArg_ParseTupleAndKeywords() 而不是 PyArg_ParseTuple() 或 PyArg_UnpackTuple() 来解析它的参数。

这是一个带有单个函数的示例模块，该函数返回 None 并接受两个打印在标准输出上的命名参数：

```c
#define PY_SSIZE_T_CLEAN
#include <Python.h> 
 
static PyObject* print_args(PyObject *self, PyObject *args, 
 PyObject *keywds) 
{ 
    char *first; 
    char *second; 
 
    static char *kwlist[] = {"first", "second", NULL}; 
 
    if (!PyArg_ParseTupleAndKeywords(args, keywds, "ss", kwlist, 
                                     &first, &second)) 
        return NULL; 
 
    printf("%s %s\n", first, second); 
 
    Py_INCREF(Py_None); 
    return Py_None; 
} 
 
 
static PyMethodDef module_methods[] = { 
    {"print_args", (PyCFunction)print_args, 
     METH_VARARGS | METH_KEYWORDS, 
     "print provided arguments"}, 
    {NULL, NULL, 0, NULL} 
}; 
 
 
static struct PyModuleDef module_definition = { 
    PyModuleDef_HEAD_INIT, 
    "kwargs", 
    "Keyword argument processing example", 
    -1, 
    module_methods 
}; 
 
 
PyMODINIT_FUNC PyInit_kwargs(void) { 
    return PyModule_Create(&module_definition); 
}
```

Python/C API 中的参数解析非常灵活，在 https://docs.python.org/3/c-api/arg.html 的官方文档中进行了广泛的描述。

PyArg_ParseTuple() 和 PyArg_ParseTupleAndKeywords() 中的格式参数允许对参数数量和类型进行细粒度控制。 Python 中已知的每个高级调用约定都可以使用此 API 用 C 进行编码，包括以下内容：

- 具有参数默认值的函数
- 参数指定为仅关键字的函数
- 参数指定为仅位置的函数
- 具有可变数量参数的函数
- 没有参数的函数

附加的绑定约定标志 METH_CLASS、METH_STATIC 和 METH_COEXIST 是为方法保留的，不能用于描述模块功能。前两个是不言自明的。它们是@classmethod 和@staticmethod 装饰器的C 对应物，并更改传递给C 函数的self 参数的含义。

METH_COEXIST 允许加载一个方法来代替现有的定义。它很少有用。这主要是在您希望提供 C 方法的实现的情况下，该实现将从定义的类型的其他功能自动生成。 Python 文档给出了 \_\_contains\_\_() 包装方法的示例，如果类型定义了 sq_contains 槽，则会生成该方法。不幸的是，使用 Python/C API 定义您自己的类和类型超出了本介绍性章节的范围。

下一节我们来看看异常处理。

#### 异常处理

与 Python 甚至 C++ 不同，C 没有用于引发和捕获异常的语法。所有错误处理通常使用函数返回值和可选的全局状态来处理，用于存储可以解释上次失败原因的详细信息。

Python/C API 中的异常处理就是围绕这个简单的原则构建的。上次发生的错误有一个全局的每线程指示器。它用于描述问题的原因。如果此状态在调用期间发生更改，还有一种标准化方法可以通知函数的调用者，例如：

- 如果函数应该返回一个指针，则返回 NULL
- 如果函数应该返回 int 类型的值，则返回 -1

Python/C API 中上述规则的唯一例外是 PyArg_*() 函数，它返回 1 表示成功，返回 0 表示失败。

为了看看这在实践中是如何工作的，让我们回忆一下前几节示例中的 fibonacci_py() 函数：

```c
static PyObject* fibonacci_py(PyObject* self, PyObject* args) { 
    PyObject *result = NULL; 
    long n; 
 
    if (PyArg_ParseTuple(args, "l", &n)) {
        result = Py_BuildValue("L", fibonacci((unsigned int) n)); 
    } 
 
    return result; 
}
```

错误处理从我们函数的最开始开始，伴随着结果变量的初始化。这个变量应该存储我们函数的返回值。它用 NULL 初始化，正如我们已经知道的那样，它是错误的指示器。这就是您通常对扩展进行编码的方式——假设错误是您代码的默认状态。

稍后我们有 PyArg_ParseTuple() 调用，它会在出现异常的情况下设置错误信息并返回 0。这是 if 语句的一部分，因此在出现异常的情况下，我们不再做任何事情，函数将返回 NULL。任何调用我们函数的人都会收到有关错误的通知。

Py_BuildValue() 也可以引发异常。它应该返回 PyObject*（指针），所以在失败的情况下，它给出 NULL。我们可以简单地将其存储为我们的结果变量并将其作为返回值传递。

但是我们的工作并没有以关注 Python/C API 调用引发的异常结束。您很可能需要将发生的错误或故障类型告知扩展用户。 Python/C API 有多个函数可以帮助您引发异常，但最常见的是 PyErr_SetString()。它使用给定的异常类型设置错误指示符，并提供附加字符串作为错误原因的解释。该函数的完整签名如下：

```c
void PyErr_SetString(PyObject* type, const char* message)
```

您可能已经在仔细查看 Python/C API 部分中的 fibonacci_py() 函数中注意到了一个有问题的问题。如果没有，现在是发现它并修复它的合适时机。幸运的是，我们终于有了合适的工具来做到这一点。

问题在于以下几行将 long 类型转换为 unsigned int 的不安全：

```c
if (PyArg_ParseTuple(args, "l", &n)) { 
    result = Py_BuildValue("L", fibonacci((unsigned int) n)); 
}
```

由于 PyArg_ParseTuple() 调用，第一个也是唯一的参数将被解释为 long 类型（“l”说明符）并存储在局部 n 变量中。然后将其转换为 unsigned int，因此如果用户使用负值从 Python 调用 fibonacci() 函数，则会出现问题。例如，当转换为无符号 32 位整数时，-1 作为有符号 32 位整数将被解释为 4294967295。这样的值会导致非常深的递归，并会导致堆栈溢出和分段错误。请注意，如果用户给出任意大的正论点，则可能会发生同样的情况。如果不完全重新设计 C fibonacci() 函数，我们就无法解决这个问题，但我们至少可以尝试确保函数输入参数满足一些先决条件。在这里，我们检查 n 参数的值是否大于或等于 0，如果不是，则引发 ValueError 异常，如下所示：

```c
static PyObject* fibonacci_py(PyObject* self, PyObject* args) { 
    PyObject *result = NULL; 
    long n; 
    long long fib; 
 
    if (PyArg_ParseTuple(args, "l", &n)) { 
        if (n<0) { 
            PyErr_SetString(PyExc_ValueError, 
                            "n must not be less than 0"); 
        } else { 
            result = Py_BuildValue("L", fibonacci((unsigned int) n)); 
        } 
    } 
 
    return result; 
}
```

关于异常处理的最后一个注意事项是全局错误状态不会自行清除。有些错误可以在你的 C 函数中优雅地处理（与在 Python 中使用 try ... except 子句相同），如果错误指示器不再有效，你需要能够清除它。该函数是 PyErr_Clear()。

C 扩展的一大优势是能够绕过 GIL，这可能不利于 Python 应用程序中的线程并发。在下一节中，我们将讨论在 C 扩展中发布 GIL 的可能性。

#### 释放 GIL

我们已经提到扩展可以是绕过 Python 的 GIL 的一种方式。一次只有一个线程可以执行 Python 代码，这是 CPython 实现的一个著名限制。多处理是规避此问题的建议方法（请参阅第 6 章，并发），但由于运行附加进程的资源开销，它可能不是某些高度并行化算法的最佳解决方案。

因为扩展主要用于大部分工作是在纯 C 中执行而不对 Python/C API 进行任何调用的情况，所以在处理非 Python 数据的同时在某些应用程序部分释放 GIL 是可能的（甚至是可取的）加工。因此，您仍然可以从拥有多个 CPU 内核和多线程应用程序设计中受益。您唯一需要做的就是用 Python/C API 提供的特定宏包装已知不使用任何 Python/C API 调用或 Python 结构的代码块。提供以下两个预处理器宏以简化释放和重新获取 GIL 的整个过程：

- Py_BEGIN_ALLOW_THREADS：这声明了当前线程状态保存的隐藏局部变量，并释放 GIL。
- Py_END_ALLOW_THREADS：这会重新获取 GIL 并从使用前一个宏声明的局部变量恢复线程状态。

当我们仔细查看我们的 fibonacci 扩展示例时，我们可以清楚地看到 fibonacci() 函数不执行任何 Python 代码，也不触及任何 Python 结构。这意味着简单地包装 fibonacci(n) 执行的 fibonacci_py() 函数可以更新以在该调用周围释放 GIL，如下所示：

```c
static PyObject* fibonacci_py(PyObject* self, PyObject* args) { 
    PyObject *result = NULL; 
    long n; 
    long long fib; 
 
    if (PyArg_ParseTuple(args, "l", &n)) { 
        if (n<0) { 
            PyErr_SetString(PyExc_ValueError, 
                            "n must not be less than 0"); 
        } else { 
            Py_BEGIN_ALLOW_THREADS; 
            fib = fibonacci(n); 
            Py_END_ALLOW_THREADS; 
 
            result = Py_BuildValue("L", fib); 
        }
     } 
 
    return result; 
}
```

关于 Python/C API 的另一个重要主题是内存管理和垃圾收集。动态编程语言中最常见的垃圾收集机制是跟踪垃圾收集，它通过跟踪是否可以从程序的根引用访问对象来工作。如果对象变得不可访问，它们可以从程序内存中释放以回收内存空间。

Python 有一个用于查找引用循环的最小跟踪垃圾收集器，但实际上使用引用计数作为主要的内存管理机制。这在普通 Python 代码中不是问题，但在编写 C 扩展时会增加一些实质性的工作。让我们在下一节更深入地探讨这个主题。

> 跟踪垃圾收集是一种非常常见的垃圾收集策略，它经常被视为垃圾收集的同义词。这就是为什么有些人认为 Python 不是垃圾收集的（因为它使用引用计数作为主要的内存管理技术）而其他人则认为它是（因为它使用跟踪来查找引用循环，并且引用计数可以理解为替代垃圾收集策略）。

#### 引用计数

最后，我们来到了 Python 中内存管理的重要话题。 Python 有自己的垃圾收集器，但它的设计目的只是为了解决引用计数算法中的循环引用问题。引用计数是管理不再需要的对象的释放的主要方法。

Python/C API 文档介绍了引用的所有权，以解释它如何处理对象的释放。 Python 中的对象从不属于扩展代码，因此不能由扩展本身创建或释放。对象的实际创建由 Python 的内存管理器管理。这就是为什么我们说 Python 中的对象归内存管理器所有。

内存管理器是 CPython 解释器的内部组件，它是唯一负责为存储在私有堆中的对象分配和释放内存的组件。可以拥有的是对对象的引用。

Python 中由引用（PyObject* 指针）表示的每个对象都有一个关联的引用计数。当它变为零时，意味着没有人持有对该对象的任何有效引用，并且可以调用与其类型关联的解除分配器。 Python/C API 提供了一些用于增加和减少引用计数的宏：

- Py_INCREF() 和 Py_DECREF()：第一个增加引用计数，第二个减少它。这些宏接受不能为 NULL 的对象指针。
- Py_XINCREF() 和 Py_XDECREF()：第一个增加引用计数，第二个减少它。这些宏接受 NULL 值，因此当您不确定是否正在处理 NULL 指针时应该使用它们。

但在我们讨论它们的细节之前，我们需要了解以下与引用所有权相关的术语：

- 所有权的传递：每当我们说函数将所有权传递给引用时，这意味着它已经增加了引用计数，当不再需要对对象的引用时，调用者有责任减少计数。大多数返回新创建对象的函数，例如 Py_BuildValue，都在这样做。如果该对象将从我们的函数返回给另一个调用者，那么所有权将再次传递。在这种情况下，我们不会减少引用计数，因为这不再是我们的责任。这就是 fibonacci_py() 函数不对结果变量调用 Py_DECREF() 的原因。
- 借用引用：当函数接收到某个 Python 对象的引用作为参数时，就会发生引用的借用。除非在其范围内明确增加，否则不应在该函数中减少此类引用的引用计数。在我们的 fibonacci_py() 函数中，self 和 args 参数就是这样的借用引用，因此我们不会对它们调用 PyDECREF()。一些 Python/C API 函数也可能返回借用的引用。值得注意的例子是 PyTuple_GetItem() 和 PyList_GetItem()。人们常说这样的引用是不受保护的。除非它们将作为函数的返回值返回，否则无需处置它们的所有权。在大多数情况下，如果我们使用此类借用引用作为其他 Python/C API 调用的参数，则应格外小心。在某些情况下，可能需要使用单独的 Py_INCREF() 调用额外保护此类引用，然后再将其用作其他函数的参数，然后在不再需要时调用 Py_DECREF()。我们将在本节末尾看到这种情况的示例。
- 被盗引用：当作为调用参数提供时，Python/C API 函数也有可能窃取引用而不是借用它。这正是两个函数的情况——PyTuple_SetItem() 和 PyList_SetItem()。他们完全接管了传递给他们的引用的责任。它们不会自己增加引用计数，而是在不再需要引用时调用 Py_DECREF()。

在编写复杂的扩展时，密切关注引用计数是最困难的事情之一。在代码在多线程设置中运行之前，可能不会注意到一些不明显的问题。

另一个常见问题是由 Python 对象模型的本质以及某些函数返回借用引用的事实引起的。当引用计数变为零时，执行释放函数。对于用户定义的类，可以定义将在那个时刻调用的 \_\_del\_\_() 方法。

这可以是任何 Python 代码，它可能会影响其他对象及其引用计数。 Python 官方文档给出了以下可能受此问题影响的代码示例：

```c
void bug(PyObject *list) { 
    PyObject *item = PyList_GetItem(list, 0); 
 
    PyList_SetItem(list, 1, PyLong_FromLong(0L)); 
    PyObject_Print(item, stdout, 0); /* BUG! */ 
}
```

它看起来完全无害，但问题实际上是我们无法知道列表对象包含哪些元素。当 PyList_SetItem() 在 list[1] 索引上设置新值时，先前存储在该索引处的对象的所有权将被释放。如果它是唯一存在的引用，则引用计数将变为 0，并且对象可能会被释放。它可能是某个用户定义的类，具有 \_\_del\_\_() 方法的自定义实现。如果在这样的 \_\_del\_\_() 执行结果中， item[0] 从列表中删除，则会出现严重问题。

注意 PyList_GetItem() 返回一个借用的引用！它在返回引用之前不会调用 Py_INCREF()。因此，在该代码中，可能会使用对不再存在的对象的引用来调用 PyObject_Print()。这将导致分段错误并使 Python 解释器崩溃。

正确的方法是在我们需要它们的整个时间内保护借用的引用，因为两者之间的任何调用都有可能导致该对象的释放。即使它们看似无关，也可能发生这种情况，如以下代码所示：

```c
void no_bug(PyObject *list) { 
    PyObject *item = PyList_GetItem(list, 0); 
 
    Py_INCREF(item); 
    PyList_SetItem(list, 1, PyLong_FromLong(0L)); 
    PyObject_Print(item, stdout, 0); 
    Py_DECREF(item); 
}
```

如您所见，使用 Python/C API 用 C 语言编写 Python 扩展可能是一项挑战。特别是如果您没有使用 C 的经验。它需要大量有关 CPython 内部结构和精确内存管理的知识。但幸运的是，自定义扩展有更简单的途径。它是 Cython，它是 Python 的一种特殊方言。我们将在下一节讨论它。

### 使用 Cython 编写扩展

Cython 既是一个优化的静态编译器，也是一种编程语言的名称，它是 Python 的超集。它可用于通过将 Python 应用程序编译为机器代码来加速它们，但也可用作用 C 或 C++ 编写的代码的“包装语言”。

作为编译器，它使用 Python/C API 执行原生 Python 代码和 Cython 方言到 Python C 扩展的源到源编译。它允许您结合 Python 和 C 的强大功能，而无需手动处理 Python/C API。

作为 Python 的超集，它提供了使用静态类型、C 库的静态链接（与共享库的动态链接相反）、与 C 头文件交互的能力以及直接控制 CPython 的 GIL 的能力。

让我们首先讨论 Cython 作为源到源编译器。

#### Cython 作为源到源编译器

对于使用 Cython 创建的扩展，您将获得的主要优势是使用它提供的超集语言。无论如何，可以使用源到源编译从纯 Python 代码创建扩展。这是 Cython 的最简单方法，因为它几乎不需要对代码进行任何更改，并且可以毫不费力地提供一些显着的性能改进。

首先，为了构建 Cython 扩展，您将需要 Cython 包。它可以使用 pip 从 PyPI 安装：

```sh
$ python3 -m pip install Cython
```

Cython 提供了一个简单的 cythonize 实用函数，可以让您轻松地将编译过程与 setuptools 包集成。假设我们想将 fibonacci() 函数的纯 Python 实现编译为 C 扩展。如果它位于 fibonacci.py 模块中，则最小的 setup.py 脚本可能如下所示：

```python
from setuptools import setup 
from Cython.Build import cythonize 
 
setup( 
    name='fibonacci', 
    ext_modules=cythonize(['fibonacci.py']) 
)
```

您可以像使用普通 C 扩展一样使用 pip 安装这样的模块：

```sh
$ python3 -m pip install -e .
Installing collected packages: fibonacci
  Running setup.py develop for fibonacci
Successfully installed fibonacci
```

上面的命令以可编辑模式安装包，因此我们可以查看此过程中生成的所有文件。如果您在自己的 shell 中执行它，您可以看到它创建了一些额外的构建工件：

```sh
$ ls -1ap
./
../
build/
fibonacci.c
fibonacci.cpython-39-darwin.so
fibonacci.egg-info/
fibonacci.py
setup.py
```


前面输出中的 fibonacci.c 是自动生成的 C 扩展代码。 Cython 将纯 Python 代码转换为原始 C 代码。在安装过程中，此 C 代码将用于构建扩展模块库。在我们的例子中，它是 fibonacci.cpython-39-darwin.so 文件。

> 您可以查看 fibonacci.c 文件，了解 Cython 在幕后做了多少工作。它实际上很长。对于我们简单的 fibonacci.py 模块，它甚至可以超过 4000 行。

Cython 用作 Python 语言的源代码编译工具时，还有另一个好处。扩展的源到源编译可以是源分发安装过程的一个完全可选的部分。如果需要安装包的环境没有 Cython 或其他任何构建先决条件，则可以将其安装为普通的纯 Python 包。用户不应注意到以这种方式分发的代码的行为有任何功能差异。分发使用 Cython 构建的扩展的一种常见方法是包含 Python/Cython 源和将从这些源文件生成的 C 代码。

这样，根据构建先决条件的存在，可以通过以下三种不同的方式安装包：

- 如果安装环境有 Cython 可用，则扩展 C 代码是从提供的 Python/Cython 源生成的。
- 如果 Cython 不可用但有可用的构建先决条件（C 编译器、Python/C API 标头），则扩展是从分布式预生成的 C 文件构建的。
- 如果上述两个都不可用，但扩展是从纯 Python 源创建的，则模块像普通 Python 代码一样安装，并跳过编译步骤。

请注意，Cython 文档说包括生成的 C 文件以及 Cython 源代码是分发 Cython 扩展的推荐方式。同一份文档说默认情况下应该禁用 Cython 编译，因为用户在他们的环境中可能没有所需的 Cython 版本，这可能会导致意外的编译问题。

> 您可以在 https://cython.readthedocs.io/src/userguide/source_files_and_compilation.html 阅读有关分发 Cython 代码的官方指南的更多信息。

无论如何，随着环境隔离的出现，这在今天似乎不再是一个令人担忧的问题。此外，Cython 是 PyPI 上可用的有效 Python 包，因此可以轻松地将其定义为特定版本中的项目需求。当然，包括这样一个先决条件是一个具有严重影响的决定，应该非常仔细地考虑。更安全的解决方案是利用 setuptools 包中的 extras_require 功能的强大功能，并允许用户决定是否要使用具有特定环境变量的 Cython，例如：

```python
import os 
 
from setuptools import setup, Extension
try: 
    # cython source to source compilation
    # available only when Cython is available
    # and specific environment variable says 
    # explicitly that Cython should be used 
    # to generate C sources 
    USE_CYTHON = bool(os.environ.get("USE_CYTHON")) 
    import Cython 
    
 
except ImportError: 
    USE_CYTHON = False 
 
ext = '.pyx' if USE_CYTHON else '.c' 
 
extensions = [Extension("fibonacci", ["fibonacci"+ext])] 
 
if USE_CYTHON: 
    from Cython.Build import cythonize 
    extensions = cythonize(extensions) 
 
setup( 
    name='fibonacci', 
    ext_modules=extensions, 
    extras_require={ 
        # Cython will be set in that specific version 
        # as a requirement if package will be installed 
        # with '[with-cython]' extra feature 
        'with-cython': ['cython==0.29.22'] 
    } 
)
```

pip 安装工具通过在包名后添加 [extra-name] 后缀来支持使用 extras 选项安装包。对于前面的示例，可以使用以下命令启用从本地源安装期间的可选 Cython 要求和编译：

```python
$ USE_CYTHON=1 pip install .[with-cython]
```

USE_CYTHON 环境变量保证 pip 将使用 Cython 将 .pyx 源编译为 C 并且 [with-cython] 保证在安装之前实际下载 Cython 编译器。

尽管您可以使用 Cython 编译纯 Python 代码，但您将从使用 Cython 方言中获得最大好处。它有一些在普通 Python 中不可用的附加功能。在下一节中，我们将仔细研究 Cython 作为一种单独的语言。

#### Cython 作为一种语言

Cython 不仅是一个编译器，也是 Python 语言的超集。 Superset 意味着允许任何有效的 Python 代码，但它可以通过附加功能进一步增强，例如支持调用 C 函数或在变量和类属性上声明 C 类型。因此，任何用 Python 编写的代码也是用 Cython 编写的，但反过来并不总是正确的。这解释了为什么使用 Cython 编译器可以如此轻松地将普通 Python 模块编译为 C。

但我们不会停留在这个简单的事实上。与其只是说我们的参考 fibonacci() 函数也是 Cython 代码，我们将尝试对其进行一些改进。这不会是任何真正的优化，因为我们仍然希望递归地实现我们的斐波那契数列。但是我们会做一些小的更新，让它从用 Cython 编写中受益更多。

Cython 源使用不同的文件扩展名。它是 .pyx 而不是 .py。 fibonacci.pyx 文件的内容可能如下所示：

```python
"""Cython module that provides fibonacci sequence function."""
def fibonacci(unsigned int n):
    """Return nth Fibonacci sequence number computed recursively."""
    if n == 0:
        return 0
    if n == 1:
        return 1
    else:
        return fibonacci(n - 1) + fibonacci(n - 2)
```

如您所见，唯一真正改变的是 fibonacci() 函数的签名。感谢 Cython 中可选的静态类型，我们可以将 n 参数声明为 unsigned int，这应该会稍微改进我们函数的工作方式。此外，它比我们以前手动编写扩展时所做的要多得多。如果 Cython 函数的参数声明为静态类型，则扩展将通过引发适当的异常来自动处理转换和溢出错误。以下是一个交互式会话示例，展示了我们用 Cython 编写的 fibonacci() 函数如何处理转换和溢出错误：

```python
>>> from fibonacci import fibonacci
>>> fibonacci(5)
5
>>> fibonacci(0)
0
>>> fibonacci(-1)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "fibonacci.pyx", line 4, in fibonacci.fibonacci
    def fibonacci(unsigned int n):
OverflowError: can't convert negative value to unsigned int
>>> fibonacci(10 ** 10)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "fibonacci.pyx", line 4, in fibonacci.fibonacci
    def fibonacci(unsigned int n):
OverflowError: value too large to convert to unsigned int
```

我们已经知道 Cython 只编译源代码，并且生成的代码使用与我们手动编写扩展的 C 代码时使用的相同的 Python/C API。请注意 fibonacci() 是一个递归函数，因此它经常调用自己。这意味着虽然我们为输入参数声明了一个静态类型，但在递归调用期间，它会将自己视为任何其他 Python 函数。所以 n-1 和 n-2 将被打包回 Python 对象，然后传递到内部 fibonacci() 实现的隐藏包装层，这将再次将其带回无符号 int 类型。这将一次又一次地发生，直到我们达到最终的递归深度。这不一定是一个问题，但涉及比实际需要更多的参数处理。

我们可以通过将更多的工作委托给对 Python 结构一无所知的纯 C 函数，从而减少 Python 函数调用和参数处理的开销。我们之前在使用纯 C 创建 C 扩展时这样做过，我们也可以在 Cython 中这样做。我们可以使用 cdef 关键字来声明仅接受和返回 C 类型的 C 风格函数，如下所示：

```python
cdef long long fibonacci_cc(unsigned int n):
    if n == 0:
        return 0
    if n == 1:
        return 1
    else:
        return fibonacci_cc(n - 1) + fibonacci_cc(n - 2)
def fibonacci(unsigned int n):
    """ Return nth Fibonacci sequence number computed recursively
    """
    return fibonacci_cc(n)
```

fibonacci_cc() 函数将无法导入到最终编译的 fibonacci 模块中。 fibonacci() 函数形成了低级 fibonacci_cc() 实现的外观。

我们还可以走得更远。通过一个普通的 C 示例，我们终于展示了如何在调用纯 C 函数期间释放 GIL，以便扩展对于多线程应用程序更好一些。在前面的示例中，我们使用了 Python/C API 标头中的 Py_BEGIN_ALLOW_THREADS 和 Py_BEGIN_ALLOW_THREADS 预处理器宏来将一段代码标记为没有 Python 调用。 Cython 语法更短且更容易记住。可以使用简单的 with nogil 语句围绕代码部分发布 GIL，如下所示：

```python
def fibonacci(unsigned int n): 
    """ Return nth Fibonacci sequence number computed recursively 
    """ 
    with nogil: 
        return fibonacci_cc(n)
```

您还可以将整个 C 风格的函数标记为无需 GIL 即可安全调用，如下所示：

```python
cdef long long fibonacci_cc(unsigned int n) nogil: 
    if n < 2: 
        return n 
    else: 
        return fibonacci_cc(n - 1) + fibonacci_cc(n - 2)
```

重要的是要知道此类函数不能将 Python 对象作为参数或返回类型。每当标记为 nogil 的函数需要执行任何 Python/C API 调用时，它必须使用 with gil 语句获取 GIL。

我们已经知道两种创建 Python 扩展的方法：使用带有 Python/C API 的普通 C 代码和使用 Cython。第一个以相当复杂和冗长的代码为代价为您提供最大的功能和灵活性，第二个使编写扩展更容易，但在背后做了很多魔法。我们还了解了扩展的一些潜在优势，因此是时候仔细研究一些潜在的缺点了。

## 使用扩展的缺点

说实话，我开始我的 Python 冒险只是因为我厌倦了用 C 和 C++ 编写软件的所有困难。事实上，当程序员意识到其他语言不能提供用户需要的东西时，他们开始学习 Python 是很常见的。

与 C、C++ 或 Java 相比，使用 Python 编程是轻而易举的。一切似乎都很简单，而且设计得很好。您可能认为没有可以绊倒的地方，也不再需要其他编程语言。

当然，没有什么比这更错的了。是的，Python 是一种了不起的语言，具有许多很酷的特性，并且它被用于许多领域。但这并不意味着它是完美的，没有任何缺点。它易于理解和编写，但这种简单性是有代价的。它并不像许多人想象的那么慢，但永远不会像 C 那样快。它具有高度的可移植性，但它的解释器不像其他一些语言的编译器那样在那么多的体系结构上可用。我们可以继续这个清单一段时间。

解决该问题的解决方案之一是编写扩展。这使我们有能力将旧 C 的一些优点带回 Python。在大多数情况下，它运行良好。问题是——我们真的使用 Python 是因为我们想用 C 扩展它吗？答案是不。在我们没有任何更好的选择的情况下，这只是一种不方便的必需品。

扩展总是伴随着成本，使用扩展的最大缺点之一是增加了复杂性。

### 额外的复杂性

用多种不同的语言开发应用程序并不是一件容易的事，这已经不是什么秘密了。 Python 和 C 是完全不同的技术，很难找到它们的共同点。也确实没有没有错误的应用程序。如果扩展在您的代码库中变得普遍，调试就会变得很痛苦。不仅因为调试 C 代码需要完全不同的工作流程和工具，还因为您需要经常在两种不同语言之间切换上下文。

我们都是人，我们的认知能力都是有限的。当然，也有能够同时高效处理多层抽象的人，但他们似乎是一个非常罕见的样本。无论您的技术多么熟练，维护此类混合解决方案总是要付出额外的代价。这将涉及在 C 和 Python 之间切换所需的额外努力和时间，或者会导致最终降低效率的额外压力。

根据 TIOBE 指数，C 仍然是最流行的编程语言之一。尽管如此，Python 程序员对其知之甚少或几乎一无所知是很常见的。就个人而言，我认为 C 应该是编程世界的通用语言，但我的观点是不太可能改变这件事。

Python 也是如此诱人且易于学习，这意味着许多程序员忘记了他们以前的所有经验，并完全转向新技术。编程不像骑自行车。如果不充分使用和打磨，这种特殊技能会很快被侵蚀。即使是具有强大 C 背景的程序员，如果他们决定长时间潜入 Python，也可能会逐渐失去他们以前的 C 熟练程度。

以上所有导致一个简单的结论——很难找到能够理解和扩展你的代码的人。对于开源包，这意味着更少的自愿贡献者。在封闭源代码中，这意味着并非所有队友都能够在不破坏事物的情况下开发和维护扩展。在扩展中调试损坏的东西肯定比在纯 Python 代码中更难。

### 更难调试

当遇到故障时，扩展可能会非常严重地中断。人们可能会认为静态类型为您提供了许多优于 Python 的优势，并允许您在编译步骤中发现很多在 Python 中很难注意到的问题。即使没有严格的测试程序和完整的测试覆盖率，这种情况也可能发生。但这只是硬币的一面。

另一方面，我们拥有所有必须手动执行的内存管理。错误的内存管理是 C 中大多数编程错误的主要原因。在最好的情况下，此类错误只会导致一些内存泄漏，逐渐吞噬您所有的环境资源。最好的情况并不意味着容易处理。如果不使用适当的外部工具（如 Valgrind），内存泄漏真的很难找到。在大多数情况下，扩展代码中的内存管理问题将导致无法在 Python 中恢复的分段错误，并且会导致解释器崩溃而不会引发可以解释原因的异常。这意味着您最终将需要配备大多数 Python 程序员通常不需要使用的其他工具。这增加了您的开发环境和工作流程的复杂性。

使用扩展的缺点意味着它们并不总是连接 Python 与其他语言的最佳工具。如果您唯一需要做的就是与已经构建的共享库进行交互，有时最好的选择是使用完全不同的方法。下一节讨论在不使用扩展的情况下与动态库交互的方法。

## 与没有扩展的动态库接口

多亏了 ctypes（标准库中的一个模块）或 cffi（PyPI 上可用的外部包），你可以在 Python 中集成每个编译的动态/共享库，无论它是用什么语言编写的。你可以在纯Python 没有任何编译步骤。这两个包被称为外部函数库。它们是用 C 编写自己的扩展的有趣替代方案。

尽管使用外部函数库不需要编写 C 代码，但这并不意味着您不需要了解任何关于 C 的知识即可有效地使用它们。 ctypes 和 cffi 都要求您对 C 以及动态库的一般工作方式有合理的理解。另一方面，它们消除了处理 Python 引用计数的负担，并大大降低了犯下痛苦错误的风险。此外，通过 ctypes 或 cffi 与 C 代码交互比编写和编译 C 扩展模块更具可移植性。

我们先来看看ctypes，它是Python标准库的一部分。

### ctypes 模块

ctypes 模块是最流行的模块，无需编写自定义 C 扩展即可从动态或共享库调用函数。原因很明显。它是标准库的一部分，因此它始终可用并且不需要任何外部依赖项。

使用共享库中的代码的第一步是加载它。让我们看看如何使用 ctypes 做到这一点。

#### 加载库
在 ctypes 中有四种类型的动态库加载器可用，并且有两种使用它们的约定。代表动态库和共享库的类是 ctypes.CDLL、ctypes.PyDLL、ctypes.OleDLL 和 ctypes.WinDLL。它们之间的区别如下：

- ctypes.CDLL：这个类代表加载的共享库。这些库中的函数使用标准调用约定并假定返回 int 类型。 GIL 在通话期间被释放。
- ctypes.PyDLL：该类的工作方式与 ctypes.CDLL 类似，但在调用过程中不会释放 GIL。执行后，会检查 Python 错误标志，如果在执行期间设置了该标志，则会引发异常。仅当加载的库直接从 Python/C API 调用函数或使用可能是 Python 代码的回调函数时才有用。
- ctypes.OleDLL：此类仅在 Windows 上可用。这些库中的函数使用 Windows 的 stdcall 调用约定并返回有关调用成功或失败的特定于 Windows 的 HRESULT 代码。 Python 将在指示失败的结果代码后自动引发 OSError 异常。
- ctypes.WinDLL：此类仅在 Windows 上可用。默认情况下，这些库中的函数使用 Windows 的 stdcall 调用约定并返回 int 类型的值。 Python 不会自动检查这些值是否表示失败。

要加载库，您可以使用适当的参数实例化前面的类之一，或者从与特定类关联的子模块调用 LoadLibrary() 函数：

- ctypes.cdll.LoadLibrary() 用于 ctypes.CDLL
- ctypes.pydll.LoadLibrary() 用于 ctypes.PyDLL
- ctypes.windll.LoadLibrary() 用于 ctypes.WinDLL
- ctypes.oledll.LoadLibrary() 用于 ctypes.OleDLL

加载共享库时的主要挑战是如何以可移植的方式找到它们。不同的系统对共享库使用不同的后缀（Windows 上的 .dll，macOS 上的 .dylib，Linux 上的 .so）并在不同的地方搜索它们。这方面的主要违规者是 Windows，它没有预定义的库命名方案。因此，我们不会讨论在此系统上使用 ctypes 加载库的细节，而将主要集中在 Linux 和 macOS 上，它们以一致且类似的方式处理此问题。

> 如果您对 Windows 平台感兴趣，请参阅官方 ctypes 文档，其中包含大量有关支持该系统的信息。它可以在 https://docs.python.org/3/library/ctypes.html 上找到。

两种库加载约定（LoadLibrary() 函数和特定的库类型类）都要求您使用完整的库名称。这意味着需要包含所有预定义的库前缀和后缀。例如，要在 Linux 上加载 C 标准库，您需要编写以下内容：

```python
>>> import ctypes
>>> ctypes.cdll.LoadLibrary('libc.so.6')
<CDLL 'libc.so.6', handle 7f0603e5f000 at 7f0603d4cbd0>
```

在这里，对于 macOS，这将是以下内容：

```python
>>> import ctypes
>>> ctypes.cdll.LoadLibrary('libc.dylib')
```

幸运的是，ctypes.util 子模块提供了一个 find_library() 函数，它允许您使用库的名称加载库，而无需任何前缀或后缀，并且可以在任何具有用于命名共享库的预定义方案的系统上工作：

```python
>>> import ctypes
>>> from ctypes.util import find_library
>>> ctypes.cdll.LoadLibrary(find_library('c'))
<CDLL 'libc.so.6', handle 7f2e82f12000 at 0x7f2e8288e220>
>>> ctypes.cdll.LoadLibrary(find_library('bz2'))
<CDLL 'libbz2.so.1.0', handle 55fb3c2d1660 at 0x7f2e827e8af0>
```

因此，如果您正在编写一个应该在 macOS 和 Linux 下都可以使用的 ctypes 包，请始终使用 ctypes.util.find_library()。

加载共享库后，就可以使用其功能了。下一节将解释使用 ctypes 调用 C 函数。

#### 使用 ctypes 调用 C 函数

当动态/共享库成功加载到 Python 对象时，常见的模式是将其存储为模块级变量，其名称与加载的库的名称相同。这些函数可以作为对象属性访问，因此调用它们就像从任何其他导入的模块调用 Python 函数，例如：

```python
>>> import ctypes
>>> from ctypes.util import find_library
>>> libc = ctypes.cdll.LoadLibrary(find_library('c'))
>>> libc.printf(b"Hello world!\n")
Hello world!
13
```

不幸的是，除整数、字符串和字节之外的所有内置 Python 类型都与 C 数据类型不兼容，因此必须包装在 ctypes 模块提供的相应类中。以下是来自 ctypes 文档的兼容数据类型的完整列表：

| ctypes type  | C type                        | Python type        |
| ------------ | ----------------------------- | ------------------ |
| c_bool       | _Bool                         | bool               |
| c_char       | char                          | 1-character bytes  |
| c_wchar      | wchar_t                       | 1-character string |
| c_byte       | char                          | int                |
| c_ubyte      | unsigned char                 | int                |
| c_short      | short                         | int                |
| c_ushort     | unsigned short                | int                |
| c_int        | int                           | int                |
| c_uint       | unsigned int                  | int                |
| c_long       | long                          | int                |
| c_ulong      | unsigned long                 | int                |
| c_longlong   | __int64 or long long          | int                |
| c_ulonglong  | unsigned __int64 or long long | int                |
| c_size_t     | size_t                        | int                |
| c_ssize_t    | ssize_t or Py_ssize_t         | int                |
| c_float      | float                         | float              |
| c_double     | double                        | float              |
| c_longdouble | long double                   | float              |
| c_char_p     | char* (NULL-terminated)       | bytes or None      |
| c_wchar_p    | wchar_t* (NULL-terminated)    | string or None     |
| c_void_p     | void*                         | int or None        |

如您所见，上表不包含将任何 Python 集合反映为 C 数组的专用类型。为 C 数组创建类型的推荐方法是简单地使用具有所需基本 ctypes 类型的乘法运算符，如下所示：

```python
>>> import ctypes
>>> IntArray5 = ctypes.c_int * 5
>>> c_int_array = IntArray5(1, 2, 3, 4, 5)
>>> FloatArray2 = ctypes.c_float * 2
>>> c_float_array = FloatArray2(0, 3.14)
>>> c_float_array[1]
3.140000104904175
```

上面的语法适用于每个基本的 ctypes 类型。

让我们在下一节中看看 Python 函数是如何作为 C 回调传递的。

#### 将 Python 函数作为 C 回调传递

将函数实现的部分工作委托给用户提供的自定义回调是一种非常流行的设计模式。接受此类回调的 C 标准库中最著名的函数是 qsort() 函数，它提供了快速排序算法的通用实现。您不太可能希望使用此算法而不是 CPython 解释器中实现的默认 TimSort，后者更适合对 Python 集合进行排序。无论如何，qsort() 似乎是高效排序算法和 C API 的典型示例，该 C API 使用许多编程书籍中的回调机制。这就是为什么我们将尝试将其用作将 Python 函数作为 C 回调传递的示例。

普通的 Python 函数类型将与 qsort() 规范要求的回调函数类型不兼容。这是来自 BSD 手册页的 qsort() 签名，其中还包含接受的回调类型（比较参数）的类型：

```c
void qsort(void *base, size_t nel, size_t width, 
           int (*compar)(const void*, const void *));
```

因此，为了从 libc 执行 qsort()，您需要传递以下内容：

- base：这是需要排序为 void* 指针的数组。
- nel：这是元素的数量，如 size_t。
- width：这是数组中单个元素的大小，如 size_t。
- compar：这是指向应该返回 int 并接受两个 void* 指针的函数的指针。它指向比较正在排序的两个元素的大小的函数。

我们已经从使用 ctypes 调用 C 函数部分知道如何使用乘法运算符从其他 ctypes 类型构造 C 数组。 nel 应该是 size_t 并且映射到 Python int，因此它不需要任何额外的包装并且可以作为 len(iterable) 传递。一旦我们知道基本数组的类型，就可以使用 ctypes.sizeof() 函数获得宽度值。我们需要知道的最后一件事是如何创建指向与 compar 参数兼容的 Python 函数的指针。

ctypes 模块包含一个 CFUNCTYPE() 工厂函数，它允许您包装 Python 函数并将它们表示为 C 可调用函数指针。第一个参数是包装函数应该返回的 C 返回类型。

其后是函数接受作为参数的 C 类型的变量列表。与 qsort() 的 compar 参数兼容的函数类型如下：

```c
CMPFUNC = ctypes.CFUNCTYPE( 
    # return type 
    ctypes.c_int, 
    # first argument type 
    ctypes.POINTER(ctypes.c_int), 
    # second argument type 
    ctypes.POINTER(ctypes.c_int), 
)
```

> CFUNCTYPE() 使用 cdecl 调用约定，因此它仅与 CDLL 和 PyDLL 共享库兼容。使用 WinDLL 或 OleDLL 加载的 Windows 上的动态库使用 stdcall 调用约定。这意味着必须使用另一个工厂将 Python 函数包装为 C 可调用函数指针。在 ctypes 中，它是 WINFUNCTYPE()。

总而言之，让我们假设我们要使用标准 C 库中的 qsort() 函数对随机打乱的整数列表进行排序。这是示例脚本，它展示了如何使用我们迄今为止所学的关于 ctypes 的所有内容来做到这一点：

```python
from random import shuffle 
 
import ctypes 
from ctypes.util import find_library 
 
libc = ctypes.cdll.LoadLibrary(find_library('c')) 
 
CMPFUNC = ctypes.CFUNCTYPE( 
    # return type 
    ctypes.c_int, 
    # first argument type 
    ctypes.POINTER(ctypes.c_int), 
    # second argument type 
    ctypes.POINTER(ctypes.c_int), 
) 
 
 
def ctypes_int_compare(a, b): 
    # arguments are pointers so we access using [0] index 
    print(" %s cmp %s" % (a[0], b[0])) 
 
    # according to qsort specification this should return: 
    # * less than zero if a < b 
    # * zero if a == b 
    # * more than zero if a > b 
    return a[0] - b[0] 
 
 
def main(): 
    numbers = list(range(5)) 
    shuffle(numbers) 
    print("shuffled: ", numbers) 
 
    # create new type representing array with length 
    # same as the length of numbers list 
    NumbersArray = ctypes.c_int * len(numbers) 
    # create new C array using a new type 
    c_array = NumbersArray(*numbers) 
 
    libc.qsort( 
        # pointer to the sorted array 
        c_array, 
        # length of the array 
        len(c_array), 
        # size of single array element 
        ctypes.sizeof(ctypes.c_int), 
        # callback (pointer to the C comparison function) 
        CMPFUNC(ctypes_int_compare) 
    ) 
    print("sorted:   ", list(c_array)) 
 
 
if __name__ == "__main__": 
    main()
```

作为回调提供的比较函数有一个额外的打印语句，因此我们可以看到它在排序过程中是如何执行的，如下所示：

```sh
$ python3 ctypes_qsort.py 
shuffled:  [4, 3, 0, 1, 2]
 4 cmp 3
 4 cmp 0
 3 cmp 0
 4 cmp 1
 3 cmp 1
 0 cmp 1
 4 cmp 2
 3 cmp 2
 1 cmp 2
sorted:    [0, 1, 2, 3, 4]
```

当然，在 Python 中使用 qsort 没有多大意义，因为 Python 有自己专门的排序算法。无论如何，将 Python 函数作为 C 回调传递是集成许多第三方库的一项非常有用的技术。

ctypes 模块在 Python 程序员中非常流行，因为它是标准库的一部分。它的缺点是许多低级类型处理和一些与加载的库交互所需的样板。这就是为什么一些开发人员更喜欢使用第三方包 CFFI 的原因，它可以简化外部函数调用的使用。我们将在下一节中介绍它。

## CFFI

CFFI 是 Python 的外部函数接口，是 ctypes 的有趣替代品。它不是标准库的一部分，但可以从 PyPI 作为 cffi 包轻松获得。它与 ctypes 不同，因为它更强调重用纯 C 声明，而不是在单个模块中提供广泛的 Python API。它要复杂得多，而且还具有一项功能，可让您使用 C 编译器自动将集成层的某些部分编译为扩展。这意味着它可以用作一种混合解决方案，填补普通 C 扩展和 ctypes 之间的空白。

因为是一个非常大的项目，不可能几段简单介绍一下。另一方面，如果不多说一点，那就太可惜了。我们已经讨论了一个使用 ctypes 集成标准库中的 qsort() 函数的示例。因此，显示这两种解决方案之间主要差异的最佳方式是使用 cffi 重新实现相同的示例。

我希望以下一段代码比几段文字更有价值：

```python
from random import shuffle 
 
from cffi import FFI 
 
ffi = FFI() 
 
ffi.cdef(""" 
void qsort(void *base, size_t nel, size_t width, 
           int (*compar)(const void *, const void *)); 
""") 
C = ffi.dlopen(None) 
 
 
@ffi.callback("int(void*, void*)") 
def cffi_int_compare(a, b): 
    # Callback signature requires exact matching of types. 
    # This involves less magic than in ctypes 
    # but also makes you more specific and requires 
    # explicit casting 
    int_a = ffi.cast('int*', a)[0] 
    int_b = ffi.cast('int*', b)[0] 
    print(" %s cmp %s" % (int_a, int_b)) 
 
    # according to qsort specification this should return: 
    # * less than zero if a < b 
    # * zero if a == b 
    # * more than zero if a > b 
    return int_a - int_b 
 
 
def main(): 
    numbers = list(range(5)) 
    shuffle(numbers) 
    print("shuffled: ", numbers) 
 
    c_array = ffi.new("int[]", numbers) 
 
    C.qsort( 
        # pointer to the sorted array 
        c_array, 
        # length of the array 
        len(c_array), 
        # size of single array element 
        ffi.sizeof('int'), 
        # callback (pointer to the C comparison function) 
        cffi_int_compare, 
    ) 
    print("sorted:   ", list(c_array)) 
 
if __name__ == "__main__": 
    main()
```

输出将类似于前面讨论 ctypes 中的 C 回调示例时呈现的输出。使用 CFFI 将 qsort 集成到 Python 中并没有比出于相同目的使用 ctypes 更有意义。无论如何，前面的示例应该显示 ctypes 和 cffi 在处理数据类型和函数回调方面的主要区别。

## 概括

本章解释了本书中最复杂的主题之一。我们讨论了构建 Python 扩展作为连接 Python 与其他语言的一种方式的原因和工具。我们首先编写仅依赖于 Python/C API 的纯 C 扩展，然后使用 Cython 重新实现它，以展示如果您只选择合适的工具，它会变得多么容易。

仍然有一些原因使事情变得困难并且只使用纯 C 编译器和 Python.h 头文件。无论如何，最好的建议是使用 Cython 之类的工具，因为这将使您的代码库更具可读性和可维护性。它还将使您免于因不谨慎的引用计数和内存管理不当而导致的大多数问题。

我们对扩展的讨论以 ctypes 和 CFFI 作为解决集成共享库问题的替代方法的介绍结束。因为它们不需要编写自定义扩展来从编译后的二进制文件中调用函数，所以它们应该是您集成闭源动态/共享库的首选工具——尤其是当您不需要使用自定义 C 代码时。

在最后几章中，我们讨论了多个复杂的主题。从高级设计模式，到并发和事件驱动编程，再到 Python 与不同语言的桥接。现在我们将继续讨论维护 Python 应用程序的主题：从测试和质量保证到打包、监控和优化任何规模的应用程序。

软件维护最重要的挑战之一是如何确保我们编写的代码是正确的。随着我们的软件不可避免地变得更加复杂，如果没有有组织的测试制度，就很难确保它正常工作。随着它变得越来越大，如果没有任何自动化，就不可能对其进行有效测试。这就是为什么在下一章中，我们将研究各种 Python 工具和技术，这些工具和技术可让您自动化测试和质量流程。
