# JavaPoet

`JavaPoet` 是一个用于生成 `.Java` 源文件的Java API。在执行注释处理或与元数据文件（如数据库模式、协议格式）交互等操作时，源文件生成非常有用。通过生成代码，您无需编写样板文件，同时还可以为元数据保留唯一的真实源。

- [JavaPoet](#javapoet)
  - [Example](#example)
    - [Code & Control Flow](#code--control-flow)
    - [$L for Literals](#l-for-literals)
    - [$S for Strings](#s-for-strings)
    - [$T for Types](#t-for-types)
      - [Import static](#import-static)
    - [$N for Names](#n-for-names)
    - [Code block format strings](#code-block-format-strings)
      - [Relative Arguments](#relative-arguments)
      - [Positional Arguments](#positional-arguments)
      - [Named Arguments](#named-arguments)
    - [Methods](#methods)
    - [Constructors](#constructors)
    - [Parameters](#parameters)
    - [Fields](#fields)
    - [Interfaces](#interfaces)
    - [Enums](#enums)
    - [Anonymous Inner Classes](#anonymous-inner-classes)
    - [Annotations](#annotations)
    - [Javadoc](#javadoc)
  - [Download](#download)
  - [License](#license)
  - [JavaWriter](#javawriter)

## Example

这里有一个（无聊的）`helloworld` 类：

```java
package com.example.helloworld;

public final class HelloWorld {
  public static void main(String[] args) {
    System.out.println("Hello, JavaPoet!");
  }
}
```

这是用JavaPoet生成它的（激动人心的）代码：

```java
MethodSpec main = MethodSpec.methodBuilder("main")
    .addModifiers(Modifier.PUBLIC, Modifier.STATIC)
    .returns(void.class)
    .addParameter(String[].class, "args")
    .addStatement("$T.out.println($S)", System.class, "Hello, JavaPoet!")
    .build();

TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
    .addMethod(main)
    .build();

JavaFile javaFile = JavaFile.builder("com.example.helloworld", helloWorld)
    .build();

javaFile.writeTo(System.out);
```

为了声明main方法，我们创建了一个使用修饰符配置的 `MethodSpec` `main`，返回类型，参数和代码声明。我们将 main 方法添加到 `HelloWorld` 类中，然后添加到 `HelloWorld.java` 文件中。

在例子中，我们将文件写入 `System.out`，但我们也可以将其作为字符串获取（`JavaFile.toString`）或将其写入文件系统（`JavaFile.writeTo`）。

[Javadoc][javadoc] 列出了完整的 JavaPoet API，我们将在下面进行探讨。

### Code & Control Flow

JavaPoet 的大多数 API 使用普通的老式不可变 Java对象。还有构建器，方法链和可变参使 API 便于使用。JavaPoet 提供了用于类和接口（`TypeSpec`）的模型，字段（`FieldSpec`），方法和构造函数（`MethodSpec`），参数（`ParameterSpec`）和
注释（`AnnotationSpec`）。

方法体未被建模，JavaPoet 使用字符串作为代码块：

```java
MethodSpec main = MethodSpec.methodBuilder("main")
    .addCode(""
        + "int total = 0;\n"
        + "for (int i = 0; i < 10; i++) {\n"
        + "  total += i;\n"
        + "}\n")
    .build();
```

生成：

```java
void main() {
  int total = 0;
  for (int i = 0; i < 10; i++) {
    total += i;
  }
}
```

手动分号，换行和缩进非常繁琐，因此 JavaPoet 提供了API使它更容易。有一个 `addStatement`，它处理分号和换行符，并且 `beginControlFlow` + `endControlFlow（）` 一起用于花括+换行符+缩进：

```java
MethodSpec main = MethodSpec.methodBuilder("main")
    .addStatement("int total = 0")
    .beginControlFlow("for (int i = 0; i < 10; i++)")
    .addStatement("total += i")
    .endControlFlow()
    .build();
```

这个例子很蹩脚，因为生成的代码是恒定的！假设不只是将0加到10，我们要使操作和范围可配置。这是一个生成方法的方法：

```java
private MethodSpec computeRange(String name, int from, int to, String op) {
  return MethodSpec.methodBuilder(name)
      .returns(int.class)
      .addStatement("int result = 1")
      .beginControlFlow("for (int i = " + from + "; i < " + to + "; i++)")
      .addStatement("result = result " + op + " i")
      .endControlFlow()
      .addStatement("return result")
      .build();
}
```

下面就是我们调用这个时得到的 `computeRange("multiply10to20", 10, 20, "*")`:

```java
int multiply10to20() {
  int result = 1;
  for (int i = 10; i < 20; i++) {
    result = result * i;
  }
  return result;
}
```

方法生成方法！而且由于JavaPoet生成源代码而不是字节码，所以您可以通读它以确保它是正确的。

某些控制流语句，例如 `if/else`，可能具有无限的控制流可能性。您可以使用 `nextControlFlow（）` 处理这些选项：

```java
MethodSpec main = MethodSpec.methodBuilder("main")
    .addStatement("long now = $T.currentTimeMillis()", System.class)
    .beginControlFlow("if ($T.currentTimeMillis() < now)", System.class)
    .addStatement("$T.out.println($S)", System.class, "Time travelling, woo hoo!")
    .nextControlFlow("else if ($T.currentTimeMillis() == now)", System.class)
    .addStatement("$T.out.println($S)", System.class, "Time stood still!")
    .nextControlFlow("else")
    .addStatement("$T.out.println($S)", System.class, "Ok, time still moving forward")
    .endControlFlow()
    .build();
```

生成

```java
void main() {
  long now = System.currentTimeMillis();
  if (System.currentTimeMillis() < now)  {
    System.out.println("Time travelling, woo hoo!");
  } else if (System.currentTimeMillis() == now) {
    System.out.println("Time stood still!");
  } else {
    System.out.println("Ok, time still moving forward");
  }
}
```

使用 `try/catch` 捕获异常也是 `nextControlFlow` 的用例：

```java
MethodSpec main = MethodSpec.methodBuilder("main")
    .beginControlFlow("try")
    .addStatement("throw new Exception($S)", "Failed")
    .nextControlFlow("catch ($T e)", Exception.class)
    .addStatement("throw new $T(e)", RuntimeException.class)
    .endControlFlow()
    .build();
```

生成

```java
void main() {
  try {
    throw new Exception("Failed");
  } catch (Exception e) {
    throw new RuntimeException(e);
  }
}
```

### $L for Literals

调用 `beginControlFlow` 和 `addStatement` 时的字符串连接令人分心。为了解决这个问题，JavaPoet 提供了一种语法，其灵感来自但不兼容 [`String.format`][formatter]。它接受 **`$L`** 以在输出中以实际值替换。这个就像 `Formatter` 的 `％s` 一样工作：

```java
private MethodSpec computeRange(String name, int from, int to, String op) {
  return MethodSpec.methodBuilder(name)
      .returns(int.class)
      .addStatement("int result = 0")
      .beginControlFlow("for (int i = $L; i < $L; i++)", from, to)
      .addStatement("result = result $L i", op)
      .endControlFlow()
      .addStatement("return result")
      .build();
}
```

字面量直接转换到输出代码，没有转义。字面量的参数可能是字符串，基本类型和一些 JavaPoet 类型，如下所述。

### $S for Strings

当生成包含字符串的代码时，我们可以使用 **`$S`**。这是一个生成3种方法的程序，每种方法
返回其自己的名称：

```java
public static void main(String[] args) throws Exception {
  TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
      .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
      .addMethod(whatsMyName("slimShady"))
      .addMethod(whatsMyName("eminem"))
      .addMethod(whatsMyName("marshallMathers"))
      .build();

  JavaFile javaFile = JavaFile.builder("com.example.helloworld", helloWorld)
      .build();

  javaFile.writeTo(System.out);
}

private static MethodSpec whatsMyName(String name) {
  return MethodSpec.methodBuilder(name)
      .returns(String.class)
      .addStatement("return $S", name)
      .build();
}
```

这个例子中，`$S` 为我们保留了双引号。

```java
public final class HelloWorld {
  String slimShady() {
    return "slimShady";
  }

  String eminem() {
    return "eminem";
  }

  String marshallMathers() {
    return "marshallMathers";
  }
}
```

### $T for Types

Java 程序员喜欢我们的类型：它们使我们的代码更易于理解。它具有对类型的丰富内置支持，包括对 `import` 语句的生成。使用 **`$T`** 代表类型：

```java
MethodSpec today = MethodSpec.methodBuilder("today")
    .returns(Date.class)
    .addStatement("return new $T()", Date.class)
    .build();

TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
    .addMethod(today)
    .build();

JavaFile javaFile = JavaFile.builder("com.example.helloworld", helloWorld)
    .build();

javaFile.writeTo(System.out);
```

That generates the following `.java` file, complete with the necessary `import`:

```java
package com.example.helloworld;

import java.util.Date;

public final class HelloWorld {
  Date today() {
    return new Date();
  }
}
```

我们通过 `Date.class` 来引用一个在生成代码时可以使用的类。下面是一个类似的示例，但是此示例引用了一个不存在的类（尚未）：

```java
ClassName hoverboard = ClassName.get("com.mattel", "Hoverboard");

MethodSpec today = MethodSpec.methodBuilder("tomorrow")
    .returns(hoverboard)
    .addStatement("return new $T()", hoverboard)
    .build();
```

该尚不存在的类也将被导入：

```java
package com.example.helloworld;

import com.mattel.Hoverboard;

public final class HelloWorld {
  Hoverboard tomorrow() {
    return new Hoverboard();
  }
}
```

ClassName 类型非常重要，使用 JavaPoet 时经常需要它，它可以标识任何声明的类。声明的类型只是 Java 丰富的类型系统的开始：我们还具有数组，参数化类型，通配符类型和类型变量。 JavaPoet 具有用于构建以下每个类的类：

```java
ClassName hoverboard = ClassName.get("com.mattel", "Hoverboard");
ClassName list = ClassName.get("java.util", "List");
ClassName arrayList = ClassName.get("java.util", "ArrayList");
TypeName listOfHoverboards = ParameterizedTypeName.get(list, hoverboard);

MethodSpec beyond = MethodSpec.methodBuilder("beyond")
    .returns(listOfHoverboards)
    .addStatement("$T result = new $T<>()", listOfHoverboards, arrayList)
    .addStatement("result.add(new $T())", hoverboard)
    .addStatement("result.add(new $T())", hoverboard)
    .addStatement("result.add(new $T())", hoverboard)
    .addStatement("return result")
    .build();
```

JavaPoet 将分解每种类型，并在可能的情况下导入其组件。

```java
package com.example.helloworld;

import com.mattel.Hoverboard;
import java.util.ArrayList;
import java.util.List;

public final class HelloWorld {
  List<Hoverboard> beyond() {
    List<Hoverboard> result = new ArrayList<>();
    result.add(new Hoverboard());
    result.add(new Hoverboard());
    result.add(new Hoverboard());
    return result;
  }
}
```

#### Import static

JavaPoet 支持`import static`。它通过显式收集类型成员名称来实现。让我们用一些静态糖增强前面的示例：

```java
ClassName namedBoards = ClassName.get("com.mattel", "Hoverboard", "Boards");

MethodSpec beyond = MethodSpec.methodBuilder("beyond")
    .returns(listOfHoverboards)
    .addStatement("$T result = new $T<>()", listOfHoverboards, arrayList)
    .addStatement("result.add($T.createNimbus(2000))", hoverboard)
    .addStatement("result.add($T.createNimbus(\"2001\"))", hoverboard)
    .addStatement("result.add($T.createNimbus($T.THUNDERBOLT))", hoverboard, namedBoards)
    .addStatement("$T.sort(result)", Collections.class)
    .addStatement("return result.isEmpty() ? $T.emptyList() : result", Collections.class)
    .build();

TypeSpec hello = TypeSpec.classBuilder("HelloWorld")
    .addMethod(beyond)
    .build();

JavaFile.builder("com.example.helloworld", hello)
    .addStaticImport(hoverboard, "createNimbus")
    .addStaticImport(namedBoards, "*")
    .addStaticImport(Collections.class, "*")
    .build();
```

JavaPoet 首先将您的 `import static` 块添加到文件中，并根据需要导入所有其他类型。

```java
package com.example.helloworld;

import static com.mattel.Hoverboard.Boards.*;
import static com.mattel.Hoverboard.createNimbus;
import static java.util.Collections.*;

import com.mattel.Hoverboard;
import java.util.ArrayList;
import java.util.List;

class HelloWorld {
  List<Hoverboard> beyond() {
    List<Hoverboard> result = new ArrayList<>();
    result.add(createNimbus(2000));
    result.add(createNimbus("2001"));
    result.add(createNimbus(THUNDERBOLT));
    sort(result);
    return result.isEmpty() ? emptyList() : result;
  }
}
```

### $N for Names

生成的代码通常是自引用的。使用 **`$N`** 来引用同一个类中另一个生成的方法。

```java
public String byteToHex(int b) {
  char[] result = new char[2];
  result[0] = hexDigit((b >>> 4) & 0xf);
  result[1] = hexDigit(b & 0xf);
  return new String(result);
}

public char hexDigit(int i) {
  return (char) (i < 10 ? i + '0' : i - 10 + 'a');
}
```

当生成上面的代码时，我们使用 `$N` 将 `hexDigit()` 方法作为参数传递给 `byteToHex()` 方法：

```java
MethodSpec hexDigit = MethodSpec.methodBuilder("hexDigit")
    .addParameter(int.class, "i")
    .returns(char.class)
    .addStatement("return (char) (i < 10 ? i + '0' : i - 10 + 'a')")
    .build();

MethodSpec byteToHex = MethodSpec.methodBuilder("byteToHex")
    .addParameter(int.class, "b")
    .returns(String.class)
    .addStatement("char[] result = new char[2]")
    .addStatement("result[0] = $N((b >>> 4) & 0xf)", hexDigit)
    .addStatement("result[1] = $N(b & 0xf)", hexDigit)
    .addStatement("return new String(result)")
    .build();
```

### Code block format strings

代码块可以通过几种方式为其占位符指定值。同一个代码块上的每个操作只能使用相同的样式。

#### Relative Arguments

将格式字符串中每个占位符的参数值传递给 `CodeBlock.add()`。在每个示例中，我们生成代码 `I ate 3 tacos`。

```java
CodeBlock.builder().add("I ate $L $L", 3, "tacos")
```

#### Positional Arguments

在格式字符串中的占位符前放置一个整数索引（从1开始），以指定要使用的参数。

```java
CodeBlock.builder().add("I ate $2L $1L", "tacos", 3)
```

#### Named Arguments

使用语法 `$argumentName:X`，其中 `X` 是格式字符，并使用包含格式字符串中所有参数键的映射调用`CodeBlock.addNamed()`。参数名称使用 “a-z”，“A-Z”，“0-9” 和 “_” 中的字符，并且必须以小写字母开头。

```java
Map<String, Object> map = new LinkedHashMap<>();
map.put("food", "tacos");
map.put("count", 3);
CodeBlock.builder().addNamed("I ate $count:L $food:L", map)
```

### Methods

以上所有方法都有一个代码体。使用 `Modifiers.ABSTRACT` 来获得没有任何方法体的方法。仅当是抽象或接口时，这才合法。

```java
MethodSpec flux = MethodSpec.methodBuilder("flux")
    .addModifiers(Modifier.ABSTRACT, Modifier.PROTECTED)
    .build();

TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addModifiers(Modifier.PUBLIC, Modifier.ABSTRACT)
    .addMethod(flux)
    .build();
```

生成

```java
public abstract class HelloWorld {
  protected abstract void flux();
}
```

修饰符的使用需要依照Java语法使用。请注意，在指定修饰符时，JavaPoet使用 [`javax.lang.model.element.Modifier`][modifier], 这在Android上不可用。此限制仅适用于代码生成代码。生成代码随处运行：JVM，Android 和 GWT。

方法还具有参数，异常，可变参数，Javadoc，注释，类型变量和返回类型。所有这些都可以使用 `MethodSpec.Builder` 来配置.

### Constructors

`MethodSpec` 有些不太恰当；它也可以用于构造函数：

```java
MethodSpec flux = MethodSpec.constructorBuilder()
    .addModifiers(Modifier.PUBLIC)
    .addParameter(String.class, "greeting")
    .addStatement("this.$N = $N", "greeting", "greeting")
    .build();

TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addModifiers(Modifier.PUBLIC)
    .addField(String.class, "greeting", Modifier.PRIVATE, Modifier.FINAL)
    .addMethod(flux)
    .build();
```

生成：

```java
public class HelloWorld {
  private final String greeting;

  public HelloWorld(String greeting) {
    this.greeting = greeting;
  }
}
```

在大多数情况下，构造函数的工作原理与普通方法一样。生成代码时，JavaPoet 会将构造函数放在普通方法之前。

### Parameters

使用 `ParametersSpec.builder()` 或 `MethodSpec` 的 `addParameter` API 声明方法和构造函数上的参数：

```java
ParameterSpec android = ParameterSpec.builder(String.class, "android")
    .addModifiers(Modifier.FINAL)
    .build();

MethodSpec welcomeOverlords = MethodSpec.methodBuilder("welcomeOverlords")
    .addParameter(android)
    .addParameter(String.class, "robot", Modifier.FINAL)
    .build();
```

尽管上面的生成 `android` 和 `robot` 参数的代码不同，但是输出是相同的：

```java
void welcomeOverlords(final String android, final String robot) {
}
```

当参数带有注释（例如，`@Nullable`）时，扩展的 `Builder` 形式是必需的。

### Fields

像参数一样，可以使用构建器或使用方便的辅助方法来创建字段：

```java
FieldSpec android = FieldSpec.builder(String.class, "android")
    .addModifiers(Modifier.PRIVATE, Modifier.FINAL)
    .build();

TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addModifiers(Modifier.PUBLIC)
    .addField(android)
    .addField(String.class, "robot", Modifier.PRIVATE, Modifier.FINAL)
    .build();
```

生成

```java
public class HelloWorld {
  private final String android;

  private final String robot;
}
```

当字段具有 Javadoc，注释或字段初始化程序时，必须使用扩展的 `Builder` 格式。字段初始值设定项使用与上述代码块相同的 [`String.format()`][formatter] 类语法：

```java
FieldSpec android = FieldSpec.builder(String.class, "android")
    .addModifiers(Modifier.PRIVATE, Modifier.FINAL)
    .initializer("$S + $L", "Lollipop v.", 5.0d)
    .build();
```

生成

```java
private final String android = "Lollipop v." + 5.0;
```

### Interfaces

请注意，接口方法必须始终为 `PUBLIC ABSTRACT`，并且接口字段必须始终为 `PUBLIC STATIC FINAL`。定义接口时，必须使用以下修饰符：

```java
TypeSpec helloWorld = TypeSpec.interfaceBuilder("HelloWorld")
    .addModifiers(Modifier.PUBLIC)
    .addField(FieldSpec.builder(String.class, "ONLY_THING_THAT_IS_CONSTANT")
        .addModifiers(Modifier.PUBLIC, Modifier.STATIC, Modifier.FINAL)
        .initializer("$S", "change")
        .build())
    .addMethod(MethodSpec.methodBuilder("beep")
        .addModifiers(Modifier.PUBLIC, Modifier.ABSTRACT)
        .build())
    .build();
```

但是，在生成代码时将省略这些修饰符。这些是默认值，因此我们不需要为了 `javac` 的利益就包括它们！

```java
public interface HelloWorld {
  String ONLY_THING_THAT_IS_CONSTANT = "change";

  void beep();
}
```

### Enums

使用 `enumBuilder` 创建枚举类型，并为每个值添加 `addEnumConstant()`：

```java
TypeSpec helloWorld = TypeSpec.enumBuilder("Roshambo")
    .addModifiers(Modifier.PUBLIC)
    .addEnumConstant("ROCK")
    .addEnumConstant("SCISSORS")
    .addEnumConstant("PAPER")
    .build();
```

生成

```java
public enum Roshambo {
  ROCK,

  SCISSORS,

  PAPER
}
```

支持充血枚举，其中枚举值覆盖方法或调用超类构造函数。这是一个示例：

```java
TypeSpec helloWorld = TypeSpec.enumBuilder("Roshambo")
    .addModifiers(Modifier.PUBLIC)
    .addEnumConstant("ROCK", TypeSpec.anonymousClassBuilder("$S", "fist")
        .addMethod(MethodSpec.methodBuilder("toString")
            .addAnnotation(Override.class)
            .addModifiers(Modifier.PUBLIC)
            .addStatement("return $S", "avalanche!")
            .returns(String.class)
            .build())
        .build())
    .addEnumConstant("SCISSORS", TypeSpec.anonymousClassBuilder("$S", "peace")
        .build())
    .addEnumConstant("PAPER", TypeSpec.anonymousClassBuilder("$S", "flat")
        .build())
    .addField(String.class, "handsign", Modifier.PRIVATE, Modifier.FINAL)
    .addMethod(MethodSpec.constructorBuilder()
        .addParameter(String.class, "handsign")
        .addStatement("this.$N = $N", "handsign", "handsign")
        .build())
    .build();
```

Which generates this:

```java
public enum Roshambo {
  ROCK("fist") {
    @Override
    public String toString() {
      return "avalanche!";
    }
  },

  SCISSORS("peace"),

  PAPER("flat");

  private final String handsign;

  Roshambo(String handsign) {
    this.handsign = handsign;
  }
}
```

### Anonymous Inner Classes

在枚举代码中，我们使用了 `TypeSpec.anonymousInnerClass()`。匿名内部类也可以用在在代码块里。它们是可以用 `$L` 引用的值：

```java
TypeSpec comparator = TypeSpec.anonymousClassBuilder("")
    .addSuperinterface(ParameterizedTypeName.get(Comparator.class, String.class))
    .addMethod(MethodSpec.methodBuilder("compare")
        .addAnnotation(Override.class)
        .addModifiers(Modifier.PUBLIC)
        .addParameter(String.class, "a")
        .addParameter(String.class, "b")
        .returns(int.class)
        .addStatement("return $N.length() - $N.length()", "a", "b")
        .build())
    .build();

TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addMethod(MethodSpec.methodBuilder("sortByLength")
        .addParameter(ParameterizedTypeName.get(List.class, String.class), "strings")
        .addStatement("$T.sort($N, $L)", Collections.class, "strings", comparator)
        .build())
    .build();
```

这生成一个方法，该方法包含一个包含方法的类：

```java
void sortByLength(List<String> strings) {
  Collections.sort(strings, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
      return a.length() - b.length();
    }
  });
}
```

定义匿名内部类的一个特别棘手的部分是超类构造函数的参数。在上面的代码中，我们传递了没有参数的空字符串：`TypeSpec.anonymousClassBuilder()`。要传递不同的参数，请使用JavaPoet 的代码块语法和逗号分隔参数。

### Annotations

简单的注解很容易：

```java
MethodSpec toString = MethodSpec.methodBuilder("toString")
    .addAnnotation(Override.class)
    .returns(String.class)
    .addModifiers(Modifier.PUBLIC)
    .addStatement("return $S", "Hoverboard")
    .build();
```

它使用 `@Override` 注解生成此方法：

```java
  @Override
  public String toString() {
    return "Hoverboard";
  }
```

使用 `AnnotationSpec.builder()` 设置注释的属性：

```java
MethodSpec logRecord = MethodSpec.methodBuilder("recordEvent")
    .addModifiers(Modifier.PUBLIC, Modifier.ABSTRACT)
    .addAnnotation(AnnotationSpec.builder(Headers.class)
        .addMember("accept", "$S", "application/json; charset=utf-8")
        .addMember("userAgent", "$S", "Square Cash")
        .build())
    .addParameter(LogRecord.class, "logRecord")
    .returns(LogReceipt.class)
    .build();
```

生成带有 `accept` 和 `userAgent` 属性的注释：

```java
@Headers(
    accept = "application/json; charset=utf-8",
    userAgent = "Square Cash"
)
LogReceipt recordEvent(LogRecord logRecord);
```

当您喜欢时，注释值可以是注释本身。使用 `$L` 进行嵌入

```java
MethodSpec logRecord = MethodSpec.methodBuilder("recordEvent")
    .addModifiers(Modifier.PUBLIC, Modifier.ABSTRACT)
    .addAnnotation(AnnotationSpec.builder(HeaderList.class)
        .addMember("value", "$L", AnnotationSpec.builder(Header.class)
            .addMember("name", "$S", "Accept")
            .addMember("value", "$S", "application/json; charset=utf-8")
            .build())
        .addMember("value", "$L", AnnotationSpec.builder(Header.class)
            .addMember("name", "$S", "User-Agent")
            .addMember("value", "$S", "Square Cash")
            .build())
        .build())
    .addParameter(LogRecord.class, "logRecord")
    .returns(LogReceipt.class)
    .build();
```

生成

```java
@HeaderList({
    @Header(name = "Accept", value = "application/json; charset=utf-8"),
    @Header(name = "User-Agent", value = "Square Cash")
})
LogReceipt recordEvent(LogRecord logRecord);
```

请注意，您可以使用相同的属性名称多次调用 `addMember()` 来填充该属性的值列表。

### Javadoc

字段，方法和类型可以用Javadoc描述：

```java
MethodSpec dismiss = MethodSpec.methodBuilder("dismiss")
    .addJavadoc("Hides {@code message} from the caller's history. Other\n"
        + "participants in the conversation will continue to see the\n"
        + "message in their own history unless they also delete it.\n")
    .addJavadoc("\n")
    .addJavadoc("<p>Use {@link #delete($T)} to delete the entire\n"
        + "conversation for all participants.\n", Conversation.class)
    .addModifiers(Modifier.PUBLIC, Modifier.ABSTRACT)
    .addParameter(Message.class, "message")
    .build();
```

生成

```java
  /**
   * Hides {@code message} from the caller's history. Other
   * participants in the conversation will continue to see the
   * message in their own history unless they also delete it.
   *
   * <p>Use {@link #delete(Conversation)} to delete the entire
   * conversation for all participants.
   */
  void dismiss(Message message);
```

在 Javadoc 中引用类型时，请使用 `$T` 以获取自动导入。

## Download

Download [the latest .jar][dl] or depend via Maven:

```xml
<dependency>
  <groupId>com.squareup</groupId>
  <artifactId>javapoet</artifactId>
  <version>1.12.1</version>
</dependency>
```

or Gradle:

```groovy
compile 'com.squareup:javapoet:1.12.1'
```

开发版本的快照可在[Sonatype的 `snapshots` 存储库][snap]中获得。

## License

Copyright 2015 Square, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

## JavaWriter

JavaPoet是 [JavaWriter][javawriter] 的后继者。新项目应首选JavaPoet，因为它具有更强大的代码模型：它了解类型并可以自动管理导入。JavaPoet是也更适合合成：而不是流式传输.java文件的内容单次从上到下，文件可以组装为声明树。

JavaWriter仍可在[GitHub][javawriter]和 [Maven Central][javawriter_maven]中使用。

[dl]:https://search.maven.org/remote_content?g=com.squareup&a=javapoet&v=LATEST
[snap]: https://oss.sonatype.org/content/repositories/snapshots/com/squareup/javapoet/
[javadoc]: https://square.github.io/javapoet/1.x/javapoet/
[javawriter]: https://github.com/square/javapoet/tree/javawriter_2
[javawriter_maven]: https://search.maven.org/#artifactdetails%7Ccom.squareup%7Cjavawriter%7C2.5.1%7Cjar
[formatter]: https://developer.android.com/reference/java/util/Formatter.html
[modifier]: https://docs.oracle.com/javase/8/docs/api/javax/lang/model/element/Modifier.html
