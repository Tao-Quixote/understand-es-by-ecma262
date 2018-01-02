# JavaScript 中的数据类型

* [给 JavaScript 中的数据分类](#category)
	* [typeof](#typeof)
	* [instanceof](#instanceof)
	* [Array.isArray](#isArray)
	* [Object.prototype.toString()](toString)
* [Number 类型](#number)
	* [Number(value) 转换规范](#draft)
	* [特殊值 NaN](#NaN)
	* [window.isNaN VS Number.isNaN](#isNaN)
	* [另一个有趣的现象](#number-toString)
* [Author Info](#author)

JavaScript 中的数据类型分类两大类：

基本数据类型：

* string
* number
* boolean
* null
* undefined

对象：

* Object
* function
* array
* Date
* RegExp

这是 ECMAScript6 之前的数据类型，在 ECMAScript 中新增了三种数据类型：`Symbol`， `Map` 和 `Set`。

关于 ES6 之前的数据类型分类，可以参考 Dr. Axel Rauschmayer 的博文 [Categorizing values in JavaScript(对 JavaScript 中的值进行分类)](http://2ality.com/2013/01/categorizing-values.html)。

## <span id="category">给 JavaScript 中的数据分类 🍭</span>

JavaScript 中有四种方式给数值分类：

* 内部属性[[class]]：该属性保存着一个标示当前值数据类型的字符串
* typeof：主要用来给基本数据类型分类，并与对象进行区分(但是在判断 null 的时候存在 bug，且 bug 无法修复)
* instanceof：用来给对象分类(Function, Array, Object, Date, RegExp...)
* Array.isArray()：该方法用来判断一个值是否是数组的实例

### <span id="typeof">typeof</span>
****

JavaScript 中的变量是松散类型的，在声明时并不会指定变量的类型，所以 JavaScript 提供了 `typeof` 关键字来检查变量中存储的数据的类型。

当我们对一个字面量或者变量使用 typeof 操作符时，会返回上面所说的的类型中的一种。但是在对 `null` 的字面量或者保存 null 值的变量使用 typeof 操作符时，返回的结果并不是 null，而是 object。

为什么 `typeof null` 的结果是 object 呢？我们从[实现(源码为 C 语言)](https://dxr.mozilla.org/classic/source/js/src/jsapi.h)的角度来看：

```c
/*
 * Type tags stored in the low bits of a jsval.
 */
#define JSVAL_OBJECT            0x0     /* untagged reference to object */
#define JSVAL_INT               0x1     /* tagged 31-bit integer value */
#define JSVAL_DOUBLE            0x2     /* tagged reference to double */
#define JSVAL_STRING            0x4     /* tagged reference to string */
#define JSVAL_BOOLEAN           0x6     /* tagged boolean value */
```

JavaScript 的实现中使用 32 位的长度存储数据，并使用低 3 位来表示数据的类型，从这段宏定义可以看出，object 类型的**类型标签(type tag)**为 `000`，即“对象”在存储时低三位的类型标签为 `000`；而 null 在 JavaScript 中其实使用的为机器码中的 NULL 指针，代表该指针的位置存储的所有数字都是 `0`，低三位当然也是 `000`，与 object 类型的低三位相同，所以 `typeof null` 返回的数据类型为 object。

接下来我们看[源码中 `typeof` 的判断逻辑](https://dxr.mozilla.org/classic/source/js/src/jsapi.c#333)：

```c
    JS_PUBLIC_API(JSType)
    JS_TypeOfValue(JSContext *cx, jsval v)
    {
        JSType type = JSTYPE_VOID;
        JSObject *obj;
        JSObjectOps *ops;
        JSClass *clasp;

        CHECK_REQUEST(cx);
        if (JSVAL_IS_VOID(v)) {  // (1) 判断 v 是否为 undefined 类型
            type = JSTYPE_VOID;
        } else if (JSVAL_IS_OBJECT(v)) {  // (2) 判断 v 是否为 object 类型(type tag 为 000)
            obj = JSVAL_TO_OBJECT(v);
            if (obj &&
                (ops = obj->map->ops,
                 ops == &js_ObjectOps
                 ? (clasp = OBJ_GET_CLASS(cx, obj),
                    clasp->call || clasp == &js_FunctionClass) // (3,4)
                 : ops->call != 0)) {  // (3) 判断是否为函数(function)
                type = JSTYPE_FUNCTION;
            } else {
                type = JSTYPE_OBJECT; (4) 是对象但不是 function 的，一律返回 object
            }
        } else if (JSVAL_IS_NUMBER(v)) {
            type = JSTYPE_NUMBER;
        } else if (JSVAL_IS_STRING(v)) {
            type = JSTYPE_STRING;
        } else if (JSVAL_IS_BOOLEAN(v)) {
            type = JSTYPE_BOOLEAN;
        }
        return type;
    }
```

* (1) 判断是否为 undefined 类型
* (2) 判断是否 object 类型，即低三位的值为 `000`
* (3) 判断是否为 callable 或者 [[Class]] 标示为 'function'，如果是则返回 'function'
* (4) 对于是对象但是并不是 function 类型的数据，类型全部返回 object

综上，`typeof null` 的返回值是 'object'。

### <span id="instanceof">instanceof</span>
****

```javascript
a instanceof Array;
```

typeof 操作符主要用来判断数值是不是基本数据类型，以便于与引用类型(Object)进行区别。而 instanceof 操作符则是用来判断两个对象之间关系的，从操作符的名称来看好像是判断 `a` 是否为 Arrat 的实例，但是在 JavaScript 中并不存在真正的 **类** 和 **实例**，所以 instanceof 操作符真正执行的流程是判断 Array 的原型(prototype) 是否在 a 的原型链上(在原型链上的位置不影响)。对基本数据类型使用 instanceof 操作符时会抛出类型错误。

在 ECMAScript5 及其之前的版本中，使用 instanceof 操作符进行判断时引擎会在内部执行如下步骤判断：

**[InstanceofOperator](https://tc39.github.io/ecma262/#sec-instanceofoperator)**

```ecma262
1、If Type(target) is not Object, throw a TypeError exception.
2、If IsCallable(target) is false, throw a TypeError exception.
3、Return ? OrdinaryHasInstance(target, V).
```

而在 ECMAScript6 这个版本中，新增了 Symbol 这种数据类型，并且定义了常量 **Symbol.hasInstance**，如果当前环境支持 Symbol 类型，并且被操作的数值有 Symbol.hasInstance 属性时，会调用该属性指向的方法来判断 a 是否为 Array 的“实例”：

**[InstanceofOperator](https://tc39.github.io/ecma262/#sec-instanceofoperator):**

```ecma262
The abstract operation InstanceofOperator(V, target) implements the generic
algorithm for determining if ECMAScript value V is an instance of object target
either by consulting target's @@hasinstance method or, if absent, determining
whether the value of target's prototype property is present in V's
prototype chain. This abstract operation performs the following steps:

判断 V 是否为 target 的实例，如果 target 对象有 @@hasinstance 属性则调用该方法；
如果没有，则判断 target 的原型(prototype) 是否在 V 的原型链上：

// 判断 target 是否为对象
1、If Type(target) is not Object, throw a TypeError exception.
// 判断 target 对象是否有 @@hasInstance 属性
2、Let instOfHandler be ? GetMethod(target, @@hasInstance).
3、If instOfHandler is not undefined, then
	// 调用 @@hasInstance 属性指向的方法，将返回结果转换为布尔值
	a、Return ToBoolean(? Call(instOfHandler, target, « V »)).
// 判断 target 的构造器是否为一个具有内部属性 [[Call]] 的函数
// 即判断 target.prototype.constructor 是否为一个具有 [[Call]] 属性的函数
4、If IsCallable(target) is false, throw a TypeError exception.
// 返回判断后的结果
5、Return ? OrdinaryHasInstance(target, V).
```

**[OrdinaryHasInstance(target, V)](https://tc39.github.io/ecma262/#sec-ordinaryhasinstance):**

```ecma262
OrdinaryHasInstance ( C, O )：
// 判断 C 的构造器是否为一个具有内部属性 [[Call]] 的函数
// 即判断 C.prototype.constructor 是否为一个具有 [[Call]] 属性的函数
1、If IsCallable(C) is false, return false.
2、If C has a [[BoundTargetFunction]] internal slot, then
	a、Let BC be C.[[BoundTargetFunction]].
	b、Return ? InstanceofOperator(O, BC).
// 判断 O 是否为对象
3、If Type(O) is not Object, return false.
// 获取 C 的原型
4、Let P be ? Get(C, "prototype").
// 判断 C 的原型 P 是否为对象
5、If Type(P) is not Object, throw a TypeError exception.
// 判断 C 的原型 P 与 O 的原型是否为同一个值(即同一个对象)
// 重复此步骤，直到遍历完 O 的原型链或者遇到原型链中的某个原型与 C 的原型 P 相同返回true
// 遍历完之后，O === null，返回 false，即 P 不在 O 的原型链上
6、Repeat, // 重复执行，直到遍历完 O 的原型链
	a、Set O to ? O.[[GetPrototypeOf]]().
	b、If O is null, return false.
	c、If SameValue(P, O) is true, return true.
```

在 ECMAScript6 的规范中，@@hasInstance 在内部被调用时也是按照 **[OrdinaryHasInstance(target, V)](https://tc39.github.io/ecma262/#sec-ordinaryhasinstance):** 中的步骤执行判断。

由于 instanceof 操作符和内部属性 @@hasInstance 在规范中定义的算法逻辑相同(有细微的差别)，所以可以使用下面的代码来模仿 @@hasInstance 属性指向的判断方法：

```javascript
function myInstanceof(value, Type) {
	return Type.prototype.isPrototypeOf(value);
}
```

### <span id="isArray">Array.isArray()</span>
****

在 JavaScript 中，`Array.isArray` 方法的本质是用来检查数据类型的，该方法特别的地方在于其只用来检查一种数据类型，即被检测的值是否为数组。我们知道，可以通过 `variable instanceof Array` 这种方式来判断一个值是否为数组，为什么还要设置一个单独的方法来判断一个变量中的值是不是数组呢？这其实是一个历史遗留问题，也是一个涉及到 web 安全的问题：

在 ECMAScript3 中，实现该规范的环境假设只存在一个 global(在浏览器中为 window 对象)，该 global 保存了所有内建方法、属性的实现。在浏览器的实现中，一个 tab 标签页中只存在一个 window 对象，该标签页中所有关于类型检测的方法、属性等，都是从该对象中获取。在浏览器发展的早期这是没有问题的，因为一个标签页中确实只有一个页面，不存在其他其他页面，一个 global 完全可以实现；但是随着浏览器的发展，我们可以在页面中通过 iframe 等手段内嵌其他页面，这些内嵌的页面与当前页面可能同域也可能不同域，并且不同页面之间可以相互通信。

在两个不同页面的通信过程中，就出现了不能判断一个数值是否为数组的情况：

* `typeof` 操作符在判断数组时的输出为 `object`；
* `Object.prototype.toString()` 方法可以通过输出是否为 **"[object Array]"** 来判断是否为数组，但是这是在 `toString()` 方法没有被修改的情况下，虽然被修改的几率比较小，但是也有点脆弱不可靠；
* `varible instanceof Array` 操作符是通过判断 **Array.prototype** 对象是否在 **varible** 的原型链上来判断 varible 是不是 Array 类型的，这在同一个 global 中是没有问题的，但是在有多个 global 时(原页面与内嵌页面中存在两个不同的 global)，因为 JavaScript 中判断两个引用类型的值是否相同时会比较两个引用类型的内存地址是否相同，而两个不同 global 中 **Array.prototype** 对象的内存地址是不一样的，这就导致了两个页面 A、B(原页面 A 与内嵌页面 B)在通信过程中传递数组 `arr` 时，在页面 A 中判断页面 B 传递来的数组时，会检查 A 页面中 global(A) 对象中的 Array.prototype(A) 对象是否在数组 `arr` 的原型链上，但是数组 `arr` 是在页面 B 中生成的，其原型链中的 Array.prototype(B) 对象在 global(B) 中，两个 Array.prototype 是不同的，所以虽然 `arr` 是数组，但是通过 `instanceof` 操作符判断的结果是 false；

以上，介绍了 `Array.isArray` 方法存在的必要性，下面看一下在 ecma262 中是如何定义其判断步骤的：

**[主要规范步骤](https://tc39.github.io/ecma262/#sec-array.isarray)如下：**

```ecma262
Return ? IsArray(arg).
```

**[IsArray(arg)](https://tc39.github.io/ecma262/#sec-isarray)**:

```ecma262
The abstract operation IsArray takes one argument argument, and performs the following steps:	// 只接受一个参数

1、If Type(argument) is not Object, return false. // 非对象返回 false
2、If argument is an Array exotic object, return true. // 是数组返回 true
3、If argument is a Proxy exotic object, then // 如果是 proxy，执行如下流程
	// 如果参数的内部属性 [[ProxyHandler]] 是 null，抛出类型异常错误
	a、If argument.[[ProxyHandler]] is null, throw a TypeError exception.
	b、Let target be argument.[[ProxyTarget]]. // 获取内部属性 [[ProxyTarget]]
	c、Return ? IsArray(target). // 递归执行内部属性 [[ProxyTarget]] 是否为数组
4、Return false. // 如不符合以上条件，返回 false
```

### <span id="toString">Object.prototype.toString()</span>

****

在 JavaScript 中，所有以 `[[propertyName]]` 形式表示的都是内部属性，这些内部属性只能在语言内部访问到，不对外部暴露，所以开发者不能直接通过代码访问这些属性，也就拿不到这些属性的值。通常情况下，这些内部属性供语言实现在内部使用来实现一些功能；也有浏览器厂商会通过非标准的方式向外部暴露一些内部属性，如 Chrome 浏览器中暴露的 `__proto__` 属性(规范中规定的基于该内部属性进行比较操作的方法为 **object.getPrototypeOf()**)。需要注意的是，虽然浏览器厂商向外暴露了这些内部属性，但这些实现是没有写在规范(ecma262) 中的，在未来可能会被移除，所以在生产环境中不建议使用这些不规范的属性。

在其他三种分析数据类型的操作符或者方法中，都不能完全准确地判断出一个数据的类型：

* typeof：适合判断基本数据类型，在判断 Array、Date、Math 等数据类型时会笼统地归类为 **object**；而且在判断 null 时会错误地判断为 **object**。
* instanceof：只能判断一个数据的类型是否为某种对象，不能判断基本数据的类型
* Array.isArray()：只能判断一个值是否为数组类型

`Object.prototype.toString()` 方法是四种方式中最全面的方式，可以判断 JavaScript 中的所有数据类型。我们知道 JavaScript 中的数据都有一个内部属性 [[class]]，该属性保存着如下字符串中的一个："Arguments", "Array", "Boolean", "Date", "Error", "Function", "JSON", "Math", "Number", "Object", "RegExp", "String"。而在使用`Object.prototype.toString()` 方法判断一个数据的类型时，该方法会读取 [[class]] 中的值然后拼接后返回。具体实现过程，我们可以看规范 ecma262 中是怎么定义的：

**[主要规范步骤](https://tc39.github.io/ecma262/#sec-object.prototype.tostring)如下：**

```ecma262
1、If the this value is undefined, return "[object Undefined]". // 判断是否为 undefined
2、If the this value is null, return "[object Null]". // 判断是否为 null
3、Let O be ! ToObject(this value). // 其他值一律转换为相应的对象，如 String、Number等
4、Let isArray be ? IsArray(O). // 判断是否为数组 Array
// 下面得到的所有数据类型，保存在 builtinTag 变量中
5、If isArray is true, let builtinTag be "Array". // 类型字符串为 Array
6、Else if O is a String exotic object, let builtinTag be "String". // 判断是否为字符串 String
// 判断是否为参数 Arguments
7、Else if O has a [[ParameterMap]] internal slot, let builtinTag be "Arguments".
// 根据是否有 [[Call]] 属性判断是否为 Function
8、Else if O has a [[Call]] internal method, let builtinTag be "Function".
// 根据是否有[[ErrorData]] 属性判断是否为 Error
9、Else if O has an [[ErrorData]] internal slot, let builtinTag be "Error".
// 根据是否有[[BooleanData]] 属性判断是否为 Boolean
10、Else if O has a [[BooleanData]] internal slot, let builtinTag be "Boolean".
// 根据是否有[[NumberData]] 属性判断是否为 Number
11、Else if O has a [[NumberData]] internal slot, let builtinTag be "Number".
// 根据是否有[[DateValue]] 属性判断是否为 Date
12、Else if O has a [[DateValue]] internal slot, let builtinTag be "Date".
// 根据是否有[[RegExpMatcher]] 属性判断是否为 RegExp
13、Else if O has a [[RegExpMatcher]] internal slot, let builtinTag be "RegExp".
14、Else, let builtinTag be "Object". // 其他情况判断为 Object
15、Let tag be ? Get(O, @@toStringTag).
// 如果 tag 不是字符串，就将 builtinTag 的值赋值给 tag
16、If Type(tag) is not String, let tag be builtinTag.
// 拼接出最终结果
17、Return the string-concatenation of "[object ", tag, and "]".
```

上面的步骤详细地介绍了 `Object.prototype.toString()` 方法判断一个数据类型的步骤，总结如下：

* 判断是否为 undefined，如果是则返回 "[object Undefined]"
* 判断是否为 null，如果是则返回 "[object Null]"
* 其他数据转换为基本包装类()，然后依次判断类型存在 tag 中，最后统一拼接为 "[object" + tag + "]" 返回

从上面的步骤可以看出，`Object.prototype.toString()` 方法判断了 JavaScript 中的所有数据类型，而且没有 `typeof null === 'object'` 这种不能修复的 bug，所以在开发中，推荐使用该方法来检测数据类型。

**⚠️注意：上面的规范摘自 2017-12-18 更新的草案，但是不知道为什么草案中没有写 Map、Set 以及 Symbol 这三种 ECMAScript6 中新增的数据类型，但是在 Chrome 浏览器中，Object.prototype.toString() 方法已经可以正确地判断这三种数据类型了：**

```javascript
let s = new Set()
Object.prototype.toString.call(s) // "[object Set]"
let m = new Map()
Object.prototype.toString.call(m) // "[object Map]"
let symbol = Symbol(symbol) // 注意，Symbol 没有 [[Construct]] 属性，所以不能使用 new 调用
Object.prototype.toString.call(symbol) // "[object Symbol]"
```

## <span id="number">Number 类型</span>

在 JavaScript 中，当我们对一个数值进行如下操作的时候，引擎会在内部自动调用 `Number(value)` 方法将数值进行数据类型转换：

* 对数值应用一元操作符(a++, a--, ++a, --a)时
* 对数值应用乘性操作符(*, /, %)时
* 对数值应用加性操作符(+, -)，如果其中一个操作数为数字时

当对数值进行如上几种操作时，引擎会将非数值的操作数转换为数字类型，然后再进行运算。那么引擎是按照什么规则来转换非数值的数值呢，我们来看一下 ecma-262 草案中是怎么规定的：

### <span id="draft">Number(value) 转换规范</span>

**[Number ( value ) - 规范步骤](https://tc39.github.io/ecma262/#sec-tonumber) 如下：**

```ecma262
// 如果没传参，则返回 0
1、If no arguments were passed to this function invocation, let n be +0.
// 如果传参，则调用 ToNumber(value) 进行转换后赋值给 n
2、Else, let n be ? ToNumber(value).
// 如果不是通过 new 关键字调用，则返回基本数据类型的 n
3、If NewTarget is undefined, return n.
// 使用 OrdinaryCreateFromConstructor 方法调用 [[Construct]] 方法创建 Number 实例
4、Let O be ? OrdinaryCreateFromConstructor(NewTarget, "%NumberPrototype%", « [[NumberData]] »).
// 设置内部属性 [[NumberData]] 的值为 n(即转换后的数字的值)
5、Set O.[[NumberData]] to n.
// 返回 Number 包装后的对象
6、Return O.
```

**<span id="tonumber">[ToNumber(argument) - 规范步骤](https://tc39.github.io/ecma262/#sec-tonumber)</span> 如下：**

ToNumber(argument) 根据下面表格中的规则将 argument 转换为 Number 类型中的一种值：

<table>
	<tbody>
		<tr>
			<th>Argument Type</th>
			<th>Result</th>
		</tr>
	<tr>
		<td>Undefined</td>
		<td>Return <emu-val>NaN</emu-val>.</td>
	</tr>
	<tr>
		<td>Null</td>
		<td>Return <emu-val>+0</emu-val>.</td>
	</tr>
	<tr>
		<td>Boolean</td>
		<td>If <var class="referenced">argument</var> is <emu-val>true</emu-val>, return 1. If <var class="referenced">argument</var> is <emu-val>false</emu-val>, return <emu-val>+0</emu-val>.</td>
	</tr>
	<tr>
		<td>Number</td>
		<td>Return <var class="referenced">argument</var> (no conversion). // 不转换，直接返回</td>
	</tr>
	<tr>
		<td>String</td>
		<td>See grammar and conversion algorithm below. // 参考 Object</td>
	</tr>
	<tr>
		<td>Symbol</td>
		<td>Throw a
			<emu-val>TypeError</emu-val>
			exception. // 抛出异常
		</td>
	</tr>
	<tr>
		<td>Object</td>
		<td>
			<p>Apply the following steps:</p>
			<emu-alg>
				<ol>
					<li>Let <var>primValue</var> be ?&nbsp;<emu-xref aoid="ToPrimitive" id="_ref_1218"><a href="#sec-toprimitive">ToPrimitive</a></emu-xref>(<var class="referenced">argument</var>, hint Number). // 求对象的初始值</li>
					<li>Return ?&nbsp;<emu-xref aoid="ToNumber" id="_ref_1219"><a href="#sec-tonumber">ToNumber</a></emu-xref>(<var>primValue</var>). // 对初始值递归调用该方法</li>
		  		</ol>
		 </emu-alg>
		</td>
	</tr>
	</tbody>
</table>

由上面的规范可知，Number(value) 的规范步骤为：

* 如果为传参则返回 **+0**
* 如果非 new 调用，则返回 [ToNumber(argument)](#tonumber)转换后的值
* 如果为 new 调用，则返回数字对应的基本包装类对象，基本数据类型值存放在该对象的内部属性 `[[NumberData]]` 中

### <span id="NaN">特殊值 NaN</span>

在 Number 所有值中，存在一个特殊值 **NaN**，这个值是由一个 “非法” 的表达式产生的，这里的非法的意思是指在数学运算中没有意义的运算，比如：

```javascript
1 / 'Hello World'
```

当引擎执行一个数学运算时，如果两个操作数中的任意一个在经过 Number(value) 转换之后仍然不是数字，那么该运算(一个运算其实就是一个表达式)在 JavaScript 中的返回值即为 **NaN**；如果两个数值都能转换为合法的数字，则表达式会返回正确的结果。

在 ecma-262 中是这么定义 NaN 的：

> The Number type has exactly 18437736874454810627 (that is, 264-253+3) values, representing the double-precision 64-bit format IEEE 754-2008 values as specified in the IEEE Standard for Binary Floating-Point Arithmetic, except that the 9007199254740990 (that is, 253-2) distinct “Not-a-Number” values of the IEEE Standard are represented in ECMAScript as a single special NaN value. (Note that the NaN value is produced by the program expression NaN.) In some implementations, external code might be able to detect a difference between various Not-a-Number values, but such behaviour is implementation-dependent; to ECMAScript code, all NaN values are indistinguishable from each other.

即由 9007199254740990 (2**53-2) 这个数值表示 “Not-a-Number” 这种情况；在某些实现中外部代码可以检测不同 NaN 的区别，但是这些行为是与具体实现关联的；在 ECMAScript 的代码中，所有 NaN 的值彼此之间的区别是模糊的，不向外部暴露的，外部代码不能检测不同 NaN 之间的区别。

这也是 **NaN** 特殊的地方，即 `NaN` 不与任何值相等，包括其本身。这种表现咋一看好像很奇怪，但是也有其存在的合理性：在 ECMAScript 的实现中，由一个特殊的值 NaN 来表示 “不是一个数字” 这种结果，但是产生 NaN 的表达式有无数多种可能，如下所示：

```javascript
1 / 'a' // expression1
1 / Symbol.iterator // expression2
```

一个 NaN 由 expression1 这个表达式返回，另外一个 NaN 由 expression2 这个表达式返回，所以不同的 NaN 可能是由不同的表达式产生的，所以 ECMAScript 在规范中规定了 NaN 与任何值都不相等，包括其自身，在 ECMAScirpt 中，是否相等是通过相等操作符(Equality Operators)来判断的，其定义如下：

```ecma262
// x === y
1、If Type(x) is different from Type(y), return false.
2、If Type(x) is Number, then
	a、If x is NaN, return false. // anchor1
	b、If y is NaN, return false. // anchor2
	c、If x is the same Number value as y, return true.
	d、If x is +0 and y is -0, return true.
	e、If x is -0 and y is +0, return true.
	f、Return false.
3、Return SameValueNonNumber(x, y).
```

即不管相等操作符的左边还是右边存在 NaN 时，比较结果为 false。

**注：1 / 0 在数学中是合法的运算，结果为 正无穷(数学符号为：∞，在 ECMAScript 中用 Infinite 表示)；同理也存在 负无穷(数学符号为：-∞，在 ECMAScript 中用 -Infinite 表示)。**

### <span id="isNaN">Number.isNaN VS window.isNaN</span>

在 ECMAScript 的实现的全局对象中存在一个检测是否为 NaN 的方法 `global.isNaN(number)`；在 ECMA2015 这个版本中新增一个判断是否为 NaN 的方法 `Number.isNaN(number)`，这两个方法在规范中的定义如下：

**[isNaN ( number )](https://tc39.github.io/ecma262/#sec-isnan-number)：**

```javascript
1、Let num be ? ToNumber(number).
2、If num is NaN, return true.
3、Otherwise, return false.
```

**[Number.isNaN ( number )](https://tc39.github.io/ecma262/#sec-number.isnan)：**

```javascript
1、If Type(number) is not Number, return false.
2、If number is NaN, return true.
3、Otherwise, return false.
```

由上面的定义可知，**isNaN(number)** 会对数据进行类型转换，而 **Number.isNaN(number)** 更像是严格相等判断。[相等操作符](./operators.md#equal)

### <span id="number-toString">另一个有趣的现象</span>

在平时的开发中我们可能会遇到过这种情况：

```javascript
2.toString() // Uncaught SyntaxError: Invalid or unexpected token
2..toString() // '2'
2.1.toString() // '2.1'
(2).toString() // '2'
```

为什么 `2.toString()` 会报错呢？关于这个问题可以参考 [Why does 2..toString() work? [duplicate]
](https://stackoverflow.com/questions/15458774/why-does-2-tostring-work) 中一个非常形象的回答：

**2.toString()：**
> The interpreter sees 2 and thinks, "oh, a number!" Then, it sees the dot and thinks, "oh, a decimal number!" And then, it goes to the next character and sees a t, and it gets confused. "2.t is not a valid decimal number," it says, as it throws a syntax error.

**2..toString()：**
> The interpreter sees 2 and thinks, "oh, a number!" Then, it sees the dot and thinks, "oh, a decimal number!" Then, it sees another dot and thinks, "oh, I guess that was the end of our number. Now, we're looking at the properties of this object (the number 2.0)." Then, it calls the toString method of the 2.0 object.

**另外一个从原理回答的答案：**

> That's because 2. is parsed as 2.0, so 2..toString() is equivalent to 2.0.toString(), which is a valid expression.

> On the other hand, 2.toString() is parsed as 2.0toString(), which is a syntax error.

即 `2..toString` => `2.0.toString()`，引擎在解析 `2.` 时会认为这是一个浮点数，所以会解析为 `2.0`，然后遇到第二个 `.` 时认为这是取属性进行操作；

而 `2.toString` => `2.0toString`，引擎解析时会判定 0 后面跟的 `toString` 是一个错误的 词法单元(token)，所以会抛出一个语法错误的异常。

## Links 🐬

参考链接如下，排名不分先后：

* [Determining with absolute accuracy whether or not a JavaScript object is an array](http://web.mit.edu/jwalden/www/isArray.html)
* [The history of “typeof null”](http://2ality.com/2013/10/typeof-null.html)
* [Categorizing values in JavaScript](http://2ality.com/2013/01/categorizing-values.html)
* [Why does 2..toString() work? [duplicate]
](https://stackoverflow.com/questions/15458774/why-does-2-tostring-work)

## <span id="author">Author Info 🌟</span>

* [GitHub](https://github.com/Tao-Quixote)
* Email: <web.taox@gmail.com>
