<a name="LWqEW"></a>
# 1 变量
> let：[https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/let](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/let)
> const：[https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/const](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/const)
> var：[https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/var](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/var)

<a name="ZJcuo"></a>
## 1.1 变量的作用域

- var：如果变量声明在函数外，就是全局作用域，否则是函数作用域，不存在用 {} 限定的块作用域。
- let、const：变量所处的代码块，包括全局作用域、函数作用域、或者任何用 {} 限定的块作用域。

另外，未使用 var、let、const 声明的变量会被视作 var，并被提升至全局作用域：
```javascript
function f() {
    msg = "hello"
}
f()
console.log(msg) // hello
```
<a name="qfft3"></a>
## 1.2 重复声明
var 变量允许在同一作用域中重复声明，而 let、const 不允许。
<a name="CEDZf"></a>
## 1.3 绑定全局对象
在全局作用域中声明的 var 变量，将会成为全局对象 window 的属性。
```javascript
var a = 1
console.log(window.a) // 1
```
<a name="PZNol"></a>
## 1.4 变量提升
var 变量声明将被提升至作用域顶部，let、const 变量不会。
```javascript
console.log(a); // undefined
var a = 1;
// 等价于：
var a;
console.log(a);
a = 1; // 仅声明被提升了，赋值没有提升
```
<a name="dmmdM"></a>
## 1.5 暂时性死区
在 let、const 变量的作用域内，从作用域开始到变量声明之前，变量处于暂时性死区内，在暂时性死区内不能访问该变量，否则将抛出 ReferenceError。
```javascript
var a = 1
{
    console.log(a); // 抛出 ReferenceError，即使外面有一个同名的变量 a，也不能访问
    let a = 2;
}
```
需要注意，暂时性死区指的是执行的顺序，而不是代码编写的位置：
```javascript
const func = () => console.log(a); // 这里是可以的，因为实际执行的时候不在 a 的暂时性死区内
let a = 1;
func();
```
一些比较隐蔽的暂时性死区：
```javascript
// 1.
let n = {a: [1,2,3]}
for(let n of n.a) { // n.a 在 let n 的暂时性死区内
    console.log(n)
}
// 2.
var x = 1
{let x  = x + 1} // x+1 在 let x 的暂时性死区内
// 3.
function f(x=y+1, y=1) { // y+1 在参数 y=1 的暂时性死区内
    console.log(x, y)
}
f()
```
<a name="og8wS"></a>
# 2 数据类型
> [https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Data_structures](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Data_structures)
> [https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Grammar_and_types](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Grammar_and_types)
> [https://wangdoc.com/javascript/types/](https://wangdoc.com/javascript/types/)

<a name="gQAi3"></a>
## 2.1 概述

1. **boolean**：布尔值，取值有 true、false。
2. **undefined**：取值只有 undefined，表示变量未赋值。
3. **null**：取值只有 null。
4. **number**：表示整数和小数，大小受限。
5. **bigint**：表示任意大小的整数。
6. **string**：utf-16 编码的字符串。
7. **object**：对象类型。**数组**和**函数**也属于 object 类型，除了 object 之外的类型也叫原始类型。
8. **symbol**：用来创建一个独一无二的值。

一般可以用 typeof 来判断类型，除了 null 类型和函数类型：
```javascript
typeof 1; // "number"
typeof 1n; // "bigint"
typeof "1"; // "string"
typeof true; // "boolean"
typeof undefined; // "undefined"
typeof null; // "object"，所以判断 null 类型只能用 === 
typeof {}; // "object"
typeof []; // "object"，数组属于 object 类型
typeof function(){}; // "function"，函数虽然属于 object 类型，但是返回的是 "function"
typeof Symbol(); // "symbol"
```
instanceof 可用于 object 类型，判断指定值是否是某个构造函数创建的对象：
```javascript
{} instanceof Object //true
```
<a name="AgAES"></a>
## 2.2 数字类型 number
用来表示整数和小数，底层都是用 64 位浮点数来存储。

- 表示的浮点数范围为： ±[Number.MIN_VALUE, Number.MAX_VALUE]，即 ±[2^-1074，2^1024]，超过这个范围的值可能转换为 Infinity 或 0。
- 表示的整数范围为：[Number.MIN_SAFE_INTEGER, Number.MAX_SAFE_INTEGER]，即 [-(2^53 − 1), 2^53 − 1]，超过这个范围就不准确，只能用浮点数近似地表示。可以用 Number.isSafeInteger() 方法判断给定的数是否在安全的整数范围之内。

number 类型有两个特殊的取值：Infinity 和 NaN。

- Infinity 表示无穷大，但是它与数学上的无穷大的行为不完全相同，详见：[https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Number/POSITIVE_INFINITY](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Number/POSITIVE_INFINITY)。
```javascript
console.log(10 / 0) // Infinity
console.log(10 / -0) // -Infinity
```

- NaN 表示一个无效的值，当运算的结果不是一个数值，通常是其中一个运算数不是数值时，可能会得到该值。
```javascript
console.log(undefined+1) // NaN
```
当 number 类型用于位运算时，会先将其截断为 32 位整数：
```javascript
let a = 0x100000000 // 36 位
console.log(a >> 4) // 0，正常 a>>4 === 0x010000000，但是由于移位前 a 被截断为 32 位，变成了 0x00000000，a>>4 就变成了 0
```
不同进制的 number 类型字面量：
```javascript
0o123、0O123 // 8进制
0xAbC、0XABC // 16进制
0b0110、0B1010 // 2进制
```
另外，由于浮点数的存储是不精确的，所以会有下面的问题：
```javascript
console.log(0.1+0.2===0.3) // false
```
<a name="LYfAq"></a>
## 2.3 大整数类型 bigint
number 类型表示的整数范围有限，而 bigint 类型可以用来表示任意大小的整数。需要注意，虽然 bigint 和 number 都可用来表示整数，但仍然是不同的类型，不能直接放在一起运算。<br />有两种方法创建 bigint 类型：
```javascript
const x = BigInt(Number.MAX_SAFE_INTEGER); // 通过 Bigint() 函数创建 bigint 类型
x + 1n === x + 2n; // false，通过在数字后添加 n 后缀创建 bigint 类型

Number.MAX_SAFE_INTEGER + 1 === Number.MAX_SAFE_INTEGER + 2; // true，因为超出了 Number 能准确的表示整数的范围
```
bigint 和 number 相同大小的值，可以用 == 来比较，但用 === 始终为 false。
<a name="zmqAW"></a>
## 2.4 字符串类型 string
> utf-16：[https://verytools.net/xtools-guide/posts/utf16-explained](https://verytools.net/xtools-guide/posts/utf16-explained)

JavaScript 中字符串使用 utf-16 编码。

- Unicode：统一码，为每种语言中的每个字符设定了统一并且唯一的二进制编码，Unicode 中的一个编码称作一个码点。
- utf-16 编码：Unicode 中大于 0xFFFF 的码点，在 utf-16 中会用两个 16 位编码共同表示，称为代理对，高代理码点范围为 0xD800 ~ 0xDBFF，低代理码点范围为 0xDC00 ~ 0xDFFF（代理对占用的码点不会用来表示字符）；其他 Unicode 码点，在utf-16 中直接用原来的编码表示，也就是一个 16 位编码。也就是说，一个 utf-16 字符可能占 2 或 4 字节。utf-16 中每个 16 位编码称作一个代码单元。

拿字符 𝄞 举例，它的 Unicode 编码为 U+1D11E，转换成 utf-16 编码：

1. 首先减去 0x10000：0x1D11E - 0x10000 = 0xD11E = 0000110100 0100011110。
2. 高 10 位为 0x0034，低 10 位为 0x011e。
3. 高代理码点：0x0034 + 0xD800 = 0xD834。
4. 低代理码点：0x011e + 0xDC00 = 0xDD1E。
5. 字符 𝄞 的 utf-16 编码为 '\uD834\uDD1E'。

JavaScript 字符串中每个 utf-16 代码单元为一个元素，即以 2 字节为单位处理一个字符，而一个 utf-16 字符可能占 2 或 4 字节，这就导致获取字符串长度、遍历字符串等都可能出现问题，以 "𝄞" 为例：
```javascript
console.log("𝄞"[0] === "\uD834") // true
console.log("𝄞"[1] === "\uDD1E") // true
console.log("𝄞".length) // 2
for (let i = 0; i < "𝄞".length; i++) {
    console.log("𝄞"[i]); // 输出乱码
}
```
可以使用正则表达式来获取正确的字符串长度、用 for...of 来遍历字符串： 
```javascript
console.log("𝄞".match(/./gu).length) // 1
for (let i of "𝄞") {
    console.log(i); // "𝄞"
}
```
一些用来处理 utf-16 的方法：
```javascript
encodeURI("𝄞") // '%F0%9D%84%9E'，
decodeURI("\uD834\uDD1E") // "𝄞"
"𝄞".charCodeAt(0) // 55348，即 0xD834。它返回的是字符串指定位置的元素的 utf-16 编码。
"𝄞".charCodeAt(1) // 56606，即 0xDD1E
"𝄞".codePointAt(0) // 119070，即 1D11E，𝄞 的 Unicode 编码。它返回的是从指定位置开始的 Unicode 码点值。
```
<a name="xrwNj"></a>
## 2.5 对象类型 object
<a name="Wp4BP"></a>
### 2.5.1 属性
对象的属性名只能为字符串或 symbol 类型。<br />有两种语法来访问对象的属性，**.** 访问法和 **[]** 访问法，**[]** 访问法的好处是能够使用变量、可以设置非合法标识符的属性名（合法的标志符只包含**数字**、**字母**、**_**、**$**，并且不以数字开头）：
```javascript
let o = {
    name: "Jack",
}
let n = "$%^&" // 这里设置的属性名不是 JS 中合法的属性名
o[n] = 1
console.log(o[n], o.name) // 1 "Jack"
```
对象中可以设置属性名为数字，但实际上会转换成字符串：
```javascript
let o = {
  1: "hello",
  "1": "bye"
}
console.log(o[1]) // "bye"，可以发现属性名 "1" 将 1 覆盖了，说明 1 本质还是 "1"
```
当访问对象中不存在的属性时，返回 undefined。
<a name="bXZvw"></a>
### 2.5.2 计算属性名
在声明对象时，可以使用计算属性名：
```javascript
let str = "name"
let o = {
  [str]: "Jack" // [] 中允许使用一个表达式，表达式的结果将作为属性名，这样得到的属性名可以是一个非合法标识符
}
```
<a name="J51z2"></a>
### 2.5.3 对象包装
当使用原始类型的值时，会发现可以在这些值上访问属性、调用方法：
```javascript
console.log("hello".length) // 5
```
这是因为发生了对象包装，原始类型的值被自动转换成了对象，比如 string 原始类型的值被转换成了 String 对象：
```javascript
new String("hello")
```
<a name="Bgugh"></a>
### 2.5.4 原型继承
详见“2 JS-原型继承和类”一章。
<a name="FM02x"></a>
## 2.6 数组

- 数组是一个 Array 类型的对象，可以通过构造函数 Array 创建，也可以使用 [] 声明。
- 一般将具有数字属性名、并且具有 length 属性的对象，称作类似数组的对象。因为它与数组类似，都具有属性名为数字的属性、并且具有 length 属性，很多数组的方法，也能通过 Function.prototype.call 方法来对类似数组的对象调用。
```javascript
var arr = ['a', 'b', 'c']
// 类似：
var arr = {
  '1': 'a',
  '2': 'b',
  '3': 'c',
  length: 3
}
```
数组元素个数范围 [0, 2^32-1]，也是 length 属性的取值范围，如果给 length 属性赋值不在这个范围内则会报错。并且 length 属性可能不能准确反映数组的元素个数，因为 length 实际等于数组的最大的数字属性名+1：
```javascript
var a = []
a[9] = 1
console.log(a.length) // 10
```
通过设置更小的 length 属性值，可以自动删除属性名 >= length 的元素，但是设置更大的 length 值，并不会自动添加元素：
```javascript
var arr = [ 'a', 'b', 'c' ];
arr.length = 2;
console.log(arr) // ["a", "b"]，'c' 被删除了
arr.length = 5;
3 in arr; // false，增大 length，并不会自动添加元素。
```
当数组中有空位时，会跳过该位置的元素：
```javascript

var a = [1, ,1];
console.log(a) // { 0: 1, 2: 1, length: 3, ...}
```
可以通过 for...of 和 for...in 来遍历数组：
```javascript
let a = ["Jack", "Leo", "Alice"]

for (let key in a) {
    console.log(key) // 1 2 3
}
for (let val of a) {
    console.log(val) // "Jack" "Leo" "Alice"
}
```
<a name="NO8or"></a>
## 2.7 函数
> [https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Functions](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Functions)
> [https://wangdoc.com/javascript/types/function](https://wangdoc.com/javascript/types/function)

函数也是特殊的对象，它实际上是一个 Function 类型的对象。
<a name="xcSQ8"></a>
### 2.7.1 函数声明方式
```javascript
function a(){}
var a = function(){}
var a = function b(){} // 函数名 b 只能在函数内部访问到
var a = new Function('x','y','return x+y') // 前面的参数是函数的参数，最后一个参数是函数体
var a = Function('x','y','return x+y') // 可以不用 new
```
函数的一些有用的属性：
```javascript
function f(a) {
  console.log(f.name) // "f"，获取函数名
  console.log(f.length) // 1，获取函数形参个数
  console.log(arguments) // { 0: 1, 1: 2, 2: 3, length: 3 ...} // 一个类似数组的对象，元素为函数的实参
}
f(1,2,3)
```
<a name="KO8Tb"></a>
### 2.7.2 函数所属的作用域
函数只会属于全局作用域、函数作用域，不会属于块作用域：
```javascript
{
  function f() { console.log(1); }
}
f(); // 1
```
<a name="WpXvv"></a>
### 2.7.3 函数提升
直接声明的函数（而不是赋值给变量的，这种情况下，如果是 var 会有变量提升），将会提升到全局作用域、或函数作用域顶部，并且在提升的 var 变量之后：
```javascript
f(); // 1
function f() { console.log(1); }
var f;
// 等价于
var f;
function f() { console.log(1); }
f();
```
<a name="bcMt6"></a>
### 2.7.4 函数内访问的外部变量的指向
函数（包括对象的方法）内访问的外部变量的指向取决于函数声明的位置，而不是函数调用的位置：
```javascript
let a = 1;
function f() {
  console.log(a); // 不管函数 f 在哪里被调用，这里访问的变量 a 始终是 let a = 1
};

{
    let a = 2;
    f(); // 1
}
```
<a name="UvwBe"></a>
### 2.7.5 闭包
闭包是一个函数以及其捆绑的周边环境状态。
```javascript
function f(start) {
  return function () { // 当内层函数用到了外层函数的变量，就导致外层函数的环境在执行完后也无法释放
    return start++;
  };
}
var inc = f(5); // 调用函数 f，产生一个新闭包，包括 f 内的 start 变量、以及返回的函数
console.log(inc()) // 5，因为函数内访问的外部变量的指向取决于函数声明的位置，所以这里访问的 start 变量是闭包中的 start 变量，每次调用 inc() 访问的都是同一个
console.log(inc()) // 6
console.log(inc()) // 7
```
如上所示，每次调用外层的 f 函数，就会创建一个新的闭包，它包括返回的内层函数、以及函数 f 的作用域中定义的变量。
<a name="X5iIa"></a>
### 2.7.6 立即调用的函数表达式
```javascript
(function(){}()); 
(function(){})();// 当 function 位于句首时，会被当作函数声明，无法立即调用，因此在外面加上一个()。只要 function 不在句首都行。
var f = function (){ return 1}(); // 表达式形式的函数不位于句首，因此也可以直接调用。
```
<a name="BlYYM"></a>
### 2.7.7 eval 函数
eval 函数可以接受一个字符串参数，它会将字符串的内容当作 JavaScript 代码解析。
```javascript
eval("console.log(1)") // 1
```
eval 没有自己的作用域，一般情况下，执行 eval 就相当于在当前位置直接执行字符串参数中的 JS 代码；但是也有例外，当不是直接通过 eval() 的方式调用时（称为别名调用），字符串参数中的代码会在全局作用域执行：
```javascript
let a = 1

{
  let a = 2
  window.eval("console.log(a)") // 1，这里不是通过 eval() 的形式调用，因此字符串参数中的代码在全局作用域执行
}
```
以下几种情况都属于别名调用：
```javascript
var e = eval; e('...')
eval.call(null, '...')
window.eval('...')
(1, eval)('...')
```
在严格模式下，eval 内创建的变量、函数不会影响到外部的作用域，但这对别名调用的 eval 无效，要对别名调用的 eval 有效的话，eval 的字符串参数应当以 'use strict;' 开头。
```javascript
'use strict'
eval("var b = 1")
console.log(b) // Uncaught ReferenceError: b is not defined
```
<a name="h6ABY"></a>
## 2.8 symbol 类型
<a name="sthlQ"></a>
### 2.8.1 普通 symbol

- 由于对象中是以字符串作为属性名的，很容易造成属性名的冲突，比如我们使用了他人提供的对象，此时想要为该对象添加新的属性，就可能与现有的属性名产生冲突。
- 调用 Symbol() 函数会返回一个 symbol 类型的值，并且每次调用该函数，返回的值都不等。为了防止属性名冲突，可以将用 symbol 类型的值作为对象的属性名。

创建 symbol：
```javascript
var a = Symbol('my symbol') // 参数是一个字符串，只是对 symbol 的一个描述
a.description // 'my symbol'
```
将 symbol 类型的值作为对象的属性名：
```javascript
let mySymbol = Symbol();

// 第一种写法
let a = {};
a[mySymbol] = 'Hello!';

// 第二种写法
let a = {
  [mySymbol]: 'Hello!' // 使用计算属性名
};

// 第三种写法
let a = {};
Object.defineProperty(a, mySymbol, { value: 'Hello!' });

// 以上写法都得到同样结果
a[mySymbol] // "Hello!"

// 方法也是属性，也可以用它作方法名
let a = {
  [mySymbol](){}
}
```
symbol 属性不是私有属性，但是 for...in、for...of、Object.keys()、Object.getOwnPropertyNames()、JSON.stringify() 都不会返回 symbol 属性，可以使用 Object.getOwnPropertySymbols()，它返回一个数组，包含所有 symbol 属性名。
<a name="YD47v"></a>
### 2.8.2 全局 symbol
使用 Symbol.for() 创建的 symbol 能够让我们重新利用已有的 symbol。它会将创建的 symbol 注册到一个全局的 symbol 注册表中，在该注册表中以其字符串参数作为键，每次调用 Symbol.for()，都会先去 symbol 注册表中查找是否已经有了相同键的 symbol，如果有就直接将其返回，否则创建新的。并且这个注册表在主页面、iframe、Service Worker 中都是共用的。
```javascript
let s1 = Symbol.for("hello")
let s2 = Symbol.for("hello")

console.log(s1 === s2, s1.description) // true "hello"
```
Symbol.keyFor() 从 symbol 全局注册表中获取指定 symbol 的键，没有则返回 undefined：
```javascript
const s = Symbol.for('name');
console.log(Symbol.keyFor(s)); // "name"
```
内置的一些Symbol值：[https://wangdoc.com/es6/symbol#%E5%86%85%E7%BD%AE%E7%9A%84-symbol-%E5%80%BC](https://wangdoc.com/es6/symbol#%E5%86%85%E7%BD%AE%E7%9A%84-symbol-%E5%80%BC)
<a name="gjWel"></a>
### 2.8.3 控制台上输出的 symbol 值的问题
控制台上 symbol 值的输出，可能会造成困惑，比如：
```javascript
let a = {
  [Symbol.iterator]() {}
}
console.log(a)
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/23122012/1691070173987-84e542cc-cccd-492c-805e-5bebfc705d49.png#averageHue=%23fdfbf9&clientId=u537f62c7-91ee-4&from=paste&height=54&id=QhYfb&originHeight=68&originWidth=412&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=8766&status=done&style=none&taskId=ub24326b2-a78e-4f37-9616-7e6b2446374&title=&width=329.6)<br />显示的是 Symbol(Symbol.iterator)，而不是 Symbol.iterator。这是因为输出的其实是 symbol 值被包装成对象后的 toString() 方法返回的字符串，而不是 symbol 值本身：
```javascript
console.log(new Object(Symbol.iterator).toString()) // "Symbol(Symbol.iterator)"
```
<a name="J9LOH"></a>
# 3 数据类型转换
<a name="bOprg"></a>
## 3.1 强制类型转换
<a name="S2IGR"></a>
### 4.1.1.Number()
```javascript
Number(324) // 324
Number('324') // 324
Number('324abc') // NaN
Number('') // 0
Number(true) // 1
Number(false) // 0
Number(undefined) // NaN
Number(null) // 0
```
参数为对象：

- 先调用对象的 valueOf() 方法，如果返回原始类型（即非object类型）的值，则对该值调用 Number() 并返回。
- 否则，调用对象的 toString() 方法，如果返回原始类型的值，则对该值调用 Number() 并返回。
- 否则报错。
<a name="ScrtL"></a>
### 4.1.2.String()
```javascript
String(123) // "123"
String('abc') // "abc"
String(true) // "true"
String(undefined) // "undefined"
String(null) // "null"
```
参数为对象：

- 先调用对象的 toString() 方法，如果返回原始类型的值，则对该值调用 String() 并返回。
- 否则，调用对象的 valueOf() 方法，如果返回原始类型的值，则对该值调用 String() 并返回。
- 否则报错。
<a name="vK7Bs"></a>
### 4.1.3.Boolean()
除了以下情况为 false，其他都为 true：
```javascript
Boolean(undefined) // false
Boolean(null) // false
Boolean(0) // false
Boolean(NaN) // false
Boolean('') // false
```
<a name="DoWZR"></a>
## 4.2.自动转换
一个地方预期是什么类型的值，就会相应地自动调用 Number()、String()、Boolean() 转换成该类型的值。<br />不同类型的值进行运算时，也会发生自动类型转换：

| 运算 | 转换规则 |
| --- | --- |
| 算数运算 | 一般情况下，用 Number() 转换。<br />特殊情况下（优先级由高到低）：<br />- 操作数存在 Date() 对象，则对该对象调用 String()。<br />- 操作数存在对象，则对该对象调用 Number()。<br />- 加法操作数存在字符串，则对另一个操作数调用 String()。<br /> |
| 比较运算 | 用 Number() 转换，除了两个操作数都是 string，这种情况不用转换。 |
| ==和!= | <br />- 操作数不是同一类型，对所有操作数调用 Number() 转换，是同一类型不用转换。<br />- undefined 和 null 类型，不会转换，有：undefined == null，并且意味着其它类型的值与这两个类型不等。<br /> |
| ===和!== | 不会进行类型转换，不同类型的值认为是不相等。 |

<a name="z4AME"></a>
# 3.运算
逻辑运算并不返回布尔值。
```javascript
x && y
//等价于
if(Boolean(x)==false) return x;
else return y;

x || y
//等价于
if(Boolean(x)==true) return x;
else return y;
```
位运算将操作数转换为32位整数进行运算（直接截断为32位整数），返回值也是32位整数。
```javascript
~~1.234 // 1
```
void 运算符，会执行后面的表达式，但不返回值，有以下作用：
```html
// 如下所示的链接中，会将javascript:后面的表达式的值作为文本替代当前页面，除非值为undefined（chrome中只有值为字符串才会替换）
<a href="javascript: document.body.style.background='red'">页面被文本red替代</a>
// 加上void之后，就不会了
<a href="javascript: void(document.body.style.background='red')">页面变为红色</a>

// void还可用于阻止事件的默认行为，事件监听代码最后返回false，点击链接后将不会跳转
<a href="http://www.baidu.com" onclick="f(); return false;">点击</a>
// 使用void效果相同
<a href="javascript: void(f())">点击</a>
```
<a name="XK9W1"></a>
# 5.错误处理
所有错误都是 Error 对象，有6个子类: SyntaxError、TypeError、ReferenceError、RangeError、URIError、EvalError，可以自定义错误对象，只需继承 Error。<br />使用 try、catch、finally 捕获错误。
```javascript
var err = new Error('出错了');
err.message // "出错了"
err.stack; // 非标准的
```

- throw 可以抛出 Error 对象，但也可以抛出其他类型的值，程序遇到 throw 就终止，并不管它抛出的是不是错误。
- 如果是在函数中 return 而非抛出错误，finally 最后也会执行，如果 finally 也有 return，那么函数最终返回的是 finally 中的 return。
```javascript
function f() {
  try {
    return 1;
  } finally {
    return 2;
  }
}
console.log(f()); // 2
```


