<!-- Uncomment below and remove the official heading before publishing via octopress -->
<!--
---
layout: post
title: Lessons learned from some of the best Ruby codebases out there (part 3)
date: TODO
comments: true
categories: ruby
keywords: ruby, gems, mutant
description: Lessons learned from some of the best Ruby codebases out there (part 3)
author: Timo Rößner
---
-->

Welcome to the third part of the series - you can find the first part [here](https://tech.blacklane.com/2016/04/23/lessons-learned-from-some-of-the-best-ruby-codebases-part-1/) and the second part [here](https://tech.blacklane.com/2016/05/04/lessons-learned-from-some-of-the-best-ruby-codebases-part-2/).

In this part I will again focus on some other highly interesting gems [Mutant](https://github.com/mbj/mutant) uses, in particular the [Abstract Type](https://github.com/dkubb/abstract_type) gem and the [Adamantium](https://github.com/dkubb/adamantium) gem.

<!--more-->

### [Abstract Type](https://github.com/dkubb/abstract_type)

The `Abstract Type` gem allows you to declare [abstract type](https://en.wikipedia.org/wiki/Abstract_type) classes and modules in an unobstrusive way:

>>
In programming languages, an abstract type is a type in a nominative type system that cannot be instantiated directly. Abstract types are also known as existential types.An abstract type may provide no implementation, or an incomplete implementation. Often, abstract types will have one or more implementations provided separately, for example, in the form of concrete subclasses that can be instantiated. It may include abstract methods or abstract properties that are shared by its subtypes.

When would you use an abstract type?

Think about your typical web shop. You might have a base user class with some common methods:

```Ruby
class User
  def full_name
    "#{first_name} + #{last_name}"
  end
end
```

And then you have 2 other, more concrete classes like this:

```Ruby
class RegisteredUser < User
  def login
    # Login via email / password
  end
end

class GuestUser < User
  def login
    # login via ephemeral link
  end
end
```

You never intend to actually instantiate an object of a `User`, only of `RegisteredUser` and `GuestUser`, so `User` is an abstract type (this is also called the [template method pattern](https://en.wikipedia.org/wiki/Template_method_pattern)).
However in its current state every other user of your classes, e.g. your fellow programmer sitting next to you, could instantiate this class accidentally and introduce a subtle bug in your application since this object would probably be usable in some senses but in other not.

With `Abstract Type` you would fix this like this:

```Ruby
class User
  include AbstractType

  # Declare abstract instance method
  abstract_method :login

  def full_name
    "#{first_name} + #{last_name}"
  end
end

User.new  # raises NotImplementedError: User is an abstract type
```

This will not only prevent bugs like above but also - and for me this is even more important - communicates your intention clearly. There's no guessing here. It's immediately clear that `User` is not supposed to be instantiated.

At this point you might think "But wait a sec, if you can't instantiate a user, what's the point of that `abstract_method` call?".

Excellent question!

Ruby being, well, Ruby, there are of course multiple ways to do think. You could use [allocate](http://ruby-doc.org/core-2.2.3/Class.html#method-i-allocate) for instance:

```Ruby
user = User.new  # raises NotImplementedError: User is an abstract type
# Ok, let's go with `allocate`
user = User.allocate
# => #<User:0x007feab3af8e30>
user.login
# NotImplementedError: User#login is not implemented
```

So how does it work? Let's start with the `include` statement and then work our way to `abstract_method`.

When you call


```Ruby
include AbstractType
```

in your class this will prompt the Ruby runtime to call the [`included`](http://ruby-doc.org/core-2.3.0/Module.html#method-i-included) callback that is defined at the top level of the gem:

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

With `create_new_method` looking like this:

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

Ok, there is a lot going on here, so let's go through it step by step:

We're dynamically overwriting [`Class.new`](http://ruby-doc.org/core-2.3.0/Class.html#method-i-new) in the class that we intend to make abstract. The important part here is that this `new` is also what will be used when we call `new` on subclasses but it doesn't affect the "original" `new` in all other, unrelated classes. That's because of Ruby's "one step to the right, then up the ancestor chain" method lookup.

If you're confused now I'd recommend to stop reading this article and to read up on the [Ruby object model](https://thesocietea.org/2015/08/metaprogramming-in-ruby-part-1/) and especially on [Ruby's method lookup](http://tech.tulentsev.com/2012/10/object-extensions-and-method-lookups-in-ruby/) and then return here.

Ok, ready to continue?

In the new, uhm, `new` we fail if `self` is equal to the abstract class. The equality check here is based on [`BasicObject#equal?`](http://ruby-doc.org/core-2.3.0/BasicObject.html#method-i-equal-3F) which checks for strict identity. Also note that we are not using `raise` here but `fail` instead (which is just an alias for `raise`), something that is kind of [controversial](https://stackoverflow.com/questions/31937632/fail-vs-raise-in-ruby-should-we-really-believe-the-style-guide) in the Ruby community.

When we're in one of the subclasses of our abstract class this guard will fail in which case we're just delegating to the original [`Class.new`](http://ruby-doc.org/core-2.3.0/Class.html#method-i-new).

With this out of the way let's get back to the `included` callback:


```Ruby
def self.included(descendant)
  super
  create_new_method(descendant)
  descendant.extend(AbstractMethodDeclarations)
end
```

The `descendant` (read: "base class" this module is included in) is extending the `AbstractMethodDeclarations` module here, thus adopting its methods as singleton methods (or class methods in layman's terms).

One of those methods is the `abstract_method` we saw in the example at the beginning:

```Ruby
def abstract_method(*names)
  names.each(&method(:create_abstract_instance_method))
  self
end
```

This is the method that allows you to declare your abstract method a la:

```Ruby
abstract_method :foo, :bar
```

A bunch of interesting things going on here. First of all, we're using a neat little trick that is kind of the reverse of Ruby [Symbol#to_proc feature](http://benjamintan.io/blog/2015/03/16/how-does-symbol-to_proc-work/).

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

and heavily relies on [`Method#to_proc`](http://ruby-doc.org/core-2.3.0/Method.html#method-i-to_proc). You can read up on how this works [here](https://andrewjgrimm.wordpress.com/2011/10/03/in-ruby-method-passes-you/).

The `self` at the end

```Ruby
def abstract_method(*names)
  names.each(&method(:create_abstract_instance_method))
  self
end
```

allows you to chain calls to `abstract_method` (of which I don't really see the point here since you rarely chain class macros, but I guess the author follows his own conventions in this case).

And what does `create_abstract_instance_method` do?

```Ruby
def create_abstract_instance_method(name)
  define_method(name) do |*|
    fail NotImplementedError, "#{self.class}##{name} is not implemented"
  end
end
```

It defines an abstract instance method that will do nothing but .... fail with a proper error message.
Exactly what we need to implement abstract types.
Note that there's also a sister method to `abstract_method` that's called `abstract_singleton_method` which does the same thing for singleton methods which I'm not showing here for the sake of brevity.

### [Adamantium](https://github.com/dkubb/adamantium)

`Adamantium` allows you to make objects immutable in a simple, unobtrusive way.

Sounds a lot like the [IceNine](https://github.com/dkubb/ice_nine) gem I covered in the [last part of the series](https://tech.blacklane.com/2016/05/04/lessons-learned-from-some-of-the-best-ruby-codebases-part-2/), doesn't it?

Well, it kind of is and it kind of isn't. `Adamantium` uses `IceNine` under the hood for the actual freezing. The real value of `Adamantium` comes from offering high level strategies that you can easily apply to your objects where `IceNine` is more of the functional base library.

Why would you need immutable objects?

With the resurgence of functional programming these days people talk a lot about "immutable data". Immutable data structures are one of the core tenets of functional programming and something that is notoriously hard to get right in Ruby.

Imagine you have a bank account model like this in Ruby:

```Ruby
class Account
  attr_reader :balance, :interest

  def initialize
    @balance = 10 # every new customer gets 10 Euro upon account creation
    @interest = 1.05
  end

  def apply_yearly_interest
    @balance = balance * interest
  end
end
```

In its current form, every other piece of code in your application could mutate it like this:

```Ruby
account = Account.new
# => #<Account:0x007fe7d1611e88 @balance=0>
account.balance
# => 10
account.instance_variable_set :@interest, 100

account.apply_yearly_interest
account.balance
# => 1000 # Whoopsie
```

`Adamantium` to the rescue!

```Ruby
require 'adamantium'

class Account
  include Adamantium
  attr_reader :balance, :interest
  memoize :interest
  # snip
end
```

Now watch what happens when I try my shenanigans from before again:

```Ruby
account = Account.new
# => #<Account:0x007fe7d1611e88 @balance=0>
account.balance
# => 10
account.instance_variable_set :@interest, 100
# RuntimeError: can't modify frozen #<Class:#<Account:0x007ff259889390>>
# from (pry):20:in `instance_variable_set'
```

Basically `Adamantium` offers 3 strategies for freezing your objects:

- deep
- flat
- noop

The default strategy is `deep`. Here's the example from the README:

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

And now let's contrast this with the `flat` strategy:

```Ruby
class FlatExample
  include Adamantium::Flat

  # Memoized method with flat frozen value (default with Adamantium::Flat)
  # Example:
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

For the sake of brevity, we'll exclude the `noop` mode and focus on the `deep` and `flat` mode.

Let's start our investigation with the `included` callback for the `Adamantium` module:

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

Ok, `included` gets passed in the class it is included on called `descendant` and then re-opens this class using `class_eval`.
We then include the [`Memoizable`](https://github.com/dkubb/memoizable) module (which I won't cover here) and extend `ModuleMethods` and `ClassMethods`.

We'll start with checking out `ModuleMethods`:

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

The `freezer` methods determines what strategy to use. As I already mentioned above you can see that `deep` is the default strategy. And then there's the `memoize` method you saw being used above in our example from the README.
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
def foo(*names, hashie = {}); end
# => SyntaxError: unexpected '=', expecting ')'
```

You *could* achieve the same using Ruby's keyword arguments like this:

```Ruby
def foo(*names, hashie: {}); end
```

but I assume this code was written with Ruby < 2 in mind which didn't support keyword arguments.

Next we determine the freezer to use via the options or use the default freezer (which is `deep` like already mentioned):

```Ruby
method_freezer = Freezer.parse(options) || freezer
```

We then iterate over __all__ the methods the object in question has and freeze them via `memoize_method`

```Ruby
methods.each { |method| memoize_method(method, method_freezer) }
```

only to return `self` at the end

```Ruby
self
```

so we can chain the `memoize` calls if we like.

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

Ok, so the deep freezing is delegated to [IceNine](https://github.com/dkubb/ice_nine) on which you can read up [here](https://tech.blacklane.com/2016/05/04/lessons-learned-from-some-of-the-best-ruby-codebases-part-2/).

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

`Deep` inherits from `self`. And what is `self` here?

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

But why are we doing this here instead of being explicit? It's making our intention explicit without duplicating names across our module. And even if you rename `Freezer` to something else, this line doesn't change, which makes it easier to refactor and less error-prone to subtle bugs.

Is it a huge deal? No. It isn't. But in a larger code base small things like can make the difference between "barely maintainable" and "updating functionality is a walk in the park".

Before we wrap this up here let's look at one last thing:

We saw how the default strategy is applied. What about that the `flat` strategy?

A quick reminder to how applying and using the `flat` strategy looked like:

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

As you can see, you just include the `Flat` submodule of `Adamantium`.

How is this implemented?

Let's look at the `included` callback:

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

We're calling `instance_exec` on the descendant and pass `self` in. What's `self` now?
It's the module constant, so...`Adamantium::Flat`.
We then include `Adamantium`. That is something we already covered above in this article. So business as usual.
But then we extend `mod`, so `Adamantium::Flat`. Why? Because this basically overwrites the `freezer` methods that we imported in our class by doing

```Ruby
include Adamantium
```

so that this now returns the `flat` strategy

```Ruby
def freezer
  Freezer::Flat
end
```

not the `deep` strategy

```Ruby
def freezer
  Freezer::Deep
end
```

anymore.

So long story short, this is just a very advanced configuration mechanism. Granted, it might seem a little complicated but at the same time it completely separates the modules from each other making it easier to update one without updating the other.

### Wrapping it up

What we discussed today:

* Use `AbstractType` for base classes  you do not intend to instantiate
* The most popular application for abstract types is probably the [template method pattern](https://en.wikipedia.org/wiki/Template_method_pattern)
* Immutable data can prevent subtle bugs and makes it easier to reason about code if you know that the underlying data can not change
* `Adamantium` offers high level strategies for making your objects immutable

That's it for part of this series. Both gems we looked at in this article are pretty small, but packed with a lot of features.

In the upcoming part 4 I will be looking at the [enterprise](https://github.com/tenderlove/enterprise) gem from Aaron Patterson because currently the Ruby eco system is definitely not leveraging the power of xml.
