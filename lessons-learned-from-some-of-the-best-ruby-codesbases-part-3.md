Welcome to ....


### [IceNine](https://github.com/dkubb/ice_nine)

*IceNine* is a library for deep freezing of objects:

```Ruby
require 'ice_nine'

hash   = IceNine.deep_freeze('a' => '1')
# TODO
``

Let's check out how this works!

Here's the relevant part of lib/ice_nine.rb:

```Ruby
module IceNine
  def self.deep_freeze(object)
    Freezer.deep_freeze(object)
  end
end
```

Ok, not much to see here, we're just delegating to lib/ice_nine/freezer.rb:

```Ruby
module IceNine
  class Freezer
    def self.deep_freeze(object)
      guarded_deep_freeze(object, RecursionGuard::ObjectSet.new)
    end
  end
end
```

Let's ignore that recursion guard thingie for now - we'll talk about this in detail - and go further down the spiral:

```Ruby
def self.guarded_deep_freeze(object, recursion_guard)
  recursion_guard.guard(object) do
    Freezer[object.class].guarded_deep_freeze(object, recursion_guard)
  end
end
```

There it is. That's where the magic happens. Like I said above, please ignore everything having to do with "recursion" for now and let's dissect this line:

```Ruby
Freezer[object.class].guarded_deep_freeze(object, recursion_guard)
```

First we need to understand:

```Ruby
Freezer[object.class]
```

So object is the object that we passed to *IceNine.deep_freeze* in the beginning.
If this was something like this

```Ruby
IceNine.deep_freeze({ foo: :bar })
```

Then the *object* would be

```Ruby
{ foo: :bar }
```

and thus *object.class* would be *Hash*.

And what does the *[]* method look like?

```Ruby
def self.[](mod)
  @freezer_cache[mod]
end
```

Now this is starting to get interesting. This *@freezer_cache* is defined at the top of the file:

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

    def self.find(name)
      freezer = name.split('::').reduce(self) do |mod, const|
        mod.const_lookup(const) or break mod
      end
      freezer if freezer < self  # only return a descendant freezer
    end
  end
end
```

Note that this is __not__ an instance variable but a [class instance variable](http://www.railstips.org/blog/archives/2006/11/18/class-and-instance-variables-in-ruby/) which means this assignment gets executed when Ruby reads the *Freezer* class since Rubys [class bodies are executable](http://yehudakatz.com/2009/06/04/the-importance-of-executable-class-bodies/).

The code above is slightly complicated but gets an incredible amount of work done. Just talking about this code probably won't help you really grok it if you are anything like me so let's go through an example and then step through the code to see what happens.

This will be our example:

```Ruby
IceNine.deep_freeze({ foo: :bar })
```
