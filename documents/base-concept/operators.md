# JavaScript 中的操作符

在 JavaScript 中使用操作符时，最经常用到，也最容易搞错的，就是在使用操作符时 JavaScript 引擎自动进行的隐式类型转换。

本文会从 ECMA 规范的层面来讲一下不同操作符在执行时的运行规则，从规范的角度来理解一个操作符的结果为什么是输出的那样，以及具体实现与规范不同的地方。

## <span id="relational">关系操作符</span>

关系操作符主要包括如下几个操作符(这里介绍的关系操作符不包括 “等于”，相等操作符单独讨论)：

* 大于(>)
* 小于(<)
* 大于等于(>=)
* 小于等于(<=)

以上几个操作符都会返回一个 **布尔值** 作为返回结果。如果只是进行基本的数值比较，跟我们在数学课上学到的内容是一样的，结果也是一样的；但是在 JavaScript 中，会使用关系操作符进行比较不只是纯数字，还可能包括各种基本数据类型(`number`, `string`, `boolean`, `null`, `undefined`)以及对象(包括 `Object`, `Function`, `Array`)之间的比较。在比较的过程中，ECMAScript 规范中规定了针对不同数据类型使用不同操作符时 **版本实现(ECMA 只规定规范，由不同厂商负责具体实现)** 需要进行相应的隐式类型转换。

接下来我们先看一下规则：

### 1、如果两个操作数都是数字

> 执行数值比较

### 2、如果两个操作数都是字符串

> 在比较字符串时，实际比较的是两个字符串对应位置字符的字符编码的值。

例如对于应为字母组成的字符串 `let str1 = 'abc'` 和 `let str2 = 'abd'` 来说，JavaScript 引擎在处理 `str1 > str2` 这个表达式时，实际的处理流程为对字符串 `'abc'` 和 `'abd'` 逐位求字符编码，然后以数字的形式比较字符编码的大小；如果两个字符串第一个字符的字符编码相同，则比较第二个字符的字符编码，依次向后，直到最后一个字符。

### 3、其中一个操作数是数字

> 需要将另外一个操作数转换为数值，然后进行数值的比较

这条规则正好补上了前面两种规则的漏洞，即当两个操作数分别为数字与字符串时的比较结果。

### 4、其中一个操作数是布尔值

> 布尔值线转换为数字，再进行比较

```javascript
true > false  // true
1 > false	// true
```

上面的例子中，布尔值会先转换为数字，然后再进行数值的比较。

**注：布尔值转换为数字时，false 会转换为 0，true 会被转换为 1。[JavaScript 中的数据类型 - Number 类型](./types.md#number)**

### 5、其中一个操作数是对象

> 如果其中一个操作数是对象，则会在比较时先调用该对象的 `valueOf()` 方法，对返回值按前面的规则进行比较；如果该对象没有 `valueOf()` 方法，则尝试调用对象的 `toString()` 方法，再次对返回值按前面的规则进行比较。

**注意：上面的对象是广义的对象，包括数组和函数。**

```javascript
// 对象进行类型转换，调用 valueOf 方法
let obj = {
	valueOf () {
		return 3
	}
}
obj > 2	// true，此时会调用 obj.valueOf 方法，该方法的返回值为3

// 对象进行类型转换，调用 toString 方法
let obj2 = {
	toString () {
		return 2
	}
}
obj2 > 1	// true，此时会调用 obj2.toString 方法，该方法的返回值为 2
```

### 从规范的角度理解关系操作符的执行流程

下面我们以 `RelationalExpression < ShiftExpression`(小于) 做例子：

**[主要流程步骤规范](https://tc39.github.io/ecma262/#sec-relational-operators-runtime-semantics-evaluation) 如下**：

```ecma262
// Runtime Semantics: Evaluation

RelationalExpression: RelationalExpression < ShiftExpression

1、 Let lref be the result of evaluating RelationalExpression.
2、 Let lval be ? GetValue(lref).	// 获取左边操作数的值
3、 Let rref be the result of evaluating ShiftExpression.
4、 Let rval be ? GetValue(rref).	// 获取右边操作数的值
	// 进行抽象的关系比较
5、 Let r be the result of performing Abstract Relational Comparison lval < rval.
6、 ReturnIfAbrupt(r). 
	// 如果比较结果为 undefined，则返回 false
7、 If r is undefined, return false. Otherwise, return r.
```

[GetValue 规范(ecma262)](https://tc39.github.io/ecma262/#sec-getvalue) 如下：

```ecma262
// GetValue 该步骤主要用来定义如何获取操作数标识符的值
1、ReturnIfAbrupt(V).
2、If Type(V) is not Reference, return V.
3、Let base be GetBase(V).
4、If IsUnresolvableReference(V) is true, throw a ReferenceError exception.
5、If IsPropertyReference(V) is true, then
	a、If HasPrimitiveBase(V) is true, then
		i、Assert: In this case, base will never be undefined or null.
		ii、Set base to ! ToObject(base).
	b、Return ? base.[[Get]](GetReferencedName(V), GetThisValue(V)).
6、Else base must be an Environment Record,
	a、Return ? base.GetBindingValue(GetReferencedName(V), IsStrictReference(V)) (see 8.1.1).
```

[Abstract Relational Comparison lval < rval 规范(ecma262)](https://tc39.github.io/ecma262/#sec-abstract-relational-comparison) 如下：

```ecma262
  // 获取基本数据类型，ToPrimitive 方法会将入参转换为 JavaScript 中的基本数据类型
1、If the LeftFirst flag is true, then
	a、Let px be ? ToPrimitive(x, hint Number).
	b、Let py be ? ToPrimitive(y, hint Number).
  // 获取基本数据类型，ToPrimitive 方法会将入参转换为 JavaScript 中的基本数据类型
2、Else the order of evaluation needs to be reversed to preserve left to right evaluation,
	a、Let py be ? ToPrimitive(y, hint Number).
	b、Let px be ? ToPrimitive(x, hint Number).
	// 如果 px 与 px 都是字符串，则按照字符串的字符编码逐一进行数值比较
3、If Type(px) is String and Type(py) is String, then
	a、If IsStringPrefix(py, px) is true, return false.
	b、If IsStringPrefix(px, py) is true, return true.
	c、Let k be the smallest nonnegative integer such that the code unit at index k within px is different
		from the code unit at index k within py. (There must be such a k, for neither String is a prefix of the other.)
	d、Let m be the integer that is the numeric value of the code unit at index k within px.
	e、Let n be the integer that is the numeric value of the code unit at index k within py.
	f、If m < n, return true. Otherwise, return false.
  // 如果并非两个参数都是字符串，首先将参数强制转换为 Number 类型，转换规则见 [JavaScript 中的数据类型 - Number 类型](./types.md#number)
4、Else,
	a、NOTE: Because px and py are primitive values evaluation order is not important.
	// 两个参数都强制转换为数字
	b、Let nx be ? ToNumber(px).
	c、Let ny be ? ToNumber(py).
	d、If nx is NaN, return undefined.
	e、If ny is NaN, return undefined.
	f、If nx and ny are the same Number value, return false.
	g、If nx is +0 and ny is -0, return false.
	h、If nx is -0 and ny is +0, return false.
	i、If nx is +∞, return false.
	j、If ny is +∞, return true.
	k、If ny is -∞, return false.
	l、If nx is -∞, return true.
	m、If the mathematical value of nx is less than the mathematical value of ny—note that these
		mathematical values are both finite and not both zero—return true. Otherwise, return false.
```

## Author Info 🌟

* [GitHub](https://github.com/Tao-Quixote)
* Email: <web.taox@gmail.com>
