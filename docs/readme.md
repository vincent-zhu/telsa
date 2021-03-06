# Design Pattern

## Harel State Machine

This module is implemented using state machine pattern. However, it is not a flat state machine, but a Harel state machine.

In Harel machine, common properties and behaviors among concrete states are further abstracted to super states, transforming a flat state space into a hierarchical structure.

Super state is not a composition of parallel or concurrent state machines. It is merely an abstraction, providing a structured state space for better understanding and easier manipulation.

## Flat State Machine

Let's start with a flat machine. 

In classical state pattern, context and states are implemented by separate classes. All state-specific resources are maintained in state class. The context class simply forwards all external requests to state class.

Each state class has `enter` and `exit` methods for constructing and destructing state-specific resources/behaviors respectively. This is a powerful way to ensure the allocation and deallocation of resources, as well as starting and stoping actions, possibly asynchronous and concurrent, to happen at the right time and place.

The iconic method of state class is the `setState` method. It destructs the current state by calling `exit` method, constructs the new state, and calls its `enter` method.

```js
class State {
  constructor (ctx) {
    this.ctx = ctx
  }

  enter () {}
  exit () {}

  setState (NextState, ...args) {
    this.exit()
    this.ctx.state = new NextState(this.ctx, ...args)
    this.ctx.state.enter()
  }
}

class ConcreteState extends State {
  constructor (ctx, ...args) {
    super(ctx)
  }
}

class Context {
  constructor () {
    this.state = new ConcreteState(this)
    this.state.enter()
  }
}
```

The first parameter of the `setState` method is a state class constructor. 

In JavaScript, a class is modeled as a pair `(c, p)`, where `c` is the constructor function and `p` is a plain object. There are built-in, mutual references between `c` and `p`:

1. `c.prototype` is `p`
2. `p.constructor` is `c`


This can be verified in a node REPL:

```
> class A {}
undefined
> A.prototype.constructor === A
true
```

So either `c` or `p` can be used to identify a class. `c` is more convenient for it's a declared name.

It is possbile to eliminate the `enter` method and merge its logic into constructor for simplicity. 

Alternatively, calling `this.enter(...args)` inside the base state class constructor and all concrete state classes implement `enter` and `exit` methods but not constructor, is also a viable pattern.

This flat state machine pattern is sufficient for many real world use cases. And I'd like to explain a critical pitfall of this pattern here, though it is irrelevent to the hierarchical state pattern which is going to be discussed later and is used in this module.

Supposing the context class is an event emitter and its state change is observed by some external objects. It emits `entering`, `entered`, `exiting` and `exited` with corresponding state name. Obviously the best place to trigger the context's `emit` method is inside `setState`:

```js
setState (NextState, ...args) {
  this.ctx.emit('exiting', this.constructor.name)
  this.exit()
  this.ctx.emit('exited', this.constructor.name)

  let next = new NextState(this.ctx, ...args)
  this.ctx.state = next

  this.ctx.emit('entering', next.constructor.name)
  this.ctx.state.enter()
  this.ctx.emit('entered', next.constructor.name)
}
```

The danger occurs when `setState` is immediately called again inside next state's `enter` method. In this case, the `setState` and `enter` methods are nested in the calling stack. `entered` event will be emitted in a last-in, first-out manner. The observer will receive `entered` event in reversed order.

We have two solutions here.

One solution is to call `setState` with `process.nextTick()` in `enter`. In this way, an **intermediate** or **maybe** state is allowed in design. This solution is simple and intuitive. But the unnecessary asynchrony may rise problem in complex scenarios.

In the other solution, the **intermediate** or **maybe** states are strictly forbidden in design. The target state must be unambiguously determined before exiting a state. Conditional logics should be encapsulated by a function, rather than a separate state, if it's going to be used in many places. This is the **recommended** way. It avoids unnecessary asynchrony by `process.nextTick()`.

The importance of the second solution arises when many state machines, possibly organized into a list or tree, shares another larger context. Or we may say it's a composition of state machines.

`process.nextTick()` is usually used to defer or batch an composition-wise operation, such as reschedule certain jobs, when responding to an exteranl event and many state machines are transitioned simultaneously. It avoids the job being triggered each time a single state machine is transitioned. If `nextTick()` is allowed for a single state machine transition, it is difficult for the composition context to determine at what time all those `nextTick()` finishes and the composition-wise deferred or batch job can begin.

> Of course all `process.nextTick` can be tracked. But it is a non-trivial job. It requires a composition-wise counter, which is incremented before calling process.nextTick in a single state machine, and decremented after the nextTick-ed job is finished.

### Re-entry

`setState` can be invoked with the same state constructor.

Denoting a `ConcreteState1` class object as `s1`:

```js
s1.setState(ConcreteState1)
```

Such an invocation will trigger an `s1.exit` and `s1.enter`.

In some cases, this behavior is tremendously useful. It immediately abandons all current jobs and deallocates all resources, then re-creates a brand-new state object. If we want to retry something or restart something under certain circumstances, this one-line code will tear down and setup everything like a breeze, providing the `enter` and `exit` methods are properly implemented. 

