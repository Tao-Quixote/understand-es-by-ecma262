# void 操作符 🗑

在阅读 HTML 和 JavaScript 代码时，我们经常会看到 `void 0` 这个表达式。

如：

```html
<!-- html -->
<a href="javascript: void 0;" click="foo()">do sm.</a>
```
或者：

```javascript
// javascript
(function (undefined) {
	...
})(void 0)
```

而且在很多兼容老版本浏览器的库中，也经常会使用 `void 0` 这个表达式。那么这个表达式的作用是什么呢？

在 [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/void) 中，是这样解释 `void` 操作符的：

> This operator allows evaluating expressions that produce a value into places where an expression that evaluates to undefined is desired.

> The void operator is often used merely to obtain the undefined primitive value, usually using "void(0)" (which is equivalent to "void 0"). In these cases, the global variable undefined can be used instead (assuming it has not been assigned to a non-default value).

其实 `void` 操作符的作用很简单，仅仅是用来获取一个原始数据类型 **undefined**；而且在这种情况下，也可以使用全局变量 **undefined** 来代替 `void` 操作符构成的表达式。

## ecma-262 中规定的 void 操作符的行为

在这一节中，我们来看一下在 ecma-262 中对 void 的定义及其行为。

在 [ecma-262](https://tc39.github.io/ecma262/#sec-unary-operators) 中，**void** 的定义为 **一元操作符(Unary Operators)**，包含 void 操作符的表达式为 一元表达式：

```ecma262
12.5 Unary Operators

Syntax
UnaryExpression[Yield, Await]:
	UpdateExpression[?Yield, ?Await]
	delete UnaryExpression[?Yield, ?Await]
	void UnaryExpression[?Yield, ?Await]
	typeof UnaryExpression[?Yield, ?Await]
	+ UnaryExpression[?Yield, ?Await]
	- UnaryExpression[?Yield, ?Await]
	~ UnaryExpression[?Yield, ?Await]
	! UnaryExpression[?Yield, ?Await]
	[+Await] AwaitExpression[?Yield]
```

### void 表达式的执行步骤

下面看一下规范中是如何定义 `void UnaryExpression` 的执行步骤的：

```ecma262
1、Let expr be the result of evaluating UnaryExpression.
2、Perform ? GetValue(expr).
3、Return undefined.
```

从上面的规范步骤中可以看到，1、void 表达式会先将 一元表达式 的结果存储在变量 **expr** 中；2、然后执行 [GetValue(expr)](#get-value)；3、最后，不管表达式的执行结果是什么，均返回 undefined。这里 void 表达式返回的 undefined 即为全局对象的 undefined 属性。

**GetValue(expr)** 的执行步骤请看下一小节。

### <span id="get-value">GetValue(expr)</span>

其实在包含 void 操作符的表达式中，GetValue(expr) 的执行结果对 void 表达式的返回值没有影响；但是，ECMAScript 的实现版本必须依据以下步骤执行该表达式，因为 void 操作符后的表达式可能会对其他的数据进行操作，所以 void 操作符后的表达式要严格遵循下面的步骤执行：

```javascript
1、ReturnIfAbrupt(V).
2、If Type(V) is not Reference, return V.
3、Let base be GetBase(V).
4、If IsUnresolvableReference(V) is true, throw a ReferenceError exception.
5、If IsPropertyReference(V) is true, then
	a、If HasPrimitiveBase(V) is true, then
		i、Assert: In this case, base will never be undefined or null.
		ii、Set base to !ToObject(base).
	b、Return ? base.[[Get]](GetReferencedName(V), GetThisValue(V)).
6、Else base must be an Environment Record,
	a、Return ? base.GetBindingValue(GetReferencedName(V), IsStrictReference(V))
```

上面的步骤翻译过来如下：

* 1、如果为中断的操作则返回，中断的操作主要指会控制程序流程的操作，如 **break**，**continue**，**return**，**throw** 等；
* 2、如果 **V** 的类型不是引用类型，则为基本数据类型，返回该基本数据类型的值；
* 3、将 GetBase(V) 获取的 V 的基本值存储在变量 base 中；
* 4、如果 V 是不可解析的引用变量，抛出一个 **ReferenceError** 异常；
* 5、如果 V 是一个引用类型的属性：
	* a、如果 V 有原始基本类型的值(即 V 的值是引用类型且为 Boolean、String、Symbol 或者 Number 中的任意一种)：
		*  i、断言：在这种情况下，base 一定为非 undefined 和 null 的值
		*  ii、将 base 的值设置为 base 这个原始数据的包装类的值
	* b、执行 base 的内部方法 [[Get]] 并返回
* 6、如果步骤走到这里，则 base 一定是一个环境变量(Environment Record)：
	* a、返回 base.GetBindingValue(GetReferencedName(V), IsStrictReference(V)) 的执行结果；

> 注意，在 5.a.ii 这步中生成的对象在该抽象操作之外不可访问，且 5.b 中的 [[Get]] 方法也应该是一个内部方法，不向外暴露。ECMAScript 的具体实现应避免在该步骤中创建真实的对象。

## void 操作符的使用场景

### 1、获取真正的 undefined

在 JavaScript 中，undefined 是全局变量的一个属性，且该属性的值也是 undefined。但是在 ECMAScript3 中，该属性可写且可配置，这就意味在 ECMAScript3 这个版本中，任意的代码都有可能在无意中覆盖全局的 undefined 变量，所以，void 操作符更像是针对此问题的一个补丁(patch)。

```javascript
// ECMAScript3

(function () {
	undefined = {a: 'a'}
})()
```

上面的代码中，IIFE 在执行之后，会将全局的 undefined 变量修改为一个对象；所以，为了解决 undefined 可能被覆盖的问题，很多的 JavaScript 库中采用了 `void 0` 这种方式来解决这个问题；在需要使用 undefined 的地方，一律使用 `void 0` 来代替字面量的 undefined。

在 ECMAScript3 之后的版本中，规范规定了 undefined 不可被覆盖，保证了 undefined 的唯一性：

```javascript
Object.getOwnPropertyDescriptor(window, undefined)
// {value: undefined, writable: false, enumerable: false, configurable: false}
```

我们可以通过使用 `Object.getOwnPropertyDescriptor` 方法来获取 undefined 属性的描述符，从上面的结果可以看到，undefined 属性被设置为 **不可写(writable: false)** 且 **不可配置(configurable: false)**。在 ECMAScript5 及其之后的版本中，修改全局 undefined 会静默失败；在严格模式下，修改全局 undefined 会报错。

虽然不允许修改全局的 undefined 属性，但是 undefined 在 JavaScript 中是一个合法的标识符，所以下面的代码是合法的：

```javascript
function foo () {
	let undefined = 3
	console.log(undefined)
}

foo(); // 3
```

在函数作用域中可以声明名为 undefined 变量，与普通变量无任何区别。

## Links 🐬

参考链接如下，排名不分先后：

* [void operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/void)
* [ecma-262 The void Operator](https://tc39.github.io/ecma262/#sec-void-operator)
* [javascript void 0](http://www.tizag.com/javascriptT/javascriptvoid.php)

## <span id="author">Author Info 🌟</span>

* [GitHub](https://github.com/Tao-Quixote)
* Email: <web.taox@gmail.com>
