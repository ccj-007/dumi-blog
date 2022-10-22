# 闭包 this 作用域

```js
/**
 *  编译的原理
 *
 *  ide  token  ast  二进制被机器识别
 *
 *  js执行过程是怎么样的
 *
 * js会先预编译，做词法环境和动态环境（执行上下文）, 在词法环境中会先定义全局的环境，然后声明的对象、变量的提升、函数的提升， 定义this。 然后将编译好的js开始运行。也就是runtime，那么他会有作用域的堆栈的执行，会一层一层执行如果内部引用了外部的变量，会不断向上查找作用域然后执行。
 *
 *
 *  this的定义
 *
 * this就是执行的上下文， 一般函数的复用会直接在window，node环境会在global执行。  优先级、new  > call apply bind > 函数调用
 * 函数调用的特殊常见  window | global | undefined(严格模式)
 * this的执行看被谁调用。
 *
 * 作用域分为静态的作用域和动态的作用域，动态作用域可以看做是执行的上下文，具体就是闭包的实现。
 *
 * 闭包的定义
 *
 * 闭包的很大的特征就是执行的作用域和定义的作用域不一致
 */

/**
 *  实现一个call
 */
Function.prototype.myCall = function (context) {
  if (context) {
    //为什么不直接this调用，而是挂载在context.fn来改变this指向
    context.fn = this;
    let arg = Array.prototype.slice.call(arguments, 1);
    let result = context.fn(...arg);
    delete context.fn;
    return result;
  } else {
    this(...arguments);
  }
};

let n1 = {
  name: 'tom',
  fn: function () {
    console.log(this.name);
    console.log(arguments);
  },
};
let n2 = {
  name: 'jack',
};
let N1 = n1.fn;
N1.call(n2, 'ccc');

/**
 * 实现一个bind
 *
 * context  执行上下文 this
 */
Function.prototype.bound = function (context) {
  const contexts = context; //context是会变化的
  const fn = this;
  const arg = Array.prototype.slice.call(arguments, 1);
  return function (...rest) {
    const args = [...arg, ...rest];
    return fn.apply(contexts, args);
  };
};

let people = {
  name: 'tom',
  fn: function () {
    console.log(this.name);
    console.log(arguments);
  },
};
let people2 = {
  name: 'jack',
};
let peo = people.fn;
let fn = peo.bound(people2, 'aaa', 'bbb');
fn('ccc');
```
