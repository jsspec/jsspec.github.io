# JSSpec
Contextualised spec runner for JavaScript in the flavour of `RSpec` (Ruby Spec runner).

## Why
This spec runner was written to enable easy context changing when writing specs for JavaScript. I've used Mocha and Jest in the past, and for me, the biggest annoyance was having to either write build methods, or repeated code when writing tests - just to confirm functionality for a single input change. `JSSpec` solves that issue by allowing you to change only the parameter that is required to change to set a new context.

So instead of having to write:
```javascript
describe('MyClass.run', () => {
  context('when option.random is set', () => {
    it('runs in random order', () => {
      const thing = new MyClass({random: true, otherSetting: false});
      expect(thing.run()).to.eql('random');
    });
  });
  context('when option.random is not set', () => {
    it('does not run in random order', () => {
      const thing = new MyClass({random: false, otherSetting: false});
      expect(thing.run()).not.to.eql('random');
    });
  });
});
```

As can be done in `RSpec`, we can now make context changes when using `JSSpec`:
```javascript
describe('MyClass.run', () => {
  subject(() => myClass.run());

  set('myClass', () => new MyClass(options));
  set('options', () => ({ random, otherSetting: false }));

  context('when option.random is set', () => {
    set('random', true);

    it('runs in random order', () => {
      expect(subject).to.eql('random');
    });
  });

  context('when option.random is not set', () => {
    set('random', false);

    it('does not run in random order', () => {
      expect(subject).not.to.eql('random');
    });
  });
});
```

In this case, we define a subject when we open a describe block and prepare all of the values that are required to test it. Then, for each test context, we set only the component that relates to that context. Everything else is unchanged, so we don't have to mention it AND the state of every `set` object is reverted at the end of each example block.

This improves readability and maintainability of the test code. Which in turn improves the likelihood of your tests doing the right thing.

## Limitations
To provide this capability, the `set` and `subject` variables are defined on the global object. This means you run the risk of breaking the world if you choose the wrong variable names. But I'm sure this will just help you choose better names. ;) In reality, `JSSpec` doesn't allow you to overwrite an existing global variable. This doesn't save you if your application code sets some object on global. Of course, assigning to global in production code is a pretty bad idea anyway.

Eventually there will be an expectation library to complement `JSSpec`. For now, `chai.expect` (or any similar assertion library) works fine.

Some of the output may be wonky, please report it via the `@jsspec/format` repo.

## Command
```
jsspec [options] files
  --random, -R     Flag to run tests in random order. Default: true
                     - thus to turn this off, you have to pass this option twice
  --format, -f     Output formatter:
                     'documentation' ('d' for short) or
                     'dot' ('o' for short)
  --require, -r    list of modules to require before executing the tests.
                     - since this takes a list, you have to break out of it before
                       listing your test files
  --files, --      flag to stop processing options and start listing test files.
                     - not needed in most cases
```

Files may be listed with targeted line, or indexed examples to run. Note that this should only be done using the summary output from a failed run as, unlike RSpec, no work is done to determine what context or example a line sits within.

Example command lines:
```bash
# Run against all `.spec.js` files in the `spec/` directory, ensuring that `expect` from `chai` is available as a global variable
# note that this requires an escape of some kind before listing the files to run, since it accepts multiple files itself
jsspec -r chai/register-expect - spec/*.spec.js

# Run specs, in order (not random)
# note that the need to toggle random on then off is deliberate
# your spec shouldn't need to run in order to pass(*)
jsspec --require chai/register-expect -RR spec/*.spec.js

# Use the dot output format
jsspec -fo spec/*.spec.js

# Run an example from a specific line (as reported in a previously failed run)
jsspec /my/working/directory/spec/this_failed.spec.js:17 # All tests should pass but this one didn't

# Run an example, that may have come from a loop, or a shared context/example group (as reported in a previously failed run)
jsspec /my/working/directory/spec/this_failed.spec.js[1:3:4:1] # Turns out this failed too
```

