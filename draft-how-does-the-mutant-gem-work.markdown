
Mutation testing is quite a hot topic these days. Let's get the definition of "mutation testing" quickly out of the way by citing [wikipedia](https://en.wikipedia.org/wiki/Mutation_testing):

>>
Mutation testing (or Mutation analysis or Program mutation) is used to design new software tests and evaluate the quality of existing software tests. Mutation testing involves modifying a program in small ways.[1] Each mutated version is called a mutant and tests detect and reject mutants by causing the behavior of the original version to differ from the mutant. This is called killing the mutant. Test suites are measured by the percentage of mutants that they kill. New tests can be designed to kill additional mutants. Mutants are based on well-defined mutation operators that either mimic typical programming errors (such as using the wrong operator or variable name) or force the creation of valuable tests (such as dividing each expression by zero). The purpose is to help the tester develop effective tests or locate weaknesses in the test data used for the program or in sections of the code that are seldom or never accessed during execution.

In Ruby land the awesome [Mutant](https://github.com/mbj/mutant) gem is by far most advanced gem in this regard.

This is the first part of an upcoming series about how `Mutant` works internally.

### Using `Mutant`

Let's start by looking at how `Mutant` on a high level and how you'd use it:

`Mutant` takes code like this

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

And turns it into something like this


```Ruby
if true
  "#{@phrase}#{" "}#{options[:name]}"
end

# Was
# "#{@phrase} #{options[:name]}" if @enabled
```

Or this

```Ruby
if @enabled
  "#{@phrase}#{" "}#{options.fetch(:name)}"
end

# Was
# "#{@phrase} #{options[:name]}" if @enabled
```

Mutant then runs your tests against each mutation

```
bundle exec mutant --include lib/
                   --require greeter
                   --use rspec  "Greeter*"
```

This will first output the current configuration:

```
Mutant configuration:
Matcher:         #<Mutant::Matcher::Config match_expressions: [Greeter*]>
Integration:     Mutant::Integration::Rspec
Expect Coverage: 100.00%
Jobs:            4
Includes:        ["lib/"]
Requires:        ["greeter"]
Subjects:        2
```

Failing tests kill the mutants, surviving mutants show a lack of test coverage:

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

### How does `Mutant` work?

Whenever I dissect codebases that have an executable I start right there, at `bin/`:

```Ruby
#!/usr/bin/env ruby

trap('INT') do |status|
  effective_status = status ? status + 128 : 128
  exit! effective_status
end

require 'mutant'

namespace =
  if ARGV.include?('--zombie')
    $stderr.puts('Running mutant zombified!')
    Mutant::Zombifier.call(
      # snip
    )
    Zombie::Mutant
  else
    Mutant
  end

Kernel.exit(namespace::CLI.run(ARGV))
```
