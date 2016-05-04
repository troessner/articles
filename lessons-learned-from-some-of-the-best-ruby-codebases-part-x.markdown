<!-- Uncomment below and remove the official heading before publishing via octopress -->
<!--
---
layout: post
title: Lessons learned from some of the best ruby codebases out there (part 3)
date: TODO
comments: true
categories: ruby
keywords: ruby, gems, mutant
description: Lessons learned from some of the best ruby codebases out there (part 3)
author: Timo Rößner
---
-->

## Lessons learned from some of the best ruby codebases out there (part 2)

Welcome to the second part of the series (you can find the first part [here](https://tech.blacklane.com/2016/04/23/lessons-learned-from-some-of-the-best-ruby-codebases-part-1/)).

In this part I will again focus on some other highly interesting gems [Mutant](https://github.com/mbj/mutant) uses in particluar the abstract_type](https://github.com/dkubb/abstract_type) gem and the [adamantium](https://github.com/dkubb/adamantium) gem.

<!--more-->

### [abstract_type](https://github.com/dkubb/abstract_type)

The *abstract_type* gem allows you to declare [abstract type](https://en.wikipedia.org/wiki/Abstract_type) classes and modules in an unobstrusive way:

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

in your class this will prompt the Ruby runtime to call the *[included](http://ruby-doc.org/core-2.3.0/Module.html#method-i-included)* callback that is at the top level of the gem:

```Ruby
module AbstractType
  def self.included(descendant)
    super
    create_new_method(descendant)
    descendant.extend(AbstractMethodDeclarations)
  end

  # snip
end
```

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

We're dynamically overwriting *[Class.new](http://ruby-doc.org/core-2.3.0/Class.html#method-i-new)* in the class that we intend to make abstract. The important part here is that this *new* is also what will be used when we call *new* on subclasses.

In the new, uhm, *new* we fail if *self* is equal to the abstract class. The equality check here is based on *[BasicObject#equal?](http://ruby-doc.org/core-2.3.0/BasicObject.html#method-i-equal-3F)* which checks for strict identity. Also note that we are not using *raise* here but *fail* instead (which is just an alias for *raise*), something that is kind of [controversial](https://stackoverflow.com/questions/31937632/fail-vs-raise-in-ruby-should-we-really-believe-the-style-guide) in the Ruby community.

When we're in one of the subclasses of our abstract class this guard will fail in which case we're just delegating to the original *Class.new*.

A lot of meat here for very little code. If you're confused now I'd recommend to read up on the [Ruby object model](https://thesocietea.org/2015/08/metaprogramming-in-ruby-part-1/).

With this out of the way let's get back to the *included* callback:


```Ruby
def self.included(descendant)
  super
  create_new_method(descendant)
  descendant.extend(AbstractMethodDeclarations)
end
```

The *descendant* (read: "base class" this module is included in) is extending the *AbstractMethodDeclarations* module here, thus adopting its methods as singleton methods.

One of those methods is the *abstract_method* we saw in the example at the beginning:

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

and heavily relies on *[Method#to_proc](http://ruby-doc.org/core-2.3.0/Method.html#method-i-to_proc)*. You can read up on how this works [here](https://andrewjgrimm.wordpress.com/2011/10/03/in-ruby-method-passes-you/).

The *self* at the end

```Ruby
def abstract_method(*names)
  names.each(&method(:create_abstract_instance_method))
  self
end
```

allows you to chain calls to *abstract_method* (of which I don't really see the point here, but I guess the author follows his own conventions in this case).

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

The default strategy is *deep*. Let's look at the example from the README

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

So how's this working? For the sake of brevity, let's exclude the *noop* mode and focus on the *deep* and *flat* mode.

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
We then include the *Memoizable* module and extend *ModuleMethods* and *ClassMethods*.

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

The *freezer* methods determines what strategy to use. As I already mentioned above you can see that *deep* is the default strategy. And then there's the *memoize* method you saw being used above in our example from the README.
There's quite a lot going on in this method so let's step through it line by line:

```Ruby
options = methods.last.kind_of?(Hash) ? methods.pop : {}
```

That's a neat trick right there. This allows us to write

```Ruby
memoize :whatever # or...
memoize :whatever, option: :bar
```

and it will still work out.

Note that you couldn't do this just with Ruby's default arguments:

```Ruby
def foo(*names, hashie = {}); # ...; end
# => SyntaxError: unexpected '=', expecting ')'
```

You __could__ achieve the same using Ruby's keyword arguments like this:

```Ruby
def foo(*names, hashie: {});  # ...; end
```

but I assume this code was written with Ruby < 2 in mind which didn't support keyword arguments.

Next we determine the freezer to use via the options or use the default freezer (which is *deep* like already mentioned):

```Ruby
method_freezer = Freezer.parse(options) || freezer
```

We then iterate over __all__ the methods the object in question has and freeze them via *memoize_method*

```Ruby
methods.each { |method| memoize_method(method, method_freezer) }
```

only to return *self* at the end

```Ruby
self
```

so we can chain the *memoize* calls if we like.

Let's check out how the actual freezing is done:

```Ruby
module Adamantium
  class Freezer
    class Deep < self
      def self.freeze(value)
        IceNine.deep_freeze!(value)
      end
    end
  end
end
```

Ok, so the deep freezing is delegated to *IceNine*. I'm not going into details now because I plan to cover *IceNine* in one of my next blog posts.

One thing I'd quickly like to talk about though is this:

```Ruby
module Adamantium
  class Freezer
    class Deep < self  # <- Wat?
      # snip
    end
  end
end
```

*Deep* inherits from *self*. And what is *self* here?

```Ruby
class Omg; puts self; class Bar < self; end; end
# => Omg
```

So

```Ruby
class Deep < self
```

is actually a fancy way of writing

```Ruby
class Deep < Freezer
```

But why are we doing this here instead of being explicit? It's making our intention explicit without duplicating names across our module. And even if you rename *Freezer* to something else, this line doesn't change, which makes it easier to refactor and less error-prone to subtle bugs.

Is it a huge deal? No. It isn't. But in a larger codebase small things like can make the difference between "bareley maintainable" and "updating functionality is a walk in the park".

Before we wrap this up here let's look at one last thing:

We saw how the default strategy is applied. What about that the *flat* strategy?

Let's first quickly compare the *deep* strategy with the *flat* strategy on a high level:

```Ruby
class FlatExample
  include Adamantium::Flat
  #
  # object = Example.new
  # object.random => ["abcdef"]
  # object.random => ["abcdef"]
  # object.random.frozen? => true
  # object.random[0].frozen? => false
  #
  def random
    [SecureRandom.hex(6)]
  end
  memoize :random
end
```

As you can see, you just include the *Flat* submodule of *Adamantium*.

How is this implemented?

Let's look at the *included* callback:

```Ruby
module Adamantium
  module Flat
    def freezer
      Freezer::Flat
    end

    def self.included(descendant)
      descendant.instance_exec(self) do |mod|
        include Adamantium
        extend mod
      end
    end
  end
end
```

That's pretty cool. Let me walk you through it:

```Ruby
descendant.instance_exec(self) do |mod|
  # snip
end
```

We're calling *instance_exec* on the descendant and pass *self* in. What's *self* now?
It's the module constant, so...*Adamantium::Flat*.
We then include *Adamantium*. That is something we already covered above in this article. So bussiness as usual.
But then we extend *mod*, so *Adamantium::Flat*. Why? Because this basically overwrites the *freezer* methods that we imported in our class by doing

```Ruby
include Adamantium
```

so that this now returns the *flat* strategy

```Ruby
def freezer
  Freezer::Flat
end
```

not the "deep" strategy

```Ruby
def freezer
  Freezer::Deep
end
```

anymore.

So long story short, this is just a very advanced configuration mechanism. Granted, it might seem a little complicated but at the same time it completely separates the modules from each other making it easier to update one without updating the other.

### Wrapping it up

That's it for part of this series. Both gems we looked at in this article are pretty small, but packed with a lot of features.

In the upcoming part 3 I will be looking at the [ast](https://github.com/whitequark/ast) gem and the [ice_nine](https://github.com/dkubb/ice_nine).
