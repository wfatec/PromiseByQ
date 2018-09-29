# 安全性

promise另一个至关重要的改进是确保callbacks和errbacks能够在将来的事件循环中被调用。这将极大的降低异步编程给控制流（control-flow）带来的风险。我们回顾之前的maybeOneOneSecondLater：

```js
var maybeOneOneSecondLater = function (callback, errback) {
    var result = defer();
    setTimeout(function () {
        if (Math.random() < .5) {
            result.resolve(1);
        } else {
            result.resolve(reject("Can't provide one."));
        }
    }, 1000);
    return result.promise;
};
```

可以看到必须将**result.resolve()**的调用放到setTimeout中，实际上就是为了确保resolve调用时，then所传入的callback和errback方法已经注册成功。让我们来再看一个简单的例子：

```js
var blah = function(){
    var result = foob().then(function(){
        return barf();
    });
    var barf = function(){
        return 10;
    };
    return result;
};
```

这个函数最终会抛出一个异常还是返回一个立即为完成状态且值为10的promise，完全取决于foob()函数的是在同一轮事件循环(event loop)中resolve，还是会在下一轮才执行resolve。在前一种情况下resolve时，其then方法还没有注册相应的回调函数，因此会抛出异常；后一种情况时，则会正常执行。如何才能避免出现这类异常呢？我们可以想办法将回调函数(callback)自动延时到下一轮事件循环才执行，此时我们就不必显式的将resove方法的执行放到setTimeout中了。

```js
var enqueue = function (callback) {
    //process.nextTick(callback); // NodeJS
    setTimeout(callback, 1); // Naïve browser solution
};

var defer = function () {
    var pending = [], value;
    return {
        resolve: function (_value) {
            if (pending) {
                value = ref(_value);
                for (var i = 0, ii = pending.length; i < ii; i++) {
                    // XXX
                    enqueue(function () {
                        value.then.apply(value, pending[i]);
                    });
                }
                pending = undefined;
            }
        },
        promise: {
            then: function (_callback, _errback) {
                var result = defer();
                _callback = _callback || function (value) {
                    return value;
                };
                _errback = _errback || function (reason) {
                    return reject(reason);
                };
                var callback = function (value) {
                    result.resolve(_callback(value));
                };
                var errback = function (reason) {
                    result.resolve(_errback(reason));
                };
                if (pending) {
                    pending.push([callback, errback]);
                } else {
                    // XXX
                    enqueue(function () {
                        value.then(callback, errback);
                    });
                }
                return result.promise;
            }
        }
    };
};

var ref = function (value) {
    if (value && value.then)
        return value;
    return {
        then: function (callback) {
            var result = defer();
            // XXX
            enqueue(function () {
                result.resolve(callback(value));
            });
            return result.promise;
        }
    };
};

var reject = function (reason) {
    return {
        then: function (callback, errback) {
            var result = defer();
            // XXX
            enqueue(function () {
                result.resolve(errback(reason));
            });
            return result.promise;
        }
    };
};
```

尽管我们做出了修改，这里还有一个安全问题---任意可以执行'then'方法的对象都被看作是一个promise对象，这会让直接调用'then'方法出现一些意外的情况：

- callback和errback方法可能在同一个循环中被调用
- callback和errback方法可能同时被调用
- callback或者errback方法可能会被多次调用

我们通过'when'方法来包裹一个promise对象，并且阻止上述意外情况的发生。此外我们也可以就此对callback和errback进行封装来使得异常情况的抛出被转移到rejection中：

```js
var when = function (value, _callback, _errback) {
    var result = defer();
    var done;

    _callback = _callback || function (value) {
        return value;
    };
    _errback = _errback || function (reason) {
        return reject(reason);
    };

    var callback = function (value) {
        try {
            return _callback(value);
        } catch (reason) {
            return reject(reason);
        }
    };
    var errback = function (reason) {
        try {
            return _errback(reason);
        } catch (reason) {
            return reject(reason);
        }
    };

    enqueue(function () {
        ref(value).then(function (value) {
            if (done)
                return;
            done = true;
            result.resolve(ref(value).then(callback, errback));
        }, function (reason) {
            if (done)
                return;
            done = true;
            result.resolve(errback(reason));
        });
    });

    return result.promise;
};
```

这样我们就成功阻止了上述意外情况的发生，提高了promise的安全性和不可变性。