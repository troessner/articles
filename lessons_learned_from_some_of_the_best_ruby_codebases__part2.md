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

The *descendant* is extending a module here, let's check this out.

*AbstractMethodDeclarations* basically has those methods...

### ast


### adamantium



That's it for part of this series. In the upcoming part 3 I will be looking at the ice_nine, morpher and diff-lcs gems.
