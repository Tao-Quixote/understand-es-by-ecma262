# toString VS valueOf VS toJSON

* [\[\[DateValue\]\]](#date-value)
* [thisTimeValue(this value)](#this-time-value)
* [toString](#to-string)
* [valueOf](#value-of)
* [toJSON](#to-json)
* [相关链接](#links)
* [Author Info](#author)

在日常的开发中，我们最常见的日期对象字符串格式如下：

```javascript
> new Date() // Mon Apr 16 2018 19:01:58 GMT+0800 (CST)
```

如果我们通过 **一元运算符(+、-)** 或者 **算数运算符(+、-、\*、/)** 来操作一个日期对象实例时，可能会看到下面这样的结果：

```javascript
+ new Date() // 1523877012742
```

而如果我们偶然通过 `JSON.parse(value[, handle[, space]])` 方法对日期对象进行序列化，还可能见到如下这种数据格式：

```javascript
let obj = {
	date: new Date()
};
JSON.stringify(obj) // "{"date":"2018-04-16T11:02:37.849Z"}"
```

要搞清楚产生上述结果的原因，我们就要弄清楚下面几个日期对象（**Date**）内置方法的工作原理。

**注：**

* 运算符产生的数据类型转换的规则请移步 [操作符](../../documents/base-concept/operators.md)。
* 构造器(constructor) 生成对象的过程请移步 [OrdinaryCreateFromConstructor](../../documents/abstract-operations/ordinary-create-from-constructor.md)。

## <span id="date-value">[[DateValue]]</span>

在搞清楚下面的操作规范之前，我们先来看一下 `Date` 类型实例对象的数据存储在什么的地方。构造器 `Date(...)` 在实现时进行了重载，有三种不同的方法签名：

* `Date()`
* `Date(value)`
* `Date ( year, month [ , date [ , hours [ , minutes [ , seconds [ , ms ] ] ] ] ] )`

针对不同的 **函数签名**，在执行时步骤会稍有不同，返回的结果有如下两种情况：

* 如果是普通的函数调用，则会返回一个 **时间字符串**，时间字符串的格式同 [toString](#to-string)，因为该抽象操作和 **`toString`** 都会执行相同的抽象操作 `Runtime Semantics: ToDateString( tv )`；
* 如果是作为构造函数调用(new Date(...))，则会执行一些列的抽象操作，最终返回一个**时间对象**

这里我们只讨论返回**时间对象**的情况(即 `new` 调用)，因为对 **时间字符串** 进行相关的操作时，会在内部进行构造调用将 **时间字符串** 包装为 **时间对象**。

这里我们以 `Date(value)` 构造函数为例，其在 ECMA262 规范中的定义如下：

***

* 1、 Let ***numberOfArgs*** be the number of arguments passed to this function call. // 使用 numberOfArgs 记录参数个数
* 2、Assert: **numberOfArgs = 1**.
* 3、If ***NewTarget*** is **undefined**, then // 判断是否为非 `new` 调用
	* a、Let ***now*** be the Number that is the ***time value*** (UTC) identifying the current time. // 将当前的 **time value(代表时间的整型数字)** 存储在 **now** 中
	* b、Return ***ToDateString(now)***. // 返回 `Date` 字符串，例："Mon Apr 16 2018 19:01:58 GMT+0800 (CST)"
* 4、Else, // 处理参数为对象的情况
	* a、If ***Type(value)*** is Object and value has a **[[DateValue]]** internal slot, then
		* i、Let **tv** be ***thisTimeValue(value)***. // 参数是对象类型且有内部 **[[DateValue]]** 属性时，将 **[[DateValue]]** 赋值给 tv 变量
	* b、Else, // 其余情况(非对象类型或者没有内部属性**[[DateValue]]**)，将参数转换为基本数据类型
		* i、Let **v** be ? ***ToPrimitive(value)***.
		* ii、If ***Type(v)*** is String, then // 基本数据类型为 string
			* 1、Let ***tv*** be the result of parsing v as a date, in exactly the same manner as for the **parse** method (20.3.3.2). If the parse resulted in an abrupt completion, ***tv*** is the Completion Record.
			* 2、ReturnIfAbrupt(tv).
		* iii、Else, // 转换为 number 类型
			* 1、Let **tv** be ? ***ToNumber(v)***.
	* c、Let **O** be ? ***OrdinaryCreateFromConstructor(NewTarget, "%DatePrototype%", « [[DateValue]] »)***. // 创建一个新的对象，新对象的原型为 **`DatePrototype`**，内置一个插槽 **`[[DateValue]]`**
	* d、Set **O.[[DateValue]]** to ***TimeClip(tv)***. // 将前面入参的处理结果经 **TimeClip(tv)** 抽象操作后，保存在新对象的内部插槽 **[[DateValue]]** 中
	* e、Return **O**. // 返回生成的新对象

***

上面只介绍了 `Date(value)` 这一种签名的构造函数，其他两种签名的构造函数其内部抽象操作大体一致，如果通过 `new` 操作符进行构造调用的话，在入参合法的情况下，一定会将入参转换为 number 类型的正数，并将其保存在新生成对象的内部插槽 **[[DateValue]]** 中。

当一个 `Date` 实例需要进行数据类型转换时，所需的时间值从 **[[DateValue]]** 中获取。

## <span id="this-time-value">thisTimeValue(this value)</span>

在 `toString` 和 `valueOf` 这两个抽象操作中，会执行抽象操作 `thisTimeValue(this value)` 来获取时间对象的值。该抽象操作的对象必须是一个对象，该对象需要具有内部插槽 [[DateValue]] 且其中的值为 [time value](https://tc39.github.io/ecma262/#sec-time-values-and-time-range)。

***
* 1、If ***Type(value)*** is Object and value has a **[[DateValue]]** internal slot, then
	* a、Return **value.[[DateValue]]**. // 如果抽象操作的参数为对象且具有内部插槽 **[[DateValue]** 时，返回插槽中存储的值(整数)
* 2、Throw a **TypeError** exception. // 否则抛出一个类型错误异常(TypeError exception)

***

## <span id="to-string">toString</span>

当我们在控制台中打印 `Date` 实例，或者作为 `alert()` 方法的参数，或者在字符串拼接中使用 `Date` 实例时，在进行自动类型转换时，引擎会自动调用 `Date.prototype.toString` 方法，将 `Date` 实例转换为字符串。ECMA262 规范中规定的抽象操作如下：

***
**[Date.prototype.toString()](https://tc39.github.io/ecma262/#sec-date.prototype.tostring)：**

* 1、Let **tv** be ? ***thisTimeValue(this value)***. // 从 [[DateValue]] 中获取时间对象中存储的时间(number 类型的整数)
* 2、Return ***ToDateString(tv)***. // 返回抽象操作 `ToDateString(tv)` 构造好的字符串

***
**[ToDateString(tv)](https://tc39.github.io/ecma262/#sec-todatestring)：**

* 1、Assert: Type(tv) is Number. // 断言 `tv` 是否为 number 类型
* 2、If **tv** is **NaN**, return "**Invalid Date**". // 如果 `tv === NaN`，返回 **Invalid Date**
* 3、Let **t** be ***LocalTime(tv)***. // 将 `tv` 中的值本地化(tv 加上(或者减去)当前时区与 UTC 的差值)
* 4、Return the ***string-concatenation*** of ***DateString(t)***, the code unit 0x0020 (SPACE), ***TimeString(t)***, and ***TimeZoneString(tv)***. // 将日期字符串、空格、时间字符串和时区字符串链接在一起

***
**[DateString 格式](https://tc39.github.io/ecma262/#sec-datestring)：**

`Tue` `Apr` `17` `2018`

<span style="color: red">`周几` `月份` `日期` `年份`</span>

以**空格** 分割。

***
**[TimeString 格式](https://tc39.github.io/ecma262/#sec-timestring)：**

`17`:`47`:`14`

<span style="color: red;">`时`:`分`:`秒`</span>

以**冒号**分割。

***
**[TimeZoneString 格式](https://tc39.github.io/ecma262/#sec-timezoneestring)：**

`GMT``+``08``00` `(CST)`

<span style="color: red;">`GMT(格林威治时间)``时区偏移符号``时区偏移小时数``时区便宜分数` `时区名称`</span>

* **`TimeZoneString`** 部分用来标示当前时区相对于格林威治时间偏移的 **小时** 和 **分**；
* 时区偏移符号是通过抽象操作 [localTZA](https://tc39.github.io/ecma262/#sec-local-time-zone-adjustment) 得到的；
* **`CST`** 部分在不同时区缩写不同，这部分表示的为时区的缩写，其中 CST 代表以下国家所在时区的缩写：
	* 美国中部时间：Central Standard Time (USA) UT-6:00
	* 澳大利亚中部时间：Central Standard Time (Australia) UT+9:30
	* 中国标准时间：China Standard Time UT+8:00
	* 古巴标准时间：Cuba Standard Time UT-4:00

***

## <span id="value-of">valueOf</span>

在所有数学计算相关的表达式中，`Date` 实例在自动类型转换时，JavaScript 引擎会自动调用 `Date.prototype.valueOf()` 方法来获取存储在时间对象内部插槽 [[DateValue]] 中的时间值：

***
**[Date.prototype.valueOf()](https://tc39.github.io/ecma262/#sec-date.prototype.valueof):**

* 1、Return ? **thisTimeValue**(**this** value). // 返回时间对象内部插槽 [[DateValue]] 中存储的时间值

***

## <span id="to-json">toJSON</span>

想查看 `JSON` 对象的小伙伴可以跳转 [Frontend-Tricks JSON](https://github.com/NinjiaHub/Frontend-Tricks/blob/master/documents/JS/json.md) 了解更多。

`Date.prototype.toJSON` 方法在平时的开发中比较少使用，只有在将日期对象序列化的时候，JavaScript 引擎才会自动调用该方法，将日期对象转换为对应的时间字符串。`Date.prototype.toJSON` 方法返回的字符串与 `Date.prototype.toString` 方法返回的字符串不同，`Date.prototype.toJSON` 方法返回 `ISOString` 格式的字符串。下面我们来看一下 ECMA262 中是怎么规定 `Date.prototype.toJSON` 内部的抽象操作的：

***

**[Date.prototype.toJSON(key)](https://tc39.github.io/ecma262/#sec-date.prototype.tojson)**

* 1、Let **O** be ? ***ToObject(this value)***. // 将调用该方法的 对象/基本数据类型 转换为对应的包装对象
* 2、Let **tv** be ? ***ToPrimitive(O, hint Number)***. // 尝试将生成的对象转换为 number 类型的基本数据类型
* 3、If ***Type(tv)*** is Number and tv is not finite, return **null**. // 如果不是有限数字，返回 null，这种情况下生成的 JSON 字符串中对应位置的键值对中的值为 "null"
* 4、Return ? **Invoke(O, "toISOString")**. // 返回对象的 `toISOString` 方法的执行结果，抽象操作 `Invoke(O, "toISOString")` 表示执行对象 `O` 上的方法 `toISOString`

***
**[Date.prototype.toISOString ( )](https://tc39.github.io/ecma262/#sec-date.prototype.toisostring)**

> This function returns a String value representing the instance in time corresponding to **this time value**. The format of the String is the Date Time string format defined in [20.3.1.15 Date Time String Format](https://tc39.github.io/ecma262/#sec-date-time-string-format). All fields are present in the String. The time zone is always UTC, denoted by the suffix Z. If this time value is not a finite Number or if the year is not a value that can be represented in that format (if necessary using extended year format), a **RangeError** exception is thrown.
> 
> 该方法返回一个字符串表示当前时间。字符串的格式定义在 [20.3.1.15 Date Time String Format](https://tc39.github.io/ecma262/#sec-date-time-string-format) 部分。字符串的所有部分均以字符串来表示。时区在任何情况下都是 UTC，用后缀 Z 来表示。如果时间的值不是有限数字或者年份不能使用 [20.3.1.15 Date Time String Format](https://tc39.github.io/ecma262/#sec-date-time-string-format) 中定义的格式，则抛出一个 **RangeError** 异常。

***
**[20.3.1.15 Date Time String Format](https://tc39.github.io/ecma262/#sec-date-time-string-format)：**

`Date.prototype.toJSON()` 方法在被调用时，JavaScript 引擎会在内部根据如下的格式，将存储在时间对象中的时间转换为字符串。

日期时间字符串格式：**`YYYY`-`MM`-`DD``T``HH`:`mm`:`ss`.`sss``Z`**

* **YYYY**	is the decimal digits of the year 0000 to 9999 in the proleptic Gregorian calendar. // 年份
* **-**	"-" (hyphen) appears literally twice in the string. // 年月日之间使用的分割符
* **MM**	is the month of the year from 01 (January) to 12 (December). // 月份
* **DD**	is the day of the month from 01 to 31. // 日期
* **T**	"T" appears literally in the string, to indicate the beginning of the time element. // 时间开始的标志
* **HH**	is the number of complete hours that have passed since midnight as two decimal digits from 00 to 24. // 小时
* **:**	":" (colon) appears literally twice in the string. // 时分秒之间使用的分隔符
* **mm**	is the number of complete minutes since the start of the hour as two decimal digits from 00 to 59. // 分钟数
* **ss**	is the number of complete seconds since the start of the minute as two decimal digits from 00 to 59. // 秒
* **.**	"." (dot) appears literally in the string. // 表示毫秒的开始
* **sss**	is the number of complete milliseconds since the start of the second as three decimal digits. // 三位数字，表示毫秒数
* **Z**	is the time zone offset specified as "Z" (for UTC) or either "+" or "-" followed by a time expression HH:mm // 用来表示时区

***

综上，日期时间字符串格式为**YYYY-MM-DDTHH:mm:ss.sssZ**，当我们解析 JSON 字符串时，遇到这种结构的字符串，应该调用 `Date` 构造函数或者 `Date.parse()` 方法来解析该部分。

## <span id="links">相关链接 🔗</span>

* [操作符](../../documents/base-concept/operators.md)
* [OrdinaryCreateFromConstructor](../../documents/abstract-operations/ordinary-create-from-constructor.md)

## <span id="author">Author Info 🐧</span>

* [GitHub](https://github.com/Tao-Quixote)
* e-mail: <web.taox@gmail.com>
