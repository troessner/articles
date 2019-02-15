## Reek 2 Just Got Released

A couple of days ago we released the new and extremely shiny [Reek](https://github.com/troessner/reek) version 2 to the world!

Reek is a static code analyzer for Ruby detecting so called [code smells](https://github.com/troessner/reek/wiki/Code-Smells). Those code smells range from rather trivial ones like [UncommunicativeParameterName](https://github.com/troessner/reek/wiki/Uncommunicative-Parameter-Name) or [TooManyMethods](https://github.com/troessner/reek/wiki/Too-Many-Methods) to high-level code smells like [FeatureEnvy](https://github.com/troessner/reek/wiki/Feature-Envy) or [DataClump](https://github.com/troessner/reek/wiki/Data-Clump).

In the most simple use case you can just run

```Bash
reek my/source/files
```

or

```Bash
echo "def dirty(x,y,z); puts 'hossa!'; end" | reek
```

### So what has happened since 1.*?

There are way too many significant changes to list them all so I restrict this list to my favourite ones (excluding the countless bugfixes):

#### Support

Parsing with the [`parser`](https://github.com/whitequark/parser) gem allows us to support all major Ruby versions:

1.9.3
2.0
2.1
2.2.

We deliberately dropped support for 1.8.

#### New smell detectors

We introduced 2 new smell detectors:

- The [`ModuleInitialize` smell detector](https://github.com/troessner/reek/wiki/Module-Initialize)
- The [`PrimaDonnaMethod` smell detector](https://github.com/troessner/reek/wiki/Prima-Donna-Method)

#### Revised configuration handling

We completely revised our configuration handling, basically there are 3 ways of passing `Reek` a configuration file:

1. Using the cli `-c` switch.
2. Having a file ending with .reek either in your current working directory or in a parent directory.
3. Having a file ending with .reek in your HOME directory.

This means that `Reek` no longer tries to nest configuration files, but instead from `Reek`'s point of view there exists only one configuration file and one configuration.

Another cool feature is support for detecting specific smells like this:

```
reek lib/ --smell FeatureEnvy
```

This can be very helpful when refactoring when you want to focus on one group of problems.

And last but not least smell detectors can be configured or disabled in code comments like this (in addition to `Reek`'s regular [configuration mechanism](https://github.com/troessner/reek/wiki/Basic-Smell-Options)):

```Ruby
# This method smells of :reek:NestedIterators
def smelly_method foo
  foo.each {|bar| bar.each {|baz| baz.qux}}
end
```

#### CLI / Output / UX

We completely restructured and revised the command line interface for `Reek`:

- We now support the yaml and html format besides the standard text format which makes `Reek` great for compiling automated reports and statistics
- We overall revamped the UX. For instance we added color to `Reek`'s output, enabled different sorting options like `sort by issue count` and added a `show-wiki-links` switch that prints out helpful links next to each warning pointing to our github wiki that explains this particular smell and how to possibly fix ist.
- Speaking of the wiki, you now should find an extensive description of each smell `Reek` might detect including possible solutions

#### rspec matchers

We made the `reek_of` / `reek_only_of` rspec matchers a little smarter - you can still just do something like:

```Ruby
expect(src).not_to reek_of(:UncommunicativeParameterName)
```

but if you really want to be more specific about this you can now add a hash of "smell_details" `reek_of` will check for as well like this:

```Ruby
expect(src).not_to reek_of(:UncommunicativeParameterName, name: 'x1' )
```

You can check out the details [here](https://github.com/troessner/reek/wiki/RSpec-matchers).
