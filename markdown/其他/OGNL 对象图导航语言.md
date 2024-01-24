# OGNL 对象图导航语言

## 背景

OGNL 代表对象图导航语言；它是一种表达式语言，用于获取和设置 Java 对象的属性，以及其他附加功能，例如列表投影和选择以及 lambda 表达式。您可以使用相同的表达式来获取和设置属性的值。

Ognl类包含用于计算 OGNL 表达式的便捷方法。您可以分两个阶段执行此操作，将表达式解析为内部形式，然后使用该内部形式设置或获取属性的值；或者您可以在单个阶段中执行此操作，并直接使用表达式的字符串形式获取或设置属性。

OGNL 最初是一种使用属性名称在 UI 组件和控制器之间建立关联的方法。随着对更复杂关联的需求不断增长，德鲁·戴维森 (Drew Davidson) 在卢克·布兰沙德 (Luke Blanshard) 的怂恿下创建了他所谓的 KVCL（键值编码语言）。随后，Luke 使用 ANTLR 重新实现了该语言，提出了新名称，并在 Drew 的怂恿下，将其填写为当前状态。后来 Luke 再次使用 JavaCC 重新实现了该语言。所有代码的进一步维护由 Drew 完成（在 Luke 的精神指导下）。

## 介绍

很多人问OGNL到底有什么用处。OGNL的一些应用 包括：

* GUI 元素（文本字段、组合框等）与模型对象之间的绑定语言。OGNL的 TypeConverter 机制使转换变得更容易，可以将值从一种类型转换为另一种类型（例如，将字符串转换为数字类型）；
* 用于在表列和 Swing TableModel 之间进行映射的数据源语言；
* Web 组件和底层模型对象之间的绑定语言；
* 对 Apache Commons BeanUtils 包或 JSTL 的 EL（仅允许简单的属性导航和基本索引属性）使用的属性获取语言的更具表现力的替代。

您在 Java 中能做的大部分事情都可以在OGNL中实现，此外还有其他附加功能，例如列表投影、 选择和lambda 表达式。

## ONGL句法

基本OGNL表达式非常简单。该语言的功能已经变得相当丰富，但您通常不需要担心该语言的更复杂的部分：简单的情况仍然如此。例如，要获取对象的 name 属性，OGNL表达式就是name。要获取标题属性返回的对象的文本属性，OGNL表达式为header.text。

OGNL表达式 的基本单元是导航链，通常简称为“链”。最简单的链条由以下部分组成：属性名称、方法调用、数组索引

| 表达式元素部分 | 例子 |
| --- | --- |
| 属性名称 | 就像上面的name和header.text示例 |
| 方法调用 | hashCode()返回当前对象的哈希码 |
| 数组索引 | Listeners[0]返回当前对象的listeners列表中的第一个 |

所有OGNL表达式都在当前对象的上下文中进行计算，并且链仅使用链中前一个链接的结果作为下一个链接的当前对象。您可以根据需要延长链条。例如这条链：

``` java
name.toCharArray()[0].numericValue.toString()
```

该表达式按照以下步骤进行计算

* 提取初始对象或根对象的名称属性（用户通过OGNL上下文提供给OGNL）；
* 对结果String调用toCharArray()方法；
* 从结果数组中提取第一个字符（索引0处的字符）；
* 从该字符获取numericValue属性（该字符表示为Character对象，Character类有一个名为getNumericValue()的方法）；
* 对生成的Integer对象调用toString()。该表达式的最终结果是最后一次toString()调用返回的String。

请注意，此示例只能用于从对象获取值，而不能用于设置值。将上述表达式传递给Ognl.setValue()方法将导致抛出InpropertiesExpressionException ，因为链中的最后一个链接既不是属性名称也不是数组索引。

## OGNL表达式

本节概述了OGNL表达式 元素的详细信息。

### 常数

OGNL有以下几种常量

* 字符串字符，如Java中（添加单引号）：由单引号或双引号分隔，并具有完整的字符转义集；
* 字符字符，也与Java中一样：由单引号分隔，也带有完整的转义集；
* 数字字符，比Java的种类多一些。除了Java的int、long、float和double之外，OGNL还允许您指定带有“b”或“B”后缀的BigDecimals，以及带有“h”或“H”后缀的BigIntegers（认为“巨大”——我们选择“h”代表BigIntegers，因为它不干扰十六进制数字）；
* 布尔（true和false）字符；
* 空字符。

