I recently started looking into [Mutant](https://github.com/mbj/mutant) and related gems for an upcoming presentation about abstract syntax trees I am working on at the moment.

While browsing the source code of *Mutant* and of the gems that *Mutant* uses I realized that those codebases are among the cleanest code bases I have ever seen in Ruby land with an amazing overall architecture and I'd love to share what I've seen and learned reading their code.

I intend to make this a series with the first part of this series not covering *Mutant* itself but the gems it uses.

Let's get started with *Mutant*s [gem dependencies](https://github.com/mbj/mutant/blob/master/mutant.gemspec).

### [parser](https://github.com/whitequark/parser)

Now that is one of my favourite gem of all times. My beloved [Reek gem](https://github.com/troessner/reek) would not be what it is without the *parser* gem. Nor would 80% of all other gems in Ruby land that have something to do with abstract syntax trees.

The *parser* gem is huge though. I intend to cover this in a latter blog post and will skip this for now.

### [Concord](https://github.com/mbj/concord)

*Concord* is a nice little library that helps you turning this:

```Ruby
class ComposedObject
  attr_reader :foo
  protected :foo

  attr_reader :bar
  protected :bar

  def initialize(foo, bar)
    @foo, @bar = foo, bar
  end
end
```

into this:

```Ruby
class ComposedObject
  include Concord.new(:foo, :bar)
end
```

What's interesting here is how we we include it:

```Ruby
include Concord.new(:foo, :bar)
```


Ok, something is including a module. Wait, it's not a module. We're calling *new* on it. Wat?
How does this work?

Here's what happens when you try to include something that is not "module'sque":

```Bash
[2] pry(main)> class Klazz; end
=> nil
[3] pry(main)> class Foo; include Klazz; end
TypeError: wrong argument type Class (expected Module)
from (pry):3:in `include'
[4] pry(main)> Klazz.class
=> Class
```

Ok, makes sense. *Klazz* class is *class*, not *Module*.
Let's contrast this with a proper module:

```Bash
[1] pry(main)> module Mod; end
=> nil
[2] pry(main)> class Foo; include Mod; end
=> Foo
[3] pry(main)> Mod.class
=> Module
```

Now let's try to emulate the code I showed you from *Mutant*:

```Bash
[1] pry(main)> class Klazz < Module; end
=> nil
[2] pry(main)> class Foo; include Klazz; end
TypeError: wrong argument type Class (expected Module)
from (pry):2:in `include'
[3] pry(main)> class Foo; include Klazz.new; end
=> Foo
[4] pry(main)> Foo.class
=> Class
```

Now it get even more confusing - *include Klazz* doesn't work, *include Klazz.new* does and the class of *Foo* is still *Class* not *Module*.  What's happening here?

Let's see how Rubinius actually handles this in [module.rb](https://github.com/rubinius/rubinius/blob/master/core/module.rb#L471):

```Ruby
def include?(mod)
  if !mod.kind_of?(Module) or mod.kind_of?(Class)
    raise TypeError, "wrong argument type #{mod.class} (expected Module)"
  end
  # snip
end
```

There it is. If the argument is not a module __or__ if it's a class we raise. Now we understand why *include Klazz* doesn't work (because it's class is *class*) but *include Klazz.new* does not raise (because its class is *Klazz*, not *Class*).

This elegant trick allows to use a class in a kind-of module context and to use a module in a kind-of class context.

### [procto](https://github.com/snusnu/procto)

Turns your ruby object into a method object. The code example from the README speaks for itself so I'll just paste it here:

```Ruby
class Printer
  include Procto.call(:print)

  def initialize(text)
    @text = text
  end

  def print
    "Hello #{text}"
  end
end

Printer.call('world') # => "Hello world"
```

The code to make this work is surprisingly small and very elegant.

```Ruby
class Procto < Module
  # The default name of the instance method to be called
  DEFAULT_NAME = :call

  def self.call(name = DEFAULT_NAME)
    new(name.to_sym)
  end
  # ...
end
```

We define a singleton method called "call" that takes a name and returns an instance of *Procto*. (Using the same "include a kind-of class as module trick" *Concord* used above).

How does the constructor look like?

```Ruby
def initialize(name)
  @block = ->(*args) { new(*args).public_send(name) }
end
```

Here things start to get interesting. We use a lambda (via the stabby lambda syntax) that takes an arbitrary number of arguments. And then it calls *new* with those. What *new* does it call? The block is executed at runtime and in the scope of....the object that *Procto* is included into, so the *new* here is not *Procto#new* but rather the object in question.

In other words, the lambda will create a new object of whatever it is included into and call whatever *name* we pass into it. So that's basically where

```Ruby
Printer.call('world')
```

comes from.

But how is this included?

```Ruby
def included(host)
  host.instance_exec(@block) do |block|
    define_singleton_method(:call, &block)
  end
end
```

TODO: Explain instance_exec and what happens here.

### [equalizer](https://github.com/dkubb/equalizer)

*equalizer* is a module to define equality, equivalence and inspection methods.

An example from the README:

```Ruby
class GeoLocation
  include Equalizer.new(:latitude, :longitude)

  attr_reader :latitude, :longitude

  def initialize(latitude, longitude)
    @latitude, @longitude = latitude, longitude
  end
end

point_a = GeoLocation.new(1, 2)
point_b = GeoLocation.new(1, 2)

point_a == point_b # => true
```

Without the *include* from above Ruby would evaluate this as *false*, not *true*.

When can this be useful?

Think about [value objects](http://www.sitepoint.com/ddd-for-rails-developers-part-2-entities-and-values/):

```
“A Value Object is an object that describes some characteristic or attribute but carries no concept of identity.” As there is no identity, two Value Objects are equal when all their attributes are equal. An example of a Value Object would be Money.
```

*Value objects* are heavily used in methodologies like [Domain Driven Design](https://en.wikipedia.org/wiki/Domain-driven_design) or when using *composed_of* in Rails (for a great introduction to this whole topic, check out [this article series](http://victorsavkin.com/ddd) from Victor Savkin).

How does equalizer do this?

