

I recently started looking into the awesome [mutant](https://github.com/mbj/mutant) gem for an [upcoming presentation](https://github.com/troessner/talks/blob/master/exploiting-rubys-ast-a-love-story-in-three-chapters.html) I am working on at the moment.

While browsing the source code of *Mutant* and of the gems that *Mutant* uses I realized that those codebases are among the cleanest code bases I have ever seen in Ruby land with an amazing overall architecture and I'd love to share what I've seen reading their code.

I intend to make this a series with the first part of this series not covering *Mutant* or `synvert* itself but the gems they use.

Let's get started with *Mutant`s [gem dependencies](https://github.com/mbj/mutant/blob/master/mutant.gemspec).

### The *parser* gem

Now that is one of my favourite gem of all times. My beloved [Reek](https://github.com/troessner/reek) would not be what it is without the *parser*. Nor would 80% of all other gems in Ruby land that have something to do with abstract syntax trees.

The *parser* gem is huge though. I intend to cover this in a latter blog post and will skip this for now.

### The [procto](https://github.com/snusnu/procto) gem

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

Ok, we define a singleton method called "call" that takes a name just returns an instance of *Procto*. (But wait! Doesn't *include* only take modules as input? Why does this work on an instance of a class as well? We'll look into this later, for now let's just ignore this)

How does the constructor look like?

```Ruby
  def initialize(name)
    @block = ->(*args) { new(*args).public_send(name) }
  end
```

Here things start to get interesting. We use a lambda (via the stabby lambda syntax) that takes an arbitrary number of arguments. And then it calls *new* with those. What *new* does it call? The block is executed at runtime and in the scope of....the object that *Procto* is included into (so the *new* here is not Procto#new).

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

### Including an instance of a class

[Concord](https://github.com/mbj/concord) is a nice little library that ...TODO...
*Mutant* makes heavy use of `Concord* e.g. like this:

```Ruby
  class Receiver
    include Concord.new(:condition_variable, :mutex, :messages)
  end
````

Ok, something is including a module. Wait, it's not a module. We're calling `new* on it. Wat?
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

Ok, makes sense. `Klazz* class is `class`, not *Module`.
Let's contrast this with a proper module:

```Bash
[1] pry(main)> module Mod; end
=> nil
[2] pry(main)> class Foo; include Mod; end
=> Foo
[3] pry(main)> Mod.class
=> Module
```

Now let's try to emulate the code I showed you from *Mutant`:

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

Now it get even more confusing - `include Klazz* doesn't work, `include Klazz.new* does and the class of `Foo* is still `Class* not *Module`.  What's happening here?

Let's see how Rubinius actually handles this in [module.rb](https://github.com/rubinius/rubinius/blob/master/core/module.rb#L471):

```Ruby
  def include?(mod)
    if !mod.kind_of?(Module) or mod.kind_of?(Class)
      raise TypeError, "wrong argument type #{mod.class} (expected Module)"
    end
    # snip
  end
```

There it is. If the argument is not a module OR if it's a class we raise. Now we understand why `include Klazz* doesn't work (because it's class is `class`) but `include Klazz.new* does not raise (because its class is 'Klazz', not `Class`).

I'm not sure what's the point this whole enchilada but let's just accept it for now ;)

Ok, now with that out of the way, where were we again? Right, *Mutant`.

`concord* uses this trick to pull off .....TODO: Why? It could also just use a singleton_method instead of #new

Additional material:

- https://stackoverflow.com/questions/10558504/can-someone-explain-the-class-superclass-class-superclass-paradox
- https://stackoverflow.com/questions/7675774/the-class-object-paradox-confusion

### [equalizer](https://github.com/dkubb/equalizer)

TODO
