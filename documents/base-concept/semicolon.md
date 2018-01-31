# 分号的使用

* [非法 token(offending token)](#offending-token)
	* [例子](#offending-token-examples)
		* [case1](offending-token-case1)
		* [case2](offending-token-case2)
		* [case3](offending-token-case3)
* [不能解析为完整的 ECMAScript 程序语句(goal nonterminal / complete ECMAScript Program)](#complete-ecmascript-program)
	* [例子](#complete-ecmascript-program-examples)
		* [case1](#complete-ecmascript-progra-case1)
		* [case2](#complete-ecmascript-progra-case2)
		* [case3](#complete-ecmascript-progra-case3)
		* [case4](#complete-ecmascript-progra-case4)
* [Author Info](#author)

在开发过程中，为了对代码进行预检测和格式规范化，我们会在项目的开发环境中引入第三方的代码检查工具，如：[ESLint](https://eslint.org/)、[JSLint](http://www.jslint.com/) 等。为了代码格式(如缩进、换行等)的统一，这些工具中往往会定义一些针对这些项的规则，如缩进使用 tab 还是空格，用几个空格等。

在针对代码规范的规则中，有一个规则比较特殊：在每一行的结尾不允许使用分号 `;` 。我们知道在 JavaScript 中，分号用来表示语句或者表达式的结束；那么在行尾不使用分号标示语句或者表达式的结束，引擎如何判断结束呢？针对这种情况，ecma-262 中提出了自动分号插入(ASI - Automatic Semicolon Insertion)规范来解决未使用分号标示语句结束的情况：

## <span id="offending-token">非法 token(offending token)</span>

在 JavaScript 引擎对 JavaScript 源码进行编译的过程中，存在一个**词法分析(Lexing)**的阶段，该阶段会将 JavaScript 源码分解为一个个有意义的**代码单元(token)**；然后经过**语法分析(Parsing)**阶段生成**抽象语法树(AST)**；最后的代码生成阶段会将抽象语法树生成机器可以执行的机器码。

**offending token**：如果一个代码单元(token)与其之前的另外一个代码单元(token)放在一起进行语法解析时不符合语法规范，则该代码单元(token)就是一个**非法的代码单元(offending token)**。

ecma-262 规范中关于遇到非法代码单元时自动插入分号的规则如下：

> When, as the source text is parsed from left to right, a token (called the offending token) is encountered that is not allowed by any production of the grammar, then a semicolon is automatically inserted before the offending token if one or more of the following conditions is true:

> * The offending token is separated from the previous token by at least one LineTerminator.
> * The offending token is }.
> * The previous token is ) and the inserted semicolon would then be parsed as the terminating semicolon of a do-while statement.

ECMAScript 的实现中，如果遇到如下情况，应该在非法代码单元(offending token)前插入分号：

* 非法代码单元(offending token)与其前的一个代码单元(token)之间有至少一个换行符 // **case1**
* 当非法代码单元(offending token)是符号 `}` 时 // **case2**
* 当非法代码单元(offending token)遇到 `do-while` 表达式的 `)` 符号时 // **case3**

### <span id="offending-token-examples">例子 🌰</span>

#### <span id="offending-token-case1">case1</span>

```javascript
if (true) var a = 0
console.log(a)
```
上面的代码中，`console` 对于上一行中的 `0` 来说是一个非法 token，并且被换行符分割，符合 **case1** 中的情况，所以会触发 ASI，上面的代码将会被转换为如下代码：

```javascript
if (true) var a = 0;
console.log(a)
```

### <span id="offending-token-case2">case2</span>

```javascript
function foo () {
	console.log(3)
}
```

上面的代码中，符号 `}` 相对于 `)` 而言是一个非法 token，符合 **case2** 中的情况，所以会触发 ASI，上面的代码将会被转换为如下代码：

```javascript
function foo () {
	console.log(3);
}
```

### <span id="offending-token-case3">case3</span>

```javascript
do {
	...
} while (true)
console.log('')
```
上面的代码中，`console` 相对于 `)` 而言是一个非法 token，符合 **case3** 中的情况，所以会触发 ASI，上面的代码将会被转换为如下代码：

```javascript
do {
	...
} while (true);
console.log('')
```

**注：case[n] 对应上面的顺序。**

## <span id="complete-ecmascript-program">不能解析为完整的 ECMAScript 程序语句(goal nonterminal / complete ECMAScript Program)</span>

第二种会执行 ASI 的情况是当一个赋值语句中存在换行且不符合语法的情况时，JavaScript 引擎会在换行符之前添加分号，将两行分成两个语句来对待。

ecma-262 规范中关于遇到不能解析为完整语句时自动插入分号的规则如下：

> When, as the source text is parsed from left to right, the end of the input stream of tokens is encountered and the parser is unable to parse the input token stream as a single instance of the goal nonterminal, then a semicolon is automatically inserted at the end of the input stream.

在这种情况中，赋值语句中是要存在换行符的，否则 JavaScript 引擎在解析时会抛出语法错误(SyntaxError)或者引用错误(ReferenceError)：

### <span id="complete-ecmascript-program-examples">例子 🌰</span>

#### <span id="complete-ecmascript-progra-case1">case1</span>

ReferenceError:

```javascript
let a = 3
let b = 4 ++a // Uncaught ReferenceError: Invalid left-hand side expression in postfix operation
```
在这个例子中，由于 `let b = 4 ++a` 会被解析为 `let b = (4++) a`，`4++` 在 ECMAScript 中是非法的，所以会报错。

#### <span id="complete-ecmascript-progra-case2">case2</span>

SyntaxError: Unexpected number：

```javascript
let a = 3
let b = a++ 4 // Uncaught SyntaxError: Unexpected number
```

上面这种情况下，JavaScript 引擎在解析到 `4` 时也会因为不符合语法而报错。

#### <span id="complete-ecmascript-progra-case3">case3</span>

SyntaxError: Unexpected identifier：

```javascript
let a = 3
let c = 4
let b = a++ c
```

同 [case2](#complete-ecmascript-progra-case2)，JavaScript 引擎在解析到标识符 `c` 时会因为不符合语法报错，而不会在适当的地方添加分号。

#### <span id="complete-ecmascript-progra-case4">case4</span>

ASI:

```javascript
// source
a = b
++c

// translated
a = b;
++c
```

在符合 ECMAScript 语法的情况下，如果 ECMAScript 的实现版本在遇到无法解析为完整且合法的语句的时候，应当在输入流(input stream)的结尾处添加分号。

**注：case[n] 对应上面的顺序。**

## <span id="author">Author Info 🌟</span>

* [GitHub](https://github.com/Tao-Quixote)
* Email: <web.taox@gmail.com>