### 参考属性

OGNL在处理属性引用时以不同的方式对待不同类型的对象。映射将所有属性引用视为元素查找或存储，并以属性名称作为键。列表和数组以类似的方式处理数字属性，以属性名称作为索引，但字符串属性的处理方式与普通对象相同。普通对象（即所有其他类型）只能处理字符串属性，并且可以通过使用“get”和“set”方法（或“is”和“set”）来处理字符串属性（如果对象具有这些属性），或者使用带有否则给出名字。

请注意此处的新术语。属性“名称”可以是任何类型，而不仅仅是字符串。但要引用非字符串属性，您必须使用我们所说的“索引”表示法。例如，要获取数组的长度，可以使用以下表达式

``` java
array.length
```

但要获取数组的元素 0，必须使用如下表达式

``` java
array[0]
```

请注意，Java 集合有一些与其相关的特殊属性。

### 索引

如上所述，“索引”符号实际上只是属性引用，尽管是属性引用的计算形式而不是常量引用。

例如，OGNL在内部处理“array.length”表达式与此表达式完全相同：

``` java
array["length"]
```

这个表达式将具有相同的结果（尽管内部形式不同）

``` java
array["len" + "gth"]
```

#### 数组和列表索引

对于Java数组和列表来说，索引相当简单，就像Java中一样。给出了一个整数索引，该元素是参考对象。如果索引超出数组或列表的范围，则会抛出IndexOutOfBoundsException，就像在 Java 中一样。

#### JavaBeans 索引属性

JavaBeans 支持索引属性的概念。具体来说，这意味着一个对象具有一组遵循以下模式的方法

* public PropertyType[] getPropertyName();
* public void setPropertyName(PropertyType[] anArray);
* public PropertyType getPropertyName(int index);
* public void setPropertyName(int index, PropertyType value);

OGNL 可以解释这一点并通过索引符号提供对属性的无缝访问。参考文档如

``` java
someProperty[2]
```

#### OGNL 对象索引属性

OGNL扩展了索引属性的概念，包括对任意对象进行索引，而不仅仅是像JavaBeans索引属性那样对整数进行索引。当查找属性作为对象索引的候选时，OGNL会查找具有以下签名的方法模式

* public PropertyType getPropertyName(IndexType index);
* public void setPropertyName(IndexType index, PropertyType value);

PropertyType和IndexType在相应的 set 和get方法中必须相互匹配。使用对象索引属性的一个实际示例是Servlet API：Session对象有两种获取和设置任意属性的方法

``` java
public Object getAttribute(String name) 
public void setAttribute(String name, Object value)
```

可以获取和设置这些属性之一的OGNL表达式是

``` java
session.attribute["foo"]
```

### 调用方法

OGNL调用方法的方式与Java的方式略有不同，因为OGNL是解释性的，并且必须在运行时选择正确的方法，除了提供的实际参数之外，没有额外的类型信息。OGNL总是选择它能找到的最具体的方法，其类型与提供的参数相匹配；如果有两个或多个同样具体且与给定参数匹配的方法，则将任意选择其中之一。

特别是，空参数匹配所有非基本类型，因此很可能导致调用意外的方法。

请注意，方法的参数是用逗号分隔的，因此不能使用逗号运算符，除非将其括在括号中。例如，

``` java
method( ensureLoaded(), name )
```

是对 2 参数方法的调用，而

``` java
method( (ensureLoaded(), name) )
```

是对 1 参数方法的调用。

### 变量引用

OGNL有一个简单的变量方案，它允许您存储中间结果并再次使用它们，或者只是命名事物以使表达式更易于理解。OGNL中的所有变量 对于整个表达式都是全局的。您可以在变量名称前面使用数字符号来引用变量，如下所示

``` java
#var
```

OGNL还将表达式求值中每个点的当前对象存储在 this 变量中，可以像任何其他变量一样引用它。例如，以下表达式对listeners的数量进行操作，如果大于 100，则返回该数字的两倍，否则返回该数字的 20

``` java
listeners.size().(#this > 100? 2*#this : 20+#this)
```

可以使用定义变量初始值的映射来调用OGNL 。调用OGNL 的标准方法定义变量根（保存初始或根对象）和上下文（保存变量本身的 映射）。

