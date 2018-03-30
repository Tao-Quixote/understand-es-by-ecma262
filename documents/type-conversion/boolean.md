# Boolean

* [自动类型转换](#auto-conversion)
* [自动装箱](#auto-boxing)
* [抽象操作 ToBoolean(value)](#toBoolean)
* [Boolean(value)](#Boolean)
* [参考链接](#see-also)
* [Author Info](#author)

在 JavaScript 中可以通过下面三种方式获取 Boolean 值：

* 字面量方式声明：`true` `false`
* 通过 Boolean() 方法将其他类型的值转换为 Boolean 值：`Boolean('a')`
* 通过 `new` 关键字对 Boolean() 进行构造调用获取 Boolean 类型的对象：`new Boolean(true)`

在日常的开发过程中，由于在运行时 JavaScript 引擎会对必要的部分进行 **自动类型转换** 和 **自动装箱**，所以后面两种方式很少用到；最常用的方式为使用字面量的方式创建 Boolean 值并赋值给变量使用。

## <span id="auto-conversion">自动类型转换</span>

在进行逻辑操作(`&&``||``!`)或者条件判断语句(`if``while`)的时候，JavaScript 引擎会对表达式的结果进行自动类型转换，将其转换为 Boolean 类型的值。

例：

```javascript
3 + 2 && false // false
```

上面的例子中，JavaScript 引擎会先计算 `3 + 2` 的值，然后对其结果进行自动类型转换，即运行抽象操作 `ToBoolean(5)`：如果抽象操作的结果为 `false`，则 **&&** 后面的表达式不会被执行，直接返回 false；如果抽象操作的结果为 `true`，则返回 **&&** 后面表达式的运算结果。

同样，条件判断语句也会对表达式的结果进行类型转换的抽象操作：

```javascript
if (express) {...}
```

上面例子中的表达式 `express` 的结果也会执行类型转换，然后根据转换后的值决定是否执行 `if` 后面的逻辑。

## <span id="auto-boxing">自动装箱</span>

假设有如下代码：

```javascript
let flag = true
flag.toString() // "true"
```

变量 **flag** 中存储的是 Boolean 值的基本数据类型，而基本数据类型不是对象，所以不具备任何方法；那么为什么可以使用 `flag.toString()` 这种方式调用呢？在运行过程中，JavaScript 引擎对基本数据类型调用相关方法的行为会进行自动装箱，即先将基本数据类型包装为对应的对象，然后调用相关方法，方法执行结束后返回执行结果并销毁装箱后的对象。所以上面的代码在执行时会有如下过程：

```javascript
let flag = true
let temp = new Boolean(flag) // 将 flag 中保存的基本数据类型进行装箱(boxing)处理
temp.toString() // 返回执行结果 "true" 后，temp 会被销毁
```

## <span id="toBoolean">抽象操作 ToBoolean(value)</span>

> The abstract operation ToBoolean converts argument to a value of type Boolean according to Table 9:

在所有需要将 value 转换为 Boolean 值的场合，JavaScript 引擎都会执行 `ToBoolean(value)` 这个抽象操作。该操作的返回值参见下表([Table 9](https://tc39.github.io/ecma262/#sec-toboolean)):

**ToBoolean(argument)**:

| Argument Type | Result |
| --- | --- |
| Undefined | Return **false**.|
| Null | Return **false**.|
| Boolean | return **argument**.|
| Number | If **argument** is **+0**, **-0**, or **NaN**, return **false**; otherwise return **true**.|
| String | If **argument** is the empty String (its length is zero), return **false**; otherwise return **true**. |
| Symbol | Return **false**.|
| Object | Return **true**.|

所有涉及类型转换为 Boolean 值的地方都会根据上表进行相关的映射然后返回值。

## <span id="Boolean">Boolean(value)</span>

手动将数据转换为 Boolean 值可以使用 `Boolean(value)` 方法(也可以使用 `!!value` 的方式)将 value 转换为 Boolean 值，ecma262 规范中规定的该构造函数的抽象执行过程如下：

**[Boolean(value)](https://tc39.github.io/ecma262/#sec-boolean-constructor-boolean-value):**

```ecma262
1、Let b be ToBoolean(value). // 将传入的 value 的值转换为基本数据类型，存储在变量 b 中
2、If NewTarget is undefined, return b. // 如果不是通过 new 操作符调用的，则直接返回 b
// 使用构造器(constructor) 构造 Boolean 实例
// 将对象的引用存储在变量 O 中
3、Let O be ? OrdinaryCreateFromConstructor(
	NewTarget,
	"%BooleanPrototype%",
	« [[BooleanData]] »).
4、Set O.[[BooleanData]] to b. // 将上面转换过类型的 b 设置为属性 [[BooleanData]] 的值
5、Return O. // 返回构造好的 Boolean 实例对象的引用
```

**`Boolean(value)`** 函数在被调用时会进行判断：

* 1、如果为普通的函数调用，则对参数进行 <span id="toBoolean">抽象操作 ToBoolean(value)</span>，并将转换后的结果返回；
* 2、如果是通过 `new` 操作符进行的构造调用，先进行前面的一步操作，然后执行 **[OrdinaryCreateFromConstructor](https://tc39.github.io/ecma262/#sec-ordinarycreatefromconstructor)** 抽象操作创建一个新的对象，设置 prototype、基本数据以及内部方法后返回这个新对象。

如果想深入了解构造函数创造新对象的过程，请移步 [OrdinaryCreateFromConstructor](../abstract-operations/ordinary-create-from-constructor.md)。

## <span id="see-also">参考链接 🔗</span>

* [Type Coversion](https://javascript.info/type-conversions)
* [Boolean Objects](tc39.github.io/ecma262/#sec-boolean-objects)

## <span id="author">Author Info 🙈</span>

* [GitHub](https://github.com/Tao-Quixote)
* Email: <web.taox@gmail.com>
