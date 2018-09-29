# 基本功能实现

想象一下，当你需要实现一个不需要立即返回结果的函数时，一个最为自然的想法就是使用回掉函数，这也是JS事件轮询机制的精髓。具体实现可能会是下面的形式：

```javascript
var oneOneSecondLater = function (callback) {
    setTimeout(function () {
        callback(1);
    }, 1000);
};
```

这是解决以上问题的一个简单方法，但是我们还有很多的改进空间。更加通用的方案是加入异常校验，同时提供callback和errback.

```javascript
var maybeOneOneSecondLater = function (callback, errback) {
    setTimeout(function () {
        if (Math.random() < .5) {
            callback(1);
        } else {
            errback(new Error("Can't provide one."));
        }
    }, 1000);
};
```

以上方法通过显式的错误处理逻辑`new Error()`来实现异常处理，但是这种错误处理方式太过机械化，同时也很难灵活的对错误信息进行处理。

## Promise

下面我们来思考一个更为一般的解决方案，让函数本身返回一个对象来表示最终的结果（successful或failed），而非之前的返回values或是直接抛出异常。这个返回的结果就是`Promise`，正如其名字（承诺）显示的那样，最终需要被resolve（执行）。我们需要可以通过调用promise上的一个函数来观察其处于完成态(fulfillment)还是拒绝态(rejection)。如果一个promise被rejected，并且此rejection未能被明确的观察，则所有衍生出来的promises将隐式的执行相同原因的reject。

接下来我们构建一个简单的‘promise’，其实就是一个包含key为then的一个对象，且在then上挂载了callback：

```javascript
var maybeOneOneSecondLater(){
    var callback;
    setTimeout(function(){
        callback(1);
    },1000);
    return {
        then : function(_callback){
            callback = _callback;
        }
    }
}

maybeOneOneSecondLater().then(callback);
```

该版本有两个缺点：

1. then方法只有最后挂载的callback生效，如果我们能够将所有的callback保存下来，并依次调用也许会更加实用；
2. 如果我们的callback注册时间超过1秒，此时promise已经构建完毕，那么这个callback也永远不会调用。

一个更通用的解决方案需要能够接收任意数量的callback，并且能够保证其能够在任何情况下注册成功。我们将通过构建一个包含两种状态(two-state)的对象来实现promise。

一个promise的初始状态为unresolved，并且此时所有callbacks均被添加到一个名为pending的观察者（observer）数组中。当该promise执行resolve时，所有的observers被唤醒并执行。当promise处于resolved状态后，新注册的callbacks将立即被调用。我们通过pending数组中的callbacks是否存在来判断state的变化，并且在resolution之后将所有callbacks清空。

```javascript
var maybeOneOneSecondLater = function () {
    var pending = [], value;
    setTimeout(function () {
        value = 1;
        for (var i = 0, ii = pending.length; i < ii; i++) {
            var callback = pending[i];
            callback(value);
        }
        pending = undefined;
    }, 1000);
    return {
        then: function (callback) {
            if (pending) {
                pending.push(callback);
            } else {
                callback(value);
            }
        }
    };
};
```

这个函数已经很好的解决了问题，下面我们将其改写为工具函数使其更便于使用。定义一个deferred对象，主要包含两个部分：

1. 注册观察者队列（observers）
2. 通知observers并执行

```javascript
var defer = function(){
    var pending = [], value;
    return{
        resolve: function(_value){
            value = _value;
            var len = pending.length
            for (var i = 0; i < len; i++){
                pending[i](value);
            }
            pending = undefined;
        },
        then: function(callback){
            if(pending){
                pending.push(callback);
            } else {
                callback(value)
            }
        }
    }
};

var oneOneSecondLater = function(){
    var result = defer();
    setTimeout(function(){
        result.resolve(1)
    },1000);
};

oneOneSecondLater().then(callback);
```

这里resolve函数有一个缺陷：可以被多次调用，并不断改变result的值。因此我们需要保护value值不被意外或恶意篡改，解决方案就是只允许resolve执行一次。

```javascript
var defer = function(){
    var pending = [], value;
    return{
        resolve: function(_value){
            if(pending){
                value = _value;
                var len = pending.length
                for (var i = 0; i < len; i++){
                    pending[i](value);
                }
                pending = undefined;
            } else {
                throw new Error("A promise can only be resolved once.")
            }            
        },
        then: function(callback){
            if(pending){
                pending.push(callback);
            } else {
                callback(value);
            }
        }
    }
};

var oneOneSecondLater = function(){
    var result = defer();
    setTimeout(function(){
        result.resolve(1)
    },1000);
    return result;
};

oneOneSecondLater().then(callback);
```

发散一下思维，我们除了以上功能外，还可以定制一些特有的处理逻辑，例如：

1. 传递一个参数来作为抛出的错误信息或者忽略后续callback的执行；
2. 可以在resolve时，实现callbacks的竞争（race）机制，忽略之后的resolutions；
当然可能还有很多的场景可以思考，这些就留给读者自行发散思维吧；