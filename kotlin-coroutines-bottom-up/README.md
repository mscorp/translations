# 自底向上学习`Kotlin`协程

--------------------------------------

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [0. 关键要点](#0-%E5%85%B3%E9%94%AE%E8%A6%81%E7%82%B9)
- [1. 协程的理解挑战](#1-%E5%8D%8F%E7%A8%8B%E7%9A%84%E7%90%86%E8%A7%A3%E6%8C%91%E6%88%98)
- [2. 示例应用（服务端）](#2-%E7%A4%BA%E4%BE%8B%E5%BA%94%E7%94%A8%E6%9C%8D%E5%8A%A1%E7%AB%AF)
- [3. 示例应用（客户端）](#3-%E7%A4%BA%E4%BE%8B%E5%BA%94%E7%94%A8%E5%AE%A2%E6%88%B7%E7%AB%AF)
- [4. 访问`REST endpoint`的服务](#4-%E8%AE%BF%E9%97%AErest-endpoint%E7%9A%84%E6%9C%8D%E5%8A%A1)
- [5. 开始我们的探索](#5-%E5%BC%80%E5%A7%8B%E6%88%91%E4%BB%AC%E7%9A%84%E6%8E%A2%E7%B4%A2)
- [6. 延续传递风格（`Continuation Passing Style`/`CPS`）](#6-%E5%BB%B6%E7%BB%AD%E4%BC%A0%E9%80%92%E9%A3%8E%E6%A0%BCcontinuation-passing-stylecps)
- [7. 暂停还是不暂停 —— 这是一个值得考虑的问题](#7-%E6%9A%82%E5%81%9C%E8%BF%98%E6%98%AF%E4%B8%8D%E6%9A%82%E5%81%9C--%E8%BF%99%E6%98%AF%E4%B8%80%E4%B8%AA%E5%80%BC%E5%BE%97%E8%80%83%E8%99%91%E7%9A%84%E9%97%AE%E9%A2%98)
- [大开关声明和标签](#%E5%A4%A7%E5%BC%80%E5%85%B3%E5%A3%B0%E6%98%8E%E5%92%8C%E6%A0%87%E7%AD%BE)
- [追踪执行](#%E8%BF%BD%E8%B8%AA%E6%89%A7%E8%A1%8C)
- [第三个调用“ fetchNewName”的请求-不暂停](#%E7%AC%AC%E4%B8%89%E4%B8%AA%E8%B0%83%E7%94%A8-fetchnewname%E7%9A%84%E8%AF%B7%E6%B1%82-%E4%B8%8D%E6%9A%82%E5%81%9C)
- [第三次调用“ fetchNewName”-暂停](#%E7%AC%AC%E4%B8%89%E6%AC%A1%E8%B0%83%E7%94%A8-fetchnewname-%E6%9A%82%E5%81%9C)
- [总结执行](#%E6%80%BB%E7%BB%93%E6%89%A7%E8%A1%8C)
- [结论](#%E7%BB%93%E8%AE%BA)
- [关于作者](#%E5%85%B3%E4%BA%8E%E4%BD%9C%E8%80%85)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

--------------------------------------

## 0. 关键要点

- `JVM`没有原生支持协程（`coroutines`）
- `Kotlin`在编译器中通过转换为状态机的方式实现协程
- `Kotlin`协程的实现使用了一个关键字，剩下的通过库来实现
- `Kotlin`使用延续传递风格（`Continuation Passing Style`/`CPS`）来实现协程
- 协程使用了调度器（`Dispatchers`），因此在`JavaFX`、`Android`、`Swing`等中使用方式略有不同

--------------------------------------

协程是一个令人着迷的主题，尽管并不是一个新话题。正如[其他地方所说的那样](https://www.youtube.com/watch?v=dWBsdh0BndM)，协程多年来已经被多次重新发现，通常是作为轻量级线程（`lightweight threading`）或『回调地狱』（`callback hell`）的解决方案。

最近在`JVM`上，协程已成为反应式编程（`Reactive Programming`）的一种替代方法。诸如[`RxJava`](https://github.com/ReactiveX/RxJava)或[`Project Reactor`](https://projectreactor.io/)之类的框架为客户端提供了一种增量处理传入信息的方式，并且对节流（`throttling`）和并行（`parallelism`）提供了广泛的支持。但是，您必须围绕反应流（`reactive streams`）上的函数式操作（`functional operations`）来重新组织代码，[在很多情况下这样做成本是大于收益的](https://www.youtube.com/watch?v=5TJiTSWktLU)。

这就是为什么像`Android`社区会对更简单的替代方案有需求的原因。`Kotlin`语言引入协程作为一个实验功能来满足这个需求，并且经过改进后已成为`Kotlin 1.3`的正式功能。`Kotlin`协程的采用范围已从`UI`开发拓展到服务器端框架（例如[`Spring 5`添加了支持](https://www.baeldung.com/spring-boot-kotlin-coroutines)），甚至是像`Arrow`之类的函数式框架（通过[`Arrow Fx`](https://arrow-kt.io/docs/effects/fx/)）。

## 1. 协程的理解挑战

不幸的是理解协程并非易事。尽管`Kotlin`专家进行了许多协程介绍分享，但主要是关于协程是什么（或应如何使用）这方面的见解和介绍。你可能会说协程是并行编程的单子🙂。

而要理解协程有挑战的其实是底层实现。在`Kotlin`协程，编译器仅实现 **_`suspend`_** 关键字，其他所有内容都由协程库处理。结果是，`Kotlin`协程非常强大和灵活，但同时也显得用无定形。对于新手来说，这是学习障碍，新手想要的是有一致的指导方针和固定的原则来学习。本文有意于提供这个基础，自底向上地介绍协程。

## 2. 示例应用（服务端）

示例应用是一个典型问题：安全有效地对`RESTful`服务进行多次调用。播放[《威利在哪里？》](https://en.wikipedia.org/wiki/Where%27s_Wally%3F)的文本版本 —— 用户要追踪一个连着一个的人名链，直到出现`Waldo`。

下面是用[`Http4k`](https://www.http4k.org/)编写的完整`RESTful`服务实现。`Http4k`是 _Marius Eriksen_ 的[著名论文](https://monkey.org/~marius/funsrv.pdf)中所写的函数式服务端架构的`Kotlin`版本实现。实现有许多其他语言，包括`Scala`（[`Http4s`](https://http4s.org/)）和`Java 8`或更高版本（[`Http4j`](https://github.com/fzakaria/http4j)）。

实现有唯一一个`endpoint`，通过`Map`实现人名链。给定一个人名，返回匹配值和200状态代码，或是返回404和错误消息。

```kotlin
fun main() {
   val names = mapOf(
       "Jane" to "Dave",
       "Dave" to "Mary",
       "Mary" to "Pete",
       "Pete" to "Lucy",
       "Lucy" to "Waldo"
   )

   val lookupName = { request: Request ->
       val name = request.path("name")
       val headers = listOf("Content-Type" to "text/plain")
       val result = names[name]
       if (result != null) {
           Response(OK)
               .headers(headers)
               .body(result)
       } else {
           Response(NOT_FOUND)
               .headers(headers)
               .body("No match for $name")
       }
   }

   routes(
       "/wheresWaldo" bind routes(
           "/{name:.*}" bind Method.GET to lookupName
       )
   ).asServer(Netty(8080))
       .start()
}
```

也就是说，我们的客户要完成的操作是执行下面的请求链：

```ruby
$ curl http://localhost:8080/wheresWaldo/Mary
Pete
$ curl http://localhost:8080/wheresWaldo/Pete
Lucy
$ curl http://localhost:8080/wheresWaldo/Lucy
Waldo
```

## 3. 示例应用（客户端）

客户端应用基于`JavaFX`库来创建桌面用户界面，为了简化任务并避免不必要的细节，使用[`TornadoFX`](https://github.com/edvin/tornadofx)，它为`JavaFX`提供了`Kotlin`的`DSL`实现。

下面是客户端视图的完整定义：

```kotlin
class HelloWorldView: View("Coroutines Client UI") {
   private val finder: HttpWaldoFinder by inject()
   private val inputText = SimpleStringProperty("Jane")
   private val resultText = SimpleStringProperty("")

   override val root = form {
       fieldset("Lets Find Waldo") {
           field("First Name:") {
               textfield().bind(inputText)
               button("Search") {
                   action {
                       println("Running event handler".addThreadId())
                       searchForWaldo()
                   }
               }
           }
           field("Result:") {
               label(resultText)
           }
       }
   }

   private fun searchForWaldo() {
       GlobalScope.launch(Dispatchers.Main) {
           println("Doing Coroutines".addThreadId())
           val input = inputText.value
           val output = finder.wheresWaldo(input)
           resultText.value = output
       }
   }
}
```

另外还使用了下面的辅助函数作为`String`类型的扩展方法：

```kotlin
fun String.addThreadId() = "$this on thread ${Thread.currentThread().id}"
```

下面是用户界面运行起来的样子：

![](images/2-1578570190101.jpg)

当用户单击按钮时，会启动一个新的协程，并通过类型为`HttpWaldoFinder`的服务对象访问`RESTful endpoint`。

`Kotlin`协程存在于`CoroutineScope`之中，`CoroutineScope`关联了表示底层并发模型的`Dispatcher`。并发模型通常是线程池，但可以是其它的。

有哪些`Dispatcher`可用取决于`Kotlin`代码的所运行环境。`Main Dispatcher`对应的是`UI`库的事件处理线程，因此（在`JVM`上）仅在`Android`、`JavaFX`和`Swing`中可用。`Kotlin Native`的协程在开始时完全不支持多线程，[但是这种情况正在改变](https://www.youtube.com/watch?v=oxQ6e1VeH4M)。在服务端，可以自己引入协程，但是缺省就可用情况会越来越多，[比如在`Spring 5`中](https://www.baeldung.com/spring-boot-kotlin-coroutines)。

在开始调用暂停方法（`suspending methods`）之前，必须要有一个协程、一个`CoroutineScope`和一个`Dispatcher`。如果是最开始的调用（如上面的代码所示），可以通过『协程构建器』（`coroutine builder`）函数（如`launch`和`async`）来启动这个过程。

调用协程构建器函数或诸如`withContext`之类的上下文函数总会创建一个新的`CoroutineScope`。在这个上下文中，一个执行任务对应的是由`Job`实例构成的一个层次结构。

任务具有一些有趣的属性，即：

- `Job`在自己完成之前，会等待自己区域中的所有协程完成。
- 取消`Job`导致其所有子`Job`被取消。
- `Job`的失败或取消会传播给他的父`Job`。

这样的设计是为了避免并发编程中的常见问题，例如在没有终止子任务的情况下终止了父任务。

## 4. 访问`REST endpoint`的服务

下面是`HttpWaldoFinder`服务的完整代码：

```kotlin
class HttpWaldoFinder : Controller(), WaldoFinder {
   override suspend fun wheresWaldo(starterName: String): String {
       val firstName = fetchNewName(starterName)
       println("Found $firstName name".addThreadId())

       val secondName = fetchNewName(firstName)
       println("Found $secondName name".addThreadId())

       val thirdName = fetchNewName(secondName)
       println("Found $thirdName name".addThreadId())

       val fourthName = fetchNewName(thirdName)
       println("Found $fourthName name".addThreadId())

       return fetchNewName(fourthName)
   }

   private suspend fun fetchNewName(inputName: String): String {
       val url = URI("http://localhost:8080/wheresWaldo/$inputName")
       val client = HttpClient.newBuilder().build()
       val handler = HttpResponse.BodyHandlers.ofString()
       val request = HttpRequest.newBuilder().uri(url).build()

       return withContext<String>(Dispatchers.IO) {
           println("Sending HTTP Request for $inputName".addThreadId())
           client
               .send(request, handler)
               .body()
       }
   }
}
```

`fetchNewName`函数的参数是已知人名，在`endpoint`中查询关联的人名。通过`HttpClient`类完成，这个类在`Java 11`及以后版本的标准库中。实际的`HTTP GET`操作在使用`IO Dispatcher`的新子协程中运行。`IO Dispatcher`代表了为长时间运行的活动（如网络调用）优化的线程池。

`wheresWaldo`函数追踪人名称链五次，以（期望能）找到`Waldo`。因为后面要反汇编生成的字节码，所以逻辑实现得尽可能简单。我们感兴趣的是，每次调用`fetchNewName`都会导致当前协程被挂起，尽管它的子协程会在运行。在这种特殊情况下，父协程在`Main Dispatcher`上运行，而子协程在`IO Dispatcher`上运行。因此，尽管子协程在执行`HTTP`请求，`UI`事件处理线程已经释放了，以处理其他用户与视图的交互。如下图所示：

![](images/3-1578570191290.jpg)

当执行`suspend`调用时，`IntelliJ`会提示在协程之间有控制权转移。请注意，如果不切换`Dispatcher`，则调用不一定会导致新协程的创建。当一个`suspend`函数调用另一个`suspend`函数时，可以在同一协程中继续执行，实际上，如果处于在同一线程上，这就是我们想要的行为。

![](images/4-1578570189602.jpg)

当执行客户端时，控制台的输出如下：

```sh
Sending HTTP Request for Lucy on thread 24
Running event handler on thread 17
Doing Coroutines on thread 17
Sending HTTP Request for Jane on thread 24
Found Dave name on thread 17
Sending HTTP Request for Dave on thread 24
Found Mary name on thread 17
Sending HTTP Request for Mary on thread 24
Found Pete name on thread 17
Sending HTTP Request for Pete on thread 26
Found Lucy name on thread 17
Sending HTTP Request for Lucy on thread 26
```

可以看到，对于上面的这次运行，`Main Dispatcher`/`UI`事件`Handler`在线程17上运行，而`IO Dispatcher`在包含线程24和26的线程池上运行。

## 5. 开始我们的探索

使用`IntelliJ`自带的字节码反汇编工具，可以窥探底层的实际情况。注意，也可以使用`JDK`自带的标准`javap`工具。

![](images/6-1578570190679.jpg)

可以看到`HttpWaldoFinder`的方法签名已经改变了，因此可以接受`continuation`对象作为额外的参数，并返回一个通用的`Object`。

```java
public final class HttpWaldoFinder extends Controller implements WaldoFinder {
  public Object wheresWaldo(String a, Continuation b)

  final synthetic Object fetchNewName(String a, Continuation b)
}
```

现在，让我们深入研究添加到这些方法中的代码，并解释`Continuation`是什么以及返回的是什么。

## 6. 延续传递风格（`Continuation Passing Style`/`CPS`）

正如`Kotlin`标准化流程（也称为`KEEP`）中的[协程提案](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md)所记录的，协程的实现基于『延续传递风格』。延续（`continuation`）对象用于存储函数在挂起期间所需的状态。

暂停函数的每个局部变量都成为延续的字段。另外还需要为各个函数的参数和当前对象（如果函数是方法）创建字段。因此，有四个参数和五个局部变量的暂停方法有至少十个字段的延续。

对于`HttpWaldoFinder`中的`wheresWaldo`方法，只有一个参数和四个局部变量，因此延续实现类型具有六个字段。如果将`Kotlin`编译器生成的字节码反汇编为`Java`源代码，可以看到确实如此：

```java
$continuation = new ContinuationImpl($completion) {
  Object result;
  int label;
  Object L$0;
  Object L$1;
  Object L$2;
  Object L$3;
  Object L$4;
  Object L$5;

  @Nullable
  public final Object invokeSuspend(@NotNull Object $result) {
     this.result = $result;
     this.label |= Integer.MIN_VALUE;
     return HttpWaldoFinder.this.wheresWaldo((String)null, this);
  }
};
```

由于所有字段都为`Object`类型，因此如何使用它们并不是很明显。但是随着我们进一步探索，可以看到：

- `L$0`持有对`HttpWaldoFinder`实例的引用。始终有值。
- `L$1`持有`starterName`参数的值。始终有值。
- `L$2`到`L$5`持有局部变量的值。在代码执行时逐步填充。`L$2`将持有`firstName`的值，依此类推。

另外还有用于最终结果的字段，以及一个名为`label`的引人注意的整数字段。

## 7. 暂停还是不暂停 —— 这是一个值得考虑的问题

在检查生成的代码时，我们需要记住，它必须处理两个用例。每次挂起函数调用另一个函数时，它要么挂起当前协程（以便另一个可以在同一线程上运行），要么继续执行当前协程。

考虑一个挂起函数，该函数从数据存储中读取一个值。它很可能会在发生I / O时挂起，但也可能会缓存结果。后续调用可以同步返回缓存的值，而不会暂停。Kotlin编译器生成的代码必须允许两个路径。

Kotlin编译器会调整每个挂起函数的返回类型，以便它可以返回实际结果或特殊值 _`COROUTINE_SUSPENDED`_。在后一种情况下，当前的协程被暂停。这就是挂起函数的返回类型从结果类型更改为“对象”的原因。

在我们的示例应用程序中，“ wheresWaldo”将重复调用“ fetchNewName”。从理论上讲，这些调用中的每一个都可以挂起或不挂起当前的协程。自从我们写了“ fetchNewName”以来，我们知道总是会发生暂停。但是，要理解所生成的代码，我们必须记住，它需要能够处理所有可能性。

## 大开关声明和标签

如果进一步查看反汇编的代码，我们会发现埋在多个嵌套标签中的switch语句。这是状态机的实现，用于控制wheresWaldo（）方法中的不同悬浮点。这是高层结构：

```java
// listing one: the generated switch statement and labels
String firstName;
String secondName;
String thirdName;
String fourthName;
Object var11;
Object var10000;
label48: {
  label47: {
     label46: {
        Object $result = $continuation.result;
        var11 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
        switch($continuation.label) {
        case 0:
            // code omitted
        case 1:
            // code omitted
        case 2:
            // code omitted
        case 3:
            // code omitted
        case 4:
            // code omitted
        case 5:
            // code omitted
        default:
           throw new IllegalStateException(
                "call to 'resume' before 'invoke' with coroutine");
        } // end of switch
        // code omitted
    } // end of label 46
    // code omitted
  } // end of label 47
  // code omitted
} // end of label 48
// code omitted
```

现在，我们可以在续篇中看到“标签”字段的用途。当我们完成“ wheresWaldo”的不同阶段时，我们将更改“ label”中的值。嵌套的标签块包含原始Kotlin代码中悬浮点之间的代码块。此“标签”值允许重新输入该代码，跳至上次暂停的位置（适当的case语句）以从延续中提取一些数据，然后中断至正确的带标签的块。

但是，如果我们的所有暂停点都没有真正暂停，则可以同步执行整个块。在生成的代码中，我们经常发现以下片段：

```java
// listing two - deciding if the current coroutine should suspend
if (var10000 == var11) {
  return var11;
}
```

正如我们在上面看到的，“ var11”已设置为*`CONTINUATION_SUSPENDED`*值，而“ var10000”保存了从调用到另一个挂起函数的返回值。因此，当发生挂起时，代码将返回（稍后再输入），如果未发生挂起，则代码将中断到适当的带标签的块，从而继续执行功能的下一部分。

再一次，请记住，生成的代码不能假定所有调用都将挂起，或者所有调用都将继续使用当前协程。它必须能够应付任何可能的组合。

## 追踪执行

当我们开始执行时，连续中的'label'值将设置为零。这是switch语句的相应分支：

```java
// listing three - the first branch of the switch
case 0:
  ResultKt.throwOnFailure($result);
  $continuation.L$0 = this;
  $continuation.L$1 = starterName;
  $continuation.label = 1;
  var10000 = this.fetchNewName(starterName, $continuation);
  if (var10000 == var11) {
     return var11;
  }
  break;
```

我们将实例和参数存储到延续对象中，然后将延续对象传递给'fetchNewName'。如前所述，编译器生成的“ fetchNewName”版本将返回实际结果或 _`COROUTINE_SUSPENDED`_ 值。

如果协程被挂起，那么我们从函数中返回，并且当我们继续时跳转到'case 1'分支。如果我们继续使用当前的协程，那么我们将跳出标记框之一的开关，转到下面的代码：

```java
// listing four - calling ‘fetchNewName’ for the second time
firstName = (String)var10000;
secondName = UtilsKt.addThreadId("Found " + firstName + " name");
boolean var13 = false;
System.out.println(secondName);
$continuation.L$0 = this;
$continuation.L$1 = starterName;
$continuation.L$2 = firstName;
$continuation.label = 2;
var10000 = this.fetchNewName(firstName, $continuation);
if (var10000 == var11) {
  return var11;
}
```

因为我们知道'var10000'包含我们想要的返回值，所以我们可以将其转换为正确的类型并将其存储在本地变量'firstName'中。然后，生成的代码使用变量'secondName'存储连接线程ID的结果，然后将其打印出来。

我们将更新延续中的字段，并添加我们从服务器检索到的值。请注意，“ label”的值现在为2。然后，我们第三次调用“ fetchNewName”。

## 第三个调用“ fetchNewName”的请求-不暂停

我们必须再次基于'fetchNewName'返回的值进行选择，如果返回的值是_`COROUTINE_SUSPENDED`_，那么我们将从当前函数返回。下次调用时，我们将遵循开关的“案例2”分支。

如果我们继续当前的协程，则执行下面的代码块。正如您所看到的，它与上面的相同，除了我们现在有更多数据要存储在延续中。

```java
// listing four - calling ‘fetchNewName’ for the third time
secondName = (String)var10000;
thirdName = UtilsKt.addThreadId("Found " + secondName + " name");
boolean var14 = false;
System.out.println(thirdName);
$continuation.L$0 = this;
$continuation.L$1 = starterName;
$continuation.L$2 = firstName;
$continuation.L$3 = secondName;
$continuation.label = 3;
var10000 = this.fetchNewName(secondName, (Continuation)$continuation);
if (var10000 == var11) {
  return var11;
}
```

对于所有剩余的调用（假设从未返回 _`COROUTINE_SUSPENDED`_），此模式将重复进行，直到到达结尾为止。

## 第三次调用“ fetchNewName”-暂停

或者，如果协程已被暂停，那么这是我们将要运行的案例块：

```java
// listing five - the third branch of the switch
case 2:
  firstName = (String)$continuation.L$2;
  starterName = (String)$continuation.L$1;
  this = (HttpWaldoFinder)$continuation.L$0;
  ResultKt.throwOnFailure($result);
  var10000 = $result;
  break label46;
```

我们从延续中提取值到函数的局部变量中。然后使用标记的中断将执行跳转到上面的清单4。因此，最终我们将在同一个地方结束。

## 总结执行

现在，我们可以重新访问代码结构清单，并对每个部分中发生的事情进行高级描述：

```java
// listing six - the generated switch statement and labels in depth
String firstName;
String secondName;
String thirdName;
String fourthName;
Object var11;
Object var10000;
label48: {
  label47: {
     label46: {
        Object $result = $continuation.result;
        var11 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
        switch($continuation.label) {
       case 0:
            // set label to 1 and make the first call to ‘fetchNewName’
            // if suspending return, otherwise break from the switch
        case 1:
            // extract the parameter from the continuation
            // break from the switch
        case 2:
            // extract the parameter and first result from the continuation
            // break to outside ‘label46’
        case 3:
            // extract the parameter, first and second results from the
            //   continuation
            // break to outside ‘label47’
        case 4:
            // extract the parameter, first, second and third results from
            //   the continuation
            // break to outside ‘label48’
        case 5:
            // extract the parameter, first, second, third and fourth
            //   results from the continuation
            // return the final result
        default:
           throw new IllegalStateException(
                "call to 'resume' before 'invoke' with coroutine");
        } // end of switch
        // store the parameter and first result in the continuation
        // set the label to 2 and make the second call to ‘fetchNewName’
        // if suspending return, otherwise proceed
    } // end of label 46
        // store the parameter, first and second results in the
        //   continuation
        // set the label to 3 and make the third call to ‘fetchNewName’
        // if suspending return, otherwise proceed
  } // end of label 47
        // store the parameter, first, second and third results in the
        //   continuation
        // set the label to 4 and make the fourth call to ‘fetchNewName’
        // if suspending return, otherwise proceed
} // end of label 48
// store the parameter, first, second, third and fourth results in the continuation
// set the label to 5 and make the fifth call to ‘fetchNewName’
// return either the final result or COROUTINE_SUSPENDED
```

## 结论

这不是一个容易理解的代码库。我们正在研究从`Kotlin`编译器中的代码生成器生成的字节码反汇编的`Java`代码。此代码生成器的输出旨在提高效率和简约性，而非清晰度。

但是，我们可以得出一些有用的结论：

1. **没有魔术**。当开发人员首次了解协程时，很容易假设存在一些特殊的“魔术”将所有东西捆绑在一起。如我们所见，生成的代码仅使用过程编程的基本构建块，例如条件和标记的中断。
2. **实现是基于延续的**。如原始`KEEP`提案中所述，通过在对象内缓存功能的状态来暂停和恢复功能。因此，对于每个挂起函数，编译器将创建一个具有`N`个字段的延续类型，其中`N`是参数数量加上字段数量加上3。最后三个保存当前对象，最终结果和索引。
3. **执行始终遵循标准模式**。如果要从暂停中恢复，则可以使用延续的`label`字段跳转到`switch`语句的适当分支。在此分支中，我们检索到到延续对象为止的数据，然后使用标记的中断跳转到如果没有发生暂停将直接执行的代码。

## 关于作者

<img src="images/Garth-Gilmour.jpg" hspace="10px" align="left" >

**[Garth Gilmour](https://twitter.com/GarthGilmour)** 是`Instil`的学习主管。他在1999年放弃了全职开发工作，先是向`C`编码员讲授`C++`，然后是`Java`到`C++`编码员，再由`C＃`教给`Java`编码器，现在向所有人讲授所有东西，但更喜欢在`Kotlin`工作。如果他算出交货的话，那一定会超过1000年前。他是20多门课程的作者，经常在聚会上演讲，在国家和国际会议上发表演讲，并共同组织了贝尔法斯特`BASH`系列开发人员活动和最近成立的`Kotlin`贝尔法斯特用户组。不在白板上时，他执教 _Krav Maga_ 并举重。

<img src="images/Eamonn-Boyle.jpg" hspace="10px" align="left" >

**[Eamonn Boyle](https://twitter.com/BoyleEamonn)** 从事开发人员，建筑师和团队领导超过15年。在过去的大约四年中，他一直是一名全职培训师和教练，为各种各样的代表撰写和提供有关各种主题的课程。这些包括从核心语言技能，框架到工具和过程的范式和技术。他还在包括`Goto`和`KotlinConf`在内的许多会议，活动和聚会中演讲并举办了研讨会。
