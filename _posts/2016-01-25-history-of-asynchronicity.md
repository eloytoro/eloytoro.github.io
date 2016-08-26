---
layout: post
title:  "History of Asyncrhononicity"
date:   2016-01-25 19:45:31 +0530
author: "Eloy Toro"
---

We all know the asynchronous programming NodeJS brings to the table as it goes as follows:

- Everything that delegates a task to another service uses a callback
- Everything that takes time to compute uses a callback
- Everything that waits for something else uses a callback

It’s so obvious, use callbacks for literally **everything**.

### The Everything Pattern

Now that we’re clear on this lets go ahead and program something that could be both asynchronous or not.

Let’s begin with the synchronous version.

```javascript
function operationSync(a, b) {
  return a + b;
}
```

Done. Super easy, lets go ahead and do it asynchronous, this way we can fully take advantages of asynchronous programming

```javascript
function operation(a, b, callback) {
  var result = operationSync(a, b);
  callback(null, result)
}
```

Isn’t it beautiful?

### Callbacks run this town

Fun part over, if you’ve been using NodeJS for a while you’ve probably seen this before, why this happens makes no sense to me, sad part is that some people find it useful to have a callback’ish version of a synchronous code just for the sake of callback’ing **everything**.

But this is only a side-effect of the spreading disease that is the callback phenomena.

In paper I’m sure it painted a pretty picture, using first-class functions to handle events that are scheduled to happen in the future… but it doesn’t take it too far to realize that this brings some issues, in fact you could stop walking after _5 steps_.

```javascript
step1(function () {
  step2(function () {
    step3(function () {
      step4(function () {
        step5(function () {
          // now we're talking!
        }):
      });
    });
  });
});
```

Meet the **Pyramid of Doom**, a place where code goes beyond the cursor and you wish everything had 44 less indent tabs. And don’t get me started on **error handling**

### Async

> A system that’s commonly used in a particular manner might as well promote that behaviour

The worst thing about callback’ing about callbacks isn’t how bad it looks or how much it convolutes your code, it’s the fact that most of the time it’s the easiest way of handling asynchronous flow and it spreaded across your code before you knew how to prevent it.