It is also possible to hand over something between two state object of the same class, for example, retried times. They can be passed as the argument of `setState`. If the logic requires a job to be retried again and again until certain accumulated effect reaches a critical point, this pattern is probably the best way. 

If the re-entry behavior is not required and harmful if triggered unexpectedly, you can check and forbid it in the `setState` method.

### Initialization and Deinitialization

The code constructing the first state object inside context constructor looks natural and trivial.

```js
this.state = new ConcreteState(this)
this.state.enter()
```

But it has duplicate logic with that constructing the next state inside `setState`. If more logics are added to `setState`, such as triggering the event emission discussed above, the logic should also be implemented in context constructor.

Essentially, `setState` is a batch job. It destructs the previous state and constructs the next state. Initialization is just a special case where the previous state is `null`, and deinitialization is the opposite case where the next state is `null`.

At first thought, `setState` is a class method and a `null` object cannot have any method. However, this is not true in JavaScript since JavaScript is a late-binding language. 

The reference to the class method can be retrieved by it's prototype,  theorectically it can be applied to a `null`, something like:

```js
State.prototype.setState.apply(null, InitConcreteState)
```

In practice, context object is a required parameter for constructing the state object, so we replace `null` with the context object.


```js
// in state class
setState (NextState, ...args) {
  if (this instanceof State) {
    this.ctx.emit('exiting', this.constructor.name)
    this.exit()
    this.ctx.emit('exited', this.constructor.name)
  }

  if (NextState) {
    let ctx = this instanceof State ? this.ctx : this
    let next = new NextState(ctx, ...args)
    this.ctx.state = next

    this.ctx.emit('entering', next.constructor.name)
    this.ctx.state.enter()
    this.ctx.emit('entered', next.constructor.name)
  }
}

// In context class constructor
State.prototype.setState.apply(this, InitConcreteState)
```

Although looks weird, this code makes sense and truly implements the DRY principle. 

> IMHO, it also reveals that in JavaScript, nothing is `static` in the sense of that in Java. The implementation of `static` keyword in ES6 is probably a mistake, for it installs the `static` members onto constructor `c`, rather than the prototype object `p`.

In most cases, the deinitialization (passing `null` as `NextState`) is not used. Explicitly constructing a final/zombie state is far more practical, because a state object can accept all methods forwarded by context object. Either ignoring the action (eg. do nothing when `stream.write` is called) or returning an error gracefully, is much better than throwing an `TypeError`.

### Builder Pattern

If the context object is an event emitter and its state change is observed, and if the state object is constructed inside the context constructor, the observer will miss the first state's `entering` or `entered` event.

In node.js official document, it is recommended to emit such an event via `process.nextTick()`. As discussed above, this faked asynchrony is unnecessary. It may poses potential problem in state machine composition.

The buider pattern perfectly fits this requirement. It is also very popular in node.js, such as event emitters and streams.

The context class should provide an `enter` method, where the first state object is constructed. A factory method is also recommended. Then we can have a familiar code pattern for constructing a context object.

```js
let x = createContextObject(...)
  .on('entering', state => {...})
  .on('entered', state => {...})
  .on('exiting', state => {...})
  .on('exited', state => {...})
  .enter()
```

## Hierarchical State Machine

Now we discuss how to construct a hierarchical state machine in JavaScript.

A real benefit of Harel machine is that the `enter` and `exit` logic are distributed into several layered states. Besides the top-level base state, there are intermediate layers of abstract states. Each intermediate state, or super state, can hold a sub-context and have behaviors of its own.

Supposing we have the following hierarchical state space:

```
     S0 (base state)
    /  \
   S1  S2
  /  \
S11  S12
```

When transitting from S11 to S12, the `setState` should execute `S11.exit` and `S12.enter` sequentially. When transitting from S11 to S2, the sequence should be `S11.exit`, `S1.exit` and `S2.enter`.

Generally speaking, when transitting from concrete state Sx to Sy, there exists a common ancester super state, denoted by Sca:

1. from Sx (inclusive) to Sca (exclusive), execute `exit` method in bottom-up sequence
2. from Sca (exclusive) to Sy (inclusive), construct and execute `enter` method in top-down sequence

In implementation, there are two ways to construct such a hierarchy. It can be implemented using a tree data structure with mutual references as `parent` and `children[]` properties.

This pattern is versatile but very awkward. It has the following pros and cons.

1. [Pro] the up-and-down sequence of calling `exit` and `enter` is straight-forward and easy to implement.
2. [Pro] the sub-context are well separated in different object, so there is no name conflicts.
3. [Con] there is no inheritence between higher layer states and lower layer ones. It's painful to implement behaviors since functions and contexts are spreaded among several objects.

The first two pros can hardly balance the last con in most case.

Another way is using JavaScript's prototypal inheritance to construct the hierarchy. Two problems arise immediately.

First, all intermediate state's sub-context and the concrete state's state-specific things are merged into a single object, the object's properties must be well designed to avoid name conflict.

