# Compositional Functions

Asynchronous programming is clearly a pain point for JavaScript developers. The async/await syntax would be a valuable addition to the JavaScript language. However the async/await syntax can only emit a Promise, which is one of any number of different asynchronous primitives used in JavaScript programs.

Other asynchronous primitives include:

1. Observable
2. Task

Some of these asynchronous primitives have valuable semantics which are a better fit for certain APIs. One such semantic is the ability to signal to the producing function that that a value is no longer required. When no observers exist for an value that will eventually be asynchronously produced, the process of producing that value may be able to be cancelled.

The current async/await syntax can only produce Promises, forever using valuable syntactic space that will not be available to compose future asynchronous primitives in the future. It also implies that the natural primitives for async programming is Promise, but that may not be the case in other systems.

A better approach is to add _extensible syntax_ to the language that can be extended by the community to compose any asynchronous primitive. This is the goal of the Compositional Function proposal.

# Compositional Functions

The compositional function proposal is intended to allow for the composition of any async primitives that results in a single result. 

## Supporting Promise Composition with Compositional Functions

Here is an example of compositional function used to produce a Promises:

```JavaScript
var getStockPrice = Promise function(name) {
    var symbol = await getStockSymbol(name);
    var stockPrice = await getStockPrice(symbol);
    return stockPrice;
}
```

If your system uses Promises as the core async primitive, you may want to alias Promise to emphasize that. By aliasing Promise to 'async' the code can match today's async/await proposal exactly:

```JavaScript
var async = Promise;

var getStockPrice = async function(name) {
    var symbol = await getStockSymbol(name);
    var stockPrice = await getStockPrice(symbol);
    return stockPrice;
}
```

The code above desugars to this:

```JavaScript
var async = Promise;

var getStockPrice = async[Symbol.fromGenerator](function*(name) {
    var symbol = yield getStockSymbol(name);
    var stockPrice = yield getStockPrice(symbol);
    return stockPrice;
});
```

Here is the definition of the Symbol.fromGenerator function:

```JavaScript
Promise[Symbol.fromGenerator] = function(genF) {
    return new Promise(function(resolve, reject) {
        var gen = genF();
        function step(nextF) {
            var next;
            try {
                next = nextF();
            } catch(e) {
                // finished with failure, reject the promise
                reject(e); 
                return;
            }
            if(next.done) {
                // finished with success, resolve the promise
                resolve(next.value);
                return;
            } 
            // not finished, chain off the yielded promise and `step` again
            Promise.resolve(next.value).then(function(v) {
                step(function() { return gen.next(v); });      
            }, function(e) {
                step(function() { return gen.throw(e); });
            });
        }
        step(function() { return gen.next(undefined); });
    });    
}
```

## Supporting Task Composition with Compositional Functions

Compositional Functions can be applied to other asynchronous Primitives. Let's define an asynchronous Task primitive. The Task primitive represents the task of creating an asynchronous result.

Let's create a constructor for Task.

```JavaScript

function Task(get) {
    var self = this;

    this.get = function(valueFn, throwFn) {
        // send memoized error
        if (self.error) {
            throwFn(self.error);
        }
        // send memoized value
        else if ('value' in self) {
            valueFn(self.value);
        }
        else {
            return get(
                value => {
                    // memoize value
                    this.value = value;
                    valueFn(value);
                }, 
                error => {
                    // memoize error
                    this.error = error;
                    throwFn(error);
                });
        }
    };
}
```

You can retrieve the value from a Task using the Task's get method:

```JavaScript
var subscription = task.get(value => console.log(value), error = console.error(error));
```

The get method returns a subscription object, which can be disposed in order to stop listening for the Tasks eventual value.

```JavaScript
var subscription = task.get(value => console.log(value), error = console.error(error));
// signal to task that we're no longer interested in result
subscription.dispose();
```

Tasks can be sequenced using the 'when' method.

```JavaScript
Task.prototype.when = function(projection) {
    var self = this;
    return new Task(function(valueFn, throwFn) {
        var subscription = 
            self.get(
                x => {
                    try {
                        var nextTask = Task.resolve(projection(x));
                        subscription = nextTask.get(valueFn, throwFn);
                    }
                    catch(e) {
                        throwFn(e);
                    }
                },
                throwFn);

        return {
          dispose: function() {
            subscription.dispose();
          }
        };
    });
};
```

You can also use the resolve function to coerce non-Task values into Tasks:

```JavaScript
Task.resolve = function(v) {
    if (v instanceof Task) {
        return v;
    }
    else {
        // Wrap value in Task that immediately sends the value
        return new Task(function(valueFn, errorFn) {
            valueFn(v);
            // NOOP disposal
            return { dispose: function() {} };
        });
    }
};
```

Now we can define the composition function on the Task constructor:

```JavaScript
Task[Symbol.fromGenerator] = function(genF) {
    return new Task(function(resolve, reject) {
        var gen = genF();
        function step(nextF) {
            var next;
            try {
                next = nextF();
            } catch(e) {
                // finished with failure, reject the promise
                reject(e); 
                return;
            }
            if(next.done) {
                // finished with success, resolve the promise
                resolve(next.value);
                return;
            } 
            // not finished, chain off the yielded promise and `step` again
            Task.resolve(next.value).when(function(v) {
                step(function() { return gen.next(v); });      
            }, function(e) {
                step(function() { return gen.throw(e); });
            });
        }
        step(function() { return gen.next(undefined); });
    });    
}
```

Now we can use compositional functions to sequence Tasks, which model asynchronous tasks that can be cancelled.

If we refactor the getStockSymbol and getStockPrice methods in the Promise example to use Tasks, we can use the same compositional style to compose the getStockPrice method from these two methods.

```JavaScript
var getStockPrice = Task function(name) {
    var symbol = await getStockSymbol(name);
    var stockPrice = await getStockPrice(symbol);
    return stockPrice;
}

var subscription = 
    getStockPrice("Johnson and Johnson").
    get(
        price => console.log(price),
        error => console.error(error));

// immediately cancel outgoing requests
subscription.dispose();
```

