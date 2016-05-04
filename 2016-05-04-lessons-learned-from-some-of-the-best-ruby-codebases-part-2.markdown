<!-- Uncomment below before publishing via octopress -->
<!--
---
layout: post
title: Lessons learned from some of the best ruby codebases out there (part 2)
date: 2016-05-04
comments: true
categories: ruby
keywords: ruby, gems, mutant
description: Lessons learned from some of the best ruby codebases out there (part 2)
author: Timo Rößner
---
-->

## Lessons learned from some of the best ruby codebases out there (part 2)

Welcome to the second part of the series (you can find the first part [here](https://tech.blacklane.com/2016/04/23/lessons-learned-from-some-of-the-best-ruby-codebases-part-1/)).

We'll continue to look at some other highly interesting gems [Mutant](https://github.com/mbj/mutant) uses.

Initially I had planned to cover multiple gems again, but after starting to look into the [IceNine](https://github.com/dkubb/ice_nine) gem I realized that this was big enough for its own blog post.

<!--more-->

### [IceNine](https://github.com/dkubb/ice_nine)

`IceNine` is a library for deep freezing objects:

```Ruby
hash_1 = { 'foo' => 'bar' }

hash_1.frozen? # => false
hash_1['foo'].frozen? # => false
hash_1['foo'] = 'whoopsie, got changed!' # => "whoopsie, got changed!"

# Let's freeze hash_1

hash_1.freeze
hash_1.frozen? # => true
hash_1['foo'].frozen? # => still false, so its not deeply frozen

# IceNine deep freezes in contrast

require 'ice_nine'

hash_2 = { 'foo' => 'bar' }

IceNine.deep_freeze hash_2

hash_2.frozen? # => true
hash_2['foo'].frozen? # => true
hash_2['foo'] = 'whoopsie, got changed!' # => RuntimeError: can't modify frozen Hash
```

### What do you need this for?

In recent years, [functional programming](https://en.wikipedia.org/wiki/Functional_programming) has become all the rage and how to introduce it into languages that are actually not functional like Ruby.

Depending on who you ask, everybody has a different definition of what "functional" means.
Me, I have 2 attributes that I count as essential for a functional language:

1. [Pure functions](https://en.wikipedia.org/wiki/Functional_programming#Pure_functions)
2. Immutable data structures

I believe Ruby will never have pure functions in that sense but you can create immutable data structures in Ruby by calling `freeze` on them. However, just calling `freeze` on an object doesn't deep-freeze it as you can in my initial code sample above.

And that's where `IceNine` comes in.

Immutable data structures might not be applicable everywhere and all the time but quite often they offer significant benefits over mutable data structures, for instance:

- It is way easier to reason about a data structure when you know that it will always have a guaranteed state
- It's less prone to bugs. Quite often the weirdest bugs occur because data has changed in a way that you hadn't foreseen
- It makes it easier to write concurrent code since immutable data structures eliminate a huge range of common pitfalls when writing concurrent code

With the rationale out of the way, let's check out how it works!

### Finding the right freezer

In my example above you could see that using `IceNine` boils down to:

```Ruby
IceNine.deep_freeze({ 'foo' => 'bar' })
```

Here's the relevant part of the `IceNine` module (defined in `lib/ice_nine.rb`):

```Ruby
module IceNine
  def self.deep_freeze(object)
    Freezer.deep_freeze(object)
  end
end
```

Ok, not much to see here, we're just delegating to `IceNine::Freezer` (defined in `lib/ice_nine/freezer.rb`):

```Ruby
module IceNine
  class Freezer
    def self.deep_freeze(object)
      guarded_deep_freeze(object, RecursionGuard::ObjectSet.new)
    end
  end
end
```

Let's ignore that recursion guard thingie for now - we'll talk about this in detail later - and go further down the spiral:

```Ruby
def self.guarded_deep_freeze(object, recursion_guard)
  recursion_guard.guard(object) do
    Freezer[object.class].guarded_deep_freeze(object, recursion_guard)
  end
end
```

There it is. That's where the magic happens.

Let's dissect this line:

```Ruby
Freezer[object.class].guarded_deep_freeze(object, recursion_guard)
```

First we need to understand:

```Ruby
Freezer[object.class]
```

`object` is the object that we passed to `IceNine.deep_freeze` in the beginning:

```Ruby
IceNine.deep_freeze({ foo: :bar })
```

So the `object` would be

```Ruby
{ foo: :bar }
```

and thus `object.class` would be `Hash`.

And what does the `[]` method look like?

```Ruby
def self.[](mod)
  @freezer_cache[mod]
end
```

Now things are starting to get interesting. This `@freezer_cache` is defined at the top of the file:

```Ruby
module IceNine
  class Freezer
    @freezer_cache = Hash.new do |cache, mod|
      freezer = nil
      mod.ancestors.each do |ancestor|
        freezer = find(ancestor.name.to_s) and break
      end
      cache[mod] = freezer
    end
  end
end
```

The code above is terse and gets a lot of work done:

- We set up `@freezer_cache` as a hash. Note that `@freezer_cache` is __not__ an instance variable but a [class instance variable](http://www.railstips.org/blog/archives/2006/11/18/class-and-instance-variables-in-ruby/) which means this assignment gets executed when Ruby reads the `Freezer` class since Rubys [class bodies are executable](http://yehudakatz.com/2009/06/04/the-importance-of-executable-class-bodies/).

- We pass a block to `Hash#new`. In case we ask the cache for a key that doesn't exist this block will be called with the hash object and the key and should return the default value (see the [Hash docs](http://ruby-doc.org/core-2.3.0/Hash.html#method-c-new))

- So the block parameter `cache` is the hash itself and `mod` is the class of the data structure we passed to `IceNine`, so `Hash` here.

- We then traverse the ancestor chain up and look for a corresponding freezer. Sticking with the hash example we would immediately find `IceNine::Freezer::Hash` since the first link in the ancestor chain of any class is....the class itself as you can see here:

```Ruby
Hash.ancestors
# => [Hash, Enumerable, Object, PP::ObjectMixin, Kernel, BasicObject]
```

Long story short: Given a `Hash` this could would find the - surprise! - `Hash` freezer.
Obviously this is kind of the most possible simple example here since there are a lot of edge cases where it's not that easy. I won't go into the details here though and focus more on the high level (I also won't go into the details of how `find` looks since that is a tad more complicated but you can go and check it out in `lib/ice_nine/freezer`).

Now, equipped with a solid understanding of how the freezer lookup works let's come back to

```Ruby
def self.guarded_deep_freeze(object, recursion_guard)
  recursion_guard.guard(object) do
    Freezer[object.class].guarded_deep_freeze(object, recursion_guard)
  end
end
```

and especially:

```Ruby
Freezer[object.class].guarded_deep_freeze(object, recursion_guard)
```

### Deep freeze

We now know that

```Ruby
Freezer[object.class]
```

part so let's look at that part:

```Ruby
guarded_deep_freeze(object, recursion_guard)
```

and imagine that

```Ruby
Freezer[object.class]
```

would have returned an `IceNine::Freezer::Hash` like we had done in the paragraph above.

This is how `IceNine::Freezer::Hash.guarded_deep_freeze` looks like:

```Ruby
module IceNine
  class Freezer
    class Hash < Object
      def self.guarded_deep_freeze(hash, recursion_guard)
        super
        # snip
        freeze_key_value_pairs(hash, recursion_guard)
      end
    end
  end
end
```

Let's see what `freeze_key_value_pairs` is all about before looking at what happens via `super`:

```Ruby
def self.freeze_key_value_pairs(hash, recursion_guard)
  hash.each do |key, value|
    Freezer.guarded_deep_freeze(key, recursion_guard)
    Freezer.guarded_deep_freeze(value, recursion_guard)
  end
end
```

Ok, that's pretty simple, isn't it? We iterate over all keys and values of the hash and deep freeze them as well.

Now what's up with `super`?

As you can see above in my initial `IceNine::Freezer::Hash` snippet this class inherits from `IceNine::Freezer::Object`:

```Ruby
module IceNine
  class Freezer
    class Object < self
      def self.guarded_deep_freeze(object, recursion_guard)
        return object unless object.respond_to?(:freeze)

        object.freeze
        freeze_instance_variables(object, recursion_guard)
        object
      end
    end
  end
end
```

Now things should start to make sense:

- We check if the object in question does support `freeze`, and if it does, we'll freeze it.
- This covers the first part in my very first example

```Ruby
IceNine.deep_freeze hash_2
hash_2.frozen? # => true
```

- We then freeze all subsequent attributes of the object, in the case of `Hash` they keys and the values

Ok, so now we know:

1. how a suitable freezer class is looked up. In our examples above we used `IceNine::Freezer::Hash`
2. how this class does the actual freezing.

This only leaves one thing left to explain: What's up with that recursion guard?

### The recursion guard

If you paid attention, you'll have noticed that `IceNine` recursively traverses any data structure you pass it in.

Remember that we started out like this in `IceNine::Freezer`:

```Ruby
def self.guarded_deep_freeze(object, recursion_guard)
  recursion_guard.guard(object) do
    Freezer[object.class].guarded_deep_freeze(object, recursion_guard)
  end
end
```

And we then restarted the whole cycle of getting the objects' class, determining the appropriate freezer, freezing it and so on in `IceNine::Freezer::Hash`:

```Ruby
module IceNine
  class Freezer
    class Hash < Object
      def self.freeze_key_value_pairs(hash, recursion_guard)
        hash.each do |key, value|
          Freezer.guarded_deep_freeze(key, recursion_guard)
          Freezer.guarded_deep_freeze(value, recursion_guard)
        end
      end
    end
  end
end
```

So there's the recursion that we start via the recursion guard.

Why does it say "guard"?
Because you might create and pass cyclic data structures like this:

```Ruby
b = { baz: nil }
a = { foo: b }
# => {:foo=>{:baz=>5}}

b[:baz] = a
# => {:foo=>{:baz=>{...}}}

# And now you can do this all day long:
a[:foo][:baz]
# => {:foo=>{:baz=>{...}}}
a[:foo][:baz][:foo]
# => {:baz=>{:foo=>{...}}}
a[:foo][:baz][:foo][:baz]
# => {:foo=>{:baz=>{...}}}
```

Without a recursion guard `IceNine` would recurse down until you'd get

```
stack level too deep (SystemStackError)
```

Let's check out `IceNine::RecursionGuard::ObjectSet`(defined in `lib/ice_nine/support/recursion_guard`):

```Ruby
module IceNine
  class RecursionGuard
    class ObjectSet < self
      def initialize
        @object_ids = {}
      end

      def guard(object)
        caller_object_id = object.__id__
        return object if @object_ids.key?(caller_object_id)
        @object_ids[caller_object_id] = nil
        yield
      end
    end
  end
end
```

The code above is incredibly simple and yet incredibly powerful at the same time.

First we get the `object_id` of the object we're looking at:

```Ruby
caller_object_id = object.__id__
```

Now the important part: If we have seen this very object already this means we're indeed traversing a cyclic structure. In this case, we just return:

```Ruby
return object if @object_ids.key?(caller_object_id)
```

If we made it until here we'er seeing this object for the first time, so we store its' `object_id`:

```Ruby
@object_ids[caller_object_id] = nil
```

And finally we yield to the block:

```Ruby
yield
```

### Making `deep_freeze` available to all objects

`IceNine` also offers a core extension in `lib/ice_nine/core_ext/object.rb`:

```Ruby
module IceNine
  module CoreExt
    module Object
      def deep_freeze
        IceNine.deep_freeze(self)
      end
    end
  end
end

Object.instance_eval { include IceNine::CoreExt::Object }
```

First we define a separate module following the convention that modules that are mixed in into core classes are defined under the `core_ext` scope. What this module does is pretty simple, it just calls `IceNine.deep_freeze` and then passes `self` to it. What will `self` be? Modules with instance methods can not be used standalone but only as mixin. Means, mixed into a class. So `self` will be the class that we mixed it in.

How are we activating this?

By

```Ruby
Object.instance_eval { include IceNine::CoreExt::Object }
```

This effectively makes `deep_freeze` available to all objects in Ruby's object space.

By the way it doesn't matter if you use `class_eval` or `instance_eval` in this case.

Both

```Ruby
Object.instance_eval { include IceNine::CoreExt::Object }
```

and

```Ruby
Object.class_eval { include IceNine::CoreExt::Object }
```

have the same outcome here. This would however make a huge difference if you defined `deep_freeze` on `Object` via `def` directly.

E.g. this

```Ruby
Object.class_eval do
  def deep_freeze
    IceNine.deep_freeze(self)
  end
end
```

defines `deep_freeze` on an instance level since `class_eval` just re-opens the class. So we're good.

This however

```Ruby
Object.instance_eval do
  def deep_freeze
    IceNine.deep_freeze(self)
  end
end
```

would __not__ work since it would create a singleton method (read: class method), not an instance method.

So you should be aware of those differences.
Most gems I have seen use the `Object.instance_eval` trick for `core_ext` and you probably should follow this pattern in your own gems.

### Wrapping it up

So let's summarize what we learned today about `IceNine`:

- it recursively traverses any data structure it gets passed
- using a recursion guard to prevent it from recursing cyclic data structure
- it maintains a cached mapping of what freezer to use for what data type
- it freezes the data it has been given
- and then goes deeper down the spiral for all children

That's it for part of this series. In the upcoming part 3 I will be looking at the [Adamantium gem](https://github.com/dkubb/adamantium) and the [abstract_type gem](https://github.com/dkubb/abstract_type).