(*) Sometimes tests need to do things in order, but you should specify this in the context options instead. JSSpec's own specs do this in a few important places.

## Use
The standard bits:

`describe` and `context` do the same thing:
`context(description, options, block)` or<br>
`context(description, block)`

Options is an object and accepts `timeout` and `random`. A `timeout` of zero means run forever if it has to. The `timeout` is set for each child example block. `random` can be set to `true` to randomise the order of execution of child blocks, or `false` to ensure they run in order. Note that child contexts are always built before their sibling examples are run.

`it(description, options, block)` or<br>
`it(description, block)`

Options only accpets a `timeout` to set for this block. blocks may be asynchronous. Tests are run synchronously, waiting for complete resolution of asynchronous code. In the future, files will be run in parallel, with tests internally running in series (synchronously).

### `set(keyName, setting)`
`set` is the lazy evaluator. The keyName must be a string, and the setting can be a value, or a function who's result will (eventually) be used as the value.
eg:
```javascript
/*
  test = 501
*/
set('test', 501)

/*
  other will resolve to the value of `test` when `other` 
  is first invoked in an example block
*/
set('other', () => test)

/*
  to set the value to a function, it has to be wrapped in another function, which is fine.
  The value of `other` will be what ever `other` is when `fn` is called.
*/
set(`fn`, () => ((...args) => doSomethingWith(other, args)))
```

The power of the lazy evaluation comes in to play when sub elements (like `test` is for `other` here) are re-set in a child context.

## `subject([optionalName], setting)`
`subject` is slightly special, it can be named just like `set`, but doesn't have to be. You can refer directly to `subject` in an example block. It's often handy to name a subject though, so you can change the subject further down in the same chain, but still refer to the original value.
```javascript
subject(() => new MyClass(options));

it('is a thing', () => {
  expect(subject).to.be.an.instanceOf(MyClass);
});
```
## Hooks
### `before([description, [options]], block)`
Define a before block to run before any `it` example block in this context. This includes nested examples. If no examples exit in the context, the `block` provided will not be run.

The `block` has access to lazy values (variables defined by `set` and `subject`), but (just like an example block) the values are reset at the end of the block execution. `before` blocks should be used to set external contexts - such as database entries, or file content - rather than setting variables to be used in the test code.

### `after([description, [options]], block)`
Define an after block to run after all `it` example block in this context. This includes nested examples. If no examples exit in the context, the `block` provided will not be run.

The `block` has access to lazy values (variables defined by `set` and `subject`), but (just like an example block) the values are reset at the end of the block execution. `after` blocks should be used to tear down external contexts - such as database entries, or file content - rather than doing anything with the test variables.

### `beforeEach([description, [options]], block)`
As per before hook, except that it runs before _every_ `it` example block in this context. This included nested examples.

### `afterEach([description, [options]], block)`
As per before hook, except that it runs before _every_ `it` example block in this context. This included nested examples.

Order of execution:
```javascript
describe('order', () => {
  before(() => console.log('before 1'));
  beforeEach(() => console.log('before each 1'));
  afterEach(() => console.log('after each 1'));
  after(() => console.log('after 1'));

  it('executes', () => console.log('block 1'));

  context('nest', () => {
    before(() => console.log('before 2 (nested)'));
    beforeEach(() => console.log('before each 2 (nested)'));
    afterEach(() => console.log('after each 2 (nested)'));
    after(() => console.log('after 2 (nested)'));

    it('executes 1', () => console.log('nested block 1'));
    it('executes 2', () => console.log('nested block 2'));
  });

  context('nest 2', () => {
    it('executes 3', () => console.log('nested block 3 (second nest)'));
  });
});
```

