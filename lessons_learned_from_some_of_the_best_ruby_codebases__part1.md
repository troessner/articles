

I recently started looking into the awesome [mutant](https://github.com/mbj/mutant) gem for an [upcoming presentation](https://github.com/troessner/talks/blob/master/exploiting-rubys-ast-a-love-story-in-three-chapters.html) I am working on at the moment.

While browsing the source code of `mutant` and of the gems that `mutant` uses I realized that those codebases are among the cleanest code bases I have ever seen in Ruby land with an amazing overall architecture.

Let's go through the most interesting bits:

### Used gems

TODO

### name < self 

TODO

### Including an instance of a class

[Concord](https://github.com/mbj/concord) is a nice little library that ...TODO...
`Mutant` makes heavy use of `Concord` e.g. like this:

```Ruby
  class Receiver
    include Concord.new(:condition_variable, :mutex, :messages)
  end
````

Ok, something is including a module. Wait, it's not a module. We're calling `new` on it. Wat?
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

Ok, makes sense. `Klazz` class is `class`, not `module`.
Let's contrast this with a proper module:

```Bash
[1] pry(main)> module Mod; end
=> nil
[2] pry(main)> class Foo; include Mod; end
=> Foo
[3] pry(main)> Mod.class
=> Module
```

Now let's try to emulate the code I showed you from `mutant`:

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

Now it get even more confusing - `include Klazz` doesn't work, `include Klazz.new` does and the class of `Foo` is still `Class` not `Module`.  What's happening here?

Let's see how Rubinius actually handles this in [module.rb](https://github.com/rubinius/rubinius/blob/master/core/module.rb#L471):

```Ruby
  def include?(mod)
    if !mod.kind_of?(Module) or mod.kind_of?(Class)
      raise TypeError, "wrong argument type #{mod.class} (expected Module)"
    end
    # snip
  end
```

There it is. If the argument is not a module OR if it's a class we raise. Now we understand why `include Klazz` doesn't work (because it's class is `class`) but `include Klazz.new` does not raise (because its class is 'Klazz', not `Class`).

I'm not sure what's the point this whole enchilada but let's just accept it for now ;)

Ok, now with that out of the way, where were we again? Right, `mutant`.

`concord` uses this trick to pull off .....TODO: Why? It could also just use a singleton_method instead of #new

Additional material:

- https://stackoverflow.com/questions/10558504/can-someone-explain-the-class-superclass-class-superclass-paradox
- https://stackoverflow.com/questions/7675774/the-class-object-paradox-confusion

### Adding the scope after `end`'s

TODO