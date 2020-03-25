# awaitility 使用手册

原文链接：[https://github.com/awaitility/awaitility/wiki/Usage](https://github.com/awaitility/awaitility/wiki/Usage)

- [awaitility 使用手册](#awaitility-%e4%bd%bf%e7%94%a8%e6%89%8b%e5%86%8c)
  - [Static imports](#static-imports)
  - [Usage examples](#usage-examples)
  - [Simple](#simple)
  - [Reusing Conditions](#reusing-conditions)
  - [Fields](#fields)
  - [Atomic, Adders and Accumulators](#atomic-adders-and-accumulators)
  - [Advanced](#advanced)
  - [Lambdas](#lambdas)
  - [Using AssertJ or Fest Assert](#using-assertj-or-fest-assert)
  - [Ignoring Exceptions](#ignoring-exceptions)
  - [Checked exceptions in Runnable lambda expressions](#checked-exceptions-in-runnable-lambda-expressions)
  - [At Least](#at-least)
    - [Ignoring uncaught exceptions](#ignoring-uncaught-exceptions)
    - [Assert that a value is maintained](#assert-that-a-value-is-maintained)
  - [Thread Handling](#thread-handling)
  - [Exception handling](#exception-handling)
  - [Deadlock detection](#deadlock-detection)
  - [Defaults](#defaults)
  - [Polling](#polling)
  - [Fixed Poll Interval](#fixed-poll-interval)
  - [Fibonacci Poll Interval](#fibonacci-poll-interval)
  - [Iterative Poll Interval](#iterative-poll-interval)
  - [Custom Poll Interval](#custom-poll-interval)
  - [Condition Evaluation Listener](#condition-evaluation-listener)
  - [Duration](#duration)
  - [Important](#important)
  - [Links and code examples](#links-and-code-examples)

## Static imports

为了有效地使用Awaitility，建议从Awaitility框架中静态导入以下方法：

- `org.awaitility.Awaitility.*`

导入以下方法也可能有用：

- `java.time.Duration.*`
- `java.util.concurrent.TimeUnit.*`
- `org.hamcrest.Matchers.*`
- `org.junit.Assert.*`

## Usage examples

## Simple

假设我们向异步系统发送 “add user” 消息，如下所示：

```java
publish(new AddUserMessage("Awaitility Rocks"));
```

在您的测试用例中，Awaitility 可以帮助您轻松地验证数据库是否已更新。最简单的形式可能是这样的：

```java
await().until(newUserIsAdded());
```

`newUserIsAdded` 是在测试用例中自己实现的方法。它指定必须满足的条件，以便 Awaitility 停止等待。

```java
private Callable<Boolean> newUserIsAdded() {
    // The condition that must be fulfilled
    return () -> userRepository.size() == 1;
}
```

当然，您也可以内联 `newUserIsAdded` 方法，只需编写：

```java
await().until(() -> userRepository.size() == 1);
```

默认情况下，Awaitility 将等待10秒，如果在此期间用户资源库的大小不等于1，它将抛出一个 [`ConditionTimeoutException`](http://static.javadoc.io/org.awaitibility/awaitibility/4.0.2/org/awaitibility/core/ConditionTimeoutException.html)，测试失败。如果需要不同的超时，可以如下定义：

```java
await().atMost(5, SECONDS).until(newUserWasAdded());
```

## Reusing Conditions

Awaitility 还支持将条件拆分为结果生成和匹配两部分部分，以便更好地重用。上面的例子也可以写成：

```java
await().until( userRepositorySize(), equalTo(1) );
```

`userRepositorySize` 方法现在返回 `Callable<Integer>`。

```java
private Callable<Integer> userRepositorySize() {
    // The suppling part of the condition
      return () -> userRepository.size();
}
```

`equalTo` 是一个标准的 [Hamcrest](http://code.google.com/p/hamcrest/) 匹配器，指定 Awaitility 的匹配部分。

现在我们可以在不同的测试中重用 `userRepositorySize`。假设我们有一个同时增加三个用户的测试：

```java
publish(new AddUserMessage("User 1"), new AddUserMessage("User 2"), new AddUserMessage("User 3"));
```

我们现在重用 `userRepositorySize`，只需更新 Hamcrest 匹配器：

```java
await().until( userRepositorySize(), equalTo(3) );
```

在这种简单的情况下，您当然也可以使用Java方法引用：

```java
await().until(userRepository::size, equalTo(3) );
```

## Fields

您还可以通过引用字段来构建供应部分。例如：

```java
await().until( fieldIn(object).ofType(int.class), equalTo(2) );
```

or:

```java
await().until( fieldIn(object).ofType(int.class).andWithName("fieldName"), equalTo(2) );
```

or:

```java
await().until( fieldIn(object).ofType(int.class).andAnnotatedWith(MyAnnotation.class), equalTo(2) );
```

## Atomic, Adders and Accumulators

如果您正在使用[Atomic](http://download.oracle.com/javase/1,5.0/docs/api/java/util/concurrent/atomic/package-summary.html)数据结构，Awaitility 提供了一种简单的方法来等待它们匹配特定值：

```java
AtomicInteger atomic = new AtomicInteger(0);
// Do some async stuff that eventually updates the atomic integer
await().untilAtomic(atomic, equalTo(1));
```

等待一个 AtomicBoolean 更简单：

```java
AtomicBoolean atomic = new AtomicBoolean(false);
// Do some async stuff that eventually updates the atomic boolean
await().untilTrue(atomic);
```

如果您正在使用 Adders，例如 [LongAdder](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/LongAdder.html)，则Awaitility 可让您简单地等待使其达到一定的值：

```java
await().untilAdder(myLongAdder, equalTo(5L))
```

同样，如果使用累加器，例如 [LongAccumulator](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/LongAccumulator.html)，则可以执行以下操作：

```java
await().untilAccumulator(myLongAccumulator, equalTo(5L))
```

## Advanced

使用100毫秒的轮询间隔，初始延迟为20毫秒，直到客户状态等于 “REGISTERED”。本示例还通过指定别名（“customer registration”）来使用命名的 await。如果您在同一测试中有多个 await，那么很容易找出哪个等待语句失败。

```java
with().pollInterval(ONE_HUNDERED_MILLISECONDS).and().with().pollDelay(20, MILLISECONDS).await("customer registration").until(
            customerStatus(), equalTo(REGISTERED));
```

您还可以指定这样的别名：

```java
await().with().alias("my alias"). ..
```

要使用非固定的轮询间隔，请参考 [轮询间隔](＃polling)文档。

## Lambdas

您可以在条件中使用 lambda 表达式：

```java
await().atMost(5, SECONDS).until(() -> userRepository.size() == 1);
```

或方法引用：

```java
await().atMost(5, SECONDS).until(userRepository::isNotEmpty);
```

或方法引用和 Hamcrest 匹配器的组合：

```java
await().atMost(5, SECONDS).until(userRepository::size, is(1));
```

您还可以使用谓词：

```java
await().atMost(5, SECONDS).until(userRepository::size, size -> size == 1);
```

有关示例，请参阅 [Jayway小组博客](http://www.jayway.com/2014/04/23/java-8-and-assertj-support-in-awaitility-1-6-0/")。

## Using AssertJ or Fest Assert

您可以使用 [AssertJ](http://joel-costigliola.github.io/assertj/)或 [Fest Assert](https://code.google.com/p/fest/) 代替 Hamcrest（实际上可以使用任何在出错时引发异常的第三方库）。

```java
await().atMost(5, SECONDS).untilAsserted(() -> assertThat(fakeRepository.getValue()).isEqualTo(1));
```

## Ignoring Exceptions

在条件评估期间忽略某些类型的异常有时很有用。例如，如果您在等待到达最终状态之前正在等待将异常作为中间状态抛出的事件。以Spring的 [SocketUtils](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/util/SocketUtils.html) 类为例，该类使您可以在给定范围查找 TCP 端口。如果在给定范围内没有端口可用，它将抛出异常。假设我们知道给定范围内的某些端口不可用，但我们要等待它们可用。这是一个示例，我们可以选择忽略“SocketUtils”引发的任何异常。例如：

```java
given().ignoreExceptions().await().until(() -> SocketUtils.findAvailableTcpPort(x,y));
```

这指示 Awaitility 在条件评估期间忽略所有捕获的异常。异常将被视为评估失败。与提供的异常类型匹配的异常，测试不会失败（除非超时）。您还可以忽略特定的异常：

```java
given().ignoreException(IllegalStateException.class).await().until(() -> SocketUtils.findAvailableTcpPort(x,y));
```

或使用 Hamcrest 匹配器：

```java
given().ignoreExceptionsMatching(instanceOf(RuntimeException.class)).await().until(() -> SocketUtils.findAvailableTcpPort(x,y));
```

或使用谓词（Java 8）：

```java
given().ignoreExceptionsMatching(e -> e.getMessage().startsWith("Could not find an available")).await().until(something());
```

您也可以忽略 `Throwable` 实例。

## Checked exceptions in Runnable lambda expressions

Java中的 `Runnable` 接口不允许您抛出已检查的异常。因此，如果您有这样的方法：

```java
public void waitUntilCompleted() throws Exception { ... }
```

可能会引发异常，如果 `untilAsserted` 将 `Runnable` 作为其参数值，则您必须捕获该异常：

```java
await().untilAsserted(() -> {
   try {
      waitUntilCompleted();
   } catch(Exception e) {
     // Handle exception
   }
});
```

幸运的是，Awaitility 通过引入传递给 `untilAsserted` 的 [ThrowingRunnable](http://static.javadoc.io/org.awaitility/awaitility/4.0.2/org/awaitility/core/ThrowingRunnable.html) 接口来解决此问题。而不是 `Runnable`。因此，您需要编写的代码如下所示：

```java
await().untilAsserted(() -> waitUntilCompleted());
```

## At Least

您可以指定 awaitility **最少** 等待一定时间。例如：

```java
await().atLeast(1, SECONDS).and().atMost(2, SECONDS).until(value(), equalTo(1));
```

如果在由 atLeast 指定的持续时间之前满足条件，则会引发异常，条件不应早于指定的时间完成。

### Ignoring uncaught exceptions

如果要将代码从使用 `Thread.sleep` 迁移到 Awaitility，请注意，在某些情况下，由于其他线程抛出异常，因此 Awaitility 测试用例可能会失败。这是因为默认情况下，Awaitility 捕获所有未捕获的异常。因此，如果您以前使用过 `Thread.sleep`，那么很有可能您没有捕获其他线程的异常。如果您对此行为感到满意，并且不希望 Awaitility 捕获这些异常，则可以使用 `dontCatchUncaughtExceptions` 禁用此功能：

```java
@Test
public void dontCatchUncaughtExample() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setMaxPoolSize(1);
    executor.afterPropertiesSet();

    executor.execute(new ErrorTestTask());
    ListenableFuture<?> future = executor.submitListenable(new ErrorTestTask());

    Awaitility.await()
              .dontCatchUncaughtExceptions()
              .atMost(1, TimeUnit.SECONDS)
              .pollInterval(10, TimeUnit.MILLISECONDS)
              .until(future::isDone);
}

private static class ErrorTestTask implements Runnable {
    @Override
    public void run() {
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        throw new RuntimeException(Thread.currentThread().getName() + " -> Error");
    }
}
```

### Assert that a value is maintained

从4.0.2版开始，可以断言某个值在特定时间段内得到维护。例如，如果您需要确保存储库中的值在 1500毫秒中的 800毫秒内保持特定值：

```java
await().during(800, MILLISECONDS).atMost(1500, MILLISECONDS).until(() -> myRepository.findById("id"), equalTo("something"));
```

Awaitility 将最多等待1500毫秒，而这样做的话，`myRepository.findById(id)` 必须等于 `something` 至少 800毫秒。

## Thread Handling

Awaitility 允许进行细粒度的线程配置。这是通过提供 Awaility 轮询条件时将使用的线程提供者或 `ExecutorService` 来完成的。请注意，这是一项高级功能，应谨慎使用。例如：

```java
given().pollThread(Thread::new).await().atMost(1000, MILLISECONDS).until(..);
```

另一种方法是指定 `ExecutorService`：

```java
ExecutorSerivce es = ...
given().pollExecutorService(es).await().atMost(1000, MILLISECONDS).until(..);
```

例如，如果您需要等待轮询 [ThreadLocal](https://docs.oracle.com/javase/7/docs/api/java/lang/ThreadLocal.html) 变量的条件，这将很有用。

在某些情况下，重要的是能够指示 Awaitility 使用与启动 Awaitility 的测试用例相同的线程。因此，Awaitility 3.0.0 引入了 [pollInSameThread](http://static.javadoc.io/org.awaitility/awaitility/4.0.2/org/awaitility/core/ConditionFactory.html#pollInSameThread--) 方法：

```java
with().pollInSameThread().await().atMost(1000, MILLISECONDS).until(...);
```

这是一项高级功能，在将 `pollInSameThread` 与永远等待（或长时间）的条件结合使用时应格外小心，因为当 Awaitility 使用与测试相同的线程时，它不会中断该线程。

## Exception handling

默认情况下，Awaitility 捕获所有线程中未捕获的 `Throwable`，并将其传播到等待线程。这意味着您的测试用例将指示失败，即使不是引发未捕获异常的测试线程也是如此。

您可以选择忽略某些异常或可抛出对象，请参见 [here](＃ignoring-exceptions)。

如果不需要在所有线程中都捕获异常，则可以使用 [dontCatchUncaughtExceptions](#ignoring-uncaught-exceptions)。

## Deadlock detection

Awaitility自动检测死锁，并将 [ConditionTimeoutException](http://static.javadoc.io/org.awaitility/awaitility/4.0.2/org/awaitility/core/ConditionTimeoutException.html) 的原因与 [DeadlockException](http://static.javadoc.io/org.awaitility/awaitility/4.0.2/org/awaitility/core/DeadlockException.html)。 `DeadlockException` 包含有关导致死锁的线程的信息。

## Defaults

如果未指定任何超时，则 Awaitility 将等待10秒钟，然后引发 [ConditionTimeoutException](http://static.javadoc.io/org.awaitility/awaitility/4.0.2/org/awaitility/core/ConditionTimeoutException.html)（如果条件尚未满足）。默认轮询间隔和轮询延迟为100毫秒。您还可以自己使用以下命令指定默认值：

```java
  Awaitility.setDefaultTimeout(..)
  Awaitility.setDefaultPollInterval(..)
  Awaitility.setDefaultPollDelay(..)
```

您还可以使用 `Awaitility.reset` 将其重置为默认值。

## Polling

请注意，由于 Awaitility 使用轮询来验证条件是否匹配，因此不建议将其用于精确的性能测试。在这些情况下，最好使用 AOP 框架，例如 AspectJ。

另请注意，[Duration.ZERO](http://static.javadoc.io/org.awaitility/awaitility/4.0.2/org/awaitility/Duration.html#ZERO) 用作所有非固定轮询间隔的起始值间隔。对于固定的轮询间隔，出于向后兼容的原因，轮询延迟等于 `FixedPollInterval` 的持续时间。

有关其他详细信息，请参见 [this blog](http://code.haleby.se/2015/11/27/non-fixed-poll-intervals-in-awaitility/)。

## Fixed Poll Interval

这是 Awaitilty 的默认轮询间隔机制。以正常方式使用DSL时，例如：

```java
with().pollDelay(100, MILLISECONDS).and().pollInterval(200, MILLISECONDS).await().until(<condition>);
```

Awaitility 将使用 [FixedPollInterval](http://static.javadoc.io/org.awaitility/awaitility/4.0.2/org/awaitility/pollinterval/FixedPollInterval.html)。这意味着 Awaitility 将在 poll delay（轮询开始之前的初始延迟，在上面的示例中为100ms）之后首次检查是否满足指定条件。除非明确指定，否则 Awaitility 将使用与轮询间隔相同的轮询延迟（请注意，这仅适用于固定轮询间隔，如上例所示）。这意味着它将首先在给定的轮询延迟后定期检查条件，然后再以给定的轮询间隔进行检查；那就是在 pollDelay 之后检查条件，然后pollDelay + pollInterval，然后pollDelay +（2 *pollInterval），依此类推。如果更改轮询间隔，则轮询延迟也将更改为与指定的轮询间隔相匹配，除非您已明确指定了轮询延迟。

## Fibonacci Poll Interval

[FibonacciPollInterval](http://static.javadoc.io/org.awaitility/awaitility/4.0.2/org/awaitility/pollinterval/FibonacciPollInterval.html) 会根据斐波那契序列生成一个非线性轮询间隔。用法示例：

```java
with().pollInterval(fibonacci()).await().until(..);
```

其中的 `fibonacci()`是从 `org.awaitility.pollinterval.FibonacciPollInterval` 静态导入的。这将生成一个 "1、2、3、5、8、13，..." 毫秒的轮询间隔。您可以配置要使用的时间单位，例如秒而不是毫秒：

```java
with().pollInterval(fibonacci(TimeUnit.SECONDS)).await().until(..);
```

或使用 english-like 的配置方式

```java
with().pollInterval(fibonacci().with().timeUnit(SECONDS).and().offset(5)).await().until(..);
```

偏移量表示斐波那契序列从该偏移量开始（默认情况下，偏移量为0）。偏移量也可以是负数（-1）以0开头（`fib（0）`= 0）。

## Iterative Poll Interval

由函数和开始持续时间生成的轮询间隔。该函数可以在持续时间内自由地执行任何操作。

例如：

```java
await().with().pollInterval(iterative(duration -> duration.multiply(2)), Duration.FIVE_HUNDRED_MILLISECONDS).until(..);
```

或使用 english-like 的配置方式

```java
await().with().pollInterval(iterative(duration -> duration.multiply(2)).with().startDuration(FIVE_HUNDRED_MILLISECONDS)).until(..);
```

这将生成一个轮询间隔序列，看起来像这样（ms）：`500，1000，2000，4000，8000，16000，...`

请注意，如果指定轮询初始延迟，则此延迟将在此轮询间隔生成第一个轮询间隔之前。有关更多信息，请参见[javadoc](http://static.javadoc.io/org.awaitility/awaitility/4.0.2/org/awaitility/pollinterval/IterativePollInterval.html)。

## Custom Poll Interval

通过实现 [PollInterval](http://static.javadoc.io/org.awaitility/awaitility/4.0.2/org/awaitility/pollinterval/PollInterval.html) 接口，可以滚动自己的轮询间隔。这是一个功能接口，因此在Java 8中，您可以像这样进行操作：

```java
await().with().pollInterval((__, previous) -> previous.multiply(2).plus(1)).until(..);
```

在此示例中，我们创建一个`PollInterval`，该函数被实现为（bi-）函数，该函数采用先前的轮询间隔持续时间并将其乘以2并加1。`__` 只是表示我们不在乎 [PollInterval](http://static.javadoc.io/org.awaitility/awaitility/4.0.2/org/awaitility/pollinterval/PollInterval.html) 提供的轮询计数。当创建的轮询间隔不是（仅）对前一个持续时间感兴趣，而是根据其被调用的次数生成其持续时间时，需要轮询计数。例如，[FibonacciPollInterval](http://static.javadoc.io/org.awaitility/awaitility/4.0.2/org/awaitility/pollinterval/FibonacciPollInterval.html) 仅使用轮询计数：

```java
await().with().pollInterval((pollCount, __) -> new Duration(fib(pollCount), MILLISECONDS)).until(..);
```

在大多数情况下，没有必要从头开始实施轮询间隔。改为向 [IterativePollInterval](＃iterative-poll-interval) 提供函数。

## Condition Evaluation Listener

Awaitility 1.6.1引入了 [Condition Evaluation Listener](http://static.javadoc.io/org.awaitility/awaitility/4.0.2/org/awaitility/core/ConditionEvaluationListener.html) 的概念。每当 Awaitility 评估条件时，都可以使用它来获取信息。例如，您可以使用它来在条件满足之前获取条件的中间值。它也可以用于打印日志。例如：

```java
with().
         conditionEvaluationListener(condition -> System.out.printf("%s (elapsed time %dms, remaining time %dms)\n", condition.getDescription(), condition.getElapsedTimeInMS(), condition.getRemainingTimeInMS())).
         await().atMost(Duration.TEN_SECONDS).until(new CountDown(5), is(equalTo(0)));
```

将以下内容打印到控制台：

```java
    org.awaitility.AwaitilityJava8Test$CountDown expected (<0> or a value less than <0>) but was <5> (elapsed time 101ms, remaining time 1899ms)
    org.awaitility.AwaitilityJava8Test$CountDown expected (<0> or a value less than <0>) but was <4> (elapsed time 204ms, remaining time 1796ms)
    org.awaitility.AwaitilityJava8Test$CountDown expected (<0> or a value less than <0>) but was <3> (elapsed time 306ms, remaining time 1694ms)
    org.awaitility.AwaitilityJava8Test$CountDown expected (<0> or a value less than <0>) but was <2> (elapsed time 407ms, remaining time 1593ms)
    org.awaitility.AwaitilityJava8Test$CountDown expected (<0> or a value less than <0>) but was <1> (elapsed time 508ms, remaining time 1492ms)
    org.awaitility.AwaitilityJava8Test$CountDown reached its end value of (<0> or a value less than <0>) (elapsed time 610ms, remaining time 1390ms)
```

有一个内置的用于记录日志的 ConditionEvaluationListener，名为 [ConditionEvaluationLogger](http://static.javadoc.io/org.awaitility/awaitility/4.0.2/org/awaitility/core/ConditionEvaluationLogger.html) 可以像这样使用：

```java
with().conditionEvaluationListener(new ConditionEvaluationLogger()).await(). ...
```

Awaitility 4.0.2在 `ConditionEvaluationListener` 接口中引入了三个新的 hook（默认方法）：

| Method        | Description   |
| ------------------ |-------------|
| `beforeEvaluation`   | 在条件评估之前调用 |
| `exceptionIgnored`   | 处理条件评估时引发的被忽略异常 |
| `onTimeout`          | 当条件超时时调用     |

## Duration

Awaitility提供了一个 [Duration](http://static.javadoc.io/org.awaitility/awaitility/1.6.5/org/awaitility/Duration.html) 类，其中包含一些预定义的持续时间值，例如 `ONE_HUNDRED_MILLISECONDS`，`FIVE_SECONDS` 和 `ONE_MINUTE`。您还可以在 `Duration` 实例上执行一些基本的数学运算。例如：

```java
new Duration(5, SECONDS).plus(17, MILLISECONDS);
```

它将返回新的持续时间5017毫秒。请注意，持续时间是不可变的，因此调用 `plus` 将返回一个新实例。当使用非固定的 [轮询间隔](＃polling) 时这个比较有用。

## Important

Awaitility 并不能确保线程安全或线程同步！这是你的责任！确保您的代码已正确同步，或者使用线程安全的数据结构，例如volatile字段或类，例如 `AtomicInteger` 和 `ConcurrentHashMap`。

## Links and code examples

1. [Awaitility test case](https://github.com/awaitility/awaitility/blob/master/awaitility/src/test/java/org/awaitility/AwaitilityTest.java)
1. [Awaitility test case Java 8](https://github.com/awaitility/awaitility/blob/master/awaitility-java8-test/src/test/java/org/awaitility/AwaitilityJava8Test.java)
1. [Field supplier test case](https://github.com/awaitility/awaitility/blob/master/awaitility/src/test/java/org/awaitility/UsingFieldSupplierTest.java)
1. [Atomic test case](https://github.com/awaitility/awaitility/blob/master/awaitility/src/test/java/org/awaitility/UsingAtomicTest.java)
1. [Presentation](http://awaitility.googlecode.com/files/awaitility-khelg-2011.pdf) from [Jayway](http://www.jayway.com)'s KHelg 2011
