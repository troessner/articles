As part of a [presentation](https://github.com/troessner/talks/blob/master/exploiting_rubys_ast_a_love_story_in_three_chapters/presentation.html) I'm hoping to give at the end of this year about abstract syntax trees and how you can leverage them I started to look into how the [Mutant](https://github.com/mbj/mutant) gem works. `Mutant` is the by far most advanced gem for doing mutation testing in Ruby. I was (and still am) amazed how incredibly well built it is on all levels, ranging from the "big picture" architecture down to the "nuts and bolts" source code.

When it comes to mutation testing I learned one thing. It's complex. Really complex. And as a consequence, `Mutant` is complex. 

I did learn a lot from reading its source and I'd like to share this with you.

**Beware though, this is going to be a long ride.**

 This article will probably take you at least half an hour to an hour. But at the end, it'll hopefully be worth your while.

### Mutation testing in a nutshell

Let's quickly talk about what mutation testing is first. Mutation testing takes code like this:

```Ruby
class Greeter
  def initialize(phrase, enabled = true)
    @phrase = phrase
    @enabled = enabled
  end

  def say_hello(name)
    "#{@phrase} #{name}" if @enabled
  end
end
```

and turns it into something like this:

```Ruby
def say_hello(name)
  if true
    "#{@phrase}#{" "}#{name}"
  end
end
```

or 

```Ruby
def say_hello(name)
  if false
    "#{@phrase}#{" "}#{name}"
  end
end
```

`Mutant` then runs tests against each mutation. Those tests might look like this:

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

So let's say the `Greeter` class from above is part of a gem called "hello_world" which follows RubyGems conventions so it basically looks like this:

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

You can find the sample application [here](https://github.com/troessner/hello_world) in case you want to check it out for yourself via:

```
git clone git@github.com:troessner/hello_world.git
cd hello_world
bundle
```

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
* use the "Greeter*" namespace to find its test subjects. We'll cover the "subjects" topic later on extensively

`Mutant` will then spit out part of its configuration first:

```
Mutant configuration:

Matcher:         #<Mutant::Matcher::Config match_expressions: [Greeter*]>
Integration:     Mutant::Integration::Rspec
Expect Coverage: 100.00%
Jobs:            4
Includes:        ["lib/"]
Requires:        ["hello_world"]
Subjects:        2
```

And then `Mutant` will run your tests against the code mutations. Failing tests kill the mutants while surviving mutants show a lack of test coverage or that your code is not concise enough.

This is how a surviving mutant would be indicated:

```
  evil:Greeter#say_hello:/Users/timo/dev/hello_world/lib/mutant/greeter.rb:7:80e67
  @@ -1,6 +1,6 @@
   def say_hello(options)
  -  if @enabled
  +  if true
       "#{@phrase}#{" "}#{name}"
     end
   end
```

This means `Mutant` replaced the `@enabled` with an unconditional `true` and none of our tests broke.

Please keep the example project and `Mutant` call from above in the back of your head - I'll refer to this example throughout the article.

### Prerequisites for this article

While doing research for my presentation, I wrote an introductory series of articles about the gems `Mutant` uses:

* [part 1](https://troessner.svbtle.com/lessons-learned-from-some-of-the-best-ruby-codebases-out-there-part-1): The [Concord](https://github.com/mbj/concord) gem and the [Procto](https://github.com/snusnu/procto) gem
* [part 2](https://troessner.svbtle.com/lessons-learned-from-some-of-the-best-ruby-codebases-out-there-part-2): The [IceNine](https://github.com/dkubb/ice_nine) gem
* [part 3](https://troessner.svbtle.com/lessons-learned-from-some-of-the-best-ruby-codebases-out-there-part-3): The [abstract type](https://en.wikipedia.org/wiki/Abstract_type) gem and the [Adamantium](https://github.com/dkubb/adamantium) gem
 
You do not need to read part 2 and 3 to fully understand everything down below but you **should** read [part 1](https://troessner.svbtle.com/lessons-learned-from-some-of-the-best-ruby-codebases-out-there-part-1) as I will refer to `Concord` and `Procto` quite often.

`Mutant` creates its mutations by utilizing the [abstract syntax tree (AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree) represented by [S-expressions](https://en.wikipedia.org/wiki/S-expression) that the ruby parser generates from your source code.

Where are ASTs positioned in Ruby compile / interpret cycle?

![Ruby compile / interpret cycle](http://i.imgur.com/T5LAaqc.jpg)

As you can see above, the ASTs generated out of so called tokens and are used by the Compiler to generate bytecode instructions that are then executed by the Ruby interpreter. Another way to describe the ASTs role is that it is the machine readable semantics of the code the developer tried to express via literal source.

If this is new territory for you I urge you to check out the links above before continuing to get the most out of this article.

Additionally it might make sense to read Pat Shaughnessy's [awesome article](http://patshaughnessy.net/2012/6/18/the-start-of-a-long-journey-how-ruby-parses-and-compiles-your-code) about how Ruby parses your code.

Ok. Ready? Then let's roll.

### Start with the binary

The great thing about gems with a binary is that it's a lot easier to start exploring them - just start with the binary.

Let's look at `bin/mutant`:

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

Kernel.exit(Mutant::CLI.run(ARGV))
```

which translates to:

* Pass all of our CLI arguments to `Mutant::CLI.run`
* `Mutant::CLI.run` runs the program and then returns the exit status
* Which we pass to `Kernel#exit` that will cause our process to terminate with this very exit status

Let's check out `Mutant::CLI.run`:

```Ruby
class CLI
  include Procto.call(:config)

  def self.run(arguments)
    Runner.call(Env::Bootstrap.call(call(arguments))).success?
  rescue Error => exception
    $stderr.puts(exception.message)
    false
  end
end
```

The `include` line is what we need to understand first:

```Ruby
include Procto.call(:config)
```


`Procto.call(:config)`: [Procto](https://github.com/snusnu/procto) turns your Ruby object into a method object.

It basically works like this:

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

Now to `run`:

```Ruby
class CLI
  def self.run(arguments)
    Runner.call(Env::Bootstrap.call(call(arguments))).success?
  rescue Error => exception
    $stderr.puts(exception.message)
    false
  end
end
```

Let's ignore the exception since that is irrelevant for our scope. So it boils down to:

```Ruby
Runner.call(Env::Bootstrap.call(call(arguments))).success?
```

Let me reformat this line to make it clearer what's happening here:

```Ruby
Runner.call(
  Env::Bootstrap.call(
    call(arguments)
  )
).success?
```

Starting from the inside - this:

```Ruby
call(arguments)
```

translates to:

```Ruby
CLI.new(arguments).config
```

since we're inside the `CLI` class and coming from `Procto` again.

So this

```Ruby
Runner.call(
  Env::Bootstrap.call(
    call(arguments)
  )
).success?
```

translates to:

* we're getting back a configuration based on our CLI arguments
* we then pass that to `Env::Bootstrap.call`
* pass whatever comes back from that to our Runner

What does the configuration look like?

### Configuration

If you check out the constructor of `Mutant::CLI` you'll see this:

```Ruby
def initialize(arguments)
  @config = Config::DEFAULT

  parse(arguments)
end
```

Ok, nice and easy. First we'll set the configuration to our defaults, then we're parsing the CLI arguments and overwrite our default configuration based on this.

What does the default configuration look like?

For that you'll have to check out the top level library file, so `lib/mutant.rb`:


```Ruby
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
    pathname:          Pathname,
    requires:          EMPTY_ARRAY,
    reporter:          Reporter::CLI.build($stdout),
    zombie:            false
  )
end
```

I did remove a lot of the `config` object above to keep it readable. I just want to highlight a couple of lines I find interesting:

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
bundle exec mutant --expected-coverage 178/197 # plus the rest of your CLI arguments
```

Back to the configuration:

```Ruby
kernel:            Kernel,
load_path:         $LOAD_PATH,
pathname:          Pathname,
requires:          EMPTY_ARRAY,
```

What I like here is how configurable `Mutant`. You can even inject another Kernel module, $LOAD_PATH or another Pathname'esque library.

But the configurability is actually just a side effect here. The main goal is to stop relying on IO globals read from a global scope and injecting them explicitly.

This lists the IO dependencies of each unit much more explicit. Makes testing easier, and allows to write a testing tool that works without message expectations (one target of the author in the near future).

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
call(arguments)
```

does (returning a `Configuration` object), let's look at:

```Ruby
Env::Bootstrap.call(
  call(arguments) # -> This part we just discussed.
)
```

What does

```Ruby
Env::Bootstrap.call configuration
```

mean?
Again, this is `Procto` at work as you can see in the class declaration:

```Ruby
class Env
  class Bootstrap
    include Procto.call(:env)
    # ...snip
  end
end
```

So

```Ruby
Env::Bootstrap.call configuration
```

translates to:

```Ruby
Env::Bootstrap.new(configuration).env
```

Let's check out the class declaration of `Env::Bootstrap` with a focus on the initializer:

```Ruby
class Env
  class Bootstrap
    def initialize(*)
      super
      @parser = Parser.new
      infect
      initialize_matchable_scopes
    end
  end
end
```

We'll go through the initializer step by step:

```Ruby
super
```

That might seem odd to the innocent reader. `Mutant` makes heavy usage of the [Concord](https://github.com/mbj/concord) gem.
`Concord` is a useful tool to cut down boilerplate code.

You can use `Concord` to transform this:

```Ruby
class Dummy
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
class Dummy
  include Concord.new(:foo, :bar)
end
```

which is a lot easier on the eyes.

Now you should understand why `super` is there. This will ensure that the `initialize` provided by `Concord` will be called as well.
Try to remember this, this is a pattern you will see throughout the `Mutant` code base.

```Ruby
@parser = Parser.new
```

`Mutant` uses the awesome `[parser](https://github.com/whitequark/parser)` gem for producing ASTs from source code and has its own thin wrapper around. That thin wrapper just maintains a cache, so subjects on the same file share the (immutable) AST.

```Ruby
infect
```

Infection is the process where `Mutant` processes includes and requires and "infects" the ruby VM with the "subjects under tests".
To put it differently, it's the point where the software that is under mutation test gets loaded.

Let's look at `infect` in detail:

```Ruby
class Env
  class Bootstrap
    def infect
      config.includes.each(&config.load_path.method(:<<))
      config.requires.each(&config.kernel.method(:require))
      @integration = config.integration.new(config).setup
    end    
  end
end
```

The first line 

```Ruby
config.includes.each(&config.load_path.method(:<<))
```

uses a neat little `Method#to_proc` feature you can read up on [here](https://andrewjgrimm.wordpress.com/2011/10/03/in-ruby-method-passes-you/). 
It is basically identical to writing it like this:

```Ruby
config.includes.each do |include|
  config.load_path << include
end
```

but the first version is arguably more concise. So technicalities aside, this will update the `load_path` with whatever you passed to `Mutant` in your CLI call via the `--include` flag like shown in the beginning:

```Ruby
bundle exec mutant --include lib/\
                   --require hello_world\
                   --use rspec  "Greeter*"
```

In analog fashion, the next line in `infect`

```Ruby
config.requires.each(&config.kernel.method(:require))
```

does require everything that we passed to `Mutant` via the `--require` flag, so our `hello_word.rb` file in `lib/` in this case.

The last line

```Ruby
@integration = config.integration.new(config).setup
```

sets up the integration with test framework of our choice. As of right now there is only one valid option and that is `RSpec`.

Now we're coming to the last line in our initializer:

```Ruby
initialize_matchable_scopes
```

Let's look at this one in detail:

```Ruby
  def initialize_matchable_scopes
    scopes = ObjectSpace.each_object(Module).each_with_object([]) do |scope, aggregate|
      expression = expression(scope) || next
      aggregate << Scope.new(scope, expression)
    end

    @matchable_scopes = scopes.sort_by { |scope| scope.expression.syntax }
  end
```

This will walk through the ObjectSpace and get **all** the modules that have been required so far. You can easily check what happens here in `pry`:

```Ruby
ObjectSpace.each_object(Module).to_a.sample 3
# => [RubyToken::TkYIELD, Random::Formatter, PryByebug::Helpers::Breakpoints]

ObjectSpace.each_object(Module).to_a.sample 3
# => [Errno::EOVERFLOW, PrettyPrint::Breakable, #<Class:Kernel>]

ObjectSpace.each_object(Module).to_a.sample 3
=> [RubyToken::TkfLBRACK, PryStackExplorer, Psych::Visitors::DepthFirst]
```

So `initialize_matchable_scopes` will take all the modules it can find and try to create an expression out of it

```Ruby
  def initialize_matchable_scopes
    scopes = ObjectSpace.each_object(Module).each_with_object([]) do |scope, aggregate|
      expression = expression(scope) || next
      # ... snip
    end
  end
```

With

```Ruby
  def expression(scope) # `scope` is one of the modules from ObjectSpace, e.g. Gem::Ext::BuildError, so a constant.
    name = scope.name # redacted for simplicity.
    # snip

    config.expression_parser.try_parse(name)
  end
```

`config.expression_parser` returns a `Mutant::Expression::Parser` by default. 

`try_parse` will then return something like this

```
#<Mutant::Expression::Namespace::Exact scope_name="Gem::Ext::BuildError">
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

This now brings us to the topic of "subjects".

### Subjects

Let's go through the `env` method step by step:

```Ruby
  def env
    subjects = matched_subjects
    # snip
  end
```

First we'll get all "matched subjects", which means all the subjects you want to mutation test. Each subject represents only a specific instance or singleton method that is later subject of mutation (`Mutant` does not yet mutate code at the class or module level).

Let's look at an example - Remember the `Greeter` example from above?

```
bundle exec mutant --include lib/\
                   --require hello_world\
                   --use rspec  "Greeter*"
```

So given we fire up `Mutant` like that, how does `subjects` look like?
`matched_subjects` returns an array of `Enumerable<Subject>`, let's check out how this looks in our console:

```
[
# The `subjects` array has 2 entries:
# First subject:

#<Mutant::Subject::Method::Instance 
  context=
    #<Mutant::Context 
      scope=Greeter
      source_path=#<Pathname:.../hello_world/lib/hello_world/greeter.rb>>
  node=s(:def, :initialize,
    s(:args,
      s(:arg, :phrase)),
        s(:ivasgn, :@phrase,
          s(:lvar, :phrase)))

>, # end of first subject

# Second subject:

#<Mutant::Subject::Method::Instance
  context=
    #<Mutant::Context
      scope=Greeter
      source_path=#<Pathname:.../hello_world/lib/hello_world/greeter.rb>>
  node= s(:def, :say_hello,
          s(:args,
            s(:arg, :name)),
          s(:if,
            s(:ivar, :@enabled),
            s(:dstr,          
              # ...snip
    > # end of second subject
]
```

Just by reading through it you should see be able to get a feeling for what's happening, but let's go through it step by step:

```
Mutant::Subject::Method::Instance
```

The `Subject` class. As you can see it has 2 attributes:

1.) A [context](https://github.com/mbj/mutant/blob/master/lib/mutant/context.rb)

```
  context=
    #<Mutant::Context 
      scope=Greeter
      source_path=#<Pathname:.../hello_world/lib/hello_world/greeter.rb>>
```

which contains:
  * a [scope](https://github.com/mbj/mutant/blob/master/lib/mutant/scope.rb) called `Greeter`. This should look familiar to you now, doesn't it? That's the module or class (in `Mutant` it's just called "namespace") from which we take our subjects.
  * a `source_path`: Again, no magic here, you can see the path to the "greeter.rb" on my local file system

2.) A [node](https://github.com/mbj/mutant/blob/master/lib/mutant/mutator/node.rb)

```
  node=s(:def, :say_hello,
         s(:args,
          # ...snip
```

representing an [abstract syntax trees (AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree) consisting of [S-expressions](https://en.wikipedia.org/wiki/S-expression).

But even without knowing **anything** about ASTs or S-expressions by looking at this

```
  node=s(:def, :initialize,
```

or this

```
node=s(:def, :say_hello,
```

you can kind of figure out that both nodes correspond to the methods in our `Greeter` class:

```Ruby
class Greeter
  def initialize(phrase)
    @enabled = true
    @phrase = phrase
  end

  def say_hello(name)
    "#{@phrase} #{name}" if @enabled
  end
end
```

This

```
s(:def, :say_hello,
  s(:args,
    # ...snip
```

is the abstract syntax tree for this code in form of S-expressions.

__Alright, so this doesn't look so complex anymore, now does it?__

* we have a class with 2 instance methods
* `Mutant` takes this class and turns it into 2 subjects
* where each subject contains a node consisting of S-Expressions that corresponds to one of our instance methods.

### Mutator

Ok, so now we have talked about "subjects" a lot. What about the "mutations"?

If you recall the `env` method in `Env::Bootstrap` from above:

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

you see this line:

```Ruby
mutations:        subjects.flat_map(&:mutations),
```

Each `Subject` can yield N mutations, `Mutant` expands these in one step here for better progress reports.

Let's check out the [Subject](https://github.com/mbj/mutant/blob/master/lib/mutant/subject.rb) class:

```Ruby
module Mutant
  class Subject
    # Mutations for this subject
    #
    # @return [Enumerable<Mutation>]
    # @return [undefined]
    def mutations
      [neutral_mutation].concat(
        Mutator.mutate(node).map do |mutant|
          Mutation::Evil.new(self, wrap_node(mutant))
        end
      )
    end
    memoize :mutations
  end
end
```

This

```Ruby
Mutator.mutate(node).
```

is the interesting part into which we're going to dive soon.
But let me give you a high level overview of what happens here first:

```Ruby
def mutations
  [neutral_mutation].concat(
    Mutator.mutate(node).map do |mutant|
      Mutation::Evil.new(self, wrap_node(mutant))
    end
  )
end
```

`Mutator.mutate(node)` takes a given node like this (re-using our example from above):

```
s(:def, :say_hello,
  s(:args,         
  # ...snip
```

creates multiple mutations out of this node and returns a `Set` of `Parser::AST::Node`.
We then iterate over this `Set` and create an evil `Mutation` out of each `Parser::AST::Node`. An evil `Mutation` is a mutation for which we expect a test to fail. Or to put it the other way round: If no test fails for this mutation we have a surviving mutant.

Finally, we'll add all of the evil mutations to a "neutral" mutation and all of this makes up the collection of mutations for one specific subject.

Now what's up with that neutral mutation?

A neutral mutation means that you're taking the original node without mutating it:

```Ruby
def neutral_mutation
  Mutation::Neutral.new(self, node)
end
```

With `Mutation::Neutral` being:

```Ruby
# Neutral mutation that should not cause mutations to fail tests
class Neutral < self
  TEST_PASS_SUCCESS = true
end
```

As you can see, in `neutral_mutation` we're basically just passing the original node with `Mutation::Neutral` having the expectation that the test pass on this one.
The purpose of the neutral mutation is kind of a sanity check to make sure that your original code doesn't break your existing specs. Especially it does not break the existing specs on unmodified source when run inside the more demanding mutation testing environment.

`Mutant` requires a test to be simultaneously:

* Idempotent
* Order dependent free
* Race condition free under separate address spaces

Lets say all your tests would touch a shared resource (database). Without neutral mutations, your evil mutations might get killed because concurrency makes your specs red, but not because the evil mutation itself that was inserted.

Neutral mutations are a safeguard for this scenario. If you see an neutral mutation being reported, your tests violate above properties, and the evil mutations dead cannot be trusted.

__2 questions now:__

1.) How does a `Mutation` look like?

2.) How does the `Mutator` work in detail?

### Mutation

Let's check out a `Mutation` first:

```Ruby
class Mutation
  include AbstractType
  include Concord::Public.new(:subject, :node)

  def self.success?(test_result)
    self::TEST_PASS_SUCCESS.equal?(test_result.passed)
  end
end
```

`Mutation` itself is an abstract base class (notice the `include AbstractType`, you can read up on this [here](https://troessner.svbtle.com/lessons-learned-from-some-of-the-best-ruby-codebases-out-there-part-3)), so it won't be instantiated directly.
As you can see it has basically just 2 attributes:

```Ruby
include Concord::Public.new(:subject, :node)
```

As you've learned above already, this is just a nice shortcut for

```Ruby
class Subject
  attr_accessor :subject, :node

  def initialize(subject, node)
    self.subject = subject
    self.node    = node
  end
end
```

Ok, so the attribute set up is nice and easy:

* `subject`: A reference back to the `Subject`. In Domain Driven Design terms this would be the link back to the root of the aggregate.
* `node`: The mutated node. So basically a mutation of

```
s(:def, :say_hello,
  s(:args,
    s(:arg, :name)),
  s(:if,
    s(:ivar, :@enabled),        
  # ...snip
```

This gets passed directly down from the `Mutator` so we'll talk about some detailed examples later on when we discuss the `Mutator` in detail.

What else is interesting?

The `success?` method:

```Ruby
def self.success?(test_result)
  self::TEST_PASS_SUCCESS.equal?(test_result.passed)
end
```

This is the method that `Mutant` uses to check what the expectation for running tests against a mutation is and is basically configured in the instantiatable subclasses:

```Ruby
class Evil < self
  TEST_PASS_SUCCESS = false
end # Evil

# Neutral mutation that should not cause mutations to fail tests
class Neutral < self
  TEST_PASS_SUCCESS = true
end # Neutral
```

As you can see, the expectation for "neutral" mutations is true while the expectation for "evil" mutations is false. 

### The mutator under the microscope

Let's recall the `Subject#mutations` method from above:

```Ruby
    def mutations
      [neutral_mutation].concat(
        Mutator.mutate(node).map do |mutant|
          Mutation::Evil.new(self, wrap_node(mutant))
        end
      )
    end
```

We talked about "neutral" and "evil" mutations, now it's time to look into how `Mutator#mutate` works under the hood:

```Ruby
module Mutant
  # Generator for mutations
  class Mutator
    # Lookup and invoke dedicated AST mutator
    #
    # @param node [Parser::AST::Node]
    # @param parent [nil,Mutant::Mutator::Node]
    #
    # @return [Set<Parser::AST::Node>]
    def self.mutate(node, parent = nil)
      self::REGISTRY.lookup(node.type).call(node, parent)
    end
  end
end
```

The [Registry](https://github.com/mbj/mutant/blob/master/lib/mutant/registry.rb) is conceptually quite simple. Just think of it as the place in `Mutant` where you basically say "I want to associate this mutator with this type of AST node".

So this:

```Ruby
self::REGISTRY.lookup(node.type)
```

will basically just check if for a given node type there is a mutator defined. And if there is this one will be returned as you can see here:

```Ruby
module Mutant
  # Registry for mapping AST types to classes
  class Registry
    include Concord.new(:contents)

    def lookup(type)
      contents.fetch(type) do
        fail RegistryError, "No entry for: #{type.inspect}"
      end
    end
  end
end
```

This all sounds very vague so let's check out the concrete example from above in pry. We'll insert a break point here in the `Mutator`:

```Ruby
module Mutant
  class Mutator
    def self.mutate(node, parent = nil)
binding.pry
      self::REGISTRY.lookup(node.type).call(node, parent)
    end
  end
end
```

`Mutator.mutate` will be called recursively to traverse the given node - I'm omitting the details here for the sake of simplicity.

So let me show you a sample run.

The first `pry` session that pops up looks like this:

```
[1] pry(Mutant::Mutator)> node
=> s(:def, :say_hello,
  s(:args,
    # snip
[2] pry(Mutant::Mutator)> node.type
=> :def
```

Ok, we're starting with the whole method.

Next:

```
[1] pry(Mutant::Mutator)> node
=> s(:args,
  s(:arg, :name))
[2] pry(Mutant::Mutator)> node.type
=> :args
```

Ok, now we're handling the arguments. I'm gonna fast-forward:

```
[1] pry(Mutant::Mutator)> node
=> s(:if,
  s(:ivar, :@enabled),
    # snip
[2] pry(Mutant::Mutator)> node.type
=> :if
```

There it is! Now we're talking. That's where we're dealing our initial example from above - remember?

That was this one:

```
  evil:Greeter#say_hello:/Users/timo/dev/hello_world/lib/mutant/greeter.rb:7:80e67
  @@ -1,6 +1,6 @@
   def say_hello(name)
  -  if @enabled
  +  if true
       "#{@phrase}#{" "}#{name}"
     end
   end
```


So we have the ":if" node - who's handling that?

If you check out the `mutator/node` [directory](https://github.com/mbj/mutant/tree/master/lib/mutant/mutator/node) you'll see all the node mutators that are available. Including the one that handles the [":if" node](https://github.com/mbj/mutant/blob/master/lib/mutant/mutator/node/if.rb).

But let's not concern ourselves with the details of the single mutators but first finish off `Mutator.mutate`. So what does this method actually return?

Let's use the console again.
First keep in mind what the original node looked like:

```Ruby
s(:def, :say_hello,
  s(:args,
    s(:arg, :name)),
  s(:if,
    s(:ivar, :@enabled),
    s(:dstr,
      s(:begin,
        s(:ivar, :@phrase)),
      s(:str, " "),
      s(:begin,
        s(:lvar, :name))), nil))
```

Now let's take samples of what `Mutator.mutate(node)` returns:

```Ruby
# [18] pry(#<Mutant::Subject::Method::Instance>)> Mutator.mutate(node).to_a.sample

 s(:def, :say_hello,
  s(:args,
    s(:arg, :name)),
  s(:if,
    s(:ivar, :@enabled),
    s(:dstr,
      s(:begin,
        s(:send, nil, :phrase)),
      s(:str, " "),
      s(:begin,
        s(:lvar, :name))), nil))
```

Interesting. See the difference to the original node?

The original:

```Ruby
s(:dstr,
  s(:begin,
    s(:ivar, :@phrase)),
```

The mutation:

```Ruby
s(:dstr,
  s(:begin,
    s(:send, nil, :phrase)),
```

This means `Mutant` replaced accessing an instance variable

```Ruby
"#{@phrase} ...."
```

with a method call:

```Ruby
"#{phrase} ...."
```

This:

```
s(:send, nil, :phrase)
```

means "Send the message `phrase` to `self`." The "nil" argument denotes that there is no explicit receiver, so `self` is the implicit one.

You can quickly check this out yourself using the `ruby-parse` executable that comes along with the `Parser` gem (so `gem install parser`):

```
($) ruby-parse -e '"#{@phrase}"'

(dstr
  (begin
    (ivar :@phrase)))

($) ruby-parse -e '"#{phrase}"'

(dstr
  (begin
    (send nil :phrase)))
```

Ok, another roll of the dice:

```Ruby
#[18] pry(#<Mutant::Subject::Method::Instance>)> Mutator.mutate(node).to_a.sample

=> s(:def, :say_hello,
  s(:args,
    s(:arg, :name)),
  s(:if,
    s(:true),
    s(:dstr,
      s(:begin,
        s(:ivar, :@phrase)),
      s(:str, " "),
      s(:begin,
        s(:lvar, :name))), nil))
```

There it is! 

```Ruby
s(:if,
  s(:true),
  s(:dstr,
    s(:begin,
      s(:ivar, :@phrase))
```

That's the mutation from our original example. 

```Ruby
if true
  "#{@phrase}#{" "}#{name}"
end
```

### Mutating single nodes

Alright, time for a recap. So far we have:

* A [configuration](https://github.com/mbj/mutant/blob/master/lib/mutant.rb#L191)
* An [environment](https://github.com/mbj/mutant/blob/master/lib/mutant/env.rb)
* A [bootstrapping process](https://github.com/mbj/mutant/blob/master/lib/mutant/env/bootstrap.rb) that ties them together
* [Subjects](https://github.com/mbj/mutant/blob/master/lib/mutant/subject.rb) that we intend to mutation test
* [A mutator](https://github.com/mbj/mutant/blob/master/lib/mutant/mutator.rb) that takes the method nodes within those subjects
* And applies [mutations](https://github.com/mbj/mutant/blob/master/lib/mutant/mutation.rb) to those subjects

The missing piece now is the logic that actually applies the mutation to the nodes. Let's focus on our good, old ":if" node from above and have again a look at `Mutator#mutate`:

```Ruby
class Mutator
  def self.mutate(node, parent = nil)
    self::REGISTRY.lookup(node.type).call(node, parent)
  end
end
```

So in case our `node.type` is `:if` the `lookup` method will return `Mutator::Node::If` defined in `mutator/node/if.rb`

This means that this:

```Ruby
self::REGISTRY.lookup(node.type).call(node, parent)
```

boils down to:

```Ruby
Mutator::Node::If.call(node, parent)
```

Since `Mutator` uses `Procto` like this:

```Ruby
include Procto.call(:output)
```

this basically means:

```Ruby
if_node = Mutator::Node::If.new(node, parent)
if_node.output
```

So let's check out the `if` mutator:

```Ruby
class Mutator
  class Node
    # Mutator for if nodes
    class If < self

      handle(:if)

      def dispatch
        emit_singletons
        mutate_condition
        mutate_if_branch
        mutate_else_branch
      end

      def mutate_condition
        # snip
        emit_type(N_TRUE,  if_branch, else_branch)
        emit_type(N_FALSE, if_branch, else_branch)
      end
    end
  end
end
```

Let's go through it step by step:

```Ruby
handle(:if)
```

Remember the `Registry` from above in the `Mutator`?

```Ruby
def self.mutate(node, parent = nil)
  self::REGISTRY.lookup(node.type).call(node, parent)
end
```

This

```Ruby
handle(:if)
```

is the part that registers the "if node" mutator at the global registry to handle ... "if" nodes (make no mistake: other mutators do handle multiple types, so this is not a fixed 1:1 relation).

Next up is the `dispatch` method. That is kind of the workhorse of all the mutators and is going to be called via `Mutator#initialize`:

```Ruby
def dispatch
  emit_singletons
  mutate_condition
  mutate_if_branch
  mutate_else_branch
end
```

Let's focus on the mutant we have been talking about from the very start: 

```
  evil:Greeter#say_hello:/Users/timo/dev/hello_world/lib/mutant/greeter.rb:7:80e67
  @@ -1,6 +1,6 @@
   def say_hello(options)
  -  if @enabled
  +  if true
       "#{@phrase}#{" "}#{name}"
     end
   end
```

For this mutation we can ignore all method calls in `dispatch` except for `mutate_condition`:

```Ruby
def dispatch
  emit_singletons # <- Ignore me.
  mutate_condition
  mutate_if_branch # <- Ignore me.
  mutate_else_branch # <- Ignore me.
end
```

So let's dive into `mutate_condition`:

```Ruby
def mutate_condition
  # snip
  emit_type(N_TRUE,  if_branch, else_branch)
  emit_type(N_FALSE, if_branch, else_branch)
end
```

What is `N_TRUE`?

It's defined in `ast/nodes`:

```Ruby
N_TRUE = s(:true)
```

You probably see by now where I'm going with this: This is the place where our `if true` mutation is generated.

Let's check out `emit_type` to understand the call in question:

```Ruby
# mutator/node.rb

def emit_type(*children)
  emit(::Parser::AST::Node.new(node.type, children))
end

# mutator.rb

def emit(object)
  # snip
  output << object
end
```

So basically a call like

```Ruby
emit_type(N_TRUE,  if_branch, else_branch)
```

will just re-create the existing if/else branch but the S-expression for the condition will be replace with `s(:true)`.

**Let's summarize this:**

- We pass the original AST to `Mutator#mutate`
- We then traverse this tree from top to down
- For every node in this tree for which we have a registered mutator we call this very mutator
- We then mutate this specific node, create a copy of our original abstract syntax tree and insert the mutated node into it
- Finally we return all of those copies back to `Mutator#mutate`

With that out of the way, let's talk about the last part of the assembly line: Taking all of what we just talked about and run it against our specs.

### The Runner

It was a long ride to get here but we're almost done.

What's missing? Right, running our specs against those mutations!

This is where the [Runner](https://github.com/mbj/mutant/blob/master/lib/mutant/runner.rb) comes into play.

Remember that our investigation how `Mutant` works started with this piece of code in the CLI module:

```Ruby
module Mutant
  class CLI
    def self.run(arguments)
      Runner.call(
        Env::Bootstrap.call(
          call(arguments)
        )
      ).success?
    end
  end
end
```

The `Runner.call` part should be something familiar for you now. Let's check out the rest of the code:


```Ruby
module Mutant
  # Runner baseclass
  class Runner
    include Adamantium::Flat, Concord.new(:env), Procto.call(:result)

    def initialize(*)
      super # You know that pattern as well - Concord.

      reporter.start(env)

      run_mutation_analysis
    end
  end
end
```

Let's ignore the `reporter` related code here and in the following since that basically does what you think it does: It collects the results and handles outputting those results.

The interesting part is the `run_mutation_analysis` at the end:

```Ruby
def run_mutation_analysis
  @result = run_driver(Parallel.async(mutation_test_config))
  reporter.report(result)
end
```

`Parallel.async` will run a given configuration asynchronously and return a corresponding driver.

What does the configuration look like?

```Ruby
# Configuration for parallel execution engine
#
# @return [Parallel::Config]
def mutation_test_config
  Parallel::Config.new(
    env:       env.actor_env,
    jobs:      config.jobs,
    processor: env.method(:kill),
    sink:      Sink.new(env),
    source:    Parallel::Source::Array.new(env.mutations)
  )
end
```

with the interesting line being:

```Ruby
source:    Parallel::Source::Array.new(env.mutations)
```

See the `env.mutations` at the end? `env` is what we pass in to the `Runner` via the constructor using `Concord`:

```Ruby
class Runner
  include Concord.new(:env)

  def initialize(*)
    super
    # snip
  end
end
```

So that's how we pass the mutations we want to run our tests against to the parallel runner.

Let's go back to `run_mutation_analysis`:

```Ruby
def run_mutation_analysis
  @result = run_driver(Parallel.async(mutation_test_config))
  reporter.report(result)
end
```

What does `run_driver` look like?

```Ruby
# Run driver
#
# @param [Driver] driver
#
# @return [Object]
#   the last returned status payload
def run_driver(driver)
  status = nil
  sleep  = env.config.kernel.method(:sleep)

  loop do
    status = driver.status
    reporter.progress(status)
    break if status.done
    sleep.call(reporter.delay)
  end

  driver.stop

  status.payload
end
```

Ok, that's conceptually simple: 

* The `driver` that was returned by `Parallel.async(mutation_test_config)` is running our specs asynchronously and in parallel
* We loop continuously over the `driver` and check its status
* As soon as the `driver` is done, i.e. all mutants have either been killed or have survived, we return the result to the reporter and exit

When it comes to how `Mutant` does finally run your tests against mutations in parallel I barely touched the surface. `Mutant` uses its own actor model and implements its own framework for parallelism on top of that.

Explaining that would be the topic of an own blog post. If you want to check this out for yourself I'd recommend to look into the [parallel namespace](https://github.com/mbj/mutant/tree/master/lib/mutant/parallel) and the [actor namespace](https://github.com/mbj/mutant/tree/master/lib/mutant/actor).

### The big picture

You made it!

Let's have a look at a diagram to finish the article.

![Mutant overview](http://i.imgur.com/QRJ2E0C.png)

You can see all the entities we talked about before. Subjects, mutations, mutators and so on. Hopefully this will reinforce what we learned today.

### What I left out

Well, a lot. I didn't go into details about

* how mutated nodes are re-inserted into the AST
* how `Mutant` actually generates Ruby code back out of ASTs for generating its warnings and for running the specs (hint: It uses the [Unparser](https://github.com/mbj/unparser) gem under the hood)
* how `Mutant` runs the right specs against the mutations
* how `Mutant` isolates concurrent mutation kills from each others

I also didn't talk about awesome `Mutant` features like the `since` flag:

```
--since REVISION             Only select subjects touched since REVISION
```

or how you can use the `--jobs` flag to play around with the degree of parallelism.

### Wrapping it up

Don't say I didn't warn you before, it was a long article indeed.
I hope you did learn something from reading it, I for sure did learn a lot browsing `Mutants` awesome code base.

In case you're not using mutation testing right now I urge you to check it out.
Applied properly, it will not only improve your specs a lot, it will also force you to focus more on maintainability and conciseness when coding.
