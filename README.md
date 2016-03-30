# JavaScript MythBusters

> A list of JavaScript tips to avoid load performance from a high level point of view.

- [JavaScript MythBusters](#javascript-mythbusters)
  * [V8 Tips](#v8-tips)
    + [Hidden Class in Constructors](#hidden-class-in-constructors)
    + [Whole than Float Number](#whole-than-float-number)
    + [Object/Arrays initialization inline](#object-arrays-initialization-inline)
    + [Monomorphic over Polymorphic types](#monomorphic-over-polymorphic-types)
    + [More safe (and fast) code with 'use strict'.](#more-safe--and-fast--code-with--use-strict-)
    + [Freeing memory by assigning 'null'.](#freeing-memory-by-assigning--null-)
  * [General Tips](#general-tips)
    + [Workflow](#workflow)
      - [Lookup table for result matching](#lookup-table-for-result-matching)
      - [Memoizaton: Cache sucesive calls.](#memoizaton--cache-sucesive-calls)
      - [Use Math native methods](#use-math-native-methods)
      - [Variable access over array/object access.](#variable-access-over-array-object-access)
      - [Setup the correct variables scope.](#setup-the-correct-variables-scope)
      - [Control flow based in boolean condition](#control-flow-based-in-boolean-condition)
    + [String](#string)
      - [Instead of String.concat, use '+='.](#instead-of-stringconcat--use-----)
    + [RegexEp](#regexep)
      - [Use RegExp in cases with sense.](#use-regexp-in-cases-with-sense)
      - [Focus RegExp on failing faster.](#focus-regexp-on-failing-faster)
      - [Use the correct RegExp method.](#use-the-correct-regexp-method)
    + [Error](#error)
      - [Avoid try/catch](#avoid-try-catch)
    + [Function](#function)
      - [Avoid .bind, is slower.](#avoid-bind--is-slower)
    + [Array](#array)
      - [Array.pop() better than Array.shift()](#arraypop---better-than-arrayshift--)
- [License](#license)

---

## V8 Tips

### Hidden Class in Constructors

JavaScript has limited compile-time type information: types can be changed at runtime, so it's natural to expect that it is expensive to reason about JS types at compile time.

```js
function Point(x, y) {
  this.x = x;
  this.y = y;
}

var p1 = new Point(11, 22); // => hidden Point class created
var p2 = new Point(33, 44); // => hidden Point class shared with p1
// At this point, p1 and p2 have a shared hidden class

p2.z = 55; // => another hidden class (Point') created, p1 !== p2
```

This might lead you to question how JavaScript performance could ever get anywhere close to C++. However, V8 has hidden types created internally for objects at runtime; objects with the same hidden class can then use the same optimized generated code.

Keep in mind:

- Initialize all object members in constructor functions (so the instances don't change type later).
- Always initialize object members in the same order.

### Whole than Float Number

V8 uses tagging to represent values efficiently when types can change.

V8 infers from the values that you use what number type you are dealing with. Once V8 has made this inference, it uses tagging to represent values efficiently, because these types can change dynamically. However, there is sometimes a cost to changing these type tags, so it's best to use number types consistently, and in general it is most optimal to use 31-bit signed integers where appropriate.

```
var i = 42;  // this is a 31-bit signed integer (whole).
var j = 4.2; // this is a double-precision floating point number (float).
```

If you actually don't need the extra of information, avoid it.


### Object/Arrays initialization inline

```js
var a = new Array();
a[0] = 77;   // Allocates
a[1] = 88;
a[2] = 0.5;  // Allocates, converts
a[3] = true; // Allocates, converts
```

is less efficient than:

```js
var a = [77, 88, 0.5, true];
```

Because in the first example the individual assignments are performed one after another, and the assignment of `a[2]` causes the Array to be converted to an Array of unboxed doubles, but then the assignment of `a[3]` causes it to be re-converted back to an Array that can contain any values (`Number` or `Object`). In the second case, the compiler knows the types of all of the elements in the literal, and the hidden class can be determined up front.

Another example:

```js
a = new Array();
for (var b = 0; b < 10; b++) {
  a[0] |= b;  // Oh no!
}
//vs.
a = new Array();
a[0] = 0;
for (var b = 0; b < 10; b++) {
  a[0] |= b;  // Much better! 2x faster.
}
```

Any name used as property name that is not a string is stringified via `.toString()`, even numbers, so 1 becomes "1".

Also, `Arrays` in JavaScript are just `Objects` with magic length property.


### Monomorphic over Polymorphic types

Operations are monomorphic if the hidden classes of inputs are always the same - otherwise they are polymorphic, meaning some of the arguments can change type across different calls to the operation. For example, the second add() call in this example causes polymorphism:

```
function add(x, y) {
  return x + y;
}

add(1, 2);      // + in add is monomorphic
add("a", "b");  // + in add becomes polymorphic
```

Monomorphic types is more predictable the Compiler and more easy to generate good code.

### More safe (and fast) code with 'use strict'.

From [MDN use strict](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode) section:

*Strict mode makes several changes to normal JavaScript semantics. First, strict mode eliminates some JavaScript silent errors by changing them to throw errors. Second, strict mode fixes mistakes that make it difficult for JavaScript engines to perform optimizations: strict mode code can sometimes be made to run faster than identical code that's not strict mode. Third, strict mode prohibits some syntax likely to be defined in future versions of ECMAScript.*

See [all use strict rules](https://msdn.microsoft.com/library/br230269%28v=vs.94%29.aspx).

In `node`, you can use the flag `--use_strict`, but is not recommended if you have dependencies untested to work with this flag.

### Freeing memory by assigning 'null'.

If you want to delete a property of a object uses

```
var foo = { bar: 'hello world' };
foo.bar = null;
```

Instead of:

```
var foo = { bar: 'hello world' };
delete foo.bar;
```

If you want do delete the object enterely, then simply `foo = null`.


Garbage Collector is interested in objects that are not referenced by any other object.

On the other hand, JavaScript engines can detect such "hot" objects and attempt to optimize them. This is easier if the object’s structure doesn’t heavily change over its lifetime and delete can trigger such changes.

This also is applicable to `Arrays`

[Writing Fast, Memory-Efficient JavaScript @ smashingmagazine](http://www.smashingmagazine.com/2012/11/writing-fast-memory-efficient-javascript/#de-referencing-misconceptions).

## General Tips

### Workflow

#### Lookup table for result matching

When we optimizing `if/else`, the goal is always to minimize the number of conditions to evaluate before taking the correct path. The easiest optimization is therefore to ensure that the most common conditions are first:

```js
if (value < 5) {
  // most frequency case
} else if (value > 5 && value < 10) {
  // second frequency case
} else {
 /// in other case
}
```

`switch` is more or less the same (maybe more semantic and more structured code). But the point is makes the reduce the path size.

Better use a direct match approach based in object/array index:

```
var lookupTable = {
  1: 'is greater',
 -1: 'is less',
  0: 'is equal'
};

var myValue = 5;

lookupTable[compare(4, myValue)];
```

Keep in mind that using an `Object` you can use `undefined` or `''` as key.

Also be careful about choose betweetn `Object` or `Array` for this purpose:

- If you need a incremental index, use `Array`.
- In other case use `Object`.

Under `ES2015`, consider use `Map` or `Set` combined with `Symbol`.

Althought is more readable, not in all scenarios use a lookup table is better: The cost of create the lookup table could be higher tan use a set of if/else statements. So, it depends about your code running time:

- If you are going to be running short time (maybe just one execution, like building a CLI tool) use if/else.
- In other case use lookup table solution.

#### Memoizaton: Cache sucesive calls.

Just calculate the value of something once, and reuse the value avoiding the cost of recalculate the same value.

If you need to control of a set of tiny values, you can use an object:

```js
var cache = {};

var cityOne = 'Murcia';
var cityTwo = 'Madrid';
var routeName = cityOne + '-' +  cityTwo;

if (!cache[routeName])
  cache[routeName] = getDistance(cityOne, cityTwo);

return cache[routeName];
```

If you actually don't know how your cache could grow in memory, be careful. You need a little more sophisticated way as [memory-cache](https://www.npmjs.com/package/memory-cache) or [lru-cache](https://www.npmjs.com/package/lru-cache) to control it.

#### Use Math native methods

No matter how optimize your JavaScript code is, it will never be faster tahn the native methods provided by the JavaScript Engine.

The reason for this is simple: the native parts of JavaScript are all writen in a lower-level language such as C++. That means these methods are compiled down to machine code as part of the browser and therefore don't have the same limitations as your code.

The `Math` object contains properties and metods designed to make mathematical operations easier. There are several matematical constans available:

| Constant      | Meaning                                              |
|---------------|------------------------------------------------------|
| `Math.E`      | The value of `E`, the base of the natural logarithms |
| `Math.LN10`   | The natural logarithm of 10                          |
| `Math.LN2`    | The natural logarithm of 2                           |
| `Math.LOG2E`  | The base-2 logarithm of `E`                          |
| `Math.Log10E` | The base-10 logarithm of `E`                         |
| `Math.PI`     | The value of π                                       |
| `Math.SQRT1_2`| The square root of 1/2                               |
| `Math.SQRT2`  | The square root of 2                                 |

Each of these values is precalculated, so there is no need for you to calculate them yourself. There are also method to handle mathematical calculations:

| Method                  | Meaning                   |
|-------------------------|---------------------------|
| `Math.abs(num)`         | The absolute value of num |
| `Math.exp(num)`         | Math.e**num               |
| `Math.log(num)`         | The logarithm of num      |
| `Math.pow(num, power)`  | num**power                |
| `Math.sqrt(num)`        | The square root of num    |
| `Math.acos(x)`          | The arc cosine of x       |


#### Variable access over array/object access.

If you need to check the result more than once.

When it comes to Javascript data, there’s pretty much four ways to access it: literal values, variables, object properties, and array items. When thinking about optimization, literal values and variables perform about the same, and are significantly faster than object properties and array items.

So whenever you reference an object property or array item multiple times, you can get a performance boost by defining a variable. (This applies to both reading and writing data).

```js
var name = myObject.name;
var value = array[3];
```

Consider this loop:

```js
// minimizing property lookups
for (var i=0, len=items.length; i < len; i++)
  process(items[i])
```

```js
var j=0
var count = items.length

while (j < count)
  process(items[j++]])
```

Maybe you can move out the first iteration of the loop if you know that `num > 0`:

```js
var k=0
var num = items.length
do
  process(items[k++]);
while (k < num)
```

Depending on the length of the array, you can save around 25% off the total loop execution time in most engines.

Another good low level tip is decrement the value:

```
var j = items.length

while (j--)
  process(items[j]])
```

Now you are using one variable for the control and based in a boolean value. Excellent!

By reversing loops and minimizing property lookups, you can see execution times that are up to 50%–60% faster than the original.

> Decreasing the work done per iteration is most effective when the loop has a complexity of O(n). When the loop is more complex than O(n), it is advisable to focus your attention on decreasing the number of iterations.

#### Setup the correct variables scope.

If you declare a global variable but you only use it in a little part of your code, then you are keeping in memory the variable all time but.

When a variable is referenced, Javascript hunts it down by looping through the different members of the scope chain. This scope chain is the set of variables available within the current scope, and across all major browsers it has at least two items: a set of local variables and a set of global variables.

Simply put, the deeper the engine has to dig into this scope chain, the longer the operation will take. It first goes through the local variables starting with this, the function arguments, then any locally defined variables, and afterwards iterates through any global variables.

**Continue reading: ES2015 tips**

Notes that with ES2015, you can use `const` and `let` as well.

`const` means that the variable can’t be reassigned. (Not to be confused with immutable values. Unlike true immutable datatypes such as those produced by Immutable.js and Mori, a `const` object can have properties mutated.):

```js
const objt = { foo: 'bar' };

objt = {};
console.log(objt); // => = { foo: 'bar' }

objt.foo = 'WAT';
console.log(objt); // => = { foo: 'WAT' }

objt.wat = 'the hell';
console.log(objt); // => = { foo: 'WAT', wat: 'the hell' }

const objt = {}
TypeError: Identifier 'objt' has already been declared
```

`let` means that the variable may be reassigned, such as a counter in a loop, or a value swap in an algorithm. It also signals that the variable will be used only in the block it’s defined in, which is not always the entire containing function.

`var` is now the weakest signal available when you define a variable in JavaScript. The variable may or may not be reassigned, and the variable may or may not be used for an entire function, or just for the purpose of a block or loop.

You need to be reassign? then `let`, else `const`.

Another important approach: If you need to deallocate a value by unsetting it, you may consider `let` over `const`.

#### Control flow based in boolean condition

Always as possible, compare with a simple boolean (for example, in loops conditions): It's a more simple (and consecuentially faster):

```js
// slow
// two boolean conditions = 0 and > 0
array.indexOf(i) >= 0

// fast
// just one boolean condition
array.indexOf(i) !== 0
```

### String

#### Instead of String.concat, use '+='.

It's no clear why this happens. We suppose that `String.concat` is part of a Class and need to wrap more things.

In general terms, `+=` is faster, but depends of your JavaScript Engine and version.

Check [test#1](https://jsperf.com/concat-vs-plus-vs-join) and [test#2](https://jsperf.com/string-concat-fast/17) benchmarks.

### RegexEp

#### Use RegExp in cases with sense.

When use with care, regexes are veru fast. However, They're suauly overkill when you are merely searching for literal strings:

```js
var endsWithSemiColon = /;$/.test(str);
```

Each time a semicolon is found, the regex advances to the next token (`$`), which checks wheter the match is at the end of the string. If not, the regex continues searching for a match until it finally makes its way throough the entire string.

In this case, a better approach is to skip all the intermediate steps required by a regex and simply check whether the last character is a semicolon:

```js
var endsWithSemiColon = str.charAt(str.length - 1) == ";";
```

This is just a bit faster than the regex-based test with small target strings, but, more importantly, the string's length no longer affects the time needed to perfom the test.

#### Focus RegExp on failing faster.

Slow regex processing is usually caused by slow failure rather than slow matching.

Reduce the use of `|` using character classes and optional components, or by pushing the alternation further back into the regex (allowing some match attemps to fail before reaching the alternation):

| Instead of | Use          |
|------------|--------------|
| `cat|bat`  | `[cb]at`     |
| `red|read` | `rea?d`      |
| `red|raw`  | `r(?:ed|aw)` |
| `(.|\r|\n)`| `[\s\S]`     |

#### Use the correct RegExp method.

First, always save the regex expression in a variable as:

```js
var reContains = /(?:^| )foo(?: |$)/;
```

Second, in general terms:

- Use `.test` if you want a faster boolean check
- Use `.match` to retrieve all matches when using the g global flag.

> It's possible that you can gain an extra of perfomance using String method to match regex instead of RegExp to match String. It's depends of your case of use and of your JavaScript Engine. Check [test#1](https://jsperf.com/regexp-test-search-vs-indexof/12), [test#2](https://jsperf.com/regex-methods-x-1/2) & [test#3](https://jsperf.com/test-vs-indexof-fast/5) benchmarks.

### Error

#### Avoid try/catch

Certain constructs like `try/catch` are considered not optimizable for the JavaScript engine, so avoid handle business logic inside. Just for pass an error as callback and this type of things.

A good approach for support optimization with throweable code is return an `Error` an later check about the type.


```js
function mayBeError() {
...
}

var result = maybError()

if (_.isError(result)) {
	/* do something under error */
}

/* in other case no error */
```

### Function

#### Make Your Constructors new-Agnostic

Even your function is called or not using `new` keyword, you can force to have the same behavior in both cases:

```js
function User(name, passwordHash) {
  if (!(this instanceof User)) return new User(name, passwordHash) // magic line!

  this.name = name
  this.passwordHash = passwordHash
}
```

Now, the behavior is the expected in both cases:

```
var x = User("baravelli", "d8b74df393528d51cd19980ae0aa028e")
var y = new User("baravelli","d8b74df393528d51cd19980ae0aa028e")

x instanceof User // true
y instanceof User // true
```


#### Avoid .bind, is slower.

In general terms, `.bind` the context with a `Function` generate need a considerable effort.

Native method is, in general JavaScript Engines slow. Instead, you can use:

- `.call` method (when you need to call the function once).
- `var self = this` is simple and effective.

> Otherwise, exists a set of alternatives, such as libraries that try to implement a better way to do the same. In special, Lodash implementation is based in bitwise, that is very fast check in JavaScript. Check the [test](https://jsperf.com/bind-vs-jquery-proxy/104) benchmark.

I usually use `.bind` more oriented to bind arguments that doesn't change around Function:

```
function sayHello(day, name) {
  console.log('Hello ' + name + ', have a good ' + day)
}

var day = 'Monday'
var sayMonday = sayHello.bind(null, day)

sayMonday('Kiko')
// => 'Hello Kiko, have a good monday'
```

### Array

#### Array.pop() better than Array.shift()

The `.shift` method removes the first element from an array and returns it.

To remove the returned item without re-addressing the array and invalidating all references to it, `shift` requires moving the entire array around.

On the other hand, `.pop` can simply subtract 1 from its length.

Then `.shift` is usually much slower than `.pop`.

# License

MIT © [Kiko Beats](http://kikobeats.com)
