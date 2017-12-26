# "Typeof null" 的历史

**Update 2013-11-05**：为了更好的解释为什么 'typeof null' 的结果是 'object'，我查阅了实现的 C 源码。

在 JavaScript 中，`typeof null` 的结果是 `object`，该结果错误地暗示了 `null` 是一个对象(null 并不是一个对象，而是一个基本数据类型(的值)，详情可以查阅我的博文 [categorizing values](http://2ality.com/2013/01/categorizing-values.html))。这其实是一个 bug，但更糟的是这个 bug 是不能修复的，因为修复这个 bug 会使已经存在的程序崩溃。下面就让我们探索一下这个 bug 的历史。

`typeof null` 这个 bug 是 JavaScript 第一个版本遗留下来的。在这个版本中，所有值都存储在 32 位的单元中，每个单元包含一个小的**类型标签(1-3 bits)**以及当前要存储值的真实数据。类型标签存储在每个单元的低位中，共有五种数据类型：

* 000: object - 当前存储的数据指向一个对象。
* 1: int - 当前存储的数据是一个 31 位的有符号整数。
* 010: double - 当前存储的数据指向一个双精度的浮点数。
* 100: string - 当前存储的数据指向一个字符串。
* 110: boolean - 当前存储的数据是布尔值。

如果最低位是 `1`，则**类型标签**标志位的长度只有一位；如果最低位是 `0`，则**类型标签**标志位的长度占三位，为存储其他四种数据类型提供了额外两个 bit 的长度。

有两种特殊数据类型：

* **undefined(JSVAL_VOID)** 的值是 `-2**30(-2 的 30 次方)` (一个超出整数范围的数字)
* **null(JSVAL_NULL)** 的值是机器码 `NULL` 指针(null 指针的值全是 0)。或者：object 类型的类型标签 + 0 的引用。

现在可以很明显地知道为什么 **typeof** 操作符会认为 **null** 是对象了：typeof 操作符检测 null 的类型标签位时发现是 `000` (存放机器码 NULL 指针的存储单元中的所有数据位都是 0，所以低三位也是 0)。下面是 typeof 操作符的机器码：

```c
    JS_PUBLIC_API(JSType)
    JS_TypeOfValue(JSContext *cx, jsval v)
    {
        JSType type = JSTYPE_VOID;
        JSObject *obj;
        JSObjectOps *ops;
        JSClass *clasp;

        CHECK_REQUEST(cx);
        if (JSVAL_IS_VOID(v)) {  // (1) 检查是否为 undefined
            type = JSTYPE_VOID;
        } else if (JSVAL_IS_OBJECT(v)) {  // (2) 检查是否为 object(低三位是 000)
            obj = JSVAL_TO_OBJECT(v);
            if (obj &&
                (ops = obj->map->ops,
                 ops == &js_ObjectOps
                 ? (clasp = OBJ_GET_CLASS(cx, obj),
                    clasp->call || clasp == &js_FunctionClass) // (3,4)
                 : ops->call != 0)) {  // (3) 检查是否为函数
                type = JSTYPE_FUNCTION;
            } else {
                type = JSTYPE_OBJECT;
            }
        } else if (JSVAL_IS_NUMBER(v)) { // 检查是否为数字
            type = JSTYPE_NUMBER;
        } else if (JSVAL_IS_STRING(v)) { // 检查是否为字符串
            type = JSTYPE_STRING;
        } else if (JSVAL_IS_BOOLEAN(v)) { // 检查是否为布尔值
            type = JSTYPE_BOOLEAN;
        }
        return type;
    }
```

上面代码的运行步骤：

* 在步骤（1）中，机器首先检查值 **v** 是否是 **undefined (VOID)**。该检查是通过比较值是否相等来完成的：

```c
 #define JSVAL_IS_VOID(v)  ((v) == JSVAL_VOID) // 译注：这是一个宏定义
```
* 步骤（2）检查值是否具有 object 类型的类型标签。如果该值具有 object 类型的类型标签，并可以调用(3)或者其内部属性 [[Class]] 标记它为函数(4)，则该值是一个函数；否则，该值就是一个对象。这就是 `typeof null` 表达式生成的结果。
* 随后检查分别为是否为 number，string 以及 boolean。甚至没有专门的步骤来检查是否为 **null**，该检查可通过如下的 C 宏定义来实现：

```c
#define JSVAL_IS_NULL(v)  ((v) == JSVAL_NULL)
```

这或许看起来是一个非常明显的 bug，但是不要忘了实现 JavaScript 第一个版本的时间非常紧迫。

**鸣谢**：感谢 Tom Schuster([@evilpies](https://twitter.com/evilpies/)) [指引](https://twitter.com/evilpies/status/393105924392374272)我去看传统 JavaScript 的[源码](http://mxr.mozilla.org/classic/source/js/src/jsapi.h)。

## Source Link 🗯

* [The history of “typeof null”](http://2ality.com/2013/10/typeof-null.html)

本文翻译自 Dr. Axel Rauschmayer 的博文，侵删。

转载请联系作者并注明出处。

## Translator Info 🌟

* [GitHub](https://github.com/Tao-Quixote)
* Email: <web.taox@gmail.com>