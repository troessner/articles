## Lessons learned from some of the best ruby codebases out there - part 2

Welcome to the second part of the series (you can find the first part [here]()).

Let's dive right into it and look at some other highly interesting gems [Mutant](https://github.com/mbj/mutant) uses.

### [abstract_type](https://github.com/dkubb/abstract_type)

The *abstract_type* allows to declare [abstract type](https://en.wikipedia.org/wiki/Abstract_type) classes and modules in an unobstrusive way:

>>
In programming languages, an abstract type is a type in a nominative type system that cannot be instantiated directly. Abstract types are also known as existential types.An abstract type may provide no implementation, or an incomplete implementation. Often, abstract types will have one or more implementations provided separately, for example, in the form of concrete subclasses that can be instantiated. It may include abstract methods or abstract properties[2] that are shared by its subtypes.

The example from the [README](https://github.com/dkubb/abstract_type) is pretty much self-explanatory:

```Ruby
class Foo
  include AbstractType

  # Declare abstract instance method
  abstract_method :bar

  # Declare abstract singleton method
  abstract_singleton_method :baz
end

Foo.new  # raises NotImplementedError: Foo is an abstract type
Foo.baz  # raises NotImplementedError: Foo.baz is not implemented

# Subclassing to allow instantiation
class Baz < Foo; end

object = Baz.new
object.bar  # raises NotImplementedError: Baz#bar is not implemented
```

So how does it work? Let's start with the *include* statement and then work our way to where we actually use *abstract_method* and *abstract_singleton_method*.

When you call


```Ruby
include AbstractType
```

in your class this will prompt the Ruby runtime to call the *included* callback:

```Ruby
def self.included(descendant)
  super
  create_new_method(descendant)
  descendant.extend(AbstractMethodDeclarations)
end
```

TODO: What's the point of calling super above?

With *create_new_method* looking like this:

```Ruby
def self.create_new_method(abstract_class)
  abstract_class.define_singleton_method(:new) do |*args, &block|
    if equal?(abstract_class)
      fail NotImplementedError, "#{inspect} is an abstract type"
    else
      super(*args, &block)
    end
  end
end
```

We're dynamically overwriting [*Class.new*](http://ruby-doc.org/core-2.2.0/Class.html#method-i-new) in the class that we intend to make "abstract". The important part here is that this *new* is also what will be used when we call *new* on subclasses.

In the new, uhm, *new* we fail if *self* is equal to the abstract class. The equality check here is based on [*BasicObject#equal?](http://ruby-doc.org/core-2.2.0/BasicObject.html#method-i-equal-3F) which checks for strict identity. Also note that we are not using *raise* here but [*fail*](https://stackoverflow.com/questions/31937632/fail-vs-raise-in-ruby-should-we-really-believe-the-style-guide) instead.

When we're in one of the subclasses of abstract_class this guard will fail in which case we're just delegating to the original *Class.new*.

A lot of meat here for very little code. If you're confused now I'd recommend to read up on the [Ruby object model](https://thesocietea.org/2015/08/metaprogramming-in-ruby-part-1/).

With this out of the way let's get back to the *included* callback:


```Ruby
def self.included(descendant)
  super
  create_new_method(descendant)
  descendant.extend(AbstractMethodDeclarations)
end
```

The *descendant* is extending the *AbstractMethodDeclarations* module here, thus adopting its methods as singleton methods:

```Ruby
def abstract_method(*names)
  names.each(&method(:create_abstract_instance_method))
  self
end
```

This is the method (or "class macro") that allows you to declare your abstract method a la:

```Ruby
abstract_method :foo, :bar
```

A bunch of interesting things going on here. First of all, we're using a neat little trick that is kind of the reverse of Ruby [Symbol#to_proc feature](https://stackoverflow.com/questions/14881125/what-does-to-proc-method-mean).

This

```Ruby
names.each(&method(:create_abstract_instance_method))
```

basically means

```Ruby
names.each do |name|
  create_abstract_instance_method name
end
```

You can read up on how this works [here](https://andrewjgrimm.wordpress.com/2011/10/03/in-ruby-method-passes-you/).

And what does *create_abstract_instance_method* do?

```Ruby
def create_abstract_instance_method(name)
  define_method(name) do |*|
    fail NotImplementedError, "#{self.class}##{name} is not implemented"
  end
end
```

It defines an abstract instance method that will do nothing but .... fail with a proper error message.
Exactly what we need to implement abstract types.
Furthermore there's a sister method to *abstract_method* that's called *abstract_singleton_method* which does the same thing for singleton methods.

### [adamantium](https://github.com/dkubb/adamantium)

*adamantium* allows you to make objects immutable in a simple, unobtrusive way.

It offers 3 strategies for doing so:

- deep
- flat
- noop

The default strategy is "deep". Let's look at the example from the README

```Ruby
require 'adamantium'
require 'securerandom'

class Example
  include Adamantium

  # Memoized method with deeply frozen value (default)
  # Example:
  #
  # object = Example.new
  # object.random => ["abcdef"]
  # object.random => ["abcdef"]
  # object.random.frozen? => true
  # object.random[0].frozen? => true
  #
  def random
    [SecureRandom.hex(6)]
  end
  memoize :random
end
```

So how's this working? For the sake of brevity, let's exclude the "noop" mode and focus on the "deep" and "flat" mode.

Let's start with the *included* callback for the *Adamantium* module:

```Ruby
module Adamantium
  def self.included(descendant)
    descendant.class_eval do
      include Memoizable
      extend ModuleMethods
      extend ClassMethods if instance_of?(Class)
    end
  end
  private_class_method :included
end
```

Ok, *included* gets passed in the class it is included on called *descendant* and then re-opens this class using *class_eval*.
We then include the *Memoizable* module (more on that later) and extend *ModuleMethods* and *ClassMethods*.

Let's check out *ModuleMethods*:

```Ruby
module Adamantium
  module ModuleMethods
    def freezer
      Freezer::Deep
    end

    # snip

    def memoize(*methods)
      options        = methods.last.kind_of?(Hash) ? methods.pop : {}
      method_freezer = Freezer.parse(options) || freezer
      methods.each { |method| memoize_method(method, method_freezer) }
      self
    end
  end
end
```

The *freezer* methods determines what strategy to use. As I already mentioned above you can see that "deep" is the default strategy. And then there's the *memoize* method you saw being used above in our example from the README.
There's quite a lot going on in this method so let's step through it line by line:

```Ruby
options = methods.last.kind_of?(Hash) ? methods.pop : {}
```

```Ruby
method_freezer = Freezer.parse(options) || freezer
```

```Ruby
methods.each { |method| memoize_method(method, method_freezer) }
```

```Ruby
self
```

Let's check out *memoize_method* to see how the actual freezing is implemented:

TODO

Ok, now we saw how the default strategy is applied. What about that the *flat* strategy?

Let's first quickly compare the *deep* strategy with the *flat* strategy on a high level:

```Ruby
TODO example from the README
```

How is this implemented?


### ast

That's it for part of this series. In the upcoming part 3 I will be looking at the ice_nine, morpher and diff-lcs gems.
