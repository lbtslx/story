# promise

标签（空格分隔）： javascript promise

---

## promise A+规范
[此处输入链接的描述][1]
## 手写promise步骤

### 1、实现promise,满足最基本的then调用

```javascript
class MyPromise{
  constructor(fn) {
    this.value = undefined;
    this.resolveCallbacks = [];
    const resolve = data => {
      this.value = data;
      this.resolveCallbacks.map(fn => fn(this.value));
    }
    const reject = err => {
      console.log(err)
    }
    fn(resolve, reject);
  }
  then(onFulfilled, onRejected) {
    this.resolveCallbacks.push(onFulfilled);
  }
}
const promise = new MyPromise((resolve, reject) => {
  setTimeout(() => {
    resolve(1);
  }, 100)
})
promise.then(data => {
  console.log(data);
})
```

### 2、实现promise, 满足resolve支持同步运行和添加状态

```javascript
const PENDING = 'PENDING';
const FULFILLED = 'FULFILLED';
const REJECTED = 'REJECTED';
class MyPromise{
  constructor(fn) {
    this.value = undefined;
    this.state = PENDING;
    this.resolveCallbacks = [];
    const resolve = data => {
      // step:2 需要延时执行，用setTimeout不放入同步线程中
      setTimeout(() => {
        if (this.state == PENDING) {
          this.value = data;
          this.resolveCallbacks.map(fn => fn(this.value));
        }
      }, 0);
      
    }
    const reject = err => {
      console.log(err)
    }
    fn(resolve, reject);
  }
  then(onFulfilled, onRejected) {
    this.resolveCallbacks.push(onFulfilled);
  }
}
const promise = new MyPromise((resolve, reject) => {
  // step:2 重点在这里，如果直接执行resolve, 这个时候constructor在then之前已经执行
  // 导致resolve在then执行之前就已经执行，then中传递的方法未进行缓存队列
  resolve(1);
})
promise.then(data => {
  console.log(data);            //1
})
promise.then(data => {
  console.log(data, 'second')   // 1 second
})
```

### 3、实现then的链式调用，then方法需要返回promise对象
```javascript
const PENDING = 'PENDING';
const FULFILLED = 'FULFILLED';
const REJECTED = 'REJECTED';
class MyPromise{
  constructor(fn) {
    this.state = PENDING;
    this.value = undefined;
    this.resolveCallbacks = [];
    const resolve = data => {
      setTimeout(() => {
        if (this.state == PENDING) {
          this.value = data;
          this.state = FULFILLED;
          this.resolveCallbacks.map(fn => fn(data));
        }
      });
    }
    const reject = err => {
      console.log(err)
    }
    fn(resolve, reject);
  }

  then(onFulfilled = val => val, onRejected = err => err) {
    if (this.state == PENDING) {
      // step3: then方法返回promise对象
      return new MyPromise((resolve, reject) => {
        this.resolveCallbacks.push((data) => {
          const ret = onFulfilled(data);
          resolve(ret);
        });
      })
    }
  }
}

const promise = new MyPromise((resolve, reject) => {
    resolve(1);
})

promise
    .then(data => {
      console.log(data);  //1
      return 2;
    })
    .then()
    .then(data => {
      console.log(data);  //2
    })
```

### 4、实现promise，实现then支持返回thenable对象

