# Opal

**Opal is a ruby to javascript compiler.** Opal aims to take ruby files
and generate efficient javascript that maintains rubys features. Opal
will, by default, generate fast and efficient code in preference to
keeping all ruby features.

Opal comes with an implementation of the ruby corelib, written in ruby,
that uses a bundled runtime (written in javascript) that tie all the
features together. Whenever possible Opal bridges to native javascript
features under the hood. The Opal gem includes the compiler used to
convert ruby sources into javascript.

Opal is [hosted on github](http://github.com/adambeynon/opal), and there
is a Freenode IRC channel at `#opal`.

## Downloads

The Opal runtime and corelib are distributed here, and are required to
run any code generated by opal.

[Opal version 0.3.19](http://opalrb.org/opal.js) _(14.9kb Minified And Gzipped)_

## Installation

Opal comes distributed as a gem, so either install with:

    gem install opal

Or add to your Gemfile:

```ruby
gem "opal"
```

## Usage

To quickly compile ruby code into javascript, use the `Opal.parse()`
method which returns a string of javascript code:

```ruby
require 'opal'

Opal.parse("[1, 2, 3, 4].each { |a| puts a }")
```

This will return a string of javascript similar to the following:

```javascript
(function() {
  // compiled ruby
}).call(Opal.top);
```

This can then be written to a file and run in any browser.

### Creating a rake task

Using a Rakefile makes it simple to build your application code.
Assuming your code is in a file `app.rb`, add a rake task:

```ruby
# Rakefile

require 'opal'

desc "Build opal application"
task :build do
  src = File.read 'app.rb'
  js  = Opal.parse src

  File.open('app.js', 'w+') do |out|
    out.write js
  end
end
```

Running `rake build` will then read your app code, compile it and then
write it out to a file ready to load in a web browser.

### Setting up html file

The generated `app.js` file can just be added into any HTML page. The
opal runtime needs to be loaded first (you can download that above).

```html
<!doctype html>
<html>
<head>
  <title>Test Opal App</title>
</head>
<body>
  <script src="opal.js"></script>
  <script src="app.js"></script>
</body>
</html>
```

When using `Opal.parse()` as above, the generated code will be run
as soon as the page loads. Open the browsers console and you should
see the 4 numbers printed to the console.

## Builder and Dependency Builder

The previous example was useful for building very simple apps using
opal. For more complex apps with dependencies, Opal provides useful
rake tasks to get started. Assuming opal is in your lib path, create
a `Rakefile` similar to:

```ruby
# Rakefile

require 'opal'

Opal::BuilderTask.new do |t|
  t.name         = 'my-first-app'
  t.dependencies = ['opal-json']
  t.files        = ['app.rb']
end
```

This simple rake task is all you need to build an app with its
dependencies.

### Building dependencies

To build the opal runtime `opal.js`, as well as `opal-json` into
`build/`, run the simple rake task:

```ruby
rake dependencies
```

This will try and find the `opal-json` gem installed, so either
install globally with `gem install opal-json`, or add it to your
Gemfile as `gem "opal-json"` and run `bundle install`.

### Building app

To build the listed files into your application, run:

```ruby
rake build
```

This will build to `/build/my-first-app.js`. To customize the output
filename, change the `name` property in the raketask.

### Running the app

You should now be able to run the built app using a standard HTML page.

```html
<!doctype html>
<html>
<head>
  <title>Test Opal App</title>
</head>
<body>
  <script src="build/opal.js"></script>
  <script src="build/opal-json.js"></script>
  <script src="build/my-first-app.js"></script>
</body>
</html>
```

### Main file

When using the `BuilderTask`, the files generated will not be run
automatically on page load. The files are registered so they can be
loaded using `require()`. Builder is clever enough though that it will,
by default, automatically require the first file in the app to be
loaded. This will appear at the bottom of `my-first-app.js`:

```javascript
Opal.define('app', function() {
  // app.rb code
});

Opal.require('app');
```

## Features And Implementation

Opal is a source-to-source compiler, so there is no VM as such and the
compiled code aims to be as fast and efficient as possible, mapping
directly to underlying javascript features and objects where possible.

### Literals

**self** is always compiled to `this`. Any context inside the generated
code is usually a function body; whether it be a method body, a block,
a class/module body or the file itself.

**true** and **false** are compiled directly into their native boolean
equivalents. This makes interaction a lot easier as there is no need
to convert values to opal specific values. It does mean that there is
only a `Boolean` ruby class available, not seperate `TrueClass` and
`FalseClass` classes.

**nil** is compiled into a `nil` reference, which inside all generated
files points to a special object which is just an instance of the ruby
`NilClass` class. This object is available externally to javascript as
`Opal.nil`.

```ruby
nil         # => nil
true        # => true
false       # => false
self        # => this
```

#### Strings

Ruby strings are compiled directly into javascript strings for
performance as well as readability. This has the side effect that Opal
does not support mutable strings - i.e. all strings are immutable.

#### Symbols

For performance reasons, symbols compile directly into strings. Opal
supports all the symbol syntaxes, but does not have a real `Symbol`
class. Symbols and Strings can therefore be used interchangeably.

```ruby
"hello world!"    # => "hello world!"
:foo              # => "foo"
<<-EOS            # => "\nHello there.\n"
Hello there.
EOS
```

#### Numbers

In Opal there is a single class for numbers; `Numeric`. To keep opal
as performant as possible, ruby numbers are mapped to native numbers.
This has the side effect that all numbers must be of the same class.
Most relevant methods from `Integer`, `Float` and `Numeric` are
implemented on this class.

```ruby
42        # => 42
3.142     # => 3.142
```

#### Arrays

Ruby arrays are compiled directly into javascript arrays. Special
ruby syntaxes for word arrays etc are also supported.

```ruby
[1, 2, 3, 4]        # => [1, 2, 3, 4]
%w[foo bar baz]     # => ["foo", "bar", "baz"]
```

#### Hash

Inside a generated ruby script, a function `__hash` is available which
creates a new hash. This is also available in javascript as `Opal.hash`
and simply returns a new instance of the `Hash` class.

```ruby
{ :foo => 100, :baz => 700 }    # => __hash("foo", 100, "baz", 700)
{ foo: 42, bar: [1, 2, 3] }     # => __hash("foo", 42, "bar", [1, 2, 3]) 
```

#### Range

Similar to hash, there is a function `__range` available to create
range instances.

```ruby
1..4        # => __range(1, 4, true)
3...7       # => __range(3, 7, false)
```

### Methods

A ruby method is just compiled directly into a function definition.
These functions are added to the constructor's prototype so they can
be called just like any javascript function. All ruby methods are
defined with a `$` prefix to try and isolate them from javascript
methods.

#### Method Calls

All method arguments are passed to the native function just like normal
javascript function calls. Therefore, the given ruby code:

```ruby
do_something 1, 2, 3
self.length
[1, 2, 3].push 5
```

Will be compiled into the easy to read javascript:

```javascript
this.$do_something(1, 2, 3);
this.$length();
[1, 2, 3].$push(5);
```

There are some certain characters which are valid as ruby method names
but not as javascript identifiers. These method calls are encoded to
keep the generated method names sane.

```ruby
self.loaded?              # => this.$loaded$p()
self.load!                # => this.$load$b()
self.loaded = true        # => this.$loaded$e(true)
self << :bar              # => this.$lshift$("bar")
```

Finally, method calls with splat arguments are also supported:

```ruby
self.push *[1, 2, 3]
# => this.$push.apply(this, [1, 2, 3])
```

#### Optimized Math Operators

In ruby, all math operators are method calls, but compiling this into
javascript would end up being too slow. For this reason, math
operators are optimized to test first if the receiver is a number, and
if so then to just carry out the math call.

```ruby
3 + 4
```

This ruby code will then be compiled into the following javascript:

```javascript
(a = 3, b = 4, typeof(a) === "number" ? a + b : a.$plus(b))
```

This ternary statement falls back on sending a method to the receiver
so all non-numeric receivers will still have the normal method call
being sent. This optimization makes math operators a **lot faster**.
Currently, the optimized method calls are `+`, `-`, `*` and `/`.

#### Method Definitions

Methods are implemented as regular javascript functions. Assuming the
following method is defined inside a class body:

```ruby
def to_s
  inspect
end
```

This would generate the following javascript. (`def.` is a local
variable set to be the class's instance prototype. It is used
for minimization of code as well as trying to be readable).

```javascript
def.$to_s = function() {
  return this.$inspect();
};
```

The defined name retains the `$` prefix outlined above, and the `self`
value for the method is `this`, which will be the receiver.

Normal arguments, splat args and optional args are all supported:

```ruby
def norm(a, b, c)

end

def opt(a, b = 100)

end

def rest(a, *b)

end
```

The generated code reads as expected:

```javascript
def.$norm = function(a, b, c) {
  return nil;
};

def.$opt = function(a, b) {
  if (b == null) b = 100;
  return nil;
};

def.$rest = function(a, b) {
  b = __slice.call(arguments, 1);
  return nil;
};
```

Currently, in opal there is no argument length checking to ensure that
the correct number of arguments get passed to a function. This can be
enabled in debug mode, but is not included in production builds as it
adds a lot of overhead to **every** method call.

### Compiled Files

As described above, a compiled ruby source gets generated into a string
of javascript code that is wrapped inside an anonymous function. This
looks similar to the following:

```javascript
(function(undefined) {
  var nil = Opal.nil, __slice = Opal.slice, __klass = Opal.klass;
  // generated code
}).call(Opal.top);
```

This function gets called with `Opal.top` as the context, which is the
top level object available to ruby (`main`). Inside the function, `nil`
is assigned to ensure a local copy is available, as well as all the
helper methods used within the generated file. There is no return value
from these functions as they are not used anywhere.

As a complete example, assuming the following code:

```ruby
puts "foo"
```

This would compile directly into:

```javascript
(function(undefined) {
  var nil = Opal.nil;
  this.$puts("foo");
}).call(Opal.top);
```

Most of the helpers are no longer present as they are not used in this
example.

### Using compiled sources

If you write the generated code as above into a file `app.js` and add
that to your HTML page, then it is obvious that `"foo"` would be
written to the browser's console.

## License

Opal is released under the MIT license.

## Change Log

**0.3.19** _(30 May 2012)_

* Add BasicObject as the root class
* Add `Opal.define` and `Opal.require` for requiring files
* Builder uses a `main` option to dictate which file to require on load
* Completely revamp runtime to reduce helper methods
* Allow native bridges (Array, String, etc) to be subclassed
* Make sure `.js` files can be built with `Opal::Builder`
* Include the current file name when raising parse errors

**0.3.18** _(20 May 2012)_

* Fix various core lib bugs
* Completely remove `require` from corelib
* Improve Builder to detect dependencies in files

**0.3.17** _(19 May 2012)_

* Revamp of Builder and Parser tools
* Remove opal-repl
* Added a lot of specs for core lib

**0.3.16** _(15 January 2012)_

* Added HEREDOCS support in parser
* Parser now handles masgn (mass/multi assignments)
* More useful DependencyBuilder class to build gems dependencies
* Blocks no longer passed as an argument in method calls

**0.3.15**

* Initial Release.