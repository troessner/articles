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

Mutant then runs your tests against each mutation. The tests might look like this:

```Ruby
# cat spec/greeter_spec.rb

require_relative 'spec_helper'
require_relative '../lib/hello_world/greeter'

RSpec.describe Greeter do
  describe '#say_hello' do
    subject { described_class.new('hola') }

    it 'says hello' do
      name = 'Lili'
      expect(subject.say_hello(name)).to eq('hola Lili')
    end
  end
end
```

So let's say the `Greeter` class from above is part of a gem called "hello_world" which, following RubyGems conventions, has a folder called "hello_world" in "lib/" and in this folder the `Greeter` class file like this:

```
├── Gemfile
├── Gemfile.lock
├── README.md
├── lib
│   ├── hello_world
│   │   └── greeter.rb
│   └── hello_world.rb
└── spec
    ├── greeter_spec.rb
    └── spec_helper.rb
```

Ypu can find the sample application [here](https://github.com/troessner/hello_world) in case you want to check it out for yourself.

This is how you'd call `Mutant` to mutation test this:

```
bundle exec mutant --include lib/\
                   --require hello_world\
                   --use rspec  "Greeter*"
```

We're telling `Mutant` to:

* include the "lib/" directory
* require "hello_world.rb"
* use RSpec as test integration
* use the "Greeter*" namespace to find its test subjects We'll come back to the test subjects so please ignore this for now

`Mutant` will then spit out part of its configuration first:

```Ruby
  Mutant configuration:
  Matcher:         #<Mutant::Matcher::Config match_expressions: [Greeter*]>
  Integration:     Mutant::Integration::Rspec
  Expect Coverage: 100.00%
  Jobs:            4
  Includes:        ["lib/"]
  Subjects:        2                   
```

And the it will run your tests against the code mutations. Failing tests kill the mutants while surviving mutants show a lack of test coverage.
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

Please keep the example project and `Mutant` call from above in the back of your head, I'll refer to this throughout the article.

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

namespace = Mutant

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

2.) What does the `Runner` do with this environment?

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

This means that the expected default coverage is 100%. Yep, 100%. You can overwrite this though. Suppose your `Mutant` run fails with this result:

```
Mutations: 197
Kills: 178
```

You can make it pass like this:

```
bundle exec mutant --expected-coverage 179/197 # plus the rest of your cli arguments
```

Back to the config:

```Ruby
load_path:         $LOAD_PATH,
requires:          EMPTY_ARRAY,
pathname:          Pathname,
```

What I like here is how configurable `Mutant`. You can even inject another $LOAD_PATH, additional requires or another Pathname'esque library.

But the configurability is actually just a sideeffect here. The main goal is to stop relying on IO globals read from a global scope and injecting them explicitly.

This lists the IO dependencies of each unit much more explicit. Makes testing easier, and allows to write a testing tool that works without message expectations (one target of mine in the near future).

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

* `super`: That might seem odd to the innocent reader. Remember, `Mutant` makes heavy usage of the `[Concord](https://github.com/mbj/concord)` gem.
`Concord` works like this:

```Ruby
TODO
```

So now you should understand why `super` is there. This will basically ...TODO

* `Parser.new`: `Mutant` uses the awesome `[parser](https://github.com/whitequark/parser)` gem for producing ASTs from source code and has its own thin wrapper around. That thin wrapper just maintains a cache, so subjects on the same file share the (immutable) AST.

* `infect`: Infection is the process where `Mutant` processes includes and requires, and "infects" the ruby VM with the "subjects under tests", its the point where the software that is under mutation test gets loaded.

```Ruby
module Mutant
  class Env
    class Bootstrap
      def infect
        config.includes.each(&config.load_path.method(:<<))
        config.requires.each(&config.kernel.method(:require))
        @integration = config.integration.new(config).setup
      end    
    end
  end
end
```

The first line 

```Ruby
config.includes.each(&config.load_path.method(:<<))
```

