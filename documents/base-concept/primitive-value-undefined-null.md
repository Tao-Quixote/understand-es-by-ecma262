# 基本数据类型 - undefined & null

ECMAScript 使用 `undefined` 和 `null` 这两个原始数据类型来表示 **空值** 这个现象，也就是信息缺失。这两个值用来表示不同类型的空值，会出现的情况也不一样。

`undefined` 用来表示没有值（既不是原始值也不是对象），而 `null` 的意思是用来表示没有对象。

## 两种数据类型出现的场景

### undefined

在以下情况会得到 undefined：

* 没有初始化的变量
* 缺失的参数
* 访问不存在的属性
* 未显式地返回值的函数，在执行后会返回 undefined

### null 的出现场景

* 原型链最顶端的元素是 null：`Object.prototype.__proto__` 或者 `Object.getPrototypeOf(Object.prototype)`
* 当字符串没有匹配到正则表达式时，返回的结果是 null：`/x/.exec('abcd')`

## 两种数据类型

在 ECMASCript 规范中，null 和 undefined 是做为两种数据类型存在的，在 9.0 的规范中，存在以下几种数据类型：

* Undefined
* Null
* Boolean
* String
* Symbol
* Number
* Object

其中关于 Undefined 类型的描述如下：

> The Undefined type has exactly one value, called undefined. Any variable that has not been assigned a value has the value undefined.
> Undefined 类型只有一个值 undefined。任何没有赋值过的变量的值都是 undefined。

其中关于 Null 类型的描述如下：

> The Null type has exactly one value, called null.
> Null 类型只有一个值 null。

## null 和 undefined 的检测

* 对于 null，可以通过严格相等来判断一个变量的值是否为 null：`x === null`；不要使用 typeof 操作符来判断 null，因为会错误的返回 object，详细请跳 ["Typeof null" 的历史](../translations/typeof-null.md)
* 对于 undefined，建议使用 `x === void 0` 来进行判断，具体原因参见 [void 操作符](./void-operator.md)

## Links 🐬

参考链接如下，排名不分先后：

* [void 操作符](./void-operator.md)

## <span id="author">Author Info 🌟</span>

* [GitHub](https://github.com/Tao-Quixote)
* Email: <web.taox@gmail.com>
