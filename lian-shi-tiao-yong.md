# 链式调用

接下来的重点就是让promises更易于组合，让新的promise可以从旧的promise中获取value值并使用。我们来看一下下面的例子：

```javascript
var twoOneSecondLater = function (callback) {
    var a, b;
    var consider = function () {
        if (a === undefined || b === undefined)
            return;
        callback(a + b);
    };
    oneOneSecondLater(function (_a) {
        a = _a;
        consider();
    });
    oneOneSecondLater(function (_b) {
        b = _b;
        consider();
    });
};

twoOneSecondLater(function (c) {
    // c === 2
});
```

这段代码显得非常的琐碎，尤其是这里需要一个哨兵值去判断是否需要调用callback。接下来我们将通过promise用更少的代码来解决上述问题，并且隐式的处理error传播，使用形式如下：

```javascript
var a = oneOneSecondLater();
var b = oneOneSecondLater();
var c = a.then(function (a) {
    return b.then(function (b) {
        return a + b;
    });
});
```

为了实现上述效果，我们需要解决一下问题：
1. “then”方法必须返回一个promise对象；
2. 返回的promise对象必须最终resolve一个callback函数的返回值；
3. callback函数的返回值必须是一个fulfilled值或者一个promise对象

将value值转成fulfilled状态的promise方法非常直接，下面的方法可以将任何value转成fulfilled状态的observers：

```javascript
var ref = function(value){
    return {
        then: function(callback){
            callback(value);
        }
    };
};
```

这里又有一个新的思考，如果入参value本身就是promise对象呢？这时我们回到问题的初衷，将value转换为promise，很显然如果value本身符合promise规范，那么直接返回value即可，改造如下：

```javascript
var ref = function(value){
    if(value && typeof value.then === "function"){ return value };
    return {
        then: function(callback){
            callback(value);
        }
    };
};
```

现在我们进一步改造一下then函数，将callback的返回值包装为promise并返回，方法如下：

```javascript
var ref = function(value){
    if(value && typeof value.then === "function"){ return value };
    return {
        then: function(callback){
            return ref(callback(value));
        }
    };
};
```

现在我们考虑一个更加复杂的情况，我们希望callback函数能够在未来的某个时间点被调用而非立即执行。让我们回到“defer”函数，并且将其中的callback包裹起来，callback的返回值将会resolve  then返回的promise。可能听起来有点拗口，其实简单说来就是我们的then函数最终会返回一个promise，这个新的promise在resolve时，传给resolve的参数就是这个callback的返回值，这也是能够实现链式调用的精髓。

此外，当resolution本身就是一个需要延时resolve的promise对象时，在何处执行resolve也是resolve方法需要处理的问题。最终需要实现的效果是：无论“defer”还是“ref”返回的promise，都能够执行then方法。如果是“ref”型promise，则行为表现和之前相同：callback将会被“then(callback)”立即调用。如果是“defer”型promise，则调用“then(callback)”时，会把callback传递给一个新的promise。因此，现在的callback将观察（observe）一个新的promise，并在执行完成后，将返回结果作为参数来执行新promise的resolve方法。

```javascript
var defer = function(){
    var pending = [], value;
    return{
        resolve: function(_value){
            if(pending){
                // 将value值封装为promise
                value = ref(_value);
                for(var i=0, len=pending.length; i<len; i++){
                    value.then(pending[i]);
                }
                pending = undefined;
            }
        },
        promise: {
            then: function(_callback){
                var result = defer();
                // 将callback封装起来以获取到其返回值，
                // 并用这个返回值来resolve “then”函数返回的promise
                var callback = function(value){
                    result.resolve(_callback(value))
                };
                if(pending){
                    pending.push(callback);
                } else {
                    value.then(callback);
                }
                return result.promise;
            }
        }
    };
};
```

这里我们使用了“thenable”类型的promise，并通过一个defer函数将“promise”和“resolve”分离开来，实现了我们最初的设想。
