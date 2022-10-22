# Promise 异步

### promise 的原理

promise 是一个构造函数，会接受一个函数，这个回调会传入 resolve 和 reject 两个函数，分别控制 pending 的状态结果 value 和 reason。 promise 的返回是一个包含 then 的函数。通过调用 reject、resolve 改变 promise 的状态 fulfilled、rejected 状态。在 then 函数的传入的回调函数中传参会得到上个 promise 对象的 resolve 或 reject 的结果。

### Promise 为什么使用了发布订阅模式

因为 resolve 之前，then 已经执行过了，我们要提前收集起来，然后在状态改变后推入微任务队列然后执行。

### then 的实现原理是什么?， 为什么可以链式调用？

在程序执行过程中，首先会以发布订阅收集所有的 then 函数中的 onFulfilled 或 onRejected，如果此时 promise 状态是 pending 状态 push 进入成功或失败的队列， then 一旦监听到 resolved | rejected 状态，那么此时 resolve 或 reject 执行能获取到 value | reason 的值（也就是上一次的结果，在 promise 返回后能在 onFulfilled 获取到 value）。在 then 的返回值一般有 promise 的情况、方法和对象、死循环、基本数据类型通过 resolvePromise 处理。then 会传入 onFulfilled 和 onRejected 方法。一旦上一级的 promise 执行了 resolve，会立即执行 onFulfilledArray 队列里面的所有 promise 对象，这个执行的时机发生在微任务。

then 传入的回调函数返回值不同情况的判断

1. 基本数据类型
2. 死循环
3. Promise
4. object 和 function

### resolvePromise 的规范

resolvePromise(promise2, x, resolve, reject)

- 如果 promise2 和 x 相等，那么 reject error;
- 如果 x 是一个 promise

  - 如果 x 是一个 pending 状态，那么 promise 必须要在 pending, 直到 x 变成 fulfilled or rejected
  - 如果 x 被 fulfilled， fulfill promise with the same value
  - 如果 x 被 rejected， reject promise with the same reason

- 如果 x 是一个 object 或者 function
  - let thenable = x.then
  - 如果 x.then 这一步出错，那么 reject promise with e as the reason
  - 如果 then 是一个函数，then.call(x, resolvePromiseFn, rejectPromise)
    resolvePromiseFn 的 入参是 y, 执行 resolvePromise(promise2, y, resolve, reject);
    rejectPromise 的 入参是 r, reject promise with r.
    如果 resolvePromise 和 rejectPromise 都调用了，那么第一个调用优先，后面的调用忽略。
    如果调用 then 抛出异常 e
    如果 resolvePromise 或 rejectPromise 已经被调用，那么忽略
    则，reject promise with e as the reason
    如果 then 不是一个 function. fulfill promise with x.

### 什么是宏任务，什么是微任务？eventloop 是什么？

宏任务： setTimeout setInterval script IO ui render V8 垃圾回收
微任务： node $nextTick mutationObserver .then .catch await

eventloop 执行过程：

js 会有一个执行栈的东西，先进后出执行同步的代码，一旦同步代码执行完毕后，会去执行宏任务和微任务，有这么一个任务队列的数据结构，会先执行宏任务，如果执行完毕，然后去执行微任务，然后执行 ui 渲染，检查 web worker 工作，执行完一轮后，会重复找宏任务、微任务。。。。

1. 一开始整个脚本作为一个宏任务执行
2. 执行过程中同步代码直接执行，宏任务进入宏任务队列，微任务进入微任务队列
3. 当前宏任务执行完出队，检查微任务列表，有则依次执行，直到全部执行完
4. 执行浏览器 UI 线程的渲染工作
5. 检查是否有 Web Worker 任务，有则执行
6. 执行完本轮的宏任务，回到 2，依此循环，直到宏任务和微任务队列都为空

