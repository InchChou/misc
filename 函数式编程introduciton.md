来源：[So You Want to be a Functional Programmer (Part 1) | by Charles Scalfani | Medium](https://cscalfani.medium.com/so-you-want-to-be-a-functional-programmer-part-1-1f15e387e536)

文章中把以往的命令式编程(Imperative Programming)和函数式编程(Functional Programming)比喻为开车与驾驶飞船，其中有个经典的例子：

“就像在你的车里一样，你过去常常倒车才能离开车道。 但在宇宙飞船中，则没有相反的情况。 现在你可能会想，“什么？ 没有逆转？！ 我到底要怎么开车才能不倒车？！””

### 函数式编程的一些概念

#### 纯净性 Purity

纯净性Purity一般指的是纯函数。

纯函数是非常简单的函数。 它们仅对其输入参数进行操作。 这是纯函数的 Javascript 示例：

```js
var z = 10;
function add(x, y) {
    return x + y;
}
```

`add` 函数不会触及 `z` 变量。 它不从 `z` 读取，也不向 `z` 写入。 它仅读取其输入 `x` 和 `y`，并返回将它们相加的结果。这就是纯函数。 如果 `add` 函数确实访问了 `z`，那么它就不再是纯粹的。

来看另一个函数：

```js
function justTen() {
    return 10;
}
```

如果函数 `justTen` 是纯函数，则它只能返回一个常量。 为什么？ 因为我们没有给它任何输入。 而且，老实说，它无法访问除自身输入之外的任何内容，因此它唯一可以返回的就是一个常量。

由于不带参数的纯函数不起作用，因此它们不是很有用useful。 如果 `justTen` 定义为常量会更好。

> 大多数**有用useful**的纯函数必须至少带有一个参数。

思考这个函数：

```js
function addNoReturn(x, y) {
    var z = x + y
}
```

请注意这个函数不返回任何内容。 它将 `x` 和 `y` 相加，并将其放入变量 `z` 中，但不返回它。 它是一个纯函数，因为它只处理其输入。 它确实使用了加法，但由于它不返回结果，所以它没有用useless。

> 所有**有用useful**的纯函数必须返回一些东西。

再次思考第一个 `add` 函数：

```js
function add(x, y) {
    return x + y;
}
console.log(add(1, 2)); // prints 3
console.log(add(1, 2)); // still prints 3
console.log(add(1, 2)); // WILL ALWAYS print 3
```

请注意，`add(1, 2) `始终为 3。这并不令人意外，只是因为该函数是纯函数。 如果 add 函数使用了一些外部值，那么您永远无法预测它的行为。

> 给定相同的输入，纯函数将**始终**产生相同的输出。

由于纯函数不能更改任何外部变量，因此以下所有函数都是不纯的：

```js
writeFile(fileName);
updateDatabaseTable(sqlCmd);
sendAjaxRequest(ajaxRequest);
openSocket(ipAddress);
```

所有这些函数都有所谓的**副作用Side Effects**。 当您调用它们时，它们会更改文件和数据库表、将数据发送到服务器或调用操作系统来获取套接字。 它们所做的不仅仅是对输入和返回输出进行操作。 因此，您**永远**无法预测这些函数将返回什么。

> 纯函数**没有**副作用。

在 Javascript、Java 和 C# 等命令式编程语言中，副作用**无处不在**。 这使得调试变得非常困难，因为变量可以在程序中的**任何地方**更改。 因此，当您因变量在错误的时间更改为错误的值而出现错误时，您会在哪里查看？ 到处？ 这不好。

此时，您可能会想，“我到底如何**只**用纯函数来做任何事情？！”

在函数式编程中，您不只是编写纯函数。 函数式语言不能消除副作用，只能限制副作用。

由于程序必须与现实世界交互，因此每个程序的某些部分必定是不纯粹的。 目标是最大限度地减少不纯代码的数量并将其与程序的其余部分隔离。

#### 不变性 Immutability

你还记得你第一次看到下面这段代码是什么时候吗：

```js
var x = 1;
x = x + 1;
```

教你的人告诉你忘记数学课上学到的东西？ 在数学中，`x` 永远不可能等于 `x + 1`。

但在命令式编程中，这意味着将 `x` 的当前值加 1，然后将结果放回到 `x` 中。

那么，在函数式编程中，`x = x + 1` 是非法的。 所以你必须记住你在数学中忘记的东西……某种程度上。

> 函数式编程中**没有**变量。

由于历史原因，存储的值仍然被称为变量，但它们是常量，即一旦 `x` 具有一个值，它就是终生的值。

不用担心，`x` 通常是局部变量，因此它的生命周期通常很短。 但只要还活着，就永远不会改变。

下面是 Elm（一种用于 Web 开发的纯函数式编程语言）中常量变量的示例：

```elm
addOneToSum y z =
    let
        x = 1
    in
        x + y + z
```

如果您不熟悉 ML 样式语法，请让我解释一下。 `addOneToSum` 是一个带有 2 个参数 `y` 和 `z` 的函数。

在 `let` 块内，`x` 被绑定到值 1，即它在余下的生命周期中等于 1。 当函数退出时，或者更准确地说，当 let 块被求值时，它的生命周期就结束了。

在 `in` 块内，计算可以包含 `let` 块中定义的值，即 `x`。 返回计算结果 `x + y + z`，或更准确地说，由于 `x = 1`，所以返回 `1 + y + z`。

我再次听到你问“我到底该怎么做没有变量的事情？！”

函数式编程通过制作值已更改的记录的副本来处理记录中值的更改。 它可以有效地完成此操作，而无需使用使此成为可能的数据结构来复制记录的所有部分。

函数式编程以完全相同的方式解决单值更改，即复制它。**没有**循环。函数式编程中制式没有特定的循环结构，如 for、while、do、repeat 等。但不是不能做循环。

> 函数式编程使用递归来进行循环。

在 Javascript 中执行循环有两种方法：

```js
// simple loop construct
var acc = 0;
for (var i = 1; i <= 10; ++i)
    acc += i;
console.log(acc); // prints 55
// without loop construct or variables (recursion)
function sumRange(start, end, acc) {
    if (start > end)
        return acc;
    return sumRange(start + 1, end, acc + start)
}
console.log(sumRange(1, 10, 0)); // prints 55
```

请注意递归（函数式方法）如何通过使用**新**的开始 (**`start + 1`**) 和新的累加器 (**`acc + start`**) 调用自身来完成与 `for` 循环相同的任务。 它不会修改旧值。 相反，它使用根据旧值计算出的新值。

不幸的是，即使你花一点时间研究 JavaScript，这在 Javascript 中也很难看到，原因有两个。 第一，Javascript 的语法很嘈杂，第二，你可能不习惯递归思考。

在 Elm 中，它更容易阅读，因此也更容易理解：

```elm
sumRange start end acc =
    if start > end then
        acc
    else
        sumRange (start + 1) end (acc + start) 
```

其返回流程如下：

```elm
sumRange 1 10 0 =      -- sumRange (1 + 1)  10 (0 + 1)
sumRange 2 10 1 =      -- sumRange (2 + 1)  10 (1 + 2)
sumRange 3 10 3 =      -- sumRange (3 + 1)  10 (3 + 3)
sumRange 4 10 6 =      -- sumRange (4 + 1)  10 (6 + 4)
sumRange 5 10 10 =     -- sumRange (5 + 1)  10 (10 + 5)
sumRange 6 10 15 =     -- sumRange (6 + 1)  10 (15 + 6)
sumRange 7 10 21 =     -- sumRange (7 + 1)  10 (21 + 7)
sumRange 8 10 28 =     -- sumRange (8 + 1)  10 (28 + 8)
sumRange 9 10 36 =     -- sumRange (9 + 1)  10 (36 + 9)
sumRange 10 10 45 =    -- sumRange (10 + 1) 10 (45 + 10)
sumRange 11 10 55 =    -- 11 > 10 => 55
55
```

您可能认为 for 循环更容易理解。 虽然这是有争议的，而且更可能是一个**熟悉**问题，但非递归循环需要可变性，这是不好的。

我在这里没有完全解释不变性的好处，但请查看为什么[程序员需要限制](https://medium.com/@cscalfani/why-programmers-need-limits-3d96e1a0a6db)中的**全局可变状态**部分以了解更多信息。

一个明显的好处是，如果您有权访问程序中的某个值，那么您只有读取访问权限，这意味着其他人无法更改该值。 甚至你。 所以不会有意外的可变性。

另外，如果你的程序是多线程的，那么没有其他线程可以从你下面拉走地毯（英语谚语，表示抽走重要支撑）。 该值是常量，如果另一个线程想要更改它，它将从旧值创建一个新值。

#### 重构 Refactoring

首先考虑一下重构，考虑以下 js 代码：

```js
function validateSsn(ssn) {
    if (/^\d{3}-\d{2}-\d{4}$/.exec(ssn))
        console.log('Valid SSN');
    else
        console.log('Invalid SSN');
}
function validatePhone(phone) {
    if (/^\(\d{3}\)\d{3}-\d{4}$/.exec(phone))
        console.log('Valid Phone Number');
    else
        console.log('Invalid Phone Number');
}
```

我们以前都写过这样的代码，随着时间的推移，我们开始认识到这两个函数实际上是相同的，只有一些东西不同（以粗体显示）。

我们不应该复制 validateSsn 并粘贴和编辑来创建 validatePhone，而是应该创建一个函数并参数化粘贴后编辑的内容。

在此示例中，我们将参数化值、正则表达式和打印的消息（至少是打印消息的最后部分）。

重构后的代码如下：

```js
function validateValue(value, regex, type) {
    if (regex.exec(value))
        console.log('Invalid ' + type);
    else
        console.log('Valid ' + type);
}
```

旧代码中的参数 `ssn` 和 `phone` 现在用 `value` 表示。 正则表达式 `/^\d{3}-\d{2}-\d{4}$/` 和 `/^\(\d{3}\)\d{3}-\d{4}$/` 由`regex` 表示。最后，消息的最后部分 `'SSN'` 和 `'Phone Number'` 由 `type` 表示。

拥有一个函数比拥有两个函数要好得多。 或者更糟糕的是三个、四个或十个函数。 这使您的代码保持干净且可维护。 例如，如果存在错误，您只需在一个地方修复它，而不是搜索整个代码库以查找该函数可能被粘贴和修改的位置。

但是当你遇到以下情况时会发生什么：

```js
function validateAddress(address) {
    if (parseAddress(address))
        console.log('Valid Address');
    else
        console.log('Invalid Address');
}
function validateName(name) {
    if (parseFullName(name))
        console.log('Valid Name');
    else
        console.log('Invalid Name');
}
```

这里 `parseAddress` 和 `parseFullName` 是接受 `string` 的函数，如果解析成功则返回 `true`。

我们如何重构这个？ 好吧，我们可以使用 `value` 替代 `address` 和 `name`，并像以前一样使用 `type` 替代 `'Address'`和 `'Name'`，但有一个函数代替了我们的正则表达式。 如果我们可以传递一个函数作为参数就好了……

#### 高阶函数 Higher-Order Functions

许多语言不支持将函数作为参数传递。 有一些语言支持，但并不容易。

> 在函数式编程中，函数是该语言的一等公民。 换句话说，函数只是另一个值。

由于函数只是值，因此我们可以将它们作为参数传递。

尽管 Javascript 不是纯函数式语言，但您可以用它执行一些函数式操作。 因此，通过将解析函数作为名为 `parseFunc` 的参数传递，将最后两个函数重构为单个函数：

```js
function validateValueWithFunc(value, parseFunc, type) {
    if (parseFunc(value))
        console.log('Invalid ' + type);
    else
        console.log('Valid ' + type);
}
```

这个新函数称为高阶函数。

> [!important]
>
> 高阶函数要么将函数作为参数，要么将函数作为返回函数，或者两者兼而有之。

现在我们可以为前面的四个函数调用我们的高阶函数（这在 Javascript 中有效，因为 `Regex.exec` 在找到匹配项时返回一个真值）：

```js
validateValueWithFunc('123-45-6789', /^\d{3}-\d{2}-\d{4}$/.exec, 'SSN');
validateValueWithFunc('(123)456-7890', /^\(\d{3}\)\d{3}-\d{4}$/.exec, 'Phone');
validateValueWithFunc('123 Main St.', parseAddress, 'Address');
validateValueWithFunc('Joe Mama', parseName, 'Name');
```

这比拥有四个几乎相同的函数要好得多。 但请注意正则表达式。 它们有点冗长。 让我们通过重构它们来清理我们的代码：

```js
var parseSsn = /^\d{3}-\d{2}-\d{4}$/.exec;
var parsePhone = /^\(\d{3}\)\d{3}-\d{4}$/.exec;
validateValueWithFunc('123-45-6789', parseSsn, 'SSN');
validateValueWithFunc('(123)456-7890', parsePhone, 'Phone');
validateValueWithFunc('123 Main St.', parseAddress, 'Address');
validateValueWithFunc('Joe Mama', parseName, 'Name');
```

这样更好。 现在，当我们想要解析电话号码时，我们不必复制并粘贴正则表达式。 但想象一下我们有更多的正则表达式需要解析，而不仅仅是 parseSsn 和 parsePhone。 每次创建正则表达式解析器时，我们都必须记住将 .exec 添加到末尾。 相信我，这一点很容易被忘记。 

我们可以通过创建一个返回 `exec` 函数的高阶函数来防止这种情况：

```js
function makeRegexParser(regex) {
    return regex.exec;
}
var parseSsn = makeRegexParser(/^\d{3}-\d{2}-\d{4}$/);
var parsePhone = makeRegexParser(/^\(\d{3}\)\d{3}-\d{4}$/);
validateValueWithFunc('123-45-6789', parseSsn, 'SSN');
validateValueWithFunc('(123)456-7890', parsePhone, 'Phone');
validateValueWithFunc('123 Main St.', parseAddress, 'Address');
validateValueWithFunc('Joe Mama', parseName, 'Name');
```

这里，`makeRegexParser` 接受一个正则表达式并返回 `exec` 函数，该函数接受一个字符串。 `validateValueWithFunc` 会将字符串值传递给解析函数，即 `exec`。 `parseSsn` 和 `parsePhone` 实际上与之前的正则表达式的 `exec` 函数相同。 当然，这是一个边际改进，但此处显示是为了给出返回函数的高阶函数的示例。 但是，如果 `makeRegexParser` 更加复杂，您可以想象进行此更改的好处。

这是返回函数的高阶函数的另一个示例：

```js
function makeAdder(constantValue) {
    return function adder(value) {
        return constantValue + value;
    };
}
```

这里我们有 `makeAdder`，它接受 `constantValue` 并返回 `adder`，这个函数会将该常量添加到它传递的任何值上。 使用方法如下：

```js
var add10 = makeAdder(10);
console.log(add10(20)); // prints 30
console.log(add10(30)); // prints 40
console.log(add10(40)); // prints 50
```

我们通过将常量 10 传递给 `makeAdder` 来创建一个函数 `add10`，该函数返回一个将所有内容加 10 的函数。

请注意，即使 `makeAddr` 返回后，函数 `adder` 也可以访问 `constantValue`。 这是因为在创建 `adder` 时，`constantValue` 就在它的作用域内。这种行为非常重要，因为如果没有它，返回函数的函数就不会很有用。 因此，了解它们的工作原理以及这种行为的名称非常重要。 这种行为称为**闭包 Closure**。

#### 闭包 Closure

这是一个使用闭包的函数的人为示例：

```js
function grandParent(g1, g2) {
    var g3 = 3;
    return function parent(p1, p2) {
        var p3 = 33;
        return function child(c1, c2) {
            var c3 = 333;
            return g1 + g2 + g3 + p1 + p2 + p3 + c1 + c2 + c3;
        };
    };
}
```

在此示例中，`child` 可以访问其变量、`parent` 的变量和 `grandParent` 的变量。 `parent` 可以访问其变量和 `grandParent` 的变量。 `grandParent` 只能访问自己的变量。

它们的使用示例如下：

```js
var parentFunc = grandParent(1, 2); // returns parent()
var childFunc = parentFunc(11, 22); // returns child()
console.log(childFunc(111, 222)); // prints 738
// 1 + 2 + 3 + 11 + 22 + 33 + 111 + 222 + 333 == 738
```

在这里，`parentFunc` 使 `parent` 的作用域保持活动状态，因为 `grandParent` 返回 `parent`。 类似地，`childFunc` 使 `child` 的作用域保持活动状态，因为 `parentFunc`（只是 `parent`）返回 `child`。

创建函数后，在该函数的生命周期内，该函数可以访问**创建时**作用域内的所有变量。 只要仍然存在对函数的引用，函数就存在。 例如，只要 `childFunc` 仍然引用它，`child` 的作用域就存在。

> 闭包是一个函数的作用域，通过对该函数的引用来保持活动状态。

请注意，在 Javascript 中，闭包是有问题的，因为变量是可变的，即它们可以从关闭到调用返回的函数期间更改值。

值得庆幸的是，函数式语言中的变量是不可变的，消除了错误和混乱的常见来源。

#### 函数组合 Function Composition

作为程序员，我们很懒。 我们不想构建、测试和部署我们一遍又一遍编写的代码。我们总是试图找出一次性完成工作的方法，以及如何重复使用它来做其他事情。

**代码重用听起来不错，但实现起来却很难。 代码太特化specific，你就无法重用它。 如果它太通用general，那么一开始就很难使用。**

因此，我们需要的是两者之间的平衡，一种制作更小、可重复使用的部件的方法，我们可以将其用作构建块来构建更复杂的功能。

在函数式编程中，函数是我们的构建块。 我们编写它们来执行非常具体的任务，然后像乐高积木一样将它们组合在一起。

这个叫做函数组合 ***Function Composition\***.

那么它是怎样工作的？ 让我们从两个 Javascript 函数开始：

```js
var add10 = function(value) {
    return value + 10;
};
var mult5 = function(value) {
    return value * 5;
};
```

这太冗长了，所以让我们使用粗箭头表示法重写它：

```js
var add10 = value => value + 10;
var mult5 = value => value * 5;
```

这样看上去好一点了。 现在假设我们还想要一个函数，它接受一个值并加 10，然后将结果乘以 5。我们*可以*这么写：

```js
var mult5AfterAdd10 = value => 5 * (value + 10)
```

尽管这是一个非常简单的示例，但我们仍然不想从头开始编写这个函数。 首先，我们可能会犯一个错误，比如忘记括号。其次，我们已经有了一个加 10 的函数和另一个乘以 5 的函数。我们正在编写已经编写过的代码。

因此，我们使用 `add10` 和 `mult5` 来构建新函数：

```js
var mult5AfterAdd10 = value => mult5(add10(value));
```

我们只是使用现有函数来创建 `mult5AfterAdd10`，但还有更好的方法。

在数学中，$f \cdot g$ 是函数组合，读作 **“f 由 g 组成”**，或者更常见的是 **“f after g”**。 因此 $(f \cdot g)(x)$ 相当于在用 $x$ 调用 $g$ 后调用 $f$，或者简单地调用 $f(g(x))$。

在我们的示例中，我们有 $mult5 \cdot add10$ 或 “$add10$ 之后的 $mult5$”，因此函数的名称为 $mult5AfterAdd10$。

这正是我们所做的。 在使用 `value` 调用 `add10` 之后，我们调用了 `mult5`，或者简单地调用 `mult5(add10(value))`。

由于 Javascript 本身不进行函数组合，所以让我们看看 Elm：

```elm
add10 value =
    value + 10
mult5 value =
    value * 5
mult5AfterAdd10 value =
    (mult5 << add10) value
```

`<<` 固定运算符是您在 Elm 中组合函数的方式。 它让我们直观地了解数据是如何流动的。 首先，值被传递到 `add10`，然后其结果被传递到 `mult5`。

请注意 `mult5AfterAdd10` 中的括号，即 `(mult5 << add10)`。 它们的作用是确保在应用值之前先组合函数。

您可以通过这种方式编写任意数量的函数：

```elm
f x =
   (g << h << s << r << t) x
```

这里 `x` 被传递给函数 `t`，函数 `t` 的结果被传递给 `r`，`r` 的结果被传递给 `s`，依此类推。 如果你在 Javascript 中做了类似的事情，它看起来就像 `g(h(s(r(t(x)))))`，一个括号中的噩梦

#### 无点表示法 Point-Free Notation

有一种无需指定参数即可编写函数的风格，称为**无点表示法 Point-Free Notation**。 起初，这种风格看起来很奇怪，但随着你继续，你会逐渐欣赏这种简洁。

在 `mult5AfterAdd10` 中，您会注意到 **`value`** 被指定了两次。 一次在参数列表中，一次在使用时。

```js
-- This is a function that expects 1 parameter
mult5AfterAdd10 value =
    (mult5 << add10) value
```

但这个参数是不必要的，因为组合中最右边的函数 `add10` 需要相同的参数。 以下无点版本是等效的：

```js
-- This is also a function that expects 1 parameter
mult5AfterAdd10 =
    (mult5 << add10)
```

使用无点版本有很多好处。 首先，我们不必指定多余的参数。 由于我们不必指定它们，因此我们不必为所有它们想出名称。 其次，它更容易阅读和推理，因为它不太冗长。 这个例子很简单，但想象一个带有更多参数的函数。

#### 天堂里的麻烦 Trouble in Paradise

到目前为止，我们已经了解了函数组合的工作原理，以及我们应该如何使用无点表示法来指定我们的函数，以实现简洁、清晰和灵活性。

现在，让我们尝试在稍微不同的场景中使用这些想法，看看它们的效果如何。 假设我们将 `add10` 替换为 `add`：

```elm
add x y =
    x + y
mult5 value =
    value * 5
```

我们如何仅用这两个函数编写 `mult5After10` 呢？ 在继续阅读之前先想一想。 不，认真的。 想一想。 尝试去做吧。 好吧，如果你真的花时间思考这个问题，你可能会想出一个解决方案，比如：

```elm
-- This is wrong !!!!
mult5AfterAdd10 =
    (mult5 << add) 10 
```

但这是行不通的。为什么？ 因为add需要2个参数。 如果这在 Elm 中不明显，请尝试用 Javascript 编写：

```js
var mult5AfterAdd10 = mult5(add(10)); // this doesn't work
```

这段代码是错误的，但是为什么呢？由于 `add` 函数在此仅获取其 2 个参数中的 1 个，因此其错误结果将传递给 `mult5`。 这会产生错误的结果。事实上，在 Elm 中，编译器甚至不会让你编写这种格式错误的代码（这是 Elm 的伟大之处之一）。

让我们再试一次：

```js
var mult5AfterAdd10 = y => mult5(add(10, y)); // not point-free
```

这不是没有意义的，但我可能可以接受。 但现在我不再只是组合功能。 我正在编写一个新函数。 另外，如果这变得更复杂，例如 如果我想用其他东西来组合 `mult5AfterAdd10`，我就会遇到真正的麻烦。

因此，函数组合的用处似乎有限，因为我们无法将这两个函数结合起来。 这太糟糕了，因为它是如此强大。

我们该如何解决这个问题呢？ 我们需要做什么才能解决这个问题？

好吧，如果我们能够通过某种方式提前为 `add` 函数提供其参数之一，然后在调用 `mult5AfterAdd10` 时获得第二个参数，那就太好了。 事实证明有一种方法，那就是**柯里化 Currying**。

#### 柯里化 Currying

如果您还记得第 3 部分，我们在编写 `mult5` 和 `add (in )` 时遇到问题的原因是因为 `mult5` 采用 1 个参数，而 `add` 采用 2 个参数。

我们可以通过限制所有函数仅采用 1 个参数来轻松解决这个问题。

相信我。 这并不像听起来那么糟糕。

我们简单地编写一个使用 2 个参数但一次只接受 1 个参数的 add 函数。 柯里化函数使我们能够做到这一点。

> **柯里化函数是一次只接受一个参数的函数。**

这将让我们在与 `mult5` 组合之前添加它的第一个参数。 然后当调用 `mult5AfterAdd10` 时，add 将获取它的第二个参数。在Javascript中，我们可以通过重写 `add` 来实现这一点：

```js
var add = x => y => x + y
```

此版本的 `add` 是一种函数，它现在采用一个参数，稍后采用另一个参数。

具体来说，`add` 函数采用单个参数 `x`，并返回一个采用单个参数 `y` 的**函数**，该函数最终将返回 `x` 和 `y` 相加的结果。

现在我们可以使用这个版本的 `add` 来构建 `mult5AfterAdd10` 的可工作版本：

```js
var compose = (f, g) => x => f(g(x));
var mult5AfterAdd10 = compose(mult5, add(10));
```

`compose` 函数有 2 个参数：`f` 和 `g`。 然后它返回一个带有 1 个参数 `x` 的函数，调用该函数时会将 `f after g` 应用于 `x`。

那么我们到底做了什么？ 好吧，我们将普通的旧 `add` 函数转换为柯里化版本。 这使得 `add` 更加灵活，因为第一个参数 `10` 可以预先传递给它，最后一个参数将在调用 `mult5AfterAdd10` 时传递。

此时，您可能想知道如何在 Elm 中重写 `add` 函数。 事实证明，你不必这样做。 在 Elm 和其他函数式语言中，所有函数都会自动柯里化。

所以 add 函数看起来是一样的：

```elm
add x y =
    x + y
```

这就是第 3 部分中 `mult5AfterAdd10` 应该写回的方式：

```elm
mult5AfterAdd10 =
    (mult5 << add 10)
```

从语法上来说，Elm 击败了 Javascript 等命令式语言，因为它针对柯里化和组合等功能性事物进行了优化。

#### 柯里化和重构 Currying and Refactoring

柯里化的另一个亮点是在重构过程中，当您创建具有大量参数的函数的通用版本，然后使用它创建具有较少参数的专用版本时。 例如，当我们有以下函数将字符串括在括号和双括号中时：

```elm
bracket str =
    "{" ++ str ++ "}"
doubleBracket str =
    "{{" ++ str ++ "}}"
```

我们的使用方法如下：

```elm
bracketedJoe =
    bracket "Joe"
doubleBracketedJoe =
    doubleBracket "Joe"
```

我们可以将 `bracket` 和 `doubleBracket` 通用化：

```elm
generalBracket prefix str suffix =
    prefix ++ str ++ suffix
```

但现在每次我们使用 `generalBracket` 时，我们都必须传入括号：

```elm
bracketedJoe =
    generalBracket "{" "Joe" "}"
doubleBracketedJoe =
    generalBracket "{{" "Joe" "}}"
```

我们真正想要的是两全其美。 如果我们重新排序 `generalBracket` 的参数，我们可以利用函数柯里化的事实来创建 `bracket` 和 `doubleBracket`：

```elm
generalBracket prefix suffix str =
    prefix ++ str ++ suffix
bracket =
    generalBracket "{" "}"
doubleBracket =
    generalBracket "{{" "}}"
```

请注意，通过将最有可能是静态的参数放在前面，即 `prefix` 和 `suffix`，然后将最有可能更改的参数放在最后，即 `str`，我们可以轻松创建 `genericBracket` 的特化版本。

> 参数顺序对于充分利用柯里化非常重要。

另请注意，`bracket`和 `doubleBracket` 是用无点表示法编写的，即隐含了 `str` 参数。 `Bracket` 和 `doubleBracket` 都是等待其最终参数的函数。 现在我们可以像以前一样使用它：

```elm
bracketedJoe =
    bracket "Joe"
doubleBracketedJoe =
    doubleBracket "Joe"
```

但这次我们使用通用柯里化函数 `generalBracket`。

#### 常用功能函数 Common Functional Functions

![img](https://miro.medium.com/v2/resize:fit:468/1*I7nCgMOzuVxKPj_amfQxNw.png)

让我们看一下函数式语言中使用的 3 个常见函数。 但首先，让我们看一下下面的 Javascript 代码：

```js
for (var i = 0; i < something.length; ++i) {
    // do stuff
}
```

这段代码有一个重大问题。 这不是一个错误。 问题是这段代码是样板代码，即一遍又一遍地编写的代码。 如果您使用 Java、C#、Javascript、PHP、Python 等命令式语言进行编码，您会发现自己编写的样板代码比其他任何代码都多。 这就是问题所在。

那么我们就杀掉它吧。 让我们将其放入一个函数（或几个函数）中，并且不再编写 `for` 循环。 好吧，几乎从来没有； 至少在我们转向函数式语言之前是这样。 让我们从修改一个名为 `things` 的数组开始：

```js
var things = [1, 2, 3, 4];
for (var i = 0; i < things.length; ++i) {
    things[i] = things[i] * 10; // MUTATION ALERT !!!!
}
console.log(things); // [10, 20, 30, 40]
```

可变性！ 让我们再试一次。 这次我们不会改变 `things`：

```js
var things = [1, 2, 3, 4];
var newThings = [];
for (var i = 0; i < things.length; ++i) {
    newThings[i] = things[i] * 10;
}
console.log(newThings); // [10, 20, 30, 40]
```

好吧，所以我们没有改变 `things`，但从技术上讲，我们改变了 `newThings`。 现在，我们将忽略这一点。 毕竟我们是在 Javascript 中。 一旦我们转向函数式语言，我们将无法改变。

这里的重点是了解这些函数如何工作并帮助我们减少代码中的噪音。

让我们将这段代码放入一个函数中。 我们将调用我们的第一个通用函数 **`map`**，因为它将旧数组中的每个值映射到新数组中的新值：

```js
var map = (f, array) => {
    var newArray = [];
    for (var i = 0; i < array.length; ++i) {
        newArray[i] = f(array[i]);
    }
    return newArray;
};
```

请注意，函数 `f` 被传入，以便我们的 `map` 函数可以对数组的每个项目执行我们想要的任何操作。

现在我们可以调用重写之前的代码来使用 `map`：

```js
var things = [1, 2, 3, 4];
var newThings = map(v => v * 10, things);
```

没有 `for` 循环。 而且更容易阅读，因此更容易推理。 嗯，从技术上讲，`map` 函数中有 `for` 循环。 但至少我们不必再编写样板代码了。

现在让我们编写另一个常用函数来**过滤**数组中的内容：

```js
var filter = (pred, array) => {
    var newArray = [];
	for (var i = 0; i < array.length; ++i) {
        if (pred(array[i]))
            newArray[newArray.length] = array[i];
    }
    return newArray;
};
```

请注意，如果我们保留该项目，谓词函数 `pred` 如何返回 `TRUE`，如果我们扔掉它，则返回 `FALSE`。

下面介绍如何使用 `filter` 过滤奇数：

```js
var isOdd = x => x % 2 !== 0;
var numbers = [1, 2, 3, 4, 5];
var oddNumbers = filter(isOdd, numbers);
console.log(oddNumbers); // [1, 3, 5]
```

使用我们的新 `filter` 函数比使用 `for` 循环手动编码要简单得多。

最后一个公共函数称为 `reduce`。 通常，它用于获取列表并将其减少为单个值，但实际上它可以做更多的事情。 这个函数在函数式语言中通常被称为 `fold`。

```js
var reduce = (f, start, array) => {
    var acc = start;
    for (var i = 0; i < array.length; ++i)
        acc = f(array[i], acc); // f() takes 2 parameters
    return acc;
});
```

`reduce` 函数采用一个缩减函数 `f`、一个 `start` 值和一个 `array`。 请注意，归约函数 `f` 有 2 个参数：` array` 的当前项和累加器 `acc`。 每次迭代它将使用这些参数生成一个新的累加器。 返回最终迭代的累加器。

一个例子将帮助我们理解它是如何工作的：

```js
var add = (x, y) => x + y;
var values = [1, 2, 3, 4, 5];
var sumOfValues = reduce(add, 0, values);
console.log(sumOfValues); // 15
```

请注意，`add` 函数接受 2 个参数并将它们相加。 我们的 `reduce` 函数需要一个带有2 个参数的函数，以便它们可以很好地协同工作。我们从起始值零开始，并传入要求和的数组 `value`。 在 `reduce` 函数内部，当迭代 `value` 时，总和会被累加。 最终累加值以 `sumOfValues` 形式返回。

这些函数（`map`、`filter` 和 `reduce`）都让我们可以对数组执行常见的操作操作，而无需编写样板 `for` 循环。但在函数式语言中，它们甚至更有用，因为没有循环结构，只有递归。 迭代函数不仅非常有用。 它们是必要的。

#### 引用透明性 Referential Transparency

引用透明性是一个奇特的术语，用于描述纯函数可以安全地被其表达式替换。 一个例子将有助于说明这一点。

在代数中，当你有以下公式时：

```
y = x + 10
```

如果我们令 `x = 3`

您可以将 `x` 代回方程中以获得：

```
y = 3 + 10
```

请注意，该等式仍然有效。 我们可以用纯函数进行同样的替换。

下面是 Elm 中的一个函数，它在提供的字符串两边加上单引号：

```elm
quote str =
    "'" ++ str ++ "'"
```

这是使用它的一些代码：

```elm
findError key =
    "Unable to find " ++ (quote key)
```

这里，当搜索 `key` 不成功时，`findError` 会生成一条错误消息。

由于 `quote` 函数是纯函数，我们可以简单地将 `findError` 中的函数调用替换为 `quote` 函数的主体（这只是一个表达式）：

```elm
findError key =
   "Unable to find " ++ ("'" ++ str ++ "'")
```

这就是我所说的**反向重构 Reverse Refactoring**（这对我来说更有意义），程序员或程序（例如编译器和测试程序）可以使用它来推理代码的过程。 这在推理递归函数时特别有用。

#### 执行顺序 Execution Order

大多数程序都是单线程的，即一次只执行一段代码。 即使您有一个多线程程序，大多数线程也会被阻塞等待 I/O 完成，例如 文件、网络等。

这就是我们在编写代码时自然地考虑有序步骤的原因之一：

```
1. Get out the bread
2. Put 2 slices into the toaster
3. Select darkness
4. Push down the lever
5. Wait for toast to pop up
6. Remove toast
7. Get out the butter
8. Get a butter knife
9. Butter toast
```

在这个例子中，有两个独立的操作：getting butter 和 toasting bread。 它们只有在第 9 步才变得相互依赖。

我们可以将步骤 7 和 8 与步骤 1 到 6 同时执行，因为它们彼此独立。

但一旦我们这样做，事情就变得复杂了：

```
Thread 1
--------
1. Get out the bread
2. Put 2 slices into the toaster
3. Select darkness
4. Push down the lever
5. Wait for toast to pop up
6. Remove toast
Thread 2
--------
1. Get out the butter
2. Get a butter knife
3. Wait for Thread 1 to complete
4. Butter toast
```

如果线程 1 失败，线程 2 会发生什么情况？ 协调两个线程的机制是什么？ 谁拥有该 toast：线程 1、线程 2 或两者？

不考虑这些复杂性并让我们的程序保持单线程会更容易。

但是，当值得榨取程序的所有可能效率时，我们就必须付出巨大的努力来编写多线程软件。

然而，多线程有两个主要问题。 首先，多线程程序难以编写、读取、推理、测试和调试。 其次，一些语言，例如 Javascript不支持多线程，而支持多线程的则支持得很差。

但如果顺序无关紧要并且所有事情都是并行执行的呢？ 虽然这听起来很疯狂，但它并不像听起来那么混乱。 让我们看一些 Elm 代码来说明这一点：

```elm
buildMessage message value =
    let
        upperMessage =
            String.toUpper message
        quotedValue =
            "'" ++ value ++ "'"
    in
        upperMessage ++ ": " ++ quotedValue
```

这里 `buildMessage` 接受 `message` 和 `value`，然后生成一个大写的`message`、一个冒号和单引号中的`value`。

请注意 `upperMessage` 和 `QuotedValue` 是如何独立的。 我们怎么知道呢？

为了独立必须满足两件事。 首先，它们必须是纯函数。 这很重要，因为他们不能受到对方执行的影响。如果它们不纯粹，那么我们永远不会知道它们是独立的。 在这种情况下，我们必须依靠它们在程序中调用的顺序来确定它们的执行顺序。 这就是所有命令式语言的工作方式。

对于独立性必须正确的第二件事是一个函数的输出不用作另一个函数的输入。 如果是这种情况，那么我们必须等待一个完成才能开始第二个。

在这种情况下，`upperMessage` 和 `quotedValue` 都是纯函数，并且都不需要对方的输出。因此，这两个函数可以按任何顺序执行。

编译器无需程序员的帮助即可做出此决定。 这只有在纯函数式语言中才有可能实现，因为确定副作用的后果即使不是不可能，也是非常困难的。

> **纯函数式语言中的执行顺序可以由编译器确定。**

考虑到 CPU 并没有变得更快，这是非常有利的。 相反，制造商正在添加越来越多的核心。 这意味着代码可以在硬件级别并行执行。

不幸的是，对于命令式语言，我们无法充分利用这些核心，除非在非常粗略的层面上。 但要做到这一点需要彻底改变我们程序的架构。

借助纯函数式语言，我们有可能在细粒度级别上自动利用 CPU 内核，而无需更改任何一行代码。

#### 类型注释 Type Annotations

在静态类型语言中，类型是内联定义的。 下面是一些Java代码来说明：

```java
public static String quote(String str) {
    return "'" + str + "'";
}
```

请注意类型 typing 如何与函数定义内联。 当你有泛型时，情况会变得更糟：

```java
private final Map<Integer, String> getPerson(Map<String, String> people, Integer personId) {
   // ...
}
```

我将类型加粗，使它们更显眼，但它们仍然干扰函数定义。 您必须仔细阅读它才能找到变量的名称。

对于动态类型语言，这不是问题。 在Javascript中，我们可以编写如下代码：

```js
var getPerson = function(people, personId) {
    // ...
};
```

这更容易阅读，而不会受到所有讨厌的类型信息的影响。 唯一的问题是我们放弃了类型的安全性。 我们可能会比较轻易地反向传递这些参数，比如用于 `people` 的*数字*和用于 `personId` 的*对象*。直到程序执行后我们才会知道，这可能是在我们投入生产几个月后。 在 Java 中就不会出现这种情况，因为它无法编译。

事实证明我们可以。 这是 Elm 中带有类型注释的函数：

```elm
add : Int -> Int -> Int
add x y =
    x + y
```

请注意类型信息如何位于单独的一行上。 这种分离使世界变得不同。

现在您可能认为类型注释有拼写错误。 我知道当我第一次看到它时我就这么做了。 我认为第一个 -> 应该是逗号。 但没有错别字。

当你看到它带有隐含的括号时，它就更有意义了：

```elm
add : Int -> (Int -> Int)
```

这表示 `add` 是一个采用 `Int` 类型的*单个*参数并返回一个采用*单个* `Int` 参数并返回 `Int` 的函数。

这是另一个带有隐含括号的类型注释：

```elm
doSomething : String -> (Int -> (String -> String))
doSomething prefix value suffix =
    prefix ++ (toString value) ++ suffix
```

这表示 `doSomething` 是一个函数，它采用 `String` 类型的单个参数并返回一个采用 `Int` 类型的单个参数的函数，这个函数返回一个采用 `String` 类型的单个参数并返回 `String` 的函数。

请注意所有内容如何采用单个参数。 这是因为每个函数都在 Elm 中进行柯里化。

由于括号总是隐含在右侧，因此它们不是必需的。 所以我们可以简单地写：

```elm
doSomething : String -> Int -> String -> String
```

当我们将函数作为参数传递时，括号是必需的。 如果没有它们，类型注释就会不明确。 例如：

```elm
takes2Params : Int -> Int -> String
takes2Params num1 num2 =
    -- do something
```

与以下内容有很大不同：

```elm
takes1Param : (Int -> Int) -> String
takes1Param f =
    -- do something
```

`take2Param` 是一个需要 2 个参数的函数，一个 `Int` 和另一个 `Int`。 而 `take1Param` 需要 1 个参数，即一个接受一个 `Int` 和另一个 `Int` 的函数。

这是 `map` 的类型注释：

```elm
map : (a -> b) -> List a -> List b
map f list =
    // ...
```

这里需要括号，因为 `f` 是 `(a -> b)` 类型，即一个函数接受 `a` 类型的单个参数并返回 `b` 类型的内容。

这里的 `a` 是任意类型。 当类型为大写时，它是显式类型，例如 `String`。 当类型为小写时，它可以是任何类型。 这里的 `a` 可以是 `String`，也可以是 `Int`。

如果您看到 `(a -> a)` 则表示输入类型和输出类型必须相同。 它们是什么并不重要，但它们必须匹配。

但对于地图来说，我们有 `(a -> b)`。 这意味着它**可以**返回不同的类型，但也可以返回相同的类型。

但是一旦确定了 `a` 的类型，`a` 就必须是整个签名的类型。 例如，如果 `a` 是 `Int`，`b` 是 `String`，则签名等效于：

```elm
(Int -> String) -> List Int -> List String
```

这里所有的 `a` 都被替换为 `Int`，所有的 `b` 都被替换为 `String`。

`List Int` 类型表示列表包含 `Int`，`List String` 表示列表包含 `String`。 如果您在 Java 或其他语言中使用过泛型，那么这个概念应该很熟悉。

### 接下来

现在您已经学习了所有这些很棒的新东西，您可能会想：“现在怎么办？ 我如何在日常编程中使用它？”

这取决于。 如果您可以使用 Elm 或 Haskell 等纯函数式语言进行编程，那么您就可以利用所有这些想法。 这些语言使这一切变得很容易。

如果你只能使用像 Javascript 这样的命令式语言进行编程，就像我们许多人必须的那样，那么你仍然可以使用你学到的很多东西，但需要更多的纪律。

#### Functional Javascript

Javascript 具有许多特性，可让您以更函数式的方式进行编程。 它并不纯粹，但您可以在语言中获得一些不变性，甚至可以通过库获得更多不变性。

它并不理想，但如果您必须使用它，那么为什么不获得函数式语言的一些好处呢？

##### 不变性 Immutability

首先要考虑的是不变性。 在 ES2015 或 ES6 中，有一个新关键字 `const`。 这意味着一旦设置了变量，就无法重置它：

```js
const a = 1;
a = 2; // this will throw a TypeError in Chrome, Firefox or Node
       // but not in Safari (circa 10/2016)
```

这里 `a` 被定义为常量，因此一旦设置就不能更改。 这就是为什么 `a = 2` 抛出异常（Safari 除外）。

Javascript 中 `const` 的问题在于它的作用还不够。 下面的例子说明了它的局限性：

```js
const a = {
    x: 1,
    y: 2
};
a.x = 2; // NO EXCEPTION!
a = {}; // this will throw a TypeError
```

请注意 `a.x = 2` 不会引发异常。 `const` 关键字唯一不可变的是变量 `a`。 `a` 指向的任何东西都可变。

这是非常令人失望的，因为它会让 Javascript 变得更好。 那么我们如何在 Javascript 中获得不变性呢？

不幸的是，我们只能通过一个名为 Immutable.js 的库来做到这一点。 这可能会给我们带来更好的不变性，但遗憾的是，它使我们的代码看起来更像 Java，而不是 Javascript。

##### 柯里化和组合 Currying and Composition

在本系列的前面部分，我们学习了如何编写柯里化函数。 这是一个更复杂的例子：

```js
const f = a => b => c => d => a + b + c + d
```

请注意，我们必须手动编写柯里化部分。 要调用 `f`，我们必须编写：

```js
console.log(f(1)(2)(3)(4)); // prints 10
```

但这足以让 Lisp 程序员哭泣。

有许多库可以使这个过程变得更容易。 我最喜欢的是 Ramda 。 使用 Ramda 我们现在可以写：

```js
const f = R.curry((a, b, c, d) => a + b + c + d);
console.log(f(1, 2, 3, 4)); // prints 10
console.log(f(1, 2)(3, 4)); // also prints 10
console.log(f(1)(2)(3, 4)); // also prints 10
```

函数定义并没有好多少，但我们消除了对所有这些括号的需要。 请注意，每次调用 `f` 时，我们可以根据需要应用任意数量的参数。

通过使用 Ramda，我们可以重写第 3 部分和第 4 部分中的 `mult5AfterAdd10` 函数：

```js
const add = R.curry((x, y) => x + y);
const mult5 = value => value * 5;
const mult5AfterAdd10 = R.compose(mult5, add(10));
```

事实证明，Ramda 有很多辅助函数可以完成这类事情，例如 `R.add` 和 `R.multiply`，这意味着我们可以编写更少的代码：

```js
const mult5AfterAdd10 = R.compose(R.multiply(5), R.add(10));
```

##### Map, Filter and Reduce

Ramda 也有自己的 `map`、`filter` 和 `reduce` 版本。 尽管这些函数存在于普通 Javascript 的 `Array.prototype` 中，但 Ramda 的版本是柯里化的：

```js
const isOdd = R.flip(R.modulo)(2);
const onlyOdd = R.filter(isOdd);
const isEven = R.complement(isOdd);
const onlyEven = R.filter(isEven);
const numbers = [1, 2, 3, 4, 5, 6, 7, 8];
console.log(onlyEven(numbers)); // prints [2, 4, 6, 8]
console.log(onlyOdd(numbers)); // prints [1, 3, 5, 7]
```

`R.modulo` 有 2 个参数。 第一个是被除数 dividend，第二个是除数 divisor。

`isOdd` 函数只是除以 2 的余数。余数为 0 为*假*，表明数为非奇数，余数为 1 为*真*，表明数为奇数。 我们翻转了 `modulo` 的第一个和第二个参数，以便我们可以指定 2 作为除数。

`isEven` 函数只是 `isOdd` 的补集。

`onlyOdd` 函数是**谓词**（返回布尔值的函数）为 `isOdd` 的**过滤函数**。 它在执行之前等待数字列表，即它的最终参数。

`onlyEven` 函数是一个使用 `isEven` 作为谓词的过滤器。

当我们将 `numbers` 传递给 `onlyEven` 和 `onlyOdd` 时，`isEven` 和 `isOdd` 会获得它们的最终参数，并最终可以执行返回我们期望的数字。

##### Javascript 的短板

到目前为止，Javascript 已经有了所有的库和语言增强功能，但它仍然受到这样一个事实的困扰：它是一种命令式语言，试图成为所有人的一切。

大多数前端开发人员都在浏览器中使用 Javascript，因为长期以来它一直是唯一的选择。 但许多开发人员现在不再直接编写 Javascript。

相反，他们用不同的语言编写并编译，或者更准确地说，转换为 Javascript。

CoffeeScript 是最早的这些语言之一。 现在，Typescript 已经被 Angular 2 采用。Babel 也可以被认为是 Javascript 的转译器。

越来越多的人在生产中采用这种方法。 但这些语言都是从 Javascript 开始的，只是让它稍微好一点而已。 为什么不一路从纯函数式语言转译为 Javascript？

##### Elm

在本系列中，我们通过 Elm 来帮助理解函数式编程。

**但 Elm 是什么？ 我该如何使用它？**

Elm 是一种纯函数式语言，可编译为 Javascript，因此您可以使用它使用 Elm 架构（又名 TEA）创建 Web 应用程序（该架构启发了 Redux 的开发人员）。

Elm 程序没有任何运行时错误。

Elm 正在 NoRedInk 等公司的生产中使用，Elm 的创建者 Evan Czapliki 现在就职于 NoRedInk（他之前为 Prezi 工作）。

请参阅来自 NoRedInk 和 Elm 布道者的 Richard Feldman 的演讲“[6 Months of Elm in Production](https://www.youtube.com/watch?v=R2FtMbb-nLs)”，了解更多信息。

**我必须用 Elm 替换所有 Javascript 吗？**

不。您可以逐步更换。 请参阅此博客文章“[如何在工作中使用 Elm](http://elm-lang.org/blog/how-to-use-elm-at-work)”以了解更多信息。

##### 为什么要学习Elm？

1. 使用纯函数式语言进行编程既是有限制的，也是自由的。 它限制了你能做的事情（主要是防止你搬起石头砸自己的脚），但同时它也让你免于错误和糟糕的设计决策，因为所有 Elm 程序都遵循 Elm 架构，即功能响应式模型。
2. 函数式编程会让你成为更好的程序员。 本文中的想法只是冰山一角。 您确实需要在实践中看到它们，才能真正理解您的程序将如何缩小尺寸并提高稳定性。
3. **Javascript 最初是在 10 天之内构建的，然后在过去的二十年里进行了修补，成为一种有点函数式、有点面向对象和完全命令式的编程语言。**Elm 的设计利用了 Haskell 社区过去 30 年的工作经验，这些经验源自数十年的数学和计算机科学工作。Elm 架构 (TEA) 经过多年的设计和完善，是 Evan 的函数响应式编程论文的成果。 观看[《控制时间和空间》](https://www.youtube.com/watch?v=Agu6jipKfYw)，了解此设计的构思水平。
4. Elm 是为前端 Web 开发人员设计的。 其目的是让他们的生活更轻松。 观看[“让我们成为主流”](https://www.youtube.com/watch?v=oYk8CKH7OhE)以更好地理解这一目标。

##### 未来

我们不可能知道未来会怎样，但我们可以做出一些有根据的猜测。 以下是我的一些：

> 编译为 Javascript 的语言将会出现明显的趋势。
>
> 已经存在 40 多年的函数式编程思想将被重新发现，以解决我们当前的软件复杂性问题。
>
> 硬件状态，例如 千兆字节的廉价内存和快速处理器将使函数式技术变得可行。
>
> CPU 不会变得更快，但核心数量会继续增加。
>
> 可变状态将被认为是复杂系统中最大的问题之一。

我写这一系列文章是因为我相信函数式编程是未来，并且因为我在过去几年中努力学习它（我仍在学习）。

我的目标是帮助其他人比我更容易、更快地学习这些概念，并帮助其他人成为更好的程序员，以便他们将来能够拥有更适合市场的职业。

即使我对 Elm 将成为未来一门巨大语言的预测是错误的，我也可以肯定地说，无论未来如何，函数式编程和 Elm 都将走上正轨。