```javascript
const PENDING = 'PENDING';
const FULFILLED = 'FULFILLED';
const REJECTED = 'REJECTED';
class MyPromise{
  constructor(fn) {
    this.state = PENDING;
    this.value = undefined;
    this.resolveCallbacks = [];
    const resolve = data => {
      setTimeout(() => {
        if (this.state == PENDING) {
          this.value = data;
          this.state = FULFILLED;
          this.resolveCallbacks.map(fn => fn(data));
        }
      });
    }
    const reject = err => {
      console.log(err)
    }
    fn(resolve, reject);
  }

  then(onFulfilled = val => val, onRejected = err => err) {
    if (this.state == PENDING) {
      return new MyPromise((resolve, reject) => {
        this.resolveCallbacks.push((data) => {
          let ret = onFulfilled(data);
          // step:4 处理thenabled
          if ((typeof ret == 'object' || typeof ret == 'function') && typeof ret.then == 'function') {
            //
            ret = ret.then(r => {
              // 标记处Question
              resolve(r);
            }); 
          } else {
            resolve(ret);
          }
          
        });
      })
    } else if (this.state == FULFILLED) {
      return new MyPromise(resolve => {
        resolve(this.value);
      })
    } else if(this.state == REJECTED) {
      return new MyPromise((resolve, reject) => {
        reject(this.value);
      })
    }
  }
}

const promise = new MyPromise((resolve, reject) => {
    resolve(1);
})
// thenable对象，对象含有then方法
promise
  .then(data => {
    console.log(data);  //1
    return 2;
  })
  .then(data => {
    console.log(data);  //2 
    return {    // 返回then对象,then方法相当于promise的构造函数
      then(res, rej){
        res(3)
      }
    }
  })
  .then(data => {
    console.log(data);  //3
  })  
```
在代码上的“标记处Question”处，上诉示例里会直接调用resolve(3), 如果3又是一个thenable对象呢？比如以下示例
```javascript
const myPromise = new Promise(res => {
    res(1);
})
myPromise.then(data => {
    return {
        then(resolver) {
            resolver({
                then(resolveFn) {
                    resolveFn('hello world')
                }
            })
        }
    }
}).then(res => console.log(res))    // 会输出hello world
```
在MyPromise只会输也最里面的包含thenable对象，所以需要对thenable中执行方法的结果进行递归，改进的方案如下
```javascript

// 处理promise
function promiseResolutionProcedure(promise2, x, resolve, reject) {
    // 处理thenable 对象
    if (
        (typeof x === 'object' || typeof x === 'function') && 
        typeof x.then === 'function'
    ){
        x.then(y => {
            promiseResolutionProcedure(promise2, y, resolve, reject);
        })
    } else {
        resolve(x);
    }
}
class MyPromise {
    constructor(fn){
        // ... 这里与上面一致
    }
    then(onFulfilled= val => val, onRejected = err => err){
        if (this.state == PENDING) {
            const promise2 =  new MyPromise((resolve, reject) => {
                this.resolveCallbacks.push(data => {
                    const x = onFulfilled(data);
                    promiseResolutionProcedure(promise2, x, resolve, reject);
                })
            });
            return promise2
        }
        // ... 后面一样
    }
}
```


### 5、实现promise，实现then支持返回promise对象
```javascript
// 可以实现如下原生promise的调用结果
const promise = new Promise(res => {
    res(1);
})
promise.then(data => {
    return new Promise(res => {
        res(2)
    })
}).then(data => {
    console.log(data);  //2
})
```
```javascript
// 这里只需要改进promiseResolutionProcedure方法
function promiseResolutionProcedure(promise2, x, resolve, reject) {
    // 处理promise对象
    if (x instanceof MyPromise) {
        if (x.state == PENDING) {
            x.then(y => {
                // 如果y值又是promise对象，所以这里需要递归
                promiseResolutionProcedure(promise2, y, resolve, reject);
            }, reject);
        } else if (x.state == FULFILLED) {
            resolve(x.value);
        } else {
            reject(x.value);
        }
        return;
    }
    // 处理thenable 对象
    if (x !== null &&  // typeof null === object
        (typeof x === 'object' || typeof x === 'function') && 
        typeof x.then === 'function'
    ){
        x.then(y => {
            promiseResolutionProcedure(promise2, y, resolve, reject);
        })
    } else {
        resolve(x);
    }
}
```
### 6、实现promise, 解决循环引用的问题
如示例，原生promise会报错，因为then自己就需要返回一个promise对象，取个名字叫promise1，如果then传递的方法返回的结果与promise1，这个就会导致循环引用了。
```javascript
const i = new MyPromise(res => res(1)});
const promise1 = i.then(data => {
    return promise1;        // 原生报错 chaining cycle....
})

// 解决方案也是修改promiseResolutionProcedure
function promiseResolutionProcedure(promise2, x, resolve, reject) {
    // then方法在返回promise对象时，将返回值赋值给了promise2变量
    if (promise2 == x) {
        throw new Error('循环引用');
    }
    // ......
}
```
### 7、实现promise，使resolve可以传递promise或thenable对象
```javascript
const promise = new MyPromise(res => {
    res(new MyPromise(r => r(1)));
})
promise.then(data => {
    console.log(data);  //原生promise会直接打印1，不是一个promise对象
})
```
改进
```javascript
class MyPromise{
  constructor(fn) {
    this.state = PENDING;
    this.value = undefined;
    this.count = count++;
    this.resolveCallbacks = [];
    const resolve = data => {
      //重点在这里
      if (data !== null && (typeof data === 'object' || typeof data === 'function') && typeof data.then ===
      'function') {
        return promiseResolutionProcedure(this, data, resolve, reject);
      }
      setTimeout(() => {
        if (this.state == PENDING) {
          this.value = data;
          this.state = FULFILLED;
          this.resolveCallbacks.map(fn => fn(data));
        }
      });
    }
    const reject = err => {
      console.log(err)
    }
    fn(resolve, reject);
  }
}
```

