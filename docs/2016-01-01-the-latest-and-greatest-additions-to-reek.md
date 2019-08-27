## The Latest And Greatest Additions To Reek

2015 has been an awesome year for `Reek`. With the release of [`Reek` 3](https://github.com/troessner/articles/blob/master/docs/2015-07-04-reek-3-has-been-released.md) we reached a level where I'd call `Reek` a stable, mature and - if used right - an incredibly useful library.

We have been quite busy since the `Reek` 3 release - here are my favourite features we have added to `Reek` since then:

### Unused private methods

`Reek` now detects unused private methods and warns you.
Code like this:

```ruby
class Car
  def start
    drive
  end

  private

  def hyper_drive; end
end
```

would cause `Reek` to emit:

```
1 warning:
  [8]:UnusedPrivateMethod: Car has the unused private instance method `hyper_drive` [https://github.com/troessner/reek/blob/master/docs/Unused-Private-Method.md]
```

Right now this smell detector can only [recognise unused private instance methods](https://github.com/troessner/reek/pull/794) - we plan to expand it to class methods as well in the future.

### Deactivate UtilityFunction for private methods

Quite some users [complained](https://github.com/troessner/reek/issues/681) about the UtilityFunction detector and I believe they rightfully did so.

In its old version it would warn on every method in question which was conflicting with a lot of users writing small, private methods for readability reasons. We [added](https://github.com/troessner/reek/pull/698) a configuration option so that you can now disable this smell detector for private methods like this:

```Yaml
---
UtilityFunction:
  public_methods_only: true
```

### Making NestedIterators smell smarter

Conceptually, every method in Ruby that takes a block is called an iterator.
However it's idiomatic Ruby to use blocks for a lot of different things and most of them have nothing to with "iterating" over anything.

For instance it's quite common to have `before` and <`after` hooks in the test framework of your choice like this:

```ruby
before do
  my_collection.each do |item|
    # do something with `item``
  end
end
```

As a consequence, `Reek` would warn you about NestedIterators which really makes no sense whatsoever in this case.

We [improved NestedIterators sensitivity](https://github.com/troessner/reek/pull/729) so that it won't count iterators without block arguments.

### Improved error messages

Unfortunately `Reek` never was the user friendliest library when it came to error messages.

Especially the warnings for complex smells like [FeatureEnvy](https://github.com/troessner/reek/blob/master/docs/Feature-Envy.md) were not really helpful:

```
FeatureEnvy: "refers to `foobar` more than self"
```

After quite some back and forth we finally converged on far more helpful error message. For instance, the new and improved error message for [FeatureEnvy](https://github.com/troessner/reek/blob/master/docs/Feature-Envy.md) is now:

```
FeatureEnvy: RedCloth#blocks refers to blk more than self (maybe move it to another class?)
[https://github.com/troessner/reek/blob/master/docs/Feature-Envy.md]
```

Far more expressive and it includes a helpful link by default you just have to click to read up on what FeatureEnvy means and how you might be able to fix it.

### What's next?

- We're [working hard](https://github.com/troessner/reek/issues/790) on our upcoming [CodeClimate](https://codeclimate.com) integration.
- We'll [streamline and clean up our API](https://github.com/troessner/reek/issues/650) which we'll then release in `Reek` 4.
- With the EOL of Ruby 2.0 approaching fast we'll drop its support in `Reek` 4 as well which will clean up our code base quite a bit.
- We have so many cool features on our [todo list](https://github.com/troessner/reek/issues?q=is%3Aopen+is%3Aissue+label%3Afeature) that I don't even know where to start.
