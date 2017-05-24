---
title: Everything you should know about Ruby Splats
description: Even though splats in Ruby are common, there are some complex edge cases that can be misleading. Let's see them together.
tags: [ruby]
header:
  image: /assets/images/splat_bazinga.jpg
  overlay_image: /assets/images/splat_bazinga.jpg
  caption: 'Photo credit: [Ryan McGuire](http://gratisography.com/){:target="_blank", rel="nofollow"}'
  overlay_filter: 0.5

---

I’m working on a new project playing with new kids in town:
Rails 5.1, Webpack(er), Bootstrap 4, React, etc.
I was working with forms when I realized
that Bootstrap 4 needs a lot of conditional CSS class to show states.
Concatenating strings in a view it does not produce a very nice code.

So I thought I could create a similar behavior replicating the
[`classNames` node package](https://github.com/JedWatson/classnames):

```ruby
model.valid?
=> true

class_string(:row, :promo, good: model.valid?, bad: !model.valid?)
=> "row promo good"
```

It concatenates all the hash's keys whose value is truthy.
For convenience,
it also allows receiving an array for CSS classes
that are always included.
This is my helper's implementation:

```ruby
def class_string(*arr, **hash)
  hash
    .map { |css, bool| css if bool }
    .compact
    .concat(arr)
    .map(&:to_s)
    .join(" ")
end
```

Later,
I found a bug in this method that amazed me.
I believed I understood completely how
the splats work in ruby,
but his edge case it astonished me:

```ruby
def splat_test(*array, **hash)
  puts "array: #{array} ; hash: #{hash}"
end

splat_test(42, 'foo' => 'foo', baz: :baz, String => String, bar: :bar)
# array: [42, {"foo"=>"foo", String=>String}] ; hash: {:baz=>:baz, :bar=>:bar}
```

{% include figure image_path="/assets/images/splat_bazinga.jpg" alt="Bazinga splat!"
caption="Bazinga splat!" %}

And this is the reason why I wrote this post.
I have been investigating how the splats work in depth
after discovering this code
and now I want to share with you.

First I'll explain the simple splat in detail,
including an unexpected benchmark result.
I will continue with every case that has come to my mind with the double splat
and the keyword arguments,
to see later why they aren't the same as a hash argument.
Finally, I'll try to explain why this code works in this way.

I will use several snippets from `pry` sessions.
For me, it is the best way to learn ruby,
hopefully, it is useful for you too.

## The simple *splat

The simple splat works with arrays.

### Ruby splat with methods

One of the main uses of the splats is to receive an undefined number of arguments:

```ruby
def splat_arguments(*args)
  args
end

splat_arguments(:linux, "ms", String)
=> [:linux, "ms", String]

splat_arguments(linux: :cool, arch: :very_cool)
=> [{:linux=>:cool, :arch=>:very_cool}]
```

It works with more arguments as well:

```ruby
def splat_more_arguments(a, *args, c)
  puts "a #{a} ; args #{args} ; c #{c}"
end

splat_more_arguments(:archlinux, :elementaryos, :gentoo, :ubuntu, :centos)
# a: archlinux ; args: [:elementaryos, :gentoo, :ubuntu] ; c: centos

splat_more_arguments(:archlinux, :centos)
# a: archlinux ; args: [] ; c centos

splat_more_arguments(:archlinux, [:elementaryos, :gentoo], :ubuntu, :centos)
# a: archlinux ; args: [[:elementaryos, :gentoo], :ubuntu] ; c: centos

splat_more_arguments(:archlinux)
# ArgumentError: wrong number of arguments (given 1, expected 2+)
```


You can avoid giving a name to the splat variable.
This way you show your intention of not using those variables
and it won't be confused with a possible bug in the future:

```ruby
def anonymous_splat_arguments(first, *, last)
  [first, last]
end

anonymous_splat_arguments(:archlinux, :elementaryos, :gentoo, :ubuntu, :centos)
=> [:archlinux, :centos]

# Other interesting use
def inspected_method(*)
  puts "before calling inspected_method"
  super
  puts "after calling inspected_method"
end
```

### Ruby splat as parameters

It can be used to pass parameters from an array:

```ruby
def splat_parameters(a, b, c)
  puts "a: #{a} ; b: #{b} ; c: #{c} "
end

distros = [:archlinux, :debian, :redhat]

splat_parameters(distros[0], distros[1], distros[2])
# a: archlinux ; b: debian ; c: redhat

# Equivalent but easier to write
splat_parameters(*distros)
# a: archlinux ; b: debian ; c: redhat

splat_parameters(:archlinux, *[:debian, :redhat])
# a: archlinux ; b: debian ; c: redhat
```

### Using splats with blocks

The previous points are also true for blocks:

```ruby
def block_splat
  yield(:linux, "ms", String)
end

block_splat do |*args|
  p args
end
# [:linux, "ms", String]

def splatted_block
  distros = [:archlinux, :elementaryos, :gentoo, :ubuntu, :centos]
  yield *distros
end

splatted_block do |best, better, good, common, secure|
  puts "#{best}, #{better}, #{good}, #{common}, #{secure}"
end
# archlinux, elementaryos, gentoo, ubuntu, centos
```

As you can see it works exactly in the same way.

### Array destructuring with splats

The simple splat can be used outside the method definitions and calls.
It can be used in assignments in a very analogous way:

```ruby
distros = [:archlinux, :elementaryos, :gentoo, :centos, :ubuntu]
best, *rest, last = distros
puts "best: #{best} ; rest: #{rest} ; last: #{last}"
# first: archlinux ; rest: [:elementaryos, :gentoo, :centos] ; last: ubuntu
```

It may surprise you, like it did to me,
that this way could be a very performant way to get data from an array.
Here is a benchmark:

```ruby
require 'benchmark/ips'

array = [1..1_000_000].to_a

Benchmark.ips do |t|
  t.report("splat") { first, *, last = array }
  t.report("first") { first, last = array.first, array.last }
  t.compare!
end
# Warming up --------------------------------------
#                splat   279.035k i/100ms
#                first   255.169k i/100ms
# Calculating -------------------------------------
#                splat      6.977M (± 3.9%) i/s -     34.879M in   5.007614s
#                first      5.526M (± 4.3%) i/s -     27.813M in   5.043826s
#
# Comparison:
#                splat:  6976888.2 i/s
#                first:  5525992.2 i/s - 1.26x  slower
```

> Using the splat is faster than calling `first` and `last` methods!

{% include figure image_path="/assets/images/red_splat.jpg" alt="Boom"
caption="Fast splat by [SidewaysSarah](https://www.flickr.com/photos/97699489@N00/5196260104/){:target='_blank', rel='nofollow'}" %}

It is important to use the anonymous splat because if not
the results will be almost identical:

```ruby
Benchmark.ips do |t|
  # Giving to the splat a name the performance goes down
  t.report("splat") { first, *unused, last = array }
  t.report("first") { first, last = array.first, array.last }
  t.compare!
end
# Warming up --------------------------------------
#                splat   251.018k i/100ms
#                first   252.530k i/100ms
# Calculating -------------------------------------
#                splat      5.628M (± 4.2%) i/s -     28.114M in   5.004803s
#                first      5.569M (± 3.7%) i/s -     28.031M in   5.041013s
#
# Comparison:
#                splat:  5628028.9 i/s
#                first:  5568622.5 i/s - same-ish: difference falls within error
```

### Flatten arrays with splats

I think this example is self-explanatory:

```ruby
distros = [:archlinux, :elementaryos, :gentoo, :centos, :ubuntu]
more_distros = [:debian, *distros, :fedora]
=> [:debian, :archlinux, :elementaryos, :gentoo, :centos, :ubuntu, :fedora]
```

### Wrap in an array with splats

Last but not least,
did you know splats can also replace the `Array()` method?

```ruby
Array("string")
=> ["string"]

wrap = *"string"
=> ["string"]

wrap = *1
=> [1]

Array(nil)
=> []

wrap = *nil
=> []

var = "string"
wrap = *var
=> ["string"]

var = 1
wrap = *var
=> [1]

var = [1]
wrap = *var
=> [1]

var = nil
wrap = *var
=> []
```

But maybe it doesn't do what you expected with a hash:

```ruby
var = {a: :a}
wrap = *var
=> [[:a, :a]]
```

And with this wrap,
we finished the simple splat.
Let's take a look at the double one!


## The double **splat operator in Ruby

The double splat in Ruby works with hashes instead of arrays,
*but not with all of them* like I thought.

{% include figure image_path="/assets/images/double_splat.jpg" alt="Double splat"
caption="Double Splat by [Epic Fireworks](https://www.flickr.com/photos/epicfireworks/2855745838/){:target='_blank', rel='nofollow'}" %}

Let's see some examples:

```ruby
def double_splat(**hash)
  hash
end

double_splat(a: :a, b: :b)
=> {:a=>:a, :b=>:b}

hash = { a: :a }
double_splat(hash)
=> {:a=>:a}
```

### Keyword arguments

This feature introduced in Ruby 2.0 is related to the double splat operator.
You can define a method with *keyword arguments*,
this means the parameters have a name.
When you want to call a keyword argument method
you have to name each argument:


```ruby
def keyword_arguments(key_a: :default_a, key_b: :default_b)
  [key_a, key_b]
end

keyword_arguments(key_a: :a, key_b: :b)
=> [:a, :b]

keyword_arguments(key_b: :b, key_a: :a)
=> [:a, :b]
```

As you can see,
the order of the arguments does not matter,
but calling this kind of methods is cumbersome.
It may be nice for methods with a long list of parameters,
however, this is usually considered a code smell.

But this is not the only feature.
They were declared with a default value,
you can use them:

```ruby
keyword_arguments(key_b: :b)
=> [:default_a, :b]

keyword_arguments
=> [:default_a, :default_b]
```

You can use a hash with all the parameters:

```ruby
hash = { key_a: :a, key_b: :b }
keyword_arguments(hash)
=> [:a, :b]
```

If you forget to name the parameters it will fail:

```ruby
keyword_arguments(:a, :b)
# ArgumentError: wrong number of arguments (given 2, expected 0)
```

The errors are very difficult to understand,
so they deserve their own section...

### Double splat and Keyword argument Errors

The last error it seems very strange:

```
ArgumentError: wrong number of arguments (given 2, expected 0)
```

> Why did it expect 0 when 2 arguments were defined?

The double splat argument and keyword arguments
are special arguments in Ruby.
You can check it with your ruby console
and a little of introspection:

```ruby
def simple_method(value)
  value
end
Object.instance_method(:simple_method).parameters
=> [[:req, :value]]

def optional_method(value=3)
  value
end
Object.instance_method(:optional_method).parameters
=> [[:opt, :value]]

def splat_method(*args)
  args
end
Object.instance_method(:splat_method).parameters
=> [[:rest, :args]]

def double_splat_method(**args)
  args
end
Object.instance_method(:double_splat_method).parameters
=> [[:key_rest, :args]]

def keyword_method(a_key: :value)
  a_key
end
Object.instance_method(:keyword_method).parameters
=> [[:key, :a_key]]

def block_method(&a_block)
  a_block
end
Object.instance_method(:block_method).parameters
=> [[:block, :a_block]]
```

The error message is confused
because you don't expect that
a keyword argument isn't an argument,
but for Ruby,
they are completely different kind of arguments.

Thus if you use regular values for double splat
and keyword arguments Ruby complains:

```ruby
double_splat_method(42)
# ArgumentError: wrong number of arguments (given 1, expected 0)

keyword_method("4, 8, 15, 16, 23, 42")
# ArgumentError: wrong number of arguments (given 1, expected 0)
```

#### Hash keys MUST be symbols!

I didn't know this before creating this post
and it was part of my initial confusion.
If you use a hash with no symbol keys,
they aren't used by the double splat and keyword arguments:

```ruby
keyword_method("a_key" => "value")
# ArgumentError: wrong number of arguments (given 1, expected 0)

keyword_method(a_key: "value")
=> "value"

double_splat_method("a_key" => "value")
# ArgumentError: wrong number of arguments (given 1, expected 0)

double_splat_method(a_key: "value")
=> {:a_key=>"value"}
```

#### Hash MUST contain exactly the same keys

If you pass more keys than
the defined in a keyword arguments method
Ruby fails again:

```ruby
keyword_method(a_key: "value")
=> "value"

keyword_method(a_key: "value", other_key: "666")
# ArgumentError: unknown keyword: other_key

hash = { a_key: "value", other_key: "666" }
keyword_method(hash)
# ArgumentError: unknown keyword: other_key
```

To avoid this kind of errors,
you can use the anonymous double splat operator:

```ruby
def keyword_arguments_no_crash(key_a: :default_a, key_b: :default_b, **)
  [key_a, key_b]
end

invalid_hash = { key_a: :hash_a, key_b: :hash_b, key_c: :hash_c }
keyword_arguments_no_crash(invalid_hash)
=> [:hash_a, :hash_b]
```

In fact, you can give it a name
and use it in the method:

```ruby
def keyword_arguments_plus(key_a: :default_a, key_b: :default_b, **plus)
  [key_a, key_b, plus]
end

invalid_hash = { key_a: :hash_a, key_b: :hash_b, key_c: :hash_c }
keyword_arguments_plus(invalid_hash)
=> [:hash_a, :hash_b, {:key_c=>:hash_c}]
```

### Mandatory keyword arguments

Since Ruby 2.1,
it is allowed to not add a default value to the arguments.
In this case, the argument is mandatory:

```ruby
def mandatory_arguments(mandatory:)
  mandatory
end

mandatory_arguments
# ArgumentError: missing keyword: mandatory

mandatory_arguments(mandatory: :learn)
=> :learn
```

It can be combined with the double splat to
make some arguments optional and another mandatory:

```ruby
def configuration(db:, **other_options)
  puts "db: #{db}, other_options: #{other_options}"
end

configuration(db: :postgres)
# db: postgres, other_options: {}

configuration(pool: 5)
# ArgumentError: missing keyword: db

configuration(db: :postgres, pool: 5)
# db: postgres, other_options: {:pool=>5}

conf = { pool: 5, db: :postgres }
configuration(conf)
# db: postgres, other_options: {:pool=>5}
```

As you can see,
**Ruby splits the hash for you,
making the code easier to read and to work with.**

{% include figure image_path="/assets/images/window_splat.jpg" alt="Splats everywhere"
caption="Splat everywhere by [Dimitri Popov](https://unsplash.com/search/splat?photo=Jlv0GcqrkJU){:target='_blank', rel='nofollow'}" %}


### The double splat operator & destructuring hashes

Finally,
you can combine keyword arguments and hashes using the double splat operator.
It will be similar to use the `merge` method:

```ruby
my_options = { logger: :stdout }

configuration(my_options.merge(db: :mongodb))
# db: mongodb, other_options: {:logger=>:stdout}

configuration(db: :mongodb, **my_options)
# db: mongodb, other_options: {:logger=>:stdout}
```


### Using double splats with blocks

All the previous points
can also be applied to the blocks in Ruby:

```ruby
def block_double_splat
  yield(kernel: :panic)
end

block_double_splat do |**args|
  p args
end
# {:kernel=>:panic}

def double_splatted_block
  distros = { best: :archlinux, common: :ubuntu }
  yield distros
end

double_splatted_block do |best:, **|
  puts "The best is #{best}"
end
# The best is archlinux
```

## Why double splat when the last argument could be a hash?

Do you remember that the double splat argument is not a regular argument?
This causes strange message errors when the arguments are not expected,
but you can also use it to take advantage of it:

```ruby
def no_using_double_splat(*args, hash_args={})
  hash_args
end

def using_double_splat(*args, **hash_args)
  hash_args
end

using_double_splat(7, 13, daenerys: :targaryen)
 => {daenerys: :targaryen}
no_using_double_splat(7, 13, daenerys: :targaryen)
 => {daenerys: :targaryen}
```

This seems to be the same behavior,
but it is different when you don't pass a hash as the last argument:

```ruby
using_double_splat(7, 13)
=> {}
no_using_double_splat(7, 13)
=> 13
```

So, if you need the last argument to always be a hash,
the double splat operator in the definition is the way to go.

## The strange behavior of the beginning

This was the code:

```ruby
def splat_test(*array, **hash)
  puts "array: #{array} ; hash: #{hash}"
end

pry(main)> splat_test(42, 'foo' => 'foo', baz: :baz, String => String, bar: :bar)
array: [42, {"foo"=>"foo", String=>String}] ; hash: {:baz=>:baz, :bar=>:bar}
```

{% include figure image_path="/assets/images/fighting_splats.jpg" alt="Fighting splats"
caption="Fighting with splats by [Ixis IT](https://www.flickr.com/photos/ixisit/14854928640/){:target='_blank', rel='nofollow'}" %}

My first mistake was thinking that
the double splat operator captures all pairs key-value of the last hash.
However, as we have seen before
**double splat operator only captures symbol keys!**

Ruby interprets the call of the method *more or less* in this way:

1. `42`: is a number and it will be the first element of the splat `*array`.

2. `'foo' => 'foo'`: the key is not a symbol, it is a string,
this means it cannot be added to the keyword `hash`.
Hence it adds the couple key-value to the second element of the `*array`.
This second element is, of course, an array.

3. `:baz => :baz`: ruby finds a new couple key-value.
This time the key is a symbol (`:baz`) and
it is added to the keyword `hash`

4. `String => String`: the key is not a symbol, it is a class.
It cannot be added to the keyword `hash`.
So ruby instead of complaining,
cleverly (maybe too cleverly for my taste),
it adds it to the splatted `array`.
Instead of being added to a new hash at the third position,
it is merged with the second and last element in this `array`.
For me, it is clearly an unexpected behavior.

5. `:bar => bar`: This time the key is a symbol.
It is added to the keyword `hash`.

Ruby does not oppose to this bizarre syntax,
it allows you mix keys of all types and
they are added to the "right" variable on the fly.
Ruby is literally splitting our last hash argument into two different hashes,
depending on the type of the key.

## Conclusions

Congratulations if you reach until here :)
This was a big review of splat features in Ruby.
Hopefully, you learned something new
among all the code snippets.

{% include figure image_path="/assets/images/end_splat.jpg" alt="Double splat"
caption="Double Splat by [Reuben Whitehouse](https://www.flickr.com/photos/rcktfld/2266691836/){:target='_blank', rel='nofollow'}" %}

The last part of this post,
Ruby splitting the hash,
is a curious feature.
I can see the point
because the keyword argument must have symbol keys,
and it can be very useful as last optional argument.
On the other hand,
it is not very intuitive and
it has to be used with caution.
A future programmer may not know all the difficulties of splats in Ruby
and bugs can easily introduce.

And what do you think about splat? Did you see something that surprised you?
