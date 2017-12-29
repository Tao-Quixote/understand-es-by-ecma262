# JavaScript 中的数据类型

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

JavaScript 中有四种方式给数值分类：

* 内部属性[[class]]：该属性保存着一个标示当前值数据类型的字符串
* typeof：主要用来给基本数据类型分类，并与对象进行区分(但是在判断 null 的时候存在 bug，且 bug 无法修复)
* instanceof：用来给对象分类(Function, Array, Object, Date, RegExp...)
* Array.isArray()：该方法用来判断一个值是否是数组的实例

## <span id="typeof">typeof</span>

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

## <span id="isArray">Array.isArray()</span>

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

## <span id="toString">Object.prototype.toString()</span>

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

## Links

参考链接如下，排名不分先后：

* [Determining with absolute accuracy whether or not a JavaScript object is an array](http://web.mit.edu/jwalden/www/isArray.html)
* [The history of “typeof null”](http://2ality.com/2013/10/typeof-null.html)
* [Categorizing values in JavaScript](http://2ality.com/2013/01/categorizing-values.html)

## Author Info 🌟

* [GitHub](https://github.com/Tao-Quixote)
* Email: <web.taox@gmail.com>