要显式为变量赋值，只需在左侧编写带有变量引用的赋值语句

``` java
#var = 99
```

### 括号表达式

正如您所期望的，括号中的表达式将作为一个单元进行计算，与任何周围的运算符分开。这可用于强制执行不同于OGNL运算符优先级所暗示的计算顺序。这也是在方法参数中使用逗号运算符的唯一方法。

### 子链表达式

如果在点后使用括号表达式，则该点处的当前对象将用作整个括号表达式中的当前对象。例如

``` java
headline.parent.(ensureLoaded(), name)
```

遍历header和parent属性，确保加载parent，然后返回（或设置）parent 的name。

顶级表达式也可以通过这种方式链接。表达式的结果是最右边的表达式元素。

``` java
ensureLoaded(), name
```

这将在根对象上调用EnsureLoaded() ，然后获取根对象的name属性作为表达式的结果。

### 集合构造器

#### Lists

要创建对象列表，请将表达式列表括在花括号中。与方法参数一样，这些表达式不能使用逗号运算符，除非将其括在括号中。这是一个例子

``` java
name in { null,"Untitled" }
```

这测试name属性是否为null或者等于"Untitled"。

上述语法将创建List接口的实例。确切的子类未定义。

#### Native Arrays

有时您想要创建 Java 本机数组，例如int[]或Integer[]。OGNL支持以类似于通常调用构造函数的方式创建这些数组，但允许从现有列表或给定大小的数组初始化本机数组。

``` java
new int[] { 1, 2, 3 }
```

这将创建一个由三个整数 1、2 和 3 组成的新int数组。

要创建包含所有null或0元素的数组，请使用替代大小构造函数

``` java
new int[5]
```

这将创建一个具有 5 个槽的int数组，全部初始化为零。

#### Maps

还可以使用特殊语法创建地图。

```java
#{ "foo" : "foo value", "bar" : "bar value" }
```

这将创建一个使用“foo”和“bar”映射初始化的Map。

希望选择特定 Map 类的高级用户可以在左大括号之前指定该类

``` java
#@java.util.LinkedHashMap@{ "foo" : "foo value", "bar" : "bar value" }
```

上面的示例将创建 JDK 1.4 类LinkedHashMap的实例，确保保留元素的插入顺序。

### 跨集合投影

OGNL提供了一种简单的方法来调用相同的方法或从集合中的每个元素中提取相同的属性并将结果存储在新集合中。我们将其称为“投影”，来自数据库术语，用于从表中选择列的子集。例如，这个表达式

``` java
listeners.{delegate}
```

返回所有listeners委托的列表。有关OGNL如何将各种类型的对象视为集合的信息，请参阅强制部分。

在投影期间，#this变量引用迭代的当前元素。

``` java
objects.{ #this instanceof String ? #this : #this.toString()}
```

上面的代码将从对象列表中生成一个新的元素列表作为字符串值。

### 从集合中选择

OGNL提供了一种简单的方法来使用表达式从集合中选择一些元素并将结果保存在新集合中。我们将其称为“选择”，来自数据库术语，用于从表中选择行的子集。例如，这个表达式

``` java
listeners.{? #this instanceof ActionListener}
```

返回作为ActionListener类实例的所有listeners的列表。

#### 选择第一个匹配项

为了从匹配列表中获取第一个匹配，您可以使用索引，例如listeners.{? true }[0]。但是，这很麻烦，因为如果匹配没有返回任何结果（或者结果列表为空），您将得到ArrayIndexOutOfBoundsException。

选择语法还可用于仅选择第一个匹配项并将其作为列表返回。如果任何元素的匹配均未成功，则结果为空列表。

``` java
objects.{^ #this instanceof String }

```

将返回作为String类实例的对象中包含的第一个元素。

#### 选择最后一场比赛

与获取第一个匹配项类似，有时您想获取最后一个匹配的元素。

``` java
objects.{$ #this instanceof String }
```

这将返回作为String类实例的对象中包含的最后一个元素

### 调用构造函数

您可以像在 Java 中一样使用new运算符创建新对象。一个区别是，您必须为 java.lang 包中的类以外的类指定完全限定的类名。

