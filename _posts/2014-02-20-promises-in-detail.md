---
layout: post
title: JavaScript Promises ... In Wicked Detail
author: Matt Greer
categories:
- promises
- tutorial
---

<div class="intro">
  This post is by Matt Greer.  You can find the original here: <a href="http://mattgreer.org/articles/promises-in-wicked-detail/">mattgreer.org/articles/promises-in-wicked-detail/</a>
</div>

I've been using Promises in my JavaScript code for a while now. They can be a little brain bending at first. I now use them pretty effectively, but when it came down to it, I didn't fully understand how they work. This article is my resolution to that. If you stick around until the end, you should understand Promises well too.

We will be incrementally creating a Promise implementation that by the end will *mostly* meet the [Promise/A+ spec](http://promises-aplus.github.io/promises-spec/), and understand how promises meet the needs of asynchronous programming along the way. This article assumes you already have some familiarity with Promises. If you don't, [promisejs.org](http://promisejs.org) is a good site to check out.

## Why?

Why bother to understand Promises to this level of detail? Really understanding how something works can increase your ability to take advantage of it, and debug it more successfully when things go wrong. I was inspired to write this article when a coworker and I got stumped on a tricky Promise scenario. Had I known then what I know now, we wouldn't have gotten stumped.

## The Simplest Use Case

Let's begin our Promise implementation as simple as can be. We want to go from this

{% highlight javascript %}
doSomething(function(value) {
  console.log('Got a value:' value);
});
{% endhighlight %}

to this

{% highlight javascript %}
doSomething().then(function(value) {
  console.log('Got a value:' value);
});
{% endhighlight %}

To do this, we just need to change `doSomething()` from this

{% highlight javascript %}
function doSomething(callback) {
  var value = 42;
  callback(value);
}
{% endhighlight %}

to this "Promise" based solution

{% highlight javascript %}
function doSomething() {
  return {
    then: function(callback) {
      var value = 42;
      callback(value);
    }
  };
}
{% endhighlight %}

<a class="fiddle" target="_blank" href="http://jsfiddle.net/city41/zdgrC/1/">fiddle</a>

This is just a little sugar for the callback pattern. It's pretty pointless sugar so far. But it's a start and yet we've already hit upon a core idea behind Promises

<div class="aside">
Promises capture the notion of an eventual value into an object
</div>

This is the main reason Promises are so interesting. Once the concept of eventuality is captured like this, we can begin to do some very powerful things. We'll explore this more later on.

### Defining the Promise type

This simple object literal isn't going to hold up. Let's define an actual `Promise` type that we'll be able to expand upon

{% highlight javascript %}
function Promise(fn) {
  var callback = null;
  this.then = function(cb) {
    callback = cb;
  };

  function resolve(value) {
    callback(value);
  }

  fn(resolve);
}
{% endhighlight %}

and reimplement `doSomething()` to use it

{% highlight javascript %}
function doSomething() {
  return new Promise(function(resolve) {
    var value = 42;
    resolve(value);
  });
}
{% endhighlight %}

There is a problem here. If you trace through the execution, you'll see that `resolve()` gets called before `then()`, which means `callback` will be `null`. Let's hide this problem in a little hack involving `setTimeout`

{% highlight javascript %}
function Promise(fn) {
  var callback = null;
  this.then = function(cb) {
    callback = cb;
  };

  function resolve(value) {
    // force callback to be called in the next
    // iteration of the event loop, giving
    // callback a chance to be set by then()
    setTimeout(function() {
      callback(value);
    }, 1);
  }

  fn(resolve);
}
{% endhighlight %}

<a class="fiddle" target="_blank" href="http://jsfiddle.net/city41/uQrza/1/">fiddle</a>

With the hack in place, this code now works ... sort of.

### This Code is Brittle and Bad

Our naive, poor Promise implementation must use asynchronicity to work. It's easy to make it fail again, just call `then()` asynchronously and we are right back to the callback being `null` again. Why am I setting you up for failure so soon? Because the above implementation has the advantage of being pretty easy to wrap your head around. `then()` and `resolve()` won't go away. They are key concepts in Promises.

## Promises have State

Our brittle code above revealed something unexpectedly. Promises have state. We need to know what state they are in before proceeding, and make sure we move through the states correctly. Doing so gets rid of the brittleness. 

<div class="aside">
  <ul>
    <li>A Promise can be <i>pending</i> waiting for a value, or <i>resolved</i> with a value.</li>
    <li>Once a Promise resolves to a value, it will always remain at that value and never resolve again.</li>
  </ul>
</div>

*(A Promise can also be rejected, but we'll get to error handling later)*

Let's explicitly track the state inside of our implementation, which will allow us to do away with our hack

{% highlight javascript %}
function Promise(fn) {
  var state = 'pending';
  var value;
  var deferred;

  function resolve(newValue) {
    value = newValue;
    state = 'resolved';

    if(deferred) {
      handle(deferred);
    }
  }

  function handle(onResolved) {
    if(state === 'pending') {
      deferred = onResolved;
      return;
    }

    onResolved(value);
  }

  this.then = function(onResolved) {
    handle(onResolved);
  };

  fn(resolve);
}
{% endhighlight %}

<a class="fiddle" target="_blank" href="http://jsfiddle.net/city41/QX85J/1/">fiddle</a>

It's getting more complicated, but the caller can invoke `then()` whenever they want, and the callee can invoke `resolve()` whenever they want. It fully works with synchronous or asynchronous code.

This is because of the `state` flag. Both `then()` and `resolve()` hand off to the new method `handle()`, which will do one of two things depending on the situation:

* The caller has called `then()` before the callee calls `resolve()`, that means there is no value ready to hand back. In this case the state will be pending, and so we hold onto the caller's callback to use later. Later when `resolve()` gets called, we can then invoke the callback and send the value on its way.
* The callee calls `resolve()` before the caller calls `then()`: In this case we hold onto the resulting value. Once `then()` gets called, we are ready to hand back the value.

Notice `setTimeout` went away? That's temporary, it will be coming back. But one thing at a time.

<div class="aside">
  With Promises, the order in which we work with them doesn't matter. We are free to call <code>then()</code> and <code>resolve()</code> whenever they suit our purposes. This is one of the powerful advantages of capturing the notion of eventual results into an object
</div>

We still have quite a few more things in the spec to implement, but our Promises are already pretty powerful. This system allows us to call `then()` as many times as we want, we will always get the same value back

{% highlight javascript %}
var promise = doSomething();

promise.then(function(value) {
  console.log('Got a value:', value);
});

promise.then(function(value) {
  console.log('Got the same value again:', value);
});
{% endhighlight %}

<p></p>

<div class="aside pitfall">
This is not completely true for the Promise implementation in this article. If the opposite happens, ie the caller calls <code>then()</code> multiple times before <code>resolve()</code> is called, only the last call to <code>then()</code> will be honored. The fix for this is to keep a running list of deferreds inside of the Promise instead of just one. I decided to not do that in the interest of keeping the article more simple, it's long enough as it is :)
</div>

## Chaining Promises

Since Promises capture the notion of asynchronicity in an object, we can chain them, map them, have them run in parallel or sequential, all kinds of useful things. Code like the following is very common with Promises

{% highlight javascript %}
getSomeData()
.then(filterTheData)
.then(processTheData)
.then(displayTheData);
{% endhighlight %}

`getSomeData` is returning a Promise, as evidenced by the call to `then()`, but the result of that first then must also be a Promise, as we call `then()` again (and yet again!) That's exactly what happens, if we can convince `then()` to return a Promise, things get more interesting.

<div class="aside">
  <code>then()</code> always returns a Promise
</div>

Here is our Promise type with chaining added in

{% highlight javascript %}
function Promise(fn) {
  var state = 'pending';
  var value;
  var deferred = null;

  function resolve(newValue) {
    value = newValue;
    state = 'resolved';

    if(deferred) {
      handle(deferred);
    }
  }

  function handle(handler) {
    if(state === 'pending') {
      deferred = handler;
      return;
    }

    if(!handler.onResolved) {
      handler.resolve(value);
      return;
    }

    var ret = handler.onResolved(value);
    handler.resolve(ret);
  }

  this.then = function(onResolved) {
    return new Promise(function(resolve) {
      handle({
        onResolved: onResolved,
        resolve: resolve
      });
    });
  };

  fn(resolve);
}
{% endhighlight %}

<a class="fiddle" target="_blank" href="http://jsfiddle.net/city41/HdzLv/2/">fiddle</a>

Hoo, it's getting a little squirrelly. Aren't you glad we're building this up slowly? The real key here is that `then()` is returning a new Promise. 

<div class="aside pitfall">
Since <code>then()</code> always returns a new Promise object, there will always be at least one Promise object that gets created, resolved and then ignored. Which can be seen as wasteful. The callback approach does not have this problem. Another ding against Promises. You can start to appreciate why some in the JavaScript community have shunned them.
</div>

What value does the second Promise resolve to? *It receives the return value of the first promise.* This is happening at the bottom of `handle()`, The `handler` object carries around both an `onResolved` callback as well as a reference to `resolve()`. There is more than one copy of `resolve()` floating around, each Promise gets their own copy of this function, and a closure for it to run within. This is the bridge from the first Promise to the second. We are concluding the first Promise at this line:

{% highlight javascript %}
var ret = handler.onResolved(value);
{% endhighlight %}

In the examples I've been using here, `handler.onResolved` is

{% highlight javascript %}
function(value) {
  console.log("Got a value:", value);
}
{% endhighlight %}

in other words, it's what was passed into the first call to `then()`. The return value of that first handler is used to resolve the second Promise. Thus chaining is accomplished

{% highlight javascript %}
doSomething().then(function(result) {
  console.log('first result', result);
  return 88;
}).then(function(secondResult) {
  console.log('second result', secondResult);
});

// the output is
//
// first result 42
// second result 88


doSomething().then(function(result) {
  console.log('first result', result);
  // not explicitly returning anything
}).then(function(secondResult) {
  console.log('second result', secondResult);
});

// now the output is
//
// first result 42
// second result undefined
{% endhighlight %}

Since `then()` always returns a new Promise, this chaining can go as deep as we like


{% highlight javascript %}
doSomething().then(function(result) {
  console.log('first result', result);
  return 88;
}).then(function(secondResult) {
  console.log('second result', secondResult);
  return 99;
}).then(function(thirdResult) {
  console.log('third result', thirdResult);
  return 200;
}).then(function(fourthResult) {
  // on and on...
});
{% endhighlight %}

What if in the above example, we wanted all the results in the end? With chaining, we would need to manually build up the result ourself

{% highlight javascript %}
doSomething().then(function(result) {
  var results = [result];
  results.push(88);
  return results;
}).then(function(results) {
  results.push(99);
  return results;
}).then(function(results) {
  console.log(results.join(', ');
});

// the output is
//
// 42, 88, 99
{% endhighlight %}

<div class="aside">
  Promises always resolve to one value. If you need to pass more than one value along, you need to create a multi-value in some fashion (an array, an object, concatting strings, etc)
</div>

A potentially better way is to use a Promise library's `all()` method or any number of other utility methods that increase the usefulness of Promises, which I'll leave to you to go and discover. 

### The Callback is Optional

The callback to `then()` is not strictly required. If you leave it off, the Promise resolves to the same value as the previous Promise

{% highlight javascript %}
doSomething().then().then(function(result) {
  console.log('got a result', result);
});

// the output is
//
// got a result 42
{% endhighlight %}

You can see this inside of `handle()`, where if there is no callback, it simply resolves the Promise and exits. `value` is still the value of the previous Promise.

{% highlight javascript %}
if(!handler.onResolved) {
  handler.resolve(value);
  return;
}
{% endhighlight %}


### Returning Promises Inside the Chain
Our chaining implementation is a bit naive. It's blindly passing the resolved values down the line. What if one of the resolved values is a Promise? For example

{% highlight javascript %}
doSomething().then(result) {
  // doSomethingElse returns a Promise
  return doSomethingElse(result)
}.then(function(finalResult) {
  console.log("the final result is", finalResult);
});
{% endhighlight %}

As it stands now, the above won't do what we want. `finalResult` won't actually be a fully resolved value, it will instead be a Promise. To get the intended result, we'd need to do

{% highlight javascript %}
doSomething().then(result) {
  // doSomethingElse returns a Promise
  return doSomethingElse(result)
}.then(function(anotherPromise) {
  anotherPromise.then(function(finalResult) {
    console.log("the final result is", finalResult);
  });
});
{% endhighlight %}

Who wants that crud in their code? Let's have the Promise implementation seamlessly handle this for us. This is simple to do, inside of `resolve()` just add a special case if the resolved value is a Promise

{% highlight javascript %}
function resolve(newValue) {
  if(newValue && typeof newValue.then === 'function') {
    newValue.then(resolve);
    return;
  }
  state = 'resolved';
  value = newValue;

  if(deferred) {
    handle(deferred);
  }
}
{% endhighlight %}

<a class="fiddle" target="_blank" href="http://jsfiddle.net/city41/38CCb/2/">fiddle</a>

We'll keep calling `resolve()` recursively as long as we get a Promise back. Once it's no longer a Promise, then proceed as before.

<div class="aside pitfall">
It <i>is</i> possible for this to be an infinite loop. The Promise/A+ spec recommends implementations detect infinite loops, but it's not required.
</div>

<div class="aside pitfall">
Also worth pointing out, this implementation does not meet the spec. Nor will we fully meet the spec in this regard in the article. For the more curious, I recommend reading the <a href="http://promises-aplus.github.io/promises-spec/#the_promise_resolution_procedure">Promise resolution procedure</a>.
</div>

Notice how loose the check is to see if `newValue` is a Promise? We are only looking for a `then()` method. This duck typing is intentional, it allows different Promise implementations to interopt with each other. It's actually quite common for Promise libraries to intermingle, as different third party libraries you use can each use different Promise implementations.

<div class="aside">
  Different Promise implementations can interopt with each other, as long as they all are following the spec properly.
</div>

With chaining in place, our implementation is pretty complete. But we've completely ignored error handling.

## Rejecting Promises

When something goes wrong during the course of a Promise, it needs to be **rejected** with a *reason*. How does the caller know when this happens? They can find out by passing in a second callback to `then()`

{% highlight javascript %}
doSomething().then(function(value) {
  console.log('Success!', value);
}, function(error) {
  console.log('Uh oh', error);
});
{% endhighlight %}

<p></p>

<div class="aside">
  As mentioned earlier, the Promise will transition from <i>pending</i> to either <i>resolved</i> or <i>rejected</i>, never both. In other words, only one of the above callbacks ever gets called.
</div>

Promises enable rejection by means of <code>reject()</code>, the evil twin of <code>resolve()</code>. Here is <code>doSomething()</code> with error handling support added

{% highlight javascript %}
function doSomething() {
  return new Promise(function(resolve, reject) {
    var result = somehowGetTheValue(); 
    if(result.error) {
      reject(result.error);
    } else {
      resolve(result.value);
    }
  });
}
{% endhighlight %}

Inside the Promise implementation, we need to account for rejection. As soon as a Promise is rejected, all downstream Promises from it also need to be rejected.

Let's see the full Promise implementation again, this time with rejection support added

{% highlight javascript %}
function Promise(fn) {
  var state = 'pending';
  var value;
  var deferred = null;

  function resolve(newValue) {
    if(newValue && typeof newValue.then === 'function') {
      newValue.then(resolve, reject);
      return;
    }
    state = 'resolved';
    value = newValue;

    if(deferred) {
      handle(deferred);
    }
  }

  function reject(reason) {
    state = 'rejected';
    value = reason;

    if(deferred) {
      handle(deferred);
    }
  }

  function handle(handler) {
    if(state === 'pending') {
      deferred = handler;
      return;
    }

    var handlerCallback;

    if(state === 'resolved') {
      handlerCallback = handler.onResolved;
    } else {
      handlerCallback = handler.onRejected;
    }

    if(!handlerCallback) {
      if(state === 'resolved') {
        handler.resolve(value);
      } else {
        handler.reject(value);
      }

      return;
    }

    var ret = handlerCallback(value);
    handler.resolve(ret);
  }

  this.then = function(onResolved, onRejected) {
    return new Promise(function(resolve, reject) {
      handle({
        onResolved: onResolved,
        onRejected: onRejected,
        resolve: resolve,
        reject: reject
      });
    });
  };

  fn(resolve, reject);
}
{% endhighlight %}

<a class="fiddle" target="_blank" href="http://jsfiddle.net/city41/rLXsL/2/">fiddle</a>

Other than the addition of `reject()` itself, `handle()` also has to be aware of rejection. Within `handle()`, either the rejection path or resolve path will be taken depending on the value of `state`. This value of `state` gets pushed into the next Promise, because calling the next Promises' `resolve()` or `reject()` sets its `state` value accordingly.

<div class="aside pitfall">
When using Promises, it's very easy to omit the error callback. But if you do, you'll never get <i>any</i> indication something went wrong. At the very least, the final Promise in your chain should have an error callback. See the section further down about swallowed errors for more info.
</div>

### Unexpected Errors Should Also Lead to Rejection

So far our error handling only accounts for known errors. It's possible an unhandled exception will happen, completely ruining everything. It's essential that the Promise implementation catch these exceptions and reject accordingly.

This means that `resolve()` should get wrapped in a try/catch block

{% highlight javascript %}
function resolve(newValue) {
  try {
    // ... as before
  } catch(e) {
    reject(e);
  }
}
{% endhighlight %}

It's also important to make sure the callbacks given to us by the caller don't throw unhandled exceptions. These callbacks are called in `handle()`, so we end up with

{% highlight javascript %}
function handle(deferred) {
  // ... as before

  var ret;
  try {
    ret = handlerCallback(value);
  } catch(e) {
    handler.reject(e);
    return;
  }

  handler.resolve(ret);
}
{% endhighlight %}

### Promises can Swallow Errors!

<div class="aside pitfall">
It's possible for a misunderstanding of Promises to lead to completely swallowed errors! This trips people up a lot
</div>

Consider this example

{% highlight javascript %}
function getSomeJson() {
  return new Promise(function(resolve, reject) {
    var badJson = "<div>uh oh, this is not JSON at all!</div>";
    resolve(badJson);
  });
}

getSomeJson().then(function(json) {
  var obj = JSON.parse(json);
  console.log(obj);
}, function(error) {
  console.log('uh oh', error);
});
{% endhighlight %}

<a class="fiddle" target="_blank" href="http://jsfiddle.net/city41/M7SRM/3/">fiddle</a>

What is going to happen here? Our callback inside `then()` is expecting some valid JSON. So it naively tries to parse it, which leads to an exception. But we have an error callback, so we're good, right?

**Nope.** *That error callback will not be invoked!* If you run this example via the above fiddle, you will get no output at all. No errors, no nothing. Pure *chilling* silence.

Why is this? Since the unhandled exception took place in our callback to `then()`, it is being caught inside of `handle()`. This causes `handle()` to reject the Promise that `then()` returned, not the Promise we are already responding to, as that Promise has already properly resolved.

<div class="aside">
Always remember, inside of <code>then()</code>'s callback, the Promise you are responding to has already resolved. The result of your callback will have no influence on this Promise
</div>

If you want to capture the above error, you need an error callback further downstream

{% highlight javascript %}
getSomeJson().then(function(json) {
  var obj = JSON.parse(json);
  console.log(obj);
}).then(null, function(error) {
  console.log("an error occured: ", error);
});
{% endhighlight %}

Now we will properly log the error.

<div class="aside pitfall">
In my experience, this is the biggest pitfall of Promises. Read onto the next section for a potentially better solution
</div>

### done() to the Rescue

Most (but not all) Promise libraries have a `done()` method. It's very similar to `then()`, except it avoids the above pitfalls of `then()`.

`done()` can be called whenever `then()` can. The key differences are it does not return a Promise, and any unhandled exception inside of `done()` is not captured by the Promise implementation. In other words, `done()` represents when the entire Promise chain has fully resolved. Our `getSomeJson()` example can be more robust using `done()`

{% highlight javascript %}
getSomeJson().done(function(json) {
  // when this throws, it won't be swallowed
  var obj = JSON.parse(json);
  console.log(obj);
});
{% endhighlight %}

`done()` also takes an error callback, `done(callback, errback)`, just like `then()` does, and since the entire Promise resolution is, well, done, you are assured of being informed of any errors that erupted.

<div class="aside pitfall">
  <code>done()</code> is not part of the Promise/A+ spec (at least not yet), so your Promise library of choice might not have it.
</div>


## Promise Resolution Needs to be Async
Early in the article we cheated a bit by using `setTimeout`. Once we fixed that hack, we've not used setTimeout since. But the truth is the Promise/A+ spec requires that Promise resolution happen asynchronously. Meeting this requirement is simple, we simply need to wrap most of `handle()`'s implementation inside of a `setTimeout` call

{% highlight javascript %}
function handle(handler) {
  if(state === 'pending') {
    deferred = handler;
    return;
  }
  setTimeout(function() {
    // ... as before
  }, 1);
}
{% endhighlight %}

This is all that is needed. In truth, real Promise libraries don't tend to use `setTimeout`. If the library is NodeJS oriented it will possibly use `process.nextTick`, for browsers it might use the new `setImmediate` or a [setImmediate shim](https://github.com/NobleJS/setImmediate) (so far only IE supports setImmediate), or perhaps an asynchronous library such as Kris Kowal's [asap](https://github.com/kriskowal/asap) (Kris Kowal also wrote [Q](https://github.com/kriskowal/q), a popular Promise library)

###Why Is This Async Requirement in the Spec?

It allows for consistency and reliable execution flow. Consider this contrived example

{% highlight javascript %}
var promise = doAnOperation();
invokeSomething();
promise.then(wrapItAllUp);
invokeSomethingElse();
{% endhighlight %}

What is the call flow here? Based on the naming you'd probably guess it is `invokeSomething()` -> `invokeSomethingElse()` -> `wrapItAllUp()`. But this all depends on if the promise resolves synchronously or asynchronously in our current implementation. If `doAnOperation()` works asynchronously, then that is the call flow. But if it works synchronously, then the call flow is actually `invokeSomething()` -> `wrapItAllUp()` -> `invokeSomethingElse()`, which is probably bad.

To get around this, Promises **always** resolve asynchronously, even if they don't have to. It reduces surprise and allows people to use Promises without having to take into consideration asynchronicity when reasoning about their code.

<div class="aside pitfall">
Promises always require at least one more iteration of the event loop to resolve. This is not necessarily true of the standard callback approach.
</div>

## Before We Wrap Up ... then/promise

There are many, full featured, Promise libraries out there. The [then](https://github.com/then) organization's [promise](https://github.com/then/promise) library takes a simpler approach. It is meant to be a simple implementation that meets the spec and nothing more. If you take a look at [their implementation](https://github.com/then/promise/blob/master/core.js), you should see it looks quite familiar. then/promise was the basis of the code for this article, we've *almost* built up the same Promise implementation. Thanks to Nathan Zadoks and Forbes Lindsay for their great library and work on JavaScript Promises. Forbes Lindsay is also the guy behind the [promisejs.org](http://promisejs.org) site mentioned at the start.

There are some differences in the real implementation and what is here in this article. That is because there are more details in the Promise/A+ spec that I have not addressed. I recommend [reading the spec](http://promises-aplus.github.io/promises-spec/), it is short and pretty straightforward. 

## Conclusion

If you made it this far, then thanks for reading! We've covered the core of Promises, which is the only thing the spec addresses. Most implementations offer much more functionality, such as `all()`, `spread()`, `race()`, `denodeify()` and much more. I recommend browsing the [API docs for Bluebird](https://github.com/petkaantonov/bluebird/blob/master/API.md) to see what all is possible with Promises. 

Once I came to understand how Promises worked and their caveats, I came to really like them. They have led to very clean and elegant code in my projects. There's so much more to talk about too, this article is just the beginning!

If you enjoyed this, you should [follow me on Twitter](http://twitter.com/cityfortyone) to find out when I write another guide like this.

## Further Reading

More great articles on Promises

* [promisejs.org](http://promisejs.org) -- great tutorial on Promises (already mentioned it a few times)
* [Q's Design Rationale](https://github.com/kriskowal/q/blob/v1/design/README.js) -- an article much like this one, but goes into even more detail. By Kris Kowal, creator of Q
* [Some debate over whether done() is a good thing](https://github.com/domenic/promises-unwrapping/issues/19)
* [Flattening Promise Chains](http://solutionoptimist.com/2013/12/27/javascript-promise-chains-2/) by Thomas Burleson. A nice article that goes into more advanced usage of Promises. If my article is the "what", then Thomas's is a nice look at the "why".

**Found a mistake?** if I made an error and you want to let me know, please [email me](mailto:matt.e.greer@gmail.com) or [file an issue](https://github.com/city41/blog/issues). Thanks!
