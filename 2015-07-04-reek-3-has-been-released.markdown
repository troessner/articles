My beloved `Reek` gem has come quite a long way. During the last months we refactored so much of the source code that it almost feels like a new and shiny gem.

Right after the release of `Reek` 2 we started to work on [`Reek` 3](https://github.com/troessner/reek/issues/401) which we released a couple of days ago.

### A stable API

The changes that I'm most exited about is that we agreed on a [public API](https://github.com/troessner/reek/issues/526)  and [implemented it as well](https://github.com/troessner/reek/pull/544). For this API to use you'll basically just do something like this:

```Ruby
require 'reek'

reporter = Reek::Report::TextReport.new
examiner = Reek::Examiner.new("class Klazz; def m(a,b,c); end; end")
reporter.add_examiner examiner
reporter.show
```

which would give you this

```
5 warnings:
Klazz has no descriptive comment (IrresponsibleModule)
Klazz#m has the name 'm' (UncommunicativeMethodName)
Klazz#m has unused parameter 'a' (UnusedParameters)
Klazz#m has unused parameter 'b' (UnusedParameters)
Klazz#m has unused parameter 'c' (UnusedParameters)
```

Getting a stable API out is something that hopefully means a lot for projects who make use of `Reek` programmatically like [Rubycritic](https://github.com/whitesmith/rubycritic).

The API is still rather small so you can quickly read up on everything you need to know in 5 minutes [here](https://github.com/troessner/reek/blob/master/docs/API.md).

### Excludable directories

We made directories excludable via configuration, a feature that was requested quite some times.

The way this works is that you just add a paragraph like this

```Yaml
exclude_paths:
  - app/views
  - app/controllers
```

to your `Reek` config and that's it - `Reek` will ignore those directories when scanning.

### Singleton methods

We fixed one of the most annoying bugs that has been around for years. Until now, `Reek` would not recognise singleton methods if they were defined with the class << self syntax, meaning that this:

```Ruby
class C
  class << self
    def m(a)
      a.to_s
    end
  end
end
```

would incorrectly report [UtilityFunction](https://github.com/troessner/reek/blob/master/docs/Utility-Function.md).
[Now](https://github.com/troessner/reek/pull/539) it will correctly recognise those methods as singleton methods.

### Compatibility

We dropped support Ruby 1.9. Time to move on.

What's next?

- An engine for [CodeClimate](https://codeclimate.com/) to make it even easier to use `Reek`
- A [rails-friendly `Reek` mode](https://github.com/troessner/reek/issues/529)
- We have quite a few awesome ideas I'd love [to get going](https://github.com/troessner/reek/issues/556)