Will result in the following output (assuming non-random ordering)
```
  order
before 1
before each 1
block 1
after each 1
    ✔︎  executes
    nest
before 2 (nested)
before each 1
before each 2 (nested)
nested block 1
after each 2 (nested)
after each 1
      ✔︎  executes 1
before each 1
before each 2 (nested)
nested block 2
after each 2 (nested)
after each 1
      ✔︎  executes 2
after 2 (nested)
    nest
before each 1
nested block 3 (second nest)
after each 1
      ✔︎  executes 3
after 1
```
## Shared examples and contexts
Shared examples and shared contexts have similar functionally, but react with the context they are invoked from differently. Both can be provided with a method that accepts arguments, and arguments can be passed when the context/examples are called upon:

`sharedExamples(name, block)` is called in with
`itBehavesLike(name, ...arguments)` where `arguments` could be empty.

`sharedContext(name, block)` is called in with `includeContext(name, ...arguments)` where again, `arguments` could be empty.

Note that `...arguments` indicates a list of arguments, not an array, think of `function.call` vs `function.apply`.


### Shared examples
If you have multiple components that require the same tests, you can avoid repeating your test code using `sharedExamples`.

```javascript
sharedExamples('an iterator', () => {
  set('target', () => subject[Symbol.iterator]());
  set('values', undefined);

  it('responds to next', () => {
    expect(target).to.respondTo('next');
  });

  it('next has `done`', () => {
    expect(target.next()).to.include.key('done');
  });

  context('with two iterable values', () => {
    set('values', () => twoValues);

    it('gets values back twice, then nothing', () => {
      expect(target.next()).to.include.key('value');
      expect(target.next()).to.include.key('value');

      let final = target.next();
      expect(final.value).to.be.undefined;
      expect(final.done).to.be.true;
    });
  });
});

describe('Map', () => {
  subject(() => new Map(values));
  set('twoValues', [[1, 2], [3, 4]]);

  itBehavesLike('an iterator');
});

describe('Set', () => {
  subject(() => new Set(values));
  set('twoValues', [1, 2]);

  itBehavesLike('an iterator');
});

describe('String', () => {
  subject(() => new String(values));
  set('twoValues', 'ab');

  itBehavesLike('an iterator');
});
```

In effect, the `sharedExamples` is invoked as a child context with the `itBehavesLike` call. The examples being called have access to the contexts lazy evaluators, and will trigger `beforeEach` and `afterEach` calls for every Example (`it`) call they contain. `before` and `after` blocks will also be triggered under their normal conditions as though the `sharedExamples` being executed were defined in a child context.

The context that is invoked will have the name `it behaves like [sharedExamples name]`. eg. `it behaves like an iterator` above. You should name your `sharedExamples` accordingly.

### Shared context
A `sharedContext` is similar to a shared example, except that the provided block is executed as though it was part of the context it is being invoked from. No child context is created. Any values `set` in the shared context block will be available to the block where `includeContext` is called.

Order of operation is important in this case. For example:

```javascript
sharedContext('changes things', () => {
  set('value', 1);

  it('has the value set in the sharedContext?', () => expect(value).to.eql(1)); // Fails once, passes once
});

context('shared after', () => {
  set('value', 0);

  includeContext('changes things');

  it('has the value set here?', () => expect(value).to.eql(0)); // This will FAIL
});

context('shared before', () => {
  includeContext('changes things');

  set('value', 0);

  it('has the value set here?', () => expect(value).to.eql(0)); // This will PASS
});
```

As per the comments, the example in the first context will fail because there is a second call to `set` for `value` which isn't accounted for. This is correct operation, but something you need to be careful of. The same is true when running examples imported from the shared context. The example in the shared context will fail when called from the second context block.


## eslint
There is an eslint plugin available which removes the 'is not defined' errors for variables defined in `set` and `subject` statements. Install with:

`npm i eslint-plugin-jsspec`

Add the following to your `.eslintrc.json` file in your spec directory:
```json
  "plugins": ["jsspec"],
  "env": {
    "jsspec/jsspec": true
  },
```
