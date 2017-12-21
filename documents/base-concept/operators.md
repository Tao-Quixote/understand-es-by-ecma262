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

### 总结

综上可以看出，在 JavaScript 引擎对两个操作数进行第一次隐式数据类型转换时，会将包装类型(Object、Array、Function) 等转换为基本数据类型，然后开始真正的比较两个操作数：1、如果两个操作数在转换之后都是字符串，则逐位对字符的字符编码进行数值比较；2、除此之外的所有数据类型，都需要转换为数字之后进行数值比较。

其实 **关系操作符** 的本质是用来比较数字的，所以除了两个操作数转换后的的基本类型都是字符串这个特例之外，其他类型的操作数都是进行数值比较的，这也符合 **关系操作符** 的本质作用。

## <span id="equal">相等操作符</span>

相等操作符按严格程度分为两类：

* 相等
	* 相等(==)
	* 不等(!=)
* 全等
	* 全等(===)
	* 不全等(!==)

### 相等和不等

这两个操作符的返回结果为布尔值；同关系操作符一样，在必要时，这两个操作符也会先转换操作数(**强制类型转换**)，然后再比较操作数的相等行：

* 如果操作数是布尔值，则转换为数字，转换规则可见 [JavaScript 中的数据类型 - Number 类型](./types.md#number)
* 如果一个数字，一个是字符串，则将字符串强制转换为数字
* 如果一个数字，一个是对象，则调用对象的 `valueOf` 方法，然后根据前面的规则比较；如果没有 `valueOf` 方法，则调用 `toString` 方法，然后将返回值根据前面的规则进行比较
* null 与 undefined 是相等的
* 在比较时，null 与 undefined 不进行强制类型转换
* NaN 与任何值都不相等，所以 NaN 与任何进行 `==` 比较结果都是 false，反之，进行 `!=` 比较结果都是 true
* 两个对象比较时，比较是否为同一个对象(从底层来理解，就是指两个标识符的指针是否指向内存中的同一块内存)

**举个例子：**

```javascript
'false' == false	// false，false 先转换为 0，然后字符串 ‘false’ 转换之后 NaN，所以结果为false
'true' == true	// false，true 先转换为 1，然后字符串 ‘false’ 转换之后 NaN，所以结果为false
0 == false	// true，因为在比较时，false 转换为 0， true 转换为 1
```

**⚠️ 注意：在进行操作符的比较时，不要跟 `if (ifStatement)` 中的条件表达式混淆，也不要类比，因为它们的规范不一致，所以，即使 if (0) {} 这个表达式中的 `0` 会转换为 false，该条件不成立，也不等于 `0 => false == false`；因为这里的转换为 false 转换为 0，0 == 0，所以结果 true；而 if 表达式中的条件表达式的返回值不要求是 布尔值，引擎会自动调用 Boolean() 转换条件表达式的返回值**

### 从规范的角度理解相等操作符的执行流程

下面我们以想等操作为例子来看规范中规定的相等操作符的执行流程：

[主要流程步骤](https://tc39.github.io/ecma262/#sec-equality-operators-runtime-semantics-evaluation)如下：

```ecma262
// Runtime Semantics: Evaluation
EqualityExpression: EqualityExpression == RelationalExpression

  // lref 作为操作符左边表达式的引用
1、Let lref be the result of evaluating EqualityExpression.
  // lval 作为操作符左边表达式的值
2、Let lval be ? GetValue(lref).
  // rref 作为操作符右边表达式的引用
3、Let rref be the result of evaluating RelationalExpression.
  // rval 作为操作符右边表达式的值
4、Let rval be ? GetValue(rref).
  // 返回执行抽象相等比较的值
5、Return the result of performing Abstract Equality Comparison rval == lval.
```

`GetValue()` 同关系操作符中的一样，为同一个步骤，这里不做赘述。

[Abstract Equality Comparison rval == lval 步骤](https://tc39.github.io/ecma262/#sec-abstract-equality-comparison)如下：

```ecma262
The comparison x == y, where x and y are values, produces true or false.
Such a comparison is performed as follows:

// 如果两个操作符类型相同，则执行下面的严格相等步骤，并返回严格相等步骤的返回值
1、If Type(x) is the same as Type(y), then
	a、Return the result of performing Strict Equality Comparison x === y.
// null 与 undefined 相等
2、If x is null and y is undefined, return true.
3、If x is undefined and y is null, return true.
// string 类型的值要转换为 number 类型
4、If Type(x) is Number and Type(y) is String, return the result of the comparison x == ! ToNumber(y).
5、If Type(x) is String and Type(y) is Number, return the result of the comparison ! ToNumber(x) == y.
// 布尔值转换为数字类型
6、If Type(x) is Boolean, return the result of the comparison ! ToNumber(x) == y.
7、If Type(y) is Boolean, return the result of the comparison x == ! ToNumber(y).
// 任一操作数为符合类型，则强制转换为基本数据类型
8、If Type(x) is either String, Number, or Symbol and Type(y) is Object, return the result of the comparison x == ToPrimitive(y).
9、If Type(x) is Object and Type(y) is either String, Number, or Symbol, return the result of the comparison ToPrimitive(x) == y.
// 以上都不符合，返回 false
10、Return false.
```

[严格相等比较运算](https://tc39.github.io/ecma262/#sec-strict-equality-comparison)：

```ecma262
The comparison x === y, where x and y are values, produces true or false.
Such a comparison is performed as follows:

1、If Type(x) is different from Type(y), return false. // 类型不一致，返回 false
// 数字类型，操作数中存在 NaN 时一律返回 false
2、If Type(x) is Number, then
3、If x is NaN, return false.
4、If y is NaN, return false.
// 数值相同，返回 true
5、If x is the same Number value as y, return true.
// +0 与 -0 相等
6、If x is +0 and y is -0, return true.
7、If x is -0 and y is +0, return true.
8、Return false.
// 非数字类型继续比较
9、Return SameValueNonNumber(x, y).
```

[SameValueNonNumber ( x, y )：非数字类型比较](https://tc39.github.io/ecma262/#sec-samevaluenonnumber)：

```ecma262
// SameValueNonNumber ( x, y )

// 判断是否同一数据类型
1、Assert: Type(x) is not Number.
2、Assert: Type(x) is the same as Type(y).
// null 和 undefined 只有一种可能且等于自身，所以返回 true
3、If Type(x) is Undefined, return true.
4、If Type(x) is Null, return true.
// 字符串逐位判断字符编码的数值是否相等
5、If Type(x) is String, then
	a、If x and y are exactly the same sequence of code units (same length and same code units at corresponding indices), return true; otherwise, return false.
// 布尔值判断是否相等
6、If Type(x) is Boolean, then
	a、If x and y are both true or both false, return true; otherwise, return false.
// Symbol 判断是否为同一 Symbol 值
7、If Type(x) is Symbol, then
	a、If x and y are both the same Symbol value, return true; otherwise, return false.
// 对象判断是否为同一个对象
8、If x and y are the same Object value, return true. Otherwise, return false.
```

### 全等和不全等

其实全等在判断过程中会执行相等的全部步骤，不过多了一个步骤，那就是类型检查。全等会在最开始的时候检查两个操作数的类型是否相等，如果两个操作数的类型不一致，则立即返回 false 结束判断；如果类型一致，再按相等的步骤进行判断。

### 总结

从规范可以看出，相等和全等是用来判断两个操作数的某一方面是否是一致的，与关系操作符最大的区别就在于 关系操作符 最主要的原始需求就是比较两个数值的大小，所以规范中才会在很多情形下将操作数强制类型转换为 Number 类型。

还需要注意的一点就是，不要拿操作数在 `if` 语句中的结果来作为操作符中的一种映射，这种映射关系根本就不成立，因为 if 语句中的条件表达式的计算规则与操作符的计算规则并不一样。

不同操作符之间的计算规则虽然有相似的地方，但是并不完全一样，所以不要盲目地生搬硬套。

## Author Info 🌟

* [GitHub](https://github.com/Tao-Quixote)
* Email: <web.taox@gmail.com>