Second, the inheritance feature of `enter` and `exit` methods must NOT be used. Instead, the up-and-down sequence of `exit` and `enter` is implemented by a manual iteration along the prototypal inheritance chain and these two methods are bound and applied forcefully.

### State Class Constructor

In flat state machine, the first parameter of the constructor is the context object. This is OK if there's only single, global context for all states. 

In hierarchical state machine, however, each intermediate/super state has its own sub-context which may need to be preserved during transition. For example, when transitting from S11 to S12 state, the S1-specific context should be preserved. This requirement can be implemented in the following method.

First, the first parameter of base class constructor should be changed from the global context object to the previous state object. A state object has all contexts inside it, either global or intermediate-specific.

Second, considering the initialization discussed in flat state machine, when constructing the first state, there is no previous state object but the global context object is required. So the type of the first parameter of the base class constructor should be `State | Context`.

```js
class State {
  constructor (soc) {
    this.ctx = soc instanceof State ? soc.ctx : soc
  }
}
```

In the constructor of a intermediate state, if the first argument is an context object, or the first argument is an state object, but is NOT a descendant of this state, in either case, a new sub-context should be created. Otherwise, the old sub-context should be copied.

```js
class SuperState1 extends State {
  constructor (soc) {
    super(soc)
    if (soc instanceof SuperState1) {
      this.ss1 = soc.ss1
    } else {
      // constructing a new sub context
      this.ss1 = {
        ...
      }
    }
  }
}
```

Noticing that the `ss1` property is `SuperState1`-specific. Be careful to choose a unique name and avoid conflicts.

In JavaScript, constructing a sub-class object using `new` keyword always calls the constructors in the top-down sequence along the inheritance path. This cannot be modified. It is possible to hijack some constructor's behavior via `return`. But this is error prone and is not suitable here.

Keep in mind that the only purpose of the intermediate state's constructor, is to create a new sub-context, or to **take over** an old one. Nothing else should be done here. 

The `enter` logic should implemented in the `enter` method, which will be called in the up-and-down `exit` and `enter` sequence in `setState`. Such logic can NOT be merged into constructor. Otherwise, during S11->S12 transition, such S1-specific logic will be triggered, which is wrong and must be avoided.

### `setState`

`setState` is tricky and unusual in hierarchical state machine, but is not difficult. 

Modern JavaScript provides an `Object.getPrototypeOf()` method to replace the non-standard `__proto__` property for accessing the prototypal object of any given object.

`Function.prototype.apply()` is used to apply the `enter` or `exit` methods along inheritance chain onto `this` object. If an intermediate state has no `enter` or `exit` method of its own, it is skipped.

```js
setState (NextState, ...args) {
  let p, qs = []
  for (p = Object.getPrototypeOf(this);
    !(NextState.prototype instanceof p.constructor);
    p.hasOwnProperty('exit') && p.exit.apply(this),
    p = Object.getPrototypeOf(p));

  this.ctx.state = new NextState(this, ...args)

  for (let q = NextState.prototype;
    q !== p;
    q.hasOwnProperty('enter') && qs.unshift(q),
    q = Object.getPrototypeOf(q));

  qs.forEach(q => q.enter.apply(this.ctx.state))
}
``` 

### Initialization

Same with that in flat state machine, we can encapsulate the construction and destruction of state solely in `setState`. Here we have even more benefit than in flat machine case, for the construction logic is more complex.

```js
// in state class
setState (NextState, ...args) {
  let p
  let qs = []

  if (this instanceof State) {
    for (p = Object.getPrototypeOf(this);
      !(NextState.prototype instanceof p.constructor);
      p.hasOwnProperty('exit') && p.exit.apply(this),
      p = Object.getPrototypeOf(p));
  }

  if (NextState) {
    this.ctx.state = new NextState(this, ...args)
    for (let q = NextState.prototype;
      q !== p;
      q.hasOwnProperty('enter') && qs.unshift(q),
      q = Object.getPrototypeOf(q));

    qs.forEach(q => q.enter.apply(this.ctx.state))
  }
}

// in context constructor or enter
State.prototype.apply(this, InitConcreteState)
``` 

## Summary

I will give some complete examples in coming days.

In short, an easy-to-understand and easy-to-use state machine pattern is invaluable for software construction, especially in the world of asynchronous and concurrent programming.

JavaScript and node.js perfectly fits the need.

The pattern discussed above are heavily used in our products. They evolves in several generations and gradually evovles into a compact and concise pattern, fully unleashing the power of JavaScript. Similar pattern implemented in other languages requires far more boiler-plate codes. And certian tricks cannot be done at all.

This is the first half and basic part of programming JavaScript concurrently, either in Browser or in Node.js. The hierarchical state machine discussed here can handle any kind of intractable concurrent problem as long as it could be modeled as a single state machine.

Both event emitter and asynchronous functions with callback are just degenerate state machines. A thorough understanding of state machine is a must-have for JavaScript programmers.

The other half is how to compose several or large quantity of individual state machines into a single one, concurrently of course. I won't talk it in near future, but we do have powerful patterns and extensive practices. When I am quite sure on the composition definitions and corresponding code patterns, I will talk it for discussion.







