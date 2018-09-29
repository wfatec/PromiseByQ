# 代码分离

基于之前的设计，我们再进一步思考，主要分为两个层面：

1. 如果能够将defferd函数中的promise和resolver部分进行分割和组合，那将大大提高其灵活性；
2. 是否可以通过某些方法来将promises和其他类型的数值进行区分，以便于维护。

- 将promise部分从resolver中分离出来可以让我们的代码更好的践行最小权限原则(the principle of least authority)。promise对象给使用者的权限主要有两个：

1. 观察（observer）promise的resolution
2. resolver接口，仅用于决定何时触发resolution

权限不能隐式的传递，实践证明，过多的权限必然会导致其滥用并最终使得代码难以维护

然而这种分离当然也存在缺点，那就是使用promise对象会增加垃圾回收处理的负担(主要是由于大量闭包的使用)

- 当然，我们也有很多的方法来对promise对象进行判别。其中最为直观和显著的方式就是通过原型继承(prototypical inheritance)

```javascript
var Promise = function(){};

var isPromise = function(value){
    return value instanceof Promise;
};

var defer = function(){
    var pending = [], value;
    var promise = new Promise();
    promise.then = function(callback){
        if(pending){
            pending.push(callback);
        }else{
            callback(value)
        }
    };
    return {
        resolve: function(_value){
            if(pending){
                value = _value;
                for(var i = 0, len = pending.length; i < len; i++){
                    pending[i](value);
                }
                pending = undefined;
            }
        },
        promise: promise
    };
};
```

使用原型继承有一个缺点，那就是能且只能是promise的实例才能在程序中起作用，这就导致一个代码耦合性的问题，使得实际使用起来不够灵活且难以维护。

另外一中实现思路就是使用“[鸭子类型（duck-typing）](https://en.wikipedia.org/wiki/Duck_typing)”，通过是否实现了约定的方法来判断是否是promise（类似于接口，约定）。在 CommonJS/Promises/A中，使用“then”来区分promise。这也会有一些缺点，例如当一个对象恰好有一个then方法时，也会被误判为promise。但在实际使用中这却不够构成困扰，毕竟这种情况出现并造成影响的概率很小。

```javascript
var isPromise = function(value){
    return value&& (typeof value.then === "function");
};

var defer = function(){
    var pending = [], value;
    return {
        resolve: function(_value){
            if(pending){
                value = _value;
                for(var i=0, len=pending.length; i<len; i++){
                    pending[i](value);
                }
                pending = undefined;
            }
        },
        promise: {
            then: function(callback){
                if(pending){
                    pending.push(callback);
                }else{
                    callback(value);
                }
            }
        }
    };
};
```