### 8、完整版，补全reject
```javascript
const PENDING = 'PENDING';
const FULFILLED = 'FULFILLED';
const REJECTED = 'REJECTED';
let count = 0;
function promiseResolutionProcedure(promise2, x, resolve, reject) {
  // 解决在then中返回自己
  if (promise2 === x) {
    throw new Error('循环引用promise');
  }
  if (x instanceof MyPromise) {
    // 返回值是promise
    if (x.state == PENDING) {
      x.then(y => {
        promiseResolutionProcedure(promise2, y, resolve, reject);
      }, reject)
    } else if (x.state == FULFILLED) {
      resolve(x.value);
    } else {
      reject(x.value);
    }
  } else if (x !== null && (typeof x === 'object' || typeof x === 'function') && typeof x.then === 'function')
  {
    // 返回值是thenable
    x.then(y => {
      // 解决y还是thenable对象
      promiseResolutionProcedure(promise2, y, resolve, reject);
    }, reject);
  } else {
    // console.log(promise2);
    resolve(x);
  }
}
class MyPromise{
  static all(promiseArr) {
    
  }
  constructor(fn) {
    this.state = PENDING;
    this.value = undefined;
    this.count = count++;
    this.resolveCallbacks = [];
    this.rejectCallbacks = [];
    const resolve = data => {
      if (data !== null && (typeof data === 'object' || typeof data === 'function') && typeof data.then ===
      'function') {
        return promiseResolutionProcedure(this, data, resolve, reject);
      }
      setTimeout(() => {
        if (this.state == PENDING) {
          this.value = data;
          this.state = FULFILLED;
          this.resolveCallbacks.map(fn => fn(data));
        }
      });
    }
    const reject = data => {
      if (data !== null && 
          (typeof data === 'object' || typeof data === 'function') && 
          typeof data.then === 'function'
      ) {
        return promiseResolutionProcedure(this, data, resolve, reject);
      }
      setTimeout(() => {
        if (this.state == PENDING) {
          this.value = data;
          this.state = REJECTED;
          this.rejectCallbacks.map(fn => fn(data));
        }
      });
    }
    fn(resolve, reject);
  }

  catch(onRejected){
    return this.then(null, onRejected);
  }

  then(
    onFulfilled = val => val,
    onRejected = err => {throw new Error(err)}
  ) {
    let promise2 = null;
    if (this.state == FULFILLED) {
      promise2 = new MyPromise((resolve, reject) => {
        let ret = onFulfilled !== null ? onFulfilled(this.value) : '';
        promiseResolutionProcedure(promise2, ret, resolve, reject);
      })
    }
    if(this.state == REJECTED) {
      promise2 = new MyPromise((resolve, reject) => {
        let ret = onRejected !== null ? onRejected(this.value) : '';
        promiseResolutionProcedure(promise2, ret, resolve, reject);
      })
    }
    if (this.state == PENDING) {
      promise2 = new MyPromise((resolve, reject) => {
        this.resolveCallbacks.push((data) => {
          let ret = onFulfilled !== null ? onFulfilled(data) : undefined;
          promiseResolutionProcedure(promise2, ret, resolve, reject);
        });
        this.rejectCallbacks.push((data) => {
          let ret = onRejected !== null ? onRejected(data) : undefined;
          promiseResolutionProcedure(promise2, ret, resolve, reject);
        });
      });
      return promise2;
    }
    
    return promise2;
  }
}
```



  [1]: https://promisesaplus.com/