仅当默认的 ClassResolver 就位时才会出现这种情况。使用自定义类解析器，可以以这样的方式映射包，从而可以对类进行更多类似 Java 的引用。有关使用ClassResolver类的详细信息，请参阅 OGNL 开发人员指南（例如，new java.util.ArrayList()，而不是简单的new ArrayList()）。

OGNL使用与重载方法调用相同的过程来选择正确的构造函数进行调用。

### 调用静态方法

您可以使用语法@class @method (args)调用静态方法。如果省略 class，则它默认为java.lang.Math，以便更轻松地调用min和max方法。如果指定类，则必须提供完全限定名称。

如果您有一个类的实例，您希望调用其静态方法，则可以通过该对象调用该方法，就像它是实例方法一样。

如果方法名称被重载，OGNL将使用与重载实例方法相同的过程来选择正确的静态方法进行调用。

### 获取静态字段

您可以使用语法@class @field引用静态字段。class必须完全合格。

### 表达式赋值

如果在OGNL表达式后跟一个带括号的表达式，并且括号前面没有点，OGNL会尝试将第一个表达式的结果视为另一个表达式来计算，并将使用带括号的表达式的结果作为根对象对于那个评价。首先，一个表达式的结果可以是任何对象；如果是抽象语法树（Abstract Syntax Tree，AST），OGNL假设它是表达式的解析形式并简单地解释它；否则，OGNL采用对象的字符串值并解析该字符串以获得AST来解释。

例如，这个表达式

``` java
#fact(30H)
```

查找fact变量，并将该变量的值解释为使用30的BigInteger表示形式作为 根对象的OGNL表达式。请参阅下面的示例，该示例使用返回其参数阶乘的表达式来设置fact变量。请注意，在OGNL的语法中，此双重求值运算符和方法调用之间存在歧义。OGNL通过调用任何看起来像方法调用的东西来解决这种歧义。例如，如果当前对象有一个包含OGNL阶乘表达式的fact属性，则无法使用此方法来调用它

``` java
fact(30H)
```

因为OGNL会将其解释为对fact方法的调用。您可以通过用括号将属性引用括起来来强制执行您想要的解释

``` java
(fact)(30H)
```

### 伪-Lambda表达式

OGNL具有简化的 lambda 表达式语法，可让您编写简单的函数。它不是一个成熟的 lambda 演算，因为没有闭包——OGNL中的所有变量都具有全局作用域和范围。

例如，下面是一个声明递归阶乘函数的OGNL表达式，然后调用它

``` java
#fact = :[#this<=1? 1 : #this*#fact(#this-1)], #fact(30H)
```

lambda表达式是括号内的所有内容。#this变量保存表达式的参数，最初为30H，然后每次连续调用表达式时都会减一。

OGNL将lambda表达式视为常量。lambda表达式的值是OGNL用作所包含表达式的解析形式的AST。

### 集合的伪属性

OGNL提供了一些特殊的集合属性。其原因是集合不遵循JavaBeans模式进行方法命名；因此，必须调用size()、length()等方法，而不是更直观地将它们称为属性。OGNL通过公开某些伪属性来纠正此问题，就像它们是内置的一样。

｜ 集合 ｜ 特殊属性 ｜
｜ --- ｜ --- ｜
｜ Collection （由Map、List和Set继承） ｜ size：集合的大小 <br> isEmpty：如果集合为空，则计算结果为true ｜
｜ List ｜ iterator：计算List上的迭代器。 ｜
｜ Map ｜ keys：计算映射中所有键的集合 <br/> value：计算Map中所有值的集合 <br/> 注意这些属性（加上size和isEmpty ）与Map的索引访问形式不同（即someMap["size"]从映射中获取“size”键，而someMap.size获取Map 的大小 ｜
｜ Set ｜ iterator：计算集合上的迭代器 ｜
｜ Iterator ｜ next ：计算迭代器中的下一个对象 <br/> hasNext：如果迭代器中有下一个对象可用，则计算结果为true ｜
｜ Enumeration ｜ next ：计算枚举中的下一个对象 <br/> hasNext：如果枚举中有下一个可用对象，则计算结果为true <br/> nextElement：下一个的同义词 <br/> hasMoreElements : hasNext的同义词 ｜

### 与 Java 运算符不同的运算符

大多数情况下，OGNL的运算符是从Java借用的，其工作方式与Java的运算符类似。请参阅OGNL参考以获得完整的讨论。这里我们描述Java中没有的或与Java不同的OGNL运算符。