注意 ⚠️：在所有任务开始的时候，由于宏任务中包括了 script，所以浏览器会先执行一个宏任务，在这个过程中你看到的延迟任务(例如 setTimeout)将被放到下一轮宏任务中来执行。

### 手写个 promise

```js
const resolvePromise = (promise2, result, resolve, reject) => {
  // 死循环，结果和then的返回promise相等
  if (result === promise2) {
    reject(new TypeError('error due to circular reference'));
  }

  // 是否已经执行过 onfulfilled 或者 onrejected
  let consumed = false;
  let thenable;

  if (result instanceof LPromise) {
    if (result.status === 'pending') {
      result.then(function (data) {
        //promsie状态改变后，resolvePromise返回结果
        resolvePromise(promise2, data, resolve, reject);
      }, reject);
    } else {
      //promise状态已改变
      result.then(resolve, reject);
    }
    return;
  }

  let isComplexResult = (target) =>
    (typeof target === 'function' || typeof target === 'object') &&
    target !== null;

  // 如果返回的是疑似 Promise 类型
  if (isComplexResult(result)) {
    try {
      thenable = result.then;
      // 如果返回的是 Promise 类型，具有 then 方法
      if (typeof thenable === 'function') {
        thenable.call(
          result,
          function (data) {
            //保证执行一次
            if (consumed) {
              return;
            }
            consumed = true;

            return resolvePromise(promise2, data, resolve, reject);
          },
          function (error) {
            //保证执行一次
            if (consumed) {
              return;
            }
            consumed = true;

            return reject(error);
          },
        );
      } else {
        resolve(result);
      }
    } catch (e) {
      if (consumed) {
        return;
      }
      consumed = true;
      return reject(e);
    }
  } else {
    //基本数据类型
    resolve(result);
  }
};

function LPromise(execute) {
  this.status = 'pending';
  this.value = null;
  this.reason = null;

  this.onFulfilledArray = [];
  this.onRejectedArray = [];

  const resolve = (value) => {
    setTimeout(() => {
      if (this.status === 'pending') {
        this.value = value;
        this.status = 'fulfilled';
        this.onFulfilledArray.forEach((func) => func(value));
      }
    }, 0);
  };
  const reject = (reason) => {
    setTimeout(() => {
      if (this.status === 'pending') {
        this.reason = reason;
        this.status = 'rejected';
        this.onRejectedArray.forEach((func) => func(reason));
      }
    }, 0);
  };

  execute(resolve, reject);
}

LPromise.prototype.then = function (onFulfilled, onRejected) {
  onFulfilled =
    typeof onFulfilled === 'function' ? onFulfilled : (data) => data;

  onRejected =
    typeof onRejected === 'function'
      ? onRejected
      : (error) => {
          throw error;
        };

  let promise2;

  if (this.status === 'fulfilled') {
    return (promise2 = new LPromise((resolve, reject) => {
      setTimeout(() => {
        try {
          let result = onFulfilled(this.value);
          resolve(result);
        } catch (e) {
          reject(e);
        }
      });
    }));
  }

  if (this.status === 'rejected') {
    return (promise2 = new LPromise((resolve, reject) => {
      setTimeout(() => {
        try {
          let result = onRejected(this.reason);
          resolve(result);
        } catch (e) {
          reject(e);
        }
      });
    }));
  }

  if (this.status === 'pending') {
    return (promise2 = new LPromise((resolve, reject) => {
      this.onFulfilledArray.push(() => {
        try {
          let result = onFulfilled(this.value);
          resolvePromise(promise2, x, resolve, reject);
        } catch (e) {
          reject(e);
        }
      });

      this.onRejectedArray.push(() => {
        try {
          let result = onRejected(this.reason);
          resolvePromise(promise2, x, resolve, reject);
        } catch (e) {
          reject(e);
        }
      });
    }));
  }
};

module.exports = LPromise;
```

### Promise 各个 api 的实现

