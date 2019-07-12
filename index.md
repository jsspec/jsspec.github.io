# JSSPEC
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

The ecosystem is not yet complete. The system does not yet have beforeEach, or afterEach hooks.

Eventually there will be an expectation library to complement `JSSpec`. For now, `chai.expect` (or any similar assertion library) works fine.

Some of the output may be wonky, please report it via the `@jsspec/format` repo.

Currently only runs against full path names - either absolute or relative to the current working directory.

## Command
```
jsspec [options] files
  --random, -R     Flag to run tests in random order. Default: true
                     - thus to turn this off, you have to pass this option twice
  --format, -f     Output formatter currently only 'documentation' (or 'd' for short)
  --require, -r    list of modules to require before executing the tests.
                     - since this takes a list, you have to break out of it before
                       listing your test files
  --files, --      flag to stop processing options and start listing test files.
                     - not needed in most cases
```

## Use
The standard bits:

`describe` and `context` do the same thing:
`context(description, options, block)` or<br>
`context(description, block)`

Options is an object and currently only accepts `timeout`. A `timeout` of zero means run forever if it has to. The `timeout` is set for each child example block.

`it(description, options, block)` or<br>
`it(description, block)`

Again, same thing for options. `timeout` set for this block only. blocks may be asynchronous. Tests are run synchronously, waiting for complete resolution of asynchronous code. In the future, files will be run in parallel, with tests internally running in series (synchronously).

### `set(keyName, settting)`
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
## `before([description, [options]], block)`
Define a before block to run before any `it` example block in this context. If no examples exit in the context, the `block` provided will not be run.

The `block` has access to lazy values (variables defined by `set` and `subject`), but (just like an example block) the values are reset at the end of the block execution. `before` blocks should be used to set external contexts - such as database entries, or file content - rather than setting variables to be used in the test code.

## `after([description, [options]], block)`
Define an after block to run after all `it` example block in this context. If no examples exit in the context, the `block` provided will not be run.

The `block` has access to lazy values (variables defined by `set` and `subject`), but (just like an example block) the values are reset at the end of the block execution. `after` blocks should be used to tear down external contexts - such as database entries, or file content - rather than doing anything with the test variables.

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