uses a neat little `Method#to_proc` feature you can read up on [here](https://andrewjgrimm.wordpress.com/2011/10/03/in-ruby-method-passes-you/). 
This line is identical to writing it like this:

```Ruby
config.includes.each do |include|
  config.load_path << include
end
```

but the first version is arguably more concise. So technicalities aside, this will whatever you passed to `Mutant` in your CLI call via the ``--include` flag like shown in the beginning:

```Ruby
  bundle exec mutant\
    --include lib\
    --require reek\
    --use rspec 'Reek*'
```

The next line in `infect`

```Ruby
config.requires.each(&config.kernel.method(:require))
```

does something similar but for our `requires`.

I won't go into detail regarding the last line:

```Ruby
@integration = config.integration.new(config).setup
```

Let's just say that this set up the integration with test framework of our choice.

* `initialize_matchable_scopes`:

```Ruby
  def initialize_matchable_scopes
    scopes = ObjectSpace.each_object(Module).each_with_object([]) do |scope, aggregate|
      expression = expression(scope) || next
      aggregate << Scope.new(scope, expression)
    end

    @matchable_scopes = scopes.sort_by { |scope| scope.expression.syntax }
  end
```

This will walk through the ObjectSpace and get **all** the modules. You can easily check what happens here in `pry`:

```Ruby
ObjectSpace.each_object(Module).to_a.sample 3
# => [RubyToken::TkYIELD, Random::Formatter, PryByebug::Helpers::Breakpoints]

ObjectSpace.each_object(Module).to_a.sample 3
# => [Errno::EOVERFLOW, PrettyPrint::Breakable, #<Class:Kernel>]

ObjectSpace.each_object(Module).to_a.sample 3
=> [RubyToken::TkfLBRACK, PryStackExplorer, Psych::Visitors::DepthFirst]
```

So `initialize_matchable_scopes` will take all the modules it can find and try to create an expression out of it:

```Ruby
  def expression(scope) # <- "scope" is one of the modules from ObjectSpace
    name = scope.name # redacted for simplicity.

    unless name.instance_of?(String)
      # snip: give proper warning to the user
      return
    end

    config.expression_parser.try_parse(name)
  end
```

Ok, we went through a lot - let's look at the big picture again:

```Ruby
  def initialize_matchable_scopes
    scopes = ObjectSpace.each_object(Module).each_with_object([]) do |scope, aggregate|
      expression = expression(scope) || next
      aggregate << Scope.new(scope, expression)
    end

    @matchable_scopes = scopes.sort_by { |scope| scope.expression.syntax }
  end
```

* We're going through the full ObjectSpace
* try to create an expression out of the given Module
* if we can we store the module along side with it

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

Ok, there's a lot that's happening here. 
Let's go through it step by step:

First we'll get all "matched subjects", which means all the subjects you want to mutation tests. I won't go into detail how the `matched_subjects` method works because that would probably deserve another blog post so let's just assume it returns your modules and classes you would like to mutation test. 

Let's look at an example - Remember the `Greeter` example from above?

```
bundle exec mutant --include lib/\
                   --require hello_world\
                   --use rspec  "Greeter*"
```

So given we fire up `mutant` like that, how does `subjects` look like?

TODO below

```
 first subject: [#<Mutant::Subject::Method::Instance context=#<Mutant::Context scope=Greeter source_path=#<Pathname:/Users/timo/dev/hello_world/lib/hello_world/greeter.rb>> node=s(:def, :initialize,
  s(:args,
    s(:arg, :phrase)),
  s(:ivasgn, :@phrase,
    s(:lvar, :phrase)))>, #<Mutant::Subject::Method::Instance context=#<Mutant::Context scope=Greeter source_path=#<Pathname:/Users/timo/dev/hello_world/lib/hello_world/greeter.rb>> node=s(:def, :say_hello,
  s(:args,
    s(:arg, :name)),
  s(:dstr,
    s(:begin,
      s(:ivar, :@phrase)),
    s(:str, " "),
    s(:begin,
      s(:lvar, :name))))>]
```

Alright, so we have:

* A [configuration](https://github.com/mbj/mutant/blob/master/lib/mutant.rb#L191)
* An [environment](https://github.com/mbj/mutant/blob/master/lib/mutant/env.rb)
* A [bootstrapping process](https://github.com/mbj/mutant/blob/master/lib/mutant/env/bootstrap.rb), that ties them together
* [Subjects](https://github.com/mbj/mutant/blob/master/lib/mutant/subject.rb) that we intend to test
* [Mutations](https://github.com/mbj/mutant/blob/master/lib/mutant/mutation.rb) for those subjects

What's missing? Right, running our specs against those mutations!

### The Runner

### The big picture

TODO: Take the ascii art below and put it into a diagram

bin -> 
   cli -> config
config ->
   bootstrap -> env
      (infection; matching)
env -> runner
runner -> mainloop
mainloop -> report

### Wrapping it up

TODO
