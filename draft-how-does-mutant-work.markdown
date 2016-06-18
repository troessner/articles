As part of a [presentation](https://github.com/troessner/talks/blob/master/exploiting_rubys_ast_a_love_story_in_three_chapters/presentation.html) I'm hoping to give at the end of this year about abstract syntax trees and how you can leverage them I started to look into how the [Mutant](https://github.com/mbj/mutant) gem works. `Mutant` is the by far most advanced gem for doing mutation testing in Ruby. I was (and still am) amazed how incredibly well built Mutant is on all levels, ranging from the "big picture" architecture down to the "nuts and bolts" source code.

But let's quickly talk about what mutation testing is first. Mutation testing takes code like this:

```Ruby
class Greeter
  def initialize(phrase)
    @enabled = true
    @phrase = phrase
  end

  def say_hello(options)
    "#{@phrase} #{options[:name]}" if @enabled
  end
end
```

and turns it into something like this:

```Ruby
if true
  "#{@phrase}#{" "}#{options[:name]}"
end
```

or 

```Ruby
if @enabled
  "#{@phrase}#{" "}#{options.fetch(:name)}"
end
```

Mutant then runs your tests against each mutation:

```
bundle exec mutant --include lib/\
                   --require ast_talk_samples\
                   --use rspec  "Greeter*"

  Mutant configuration:
  Matcher:         #<Mutant::Matcher::Config match_expressions: [Greeter*]>
  Integration:     Mutant::Integration::Rspec
  Expect Coverage: 100.00%
  Jobs:            4
  Includes:        ["lib/"]
  Subjects:        2                   
```

Failing tests kill the mutants while surviving mutants show a lack of test coverage.
This is how a surviving mutant would be indicated:

```
  evil:Greeter#say_hello:/Users/timo/dev/ast_talk_samples/lib/mutant/greeter.rb:7:80e67
  @@ -1,6 +1,6 @@
   def say_hello(options)
  -  if @enabled
  +  if true
       "#{@phrase}#{" "}#{options[:name]}"
     end
   end
```

This means `Mutant` replaced the `@enabled` with an unconditional `true` and none of our tests broke.

When it comes to mutation testing I learned one thing. It's complex. Really complex. And as a consequence, `Mutant` is complex. 
I did learn a lot from reading it's source and I'd like to share this with you.
Beware though, this is going to be a long ride - but at the end, it'll be worth your while.

While doing research for my presentation, I wrote an introductory series of articles about the gems `Mutant` uses ([part 1](https://troessner.svbtle.com/lessons-learned-from-some-of-the-best-ruby-codebases-out-there-part-1), [part 2](https://troessner.svbtle.com/lessons-learned-from-some-of-the-best-ruby-codebases-out-there-part-2) and [part 3](https://troessner.svbtle.com/lessons-learned-from-some-of-the-best-ruby-codebases-out-there-part-3)). It probably makes sense if you check the series out first to be able to fully follow everything down below.

Ok. Ready? Then let's roll.

### Our entry point: bin/mutant

The great thing about gems with a binary is that it's a lot easier to start exploring them - just start at the binary.

Let's look at bin/mutant:

```Ruby
require 'mutant'

namespace =
  if ARGV.include?('--zombie')
    $stderr.puts('Running mutant zombified!')
    Mutant::Zombifier.call(
      namespace:        :Zombie,
      load_path:        $LOAD_PATH,
      kernel:           Kernel,
      pathname:         Pathname,
      require_highjack: Mutant::RequireHighjack
        .method(:call)
        .to_proc
        .curry
        .call(Kernel),
      root_require:     'mutant',
      includes: %w[
        mutant
        unparser
        morpher
        adamantium
        equalizer
        anima
        concord
      ]
    )
    Zombie::Mutant
  else
    Mutant
  end

Kernel.exit(namespace::CLI.run(ARGV))
```

Ok, that looks friggin complicated already. Let's keep things simple and ignore everything that relates to the, uhm, "zombie". This would warrant a blog post of its own. 
Here's our new executable:

```Ruby
require 'mutant'

namespace =
  if ARGV.include?('--zombie')
    # Stuff we are going to ignore
    # ...snip
    Zombie::Mutant
  else
    Mutant
  end

Kernel.exit(namespace::CLI.run(ARGV))
```

That looks a lot easier to understand. So we're basically just using the regular `Mutant` namespace in case we didn't do the zombie thing so basically it boils down to the last line looking like this:


```Ruby
Kernel.exit(Mutant::CLI.run(ARGV))
```

which translates to:

* Pass all of our CLI arguments to `Mutant::CLI.run`
* `Mutant::CLI.run` runs the program and then returns the exit status
* Which we pass to `Kernel#exit` that will cause our process to terminate with this very exit status

Let's check out `Mutant::CLI.run`:

```Ruby
module Mutant
  class CLI
    include Adamantium::Flat, Equalizer.new(:config), Procto.call(:config)

    def self.run(arguments)
      Runner.call(Env::Bootstrap.call(call(arguments))).success?
    rescue Error => exception
      $stderr.puts(exception.message)
      false
    end
  end
end
```

The `include` line is what we need to understand first:

```Ruby
include Adamantium::Flat, Equalizer.new(:config), Procto.call(:config)
```

* `Adamantium::Flat`: This makes instances of this class immutable using the [Adamantium gem](https://github.com/dkubb/adamantium) (for a low level introduction check out [this blog post](https://troessner.svbtle.com/lessons-learned-from-some-of-the-best-ruby-codebases-out-there-part-3)))
* `Equalizer.new(:config)`: This defines equality, equivalency and hash methods automatically for whatever `config` returns and is based on the [Equalizer](https://github.com/dkubb/equalizer) gem
* `Procto.call(:config)`: [Procto](https://github.com/snusnu/procto) turns your Ruby object into a method object. Please check out [this blog post](https://troessner.svbtle.com/lessons-learned-from-some-of-the-best-ruby-codebases-out-there-part-3)) to read up on it.

It is important that you memorize this, at least the `Procto` pattern because `Mutant` makes heavy use of those gems in almost every module.

Now to `run`:

```Ruby
module Mutant
  class CLI
    def self.run(arguments)
      Runner.call(Env::Bootstrap.call(call(arguments))).success?
    rescue Error => exception
      $stderr.puts(exception.message)
      false
    end
  end
end
```

Narrowing it down further this is the line we're interested in:

```Ruby
Runner.call(Env::Bootstrap.call(call(arguments))).success?
```

That line does pack quite a punch, doesn't it?

Let me reformat this line to make clearer what's happening here:

```Ruby
Runner.call(
  Env::Bootstrap.call(
    call(arguments)
  )
).success?
```

Starting from the inside

```Ruby
  call(arguments)
```

translates to:

```Ruby
CLI.new(arguments).config
```

This is the part that `Procto` provides.

So:

* we're getting back a config based on our CLI arguments
* then pass that to `Env::Bootstrap.call`
* pass whatever comes back from that to our Runner

**This leads us to 2 questions:**

1.) What is the `config` and what is `Env::Bootstrap`?

2.) What does the `Runner` do with this environment.

### Configuration

If you check out the constructor of `Mutant::CLI` you'll see this:

```Ruby
def initialize(arguments)
  @config = Config::DEFAULT

  parse(arguments)
end
```

Ok, nice and easy. First we'll set the configuration to our defaults, then we're parsing the cli arguments and overwrite our configuration based on this.

What does the default configuration look like?
For that you'll have to check out the top level library file, so lib/mutant.rb:


```Ruby
module Mutant
  # Reopen class to initialize constant to avoid dep circle
  class Config
    DEFAULT = new(
      debug:             false,
      expected_coverage: Rational(1),
      # snip
      fail_fast:         false,
      includes:          EMPTY_ARRAY,
      # snip
      kernel:            Kernel,
      load_path:         $LOAD_PATH,
      matcher:           Matcher::Config::DEFAULT,
      open3:             Open3,
      pathname:          Pathname,
      requires:          EMPTY_ARRAY,
      reporter:          Reporter::CLI.build($stdout),
      zombie:            false
    )
  end # Config
end # Mutant
```

I did remove a lot of the config object above to keep it readable. I just want to highlight a couple of lines I find interesting:

```Ruby
expected_coverage: Rational(1)
```

This means that the expected coverage is 100%. Yep, 100%.

```Ruby
reporter:          Reporter::CLI.build($stdout)
```

The default reporter. Keep that in mind, we'll come back to this later.

```Ruby
load_path:         $LOAD_PATH,
requires:          EMPTY_ARRAY,
pathname:          Pathname,
```

What I like here is how configurable `Mutant`. You can even inject another $LOAD_PATH, additional requires or another Pathname'esque library.

**The big picture for now:**

TODO: proper picture

bin -> cli -> config

### Environment

So where were we?
Keep in mind that we are still talking about this:

```Ruby
Runner.call(
  Env::Bootstrap.call(
    call(arguments) # -> This part we just discussed.
  )
).success?
```

Now that we know what 

```Ruby
call(arguments) # -> This part we just discussed.
```

does, let's look at:

```Ruby
Env::Bootstrap.call(
  call(arguments) # -> This part we just discussed.
)
```

Again, this is `Procto` at work.

```Ruby
Env::Bootstrap.call configuration
```

means

```Ruby
 # "some_method" is what we configure when including Procto
Env::Bootstrap.new(configuration).some_method
```

So let's check out the class declaration of Env::Bootstrap with a focus on the initializer:

```Ruby
module Mutant
  class Env
    class Bootstrap
      include Procto.call(:env) # -> That's the "some_method" from above

      def initialize(*)
        super
        @parser = Parser.new
        infect
        initialize_matchable_scopes
      end
    end
  end
end
```

Step by step:

* `super`: TODO: Explain Concord?
* `Parser.new`: `Mutant` uses the awesome `[parser](https://github.com/whitequark/parser)` gem for producing ASTs from source code and has its own thin wrapper around
* `infect`: TODO
* `initialize_matchable_scopes`: TODO

Now that we know how the initializer works we know half of the story of the call to 

```Ruby
Runner.call(
  Env::Bootstrap.call(
    call(arguments)
  )
).success?
```

But what happens when we call `Env::Bootstrap.call`? If you remember the `include` statement in the class declaration:

```Ruby
module Mutant
  class Env
    class Bootstrap
      include Procto.call(:env)

      def initialize(*)
        # snip, we just discussed this
      end
    end
  end
end
```

you can see that we actually call `env` on our `Env::Bootstrap` object:

```Ruby
  def env
    subjects = matched_subjects
    Env.new(
      actor_env:        Actor::Env.new(Thread),
      config:           config,
      integration:      @integration,
      matchable_scopes: matchable_scopes,
      mutations:        subjects.flat_map(&:mutations),
      parser:           parser,
      selector:         Selector::Expression.new(@integration),
      subjects:         subjects
    )
  end
```

TODO: Explain that :)

```
TODO: Debug your way through in cli.rb via pry
args = call(arguments); nil
env = Env::Bootstrap.new args
cd env
wtf are matched_subjects?
```

**The big picture now:**

TODO: proper picture

bin -> cli -> config -> environment -> Runner


### The Runner