Thankfully there had been many ways to patch this, meet the [async](https://github.com/caolan/async) module, something you wish you had used before or you already use for anything that requires stacking any amount of asynchronous calls.

```javascript
// does the same as above but with much less code
async.series([
  step1,
  step2,
  step3,
  step4,
  step5
], function () {
  // finished!
});
```

That’s great! But truthfully it’s only a patch to the whole callback issue. Used propperly async can reduce your code to the seemingly bare minimum, but it’s not enough.

### Promises

You’ve probably heard of these, if you haven’t I suggest you [domenic’s article on promises](https://gist.github.com/domenic/3889970), it brings a lot of insight to the concept of promises, which sometimes are very mind-bending even though they seem pretty simple.

So basically promises resemble _asynchronous processes that can either be fulfilled or rejected in the foreseeable future_.

That seems like a callback to me, doesnt it?

```javascript
step1()
  .then(step2)
  .then(step3)
  .then(step4)
  .then(step5)
  .then(/* we're done here */)
```

Promises have beautiful inner workings, in a way they’re could **pipe** into one another, they are immutable, they _solve_ the issue.

### 'Next' Level Stragegies

ES6 is a game changer for NodeJS, Promises are now a built-in javascript type and paired up with [generators](https://ponyfoo.com/articles/es6-generators-in-depth) spawn a new behaviour dominates the scene. But first, let’s recapitulate on the basics of generators.

Syntax:

```javascript
var generator = function* () {
  yield 1;
  yield 2;
  yield 3;
}

var instance = generator();

console.log(instance.next()) // { value: 1, done: false }
console.log(instance.next()) // { value: 2, done: false }
console.log(instance.next()) // { value: 3, done: false }
console.log(instance.next()) // { value: undefined, done: true }
```

Remember you can pass arguments into the generator by calling `.next()` with a value.
This way the `yield` expression stops evaluation until the generator is resumed by calling `.next(value)` in which case code executiong resumes and the `yield` expression evaluates to `value`.

```javascript
var generator = function* () {
  return (yield) + (yield);
}

var instance = generator();
instance.next(); // { value: undefined, done: false }
instance.next(1); // { value: undefined, done: false }
instance.next(2); // { value: 3, done: true }
```

### The Compose Pattern

Meet [co](https://github.com/tj/co), the module that brings the two together and changed the asynchronous flow. It relies on the idea that:

* Generators can stop execution and be resumed at any given time (now, later or maybe never)
* Yielding promises allows a *handler* to wait for them to resolve and result the generator’s executiong with the promise’s resolved value passed through `.next(result)`
* You could yield arrays, objects, functions, generators or basically anything that could return or contain promises to control _when_ they start yielding.

This means that any asynchronous task can be represented by a promise, and if instead of _taking_ callbacks as parameters async methods would _return_ promises you could use as follows:

```javascript
var promsie = co(function* () {
  yield step1();
  yield step2();
  yield step3();
  yield step4();
  yield step5();
});
```

Another example, suppose that `getRecentTweets()` returns a promise that resolves to an array of tweets as soon as they arrive from the twitter server.

```javascript
co(function* () {
  var tweets = yield getRecentTweets();
});
```

Javascript tells us how it’s going to evaluate each of the expressions

* Splits the whole line into the right side and left side of the assignment
* In order to evaluate `yield ...` we must first clear what’s being yielded
* `getRecentTweets()` is called and it returns a promise, which current status is pending
* It yields this promise to the handler, it doesn’t result into any evaluation yet, because the generator execution stops completely and it’s state is kept in memory.
* The handler waits for the promise to resolve and calls `.next(result)` into the generator when it does (in this case an array of tweets)
* The `yield` expression evaluates to the result passed onto the generator.
* The assignment sets `tweets` to be equal to the previous result.

Oh and another thing, the `co(/*...*/)` call evaluates to a Promise, so you could wait for it to finish, or even better, yield composed generators inside generators!

Remember when I said promises could be `rejected`? Turns out that when they do `co` throws into the generator, so you could do `try/catch` statements on top of async processes, sweet right?

```javascript
try {
  var tweets = yield getRecentTweets();
} catch (err) {
  console.log('Server timeout!')
}
```

Best thing about this is that you can decide how internally you want to catch errors by specific handlers, or let the more general error handlers come up with a solution.

### Control Flow

Say you want to do parallel async processes, or perhaps some of your tasks depend upon the completion of others.

The new co syntax helps you do rather easily using ES6 destructuring (currently behind the `--harmony_destructuring` flag).

```javascript
// both Promises have to resolve before continuing
var {tweets, user} = yield {
  tweets: getRecentTweets(),
  user: getUser('eloytoro')
};

// Now that both async tasks finished we can do the next task
yield favouriteUserTweets(user, tweets);
```

If you're a NodeJS backend developer I suggest you to take a look at [koa](http://koajs.com/), a router that wraps everything using co and allows you to handle async web development using the compose pattern.

### What’s being yielded at the moment

There’s a new javascript specification coming up (ES7) which takes the compose pattern to the native level with the `async/await` syntax.
It’s basically syntax sugar for the code above: the `async` keyword, when prepended to a function declaration, allows the body to `await` asynchronous processes and wraps the function’s returned value within a promise.

```javascript
async () => {
  await step1();
  await step2();
  await step3();
  await step4();
  await step5();
}
```

And a peak into the future of what current avant garde ES6 code will look like after the new specification is implemented into NodeJS

```javascript
// is this even javascript?
var userTweets = async (username) => {
  const {tweets, user} = await {
    tweets: getRecentTweets(),
    user: getUser(username)
  };

  await favouriteUserTweets(user, tweets);
};

userTweets('eloytoro')
  .then(tweets => console.log(tweets))
  // you could also catch anything thrown by the function or the async methods
  .catch(err => console.error(err))
```

But don’t miss out on anything! Javascript is mutating really fast towards asynchronous process handling, and it’s becoming a [huge trend](http://venturebeat.com/2015/08/19/here-are-the-top-10-programming-languages-used-on-github/) in modern development, so stay put for more news!