* 逗号(,)或序列运算符。该运算符是从C借用的。逗号用于分隔两个独立的表达式。这些表达式中第二个的值是逗号表达式的值。这是一个例子

``` java
ensureLoaded(), name
```

当计算此表达式时，将调用EnsureLoaded方法（大概是为了确保对象的所有部分都在内存中），然后检索name属性（如果获取值）或替换（如果设置）。

* 使用大括号({})的列表结构。您可以通过将值括在花括号中来创建内联列表，如下例所示

``` java
{ null, true, false }
```

* in运算符（而不是in，它的非运算）。这是一个包含测试，用于查看某个值是否在集合中。例如

``` java
name in {null,"Untitled"} || name
```

### 设置值与获取值

如前所述，由于表达式的性质，某些可获取的值也不可设置。例如

``` java
names[0].location
```

是一个可设置的表达式 - 表达式的最终组成部分解析为可设置的属性。

然而，有些表达方式，例如

``` java
names[0].length + 1
```

不可设置，因为它们无法解析为对象中的可设置属性。它只是一个计算值。如果您尝试使用任何Ognl.setValue()方法计算此表达式，它将失败并出现InpropertyExpressionException。

还可以使用包含“=”运算符的get表达式来设置变量。当get表达式需要设置变量作为执行的副作用时，这非常有用。

### 对象类型强转

这里我们描述OGNL如何将对象解释为各种类型。请参阅下文，了解OGNL如何将对象强制转换为布尔值、数字、整数和集合。

#### 将对象解释为布尔值

任何需要布尔值的对象都可以使用。OGNL将对象解释为布尔值，如下所示

* 如果对象是 Boolean ，则提取并返回其值；
* 如果对象是Number，则将其双精度浮点值与零进行比较；非零被视为true，零被视为false；
* 如果对象是Character ，当且仅当其char值非零时，其布尔值为true ；
* 否则，当且仅当它不为null时，其布尔值才为true。

#### 将对象解释为数字

数值运算符尝试将其参数视为数字。基本基元类型包装类（Integer、Double等，包括被视为整数的Character和Boolean）以及java.math包中的“大”数字类（BigInteger和BigDecimal）被识别为特殊的数字类型。给定某个其他类的对象，OGNL尝试将对象的字符串值解析为数字。

采用两个参数的数值运算符使用以下算法来决定结果应该是什么类型。如果结果不适合给定类型，则实际结果的类型可能更宽。

* 如果两个参数的类型相同，则结果将尽可能具有相同的类型；
* 如果任一参数不属于可识别的数字类，则对于该算法的其余部分，它将被视为Double ；
* 如果两个参数都是实数的近似值（Float、Double或BigDecimal），则结果将是更宽的类型；
* 如果两个参数都是整数（Boolean、Byte、Character、Short、Integer、Long或BigInteger），则结果将是较宽的类型；
* 如果一个参数是实数类型，另一个参数是整数类型，则如果整数比“int”窄，则结果将为实数类型；BigDecimal如果整数是BigInteger；或实际类型的较宽者，否则为Double。

#### 将对象解释为整数

仅对整数起作用的运算符（如位移运算符）将其参数视为数字，但BigDecimal和BigInteger被作为BigInteger进行操作，所有其他类型的数字被作为Long进行操作。对于BigInteger情况，这些运算符的结果仍然是BigInteger；对于Long情况，结果表示为相同类型的参数（如果适合），否则表示为Long。

#### 将对象解释为集合

投影和选择运算符（e1.{e2}和e1.{?e2}）以及in运算符都将其参数之一视为集合并对其进行遍历。根据参数的类别，此操作的完成方式有所不同：

* Java数组是从前往后走的；
* java.util.Collection的成员通过遍历其迭代器来遍历；
* java.util.Map的成员通过遍历迭代器遍历它们的值来遍历；
* java.util.Iterator和java.util.Enumeration的成员通过迭代来遍历；
* java.lang.Number的成员通过返回小于从零开始的给定数字的整数来“遍历”；
* 所有其他对象都被视为仅包含其自身的单例集合。

## 相关文档

[commons-ognl官网](https://commons.apache.org/dormant/commons-ognl/)

[commons-ognl官方用户文档](https://commons.apache.org/dormant/commons-ognl/language-guide.html)
