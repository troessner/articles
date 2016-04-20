## Lessons learned from some of the best ruby codebases out there

I recently started looking into [Mutant](https://github.com/mbj/mutant) and related gems for an upcoming presentation about abstract syntax trees I am working on at the moment.

While browsing the source code of *Mutant* and of the gems that *Mutant* uses I realized that those codebases are among the cleanest code bases I have ever seen in Ruby land with an amazing overall architecture and I'd love to share what I've seen and learned reading their code.

I intend to make this a series with the first part of this series not covering *Mutant* itself but the gems it uses.

Let's get started with *Mutant*s [gem dependencies](https://github.com/mbj/mutant/blob/master/mutant.gemspec).

### [Concord](https://github.com/mbj/concord)

*Concord* is a nice little library that helps you turning this:

```Ruby
class ComposedObject
  attr_reader :foo, :bar
  protected :foo; :bar

  def initialize(foo, bar)
    @foo = foo
    @bar = bar
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

```
[1] pry(main)> class Foo; end
=> nil
[2] pry(main)> class Bar; include Foo; end
TypeError: wrong argument type Class (expected Module)
from (pry):2:in `include'
[3] pry(main)> Foo.class
=> Class
```

Ok, makes sense. *Foo*'s class is *Class*, not *Module*.

Let's contrast this with a proper module:

```
[1] pry(main)> module Mod; end
=> nil
[2] pry(main)> class Foo; include Mod; end
=> Foo
[3] pry(main)> Mod.class
=> Module
```

Now let's try to emulate the code I showed you from *Mutant*:

```
[1] pry(main)> class Foo < Module; end
=> nil
[2] pry(main)> class Bar; include Foo; end
TypeError: wrong argument type Class (expected Module)
from (pry):2:in `include'
[3] pry(main)> class Bar; include Foo.new; end
=> Bar
[4] pry(main)> Foo.class
=> Class
```

Now it gets even more confusing - *include Foo* doesn't work, *include Foo.new* does but the class of *Foo* is still *Class* not *Module*.  What's happening here?

Let's see how Rubinius actually handles this in [module.rb](https://github.com/rubinius/rubinius/blob/master/core/module.rb#L471) (the behaviour below is identical to MRI Ruby):

```Ruby
def include?(mod)
  if !mod.kind_of?(Module) or mod.kind_of?(Class)
    raise TypeError, "wrong argument type #{mod.class} (expected Module)"
  end
  # snip
end
```

There it is. If the argument is not a module __or__ if it's a class we raise.

Now we understand why *include Foo* doesn't work - because it's class is *Class* - but *include Foo.new* does not raise - because its class is *Foo*, not *Class*.

This elegant trick of having a class inherit from *Module* allows us to use a class in a kind-of module context and to use a module in a kind-of class context.

### [Procto](https://github.com/snusnu/procto)

Turns your ruby object into a method object. The code example from the README speaks for itself so I'll just paste it here:

```Ruby
class Printer
  include Procto.call(:print)

  def initialize(text)
    @text = text
  end

  def print
    "Hello #{@text}"
  end
end

Printer.call('world') # => "Hello world"
```

The code to make this work is surprisingly small and very elegant.

Let's start with looking at this part

```Ruby
Procto.call(:print)
```

from the example above without paying attention to what happens when it is included:

```Ruby
class Procto < Module
  # The default name of the instance method to be called
  DEFAULT_NAME = :call

  def self.call(name = DEFAULT_NAME)
    new(name.to_sym)
  end
  # snip
end
```

We define a singleton method called *call* that takes a name and returns an instance of *Procto*. (Using the same "include a kind-of class as module trick" *Concord* used above).

How does the constructor look like?

```Ruby
def initialize(name)
  @block = ->(*args) { new(*args).public_send(name) }
end
```

Here things start to get interesting. We use a lambda (via the stabby lambda syntax) that takes an arbitrary number of arguments. And then it calls *new* with those. What *new* does it call? The block is executed at runtime and in the scope of....the object that *Procto* is included into, so the *new* here is not *Procto#new* but rather the object it has been included into.

In other words, the lambda will create a new object of whatever it is included into and call whatever *name* we pass into it.

That's basically the

```Ruby
Procto.call(:print)
```

part.

So what happens when this is included?

```Ruby
class Procto < Module
  # snip
  def included(host)
    host.instance_exec(@block) do |block|
      define_singleton_method(:call, &block)
    end
  end
end
```

We're using the *included* callback here (note that it is __not__ *self.included* here due to  using the kind-of-class-module set up).

In this callback, we're using *instance_exec* on the host class, which would be *Printer* from the example above. This effectively puts us at class scope.

Furthermore we pass our *@block* instance variable to *[BasicObject#instance_exec](http://ruby-doc.org/core-2.3.0/BasicObject.html#method-i-instance_exec)* which *instance_exec* then passes as block-local variable *block* to the, uhm, block (sorry, that's a lot of "block" in one sentence, I know).

Note that this is not a typo: We need to pass *@block* explicitly to *instance_exec* because instance variables will not be visible in the block that *instance_exec* takes (for details see [here](https://stackoverflow.com/questions/3071532/how-does-instance-eval-work-and-why-does-dhh-hate-it?lq=1), [here](https://groups.google.com/forum/#!msg/ruby-sunspot/5xanVMY6ekQ/JSc-3olBRNAJ) or [here](http://www.skorks.com/2013/03/a-closure-is-not-always-a-closure-in-ruby/)).

Now by using *[Object.define_singleton_method](http://ruby-doc.org/core-2.3.0/Object.html#method-i-define_singleton_method)* we define the singleton method *call* (or, in layman's terms: class method) so we can finally do

```Ruby
Printer.call('world')
```

### [Equalizer](https://github.com/dkubb/equalizer)

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

>>
A Value Object is an object that describes some characteristic or attribute but carries no concept of identity.
As there is no identity, two Value Objects are equal when all their attributes are equal.
An example of a Value Object would be Money.

*Value objects* are heavily used in methodologies like [Domain Driven Design](https://en.wikipedia.org/wiki/Domain-driven_design) or when using *composed_of* in Rails (for a great introduction to this whole topic, check out [this article series](http://victorsavkin.com/ddd) from Victor Savkin).

How does equalizer do this?

Let's look again at how it is included:

```Ruby
include Equalizer.new(:latitude, :longitude)
```

Pattern looks familiar, doesn't it?

Ok, let's check out the initializer:

```Ruby
def initialize(*keys)
  @keys = keys
  define_methods
  freeze
end
```

Nothing really to see here so let's check out *define_methods*:

```Ruby
def define_methods
  define_cmp_method
  define_hash_method
  define_inspect_method
end
```

Let's focus on *define_cmp_method* and ignore the other two methods for now:

```Ruby
def define_cmp_method
  keys = @keys
  define_method(:cmp?) do |comparator, other|
    keys.all? do |key|
      __send__(key).public_send(comparator, other.__send__(key))
    end
  end
  private :cmp?
end
```
This whole chunk of code looks a little more complex than it actually is:

We define a method called *cmp?* that takes something that will denote the actual comparison and then *other*, so the thing we are comparing it to.

We then check all keys (read: attributes) and if they satisfy this monster:

```Ruby
__send__(key).public_send(comparator, other.__send__(key))
```

What's happening here?

- ```__send__(key)``` will basically just return the attribute *key* points to
- on this value we now call the *comparator*
- and pass in the value of the same attribute for the object we're comparing to
- so if we assume *==* as *comparator* for example this whole line boils down to
  ```
    attr_on_object == attr_on_other_object
  ```

Not so complicated anymore, is it?

With this out of the way, how is it used?

```Ruby
def ==(other)
  # snip
  other.kind_of?(self.class) && cmp?(__method__, other)
end
```

So first we check if the objects have the same class.
Then comes the interesting part: We call *cmp?* and pass ```__method__``` to it which is a special identifier that Ruby sets up for when you enter a method and that gives you the actual method name back.
So we could have also written:

```Ruby
cmp?(:==, other)
```

Why on earth would you then just not write it that way, you might ask?

There are multiple reasons for this. First and foremost, it communicates intent clearer. Whoever reads this just __knows__ that you are referring to the same thing. Second, it makes it easier to refactor - in case you would change *==* to something else you could leave the rest of this method untouched.

With this quick digression, let's come back to this one:


```Ruby
def ==(other)
  # snip
  cmp?(__method__, other)
end
```

Taking into account what we talked about above it's now obvious what it does: It takes the *other* object we're comparing our current object to and calls *==* for comparison of all attributes both objects have in common.

### Wrapping it up

So what did all 3 gems have in common?

The top level class was inheriting from *Module* so if you take only one thing away from this post it should be this neat little trick that kind of blurs the distinction between classes and modules in Ruby even further and that can help to increase the elegance of your code significantly in my mind.

That's it for part of this series. In the upcoming part 2 I will be looking at the [Adamantium gem](https://github.com/dkubb/adamantium), the [ast gem](https://github.com/whitequark/ast) and the [abstract_type gem](https://github.com/dkubb/abstract_type).
