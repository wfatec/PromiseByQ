# 错误传递

为了实现错误传递，我们再引入errback参数。我们首先使用一个新的promise类型，与之前实现的“ref”型promise非常类似，区别只是这里调用的是errback并传入错误信息，而不是调用callback。

```javascript
var reject = function(reason){
    return {
        then: function(callback, errback){
            return ref(errback(reason));
        }
    }
}
```

我们通过下面一个简单的例子来观察一下reject方法的使用：

```javascript
reject("Error!").then(function(value){
    // 不会执行
}, function(reason){
    // resoon === "Error!"
})
```

现在我们来对之前调用promise接口的一个用例进行修改，主要针对错误处理部分：

```javascript
var maybeOneOneSecondLater = function(){
    var result = defer();
    setTimeout(function(){
        if(Math.random() < .5){
            result.resolve(1);
        } else {
            result.resolve(reject("Can't provide one"));
        }
    }, 1000);
    return result.promise;
};
```

为了让上述代码生效，我们需要继续对defer进行改造，pending数组的callback要被换成[callback,errback]：

```javascript
var defer = function(){
    var pending = [], value;
    return {
        resolve: function(_value){
            if(pending){
                value = ref(_value);
                for(var i=0, len=pending.length; i < len; i++){
                    // value.then(pending[i][0],pending[i][1]), 不够优雅
                    // 此时pending[i]是[callback,errback]形式的数组，
                    // 故可用apply直接将pending作为参数传入then方法
                    value.then.apply(value, pending[i]);
                }
                pending = undefined;
            }
        },
        promise: {
            then: function(_callback, _errback){
                var result = defer();
                var callback = function(value){
                    result.resolve(_callback(value));
                };
                var errback = function(reason){
                    result.resolve(_errback(reason));
                };
                if(pending){
                    pending.push([callback,errback]);
                } else {
                    value.then(callback, errback);
                }
                return result.promise;
            }
        }
    };
};
```

现在我们的"defer"还有一个值得注意的问题：所有的"then"方法必须传入errback，否则当调用一个不存在的函数时就会抛出异常。有一个最简单的解决方式，就是传入默认的errback来处理rejection的情况。同样的道理，我们也可以传入一个默认的callback方法来处理fulfilled的value值。

```js
var defer = function () {
    var pending = [], value;
    return {
        resolve: function (_value) {
            if (pending) {
                value = ref(_value);
                for (var i = 0, ii = pending.length; i < ii; i++) {
                    value.then.apply(value, pending[i]);
                }
                pending = undefined;
            }
        },
        promise: {
            then: function (_callback, _errback) {
                var result = defer();
                // provide default callbacks and errbacks
                _callback = _callback || function (value) {
                    // by default, forward fulfillment
                    return value;
                };
                _errback = _errback || function (reason) {
                    // by default, forward rejection
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
                    value.then(callback, errback);
                }
                return result.promise;
            }
        }
    };
};
```

现在，我们已经实现了完整的错误传递机制。同时无论是串行方式或是并行方式，我们都能够轻松的从其他promise对象中生成新的promise。再次回到之前的问题，通过promise实现分布式求和：

```javascript
promises.reduce(function(accumulating, promise){
    return accumulating.then(function(accumulated){
        return promise.then(function(value){
            retuen accumulated + value;
        });
    });
}, ref(0)) // 从resolve结果为0的promise开始计算，这样我们可以像其他组合的promise一样调用then方法
            // 有利于代码的一致性
.then(function(sum){
    // sum即为最终的求和结果
})
```