```js
function myPromiseAll(promises) {
  let arr = [],
    count = 0;
  return new Promise((resolve, reject) => {
    promises.forEach((promise, index) => {
      Promise.resolve((res) => {
        arr[index] = res;
        count++;
        if (count === promise.length) resolve(arr);
      }).catch((e) => {
        reject(e);
      });
    });
  });
}

Promise.resolve = function (value) {
  if (value instanceof Promise) {
    return value;
  }

  return new Promise((resolve) => resolve(value));
};

Promise.reject = function (reason) {
  return new Promise((resolve, reject) => reject(reason));
};

//返回迭代的数组中第一个reject | fulfilled包装后的实例
Promise.race = function (promiseArr) {
  return new Promise((resolve, reject) => {
    promiseArr.forEach((p) => {
      Promise.resolve(p)
        .then((val) => {
          resolve(val);
        })
        .catch((e) => {
          reject(e);
        });
    });
  });
};

//allSettled 用数组记录对应的状态和值
Promise.allSettled = function (promiseArr) {
  let result = [];

  return new Promise((resolve, reject) => {
    promiseArr.forEach((p, i) => {
      Promise.resolve(p).then(
        (val) => {
          result.push({
            status: 'fulfilled',
            value: val,
          });
          if (result.length === promiseArr.length) {
            resolve(val);
          }
        },
        (err) => {
          result.push({
            status: 'rejected',
            reason: err,
          });
          if (result.length === promiseArr.length) {
            resolve(val);
          }
        },
      );
    });
  });
};

//Promise.all 无论如何返回第一个完成状态，如果是空数组或全部都是rejected状态的，就返回AggregateError错误

Promise.any = function (promiseArr) {
  let index = 0;
  return new Promise((resolve, reject) => {
    if (promiseArr.length === 0) return;
    promiseArr.forEach((p, i) => {
      Promise.resolve(p).then(
        (val) => {
          resolve(val);
        },
        (err) => {
          index++;
          if (index === i) {
            reject(new AggregateError('all promise were rejected'));
          }
        },
      );
    });
  });
};
```

### Promise 如何并发 | 顺序执行

```js
const promiseArrGenerator = (num) =>
  new Array(num).fill(0).map((item, index) => () => {
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve(index);
      }, Math.random() * 100);
    });
  });

const proArr = promiseArrGenerator(100);

// Promise.all(proArr.map(fn => fn())).then(res => console.log(res))

//如何顺序执行
// proArr.forEach(fn => fn().then(res => console.log(res)))

//用reduce实现
const promiseReduce = (proArr) => {
  proArr.reduce((prePromise, nextPromise) => {
    return prePromise.then((res) => {
      res && console.log(res);
      return nextPromise();
    });
  }, Promise.resolve(-1));
};
// promiseReduce(proArr)
```

### 设计一个并发的 pipe, 如 chrome 浏览器最大的 6 个并发限制

```js
const promisePipe = (proArr, pipeNums) => {
  if (pipeNums > proArr.length) {
    Promise.all(proArr.map((fn) => fn())).then((res) => console.log(res));
  }

  let _arr = [...proArr];

  for (let i = 0; i < pipeNums; i++) {
    run(_arr.shift());
  }

  function run(promiseFn) {
    promiseFn().then((res) => {
      console.log(res);
      if (_arr.length) run(_arr.shift());
    });
  }
};
promisePipe(proArr, 1);
```

### Promise 一旦启动，就无法停止，那么如何实现一个 abort 方法来停止

```js
const controller = new AbortController();
const signal = controller.signal;

function abortPromise(proArr) {
  let flag = true;
  signal.addEventListener('abort', () => {
    console.log('信号终止');
    flag = false;
  });
  proArr.forEach((fn) => {
    fn().then((res) => {
      if (!flag) {
        throw new Error('abort');
      } else {
        console.log(res);
      }
    });
  });
}
abortPromise(proArr);

setTimeout(() => {
  controller.abort();
}, 5000);
```
