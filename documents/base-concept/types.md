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

这是 ECMAScript6 之前的数据类型，在 ECMAScript 中新增了三种数据类型：`Symbol`， `Map` 和 `Set`。

关于 ES6 之前的数据类型分类，可以参考 Dr. Axel Rauschmayer 的博文 [Categorizing values in JavaScript(对 JavaScript 中的值进行分类)](http://2ality.com/2013/01/categorizing-values.html)。

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

## Links

* [The history of “typeof null”](http://2ality.com/2013/10/typeof-null.html)
* [Categorizing values in JavaScript](http://2ality.com/2013/01/categorizing-values.html)

## Author Info 🌟

* [GitHub](https://github.com/Tao-Quixote)
* Email: <web.taox@gmail.com>
