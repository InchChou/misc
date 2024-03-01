来源：[So You Want to be a Functional Programmer (Part 1) | by Charles Scalfani | Medium](https://cscalfani.medium.com/so-you-want-to-be-a-functional-programmer-part-1-1f15e387e536)

文章中把以往的命令式编程(Imperative Programming)和函数式编程(Functional Programming)比喻为开车与驾驶飞船，其中有个经典的例子：

“就像在你的车里一样，你过去常常倒车才能离开车道。 但在宇宙飞船中，则没有相反的情况。 现在你可能会想，“什么？ 没有逆转？！ 我到底要怎么开车才能不倒车？！””

### 函数式编程的一些概念

#### 纯净性Purity

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