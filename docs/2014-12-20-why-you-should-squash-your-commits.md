## Why You Should Squash Your Commits

When you look at bigger pull requests at open-source projects or in closed-source projects of your company, you can see that they often contain dozens and dozens of commits with a lot of them containing useless commit messages like "Fix.", "Typo", and so on.

"Useless" in th sense of the bigger picture - while you might care for a "typo fix" during the lifecycle of your pull request you probably won't a couple of weeks down the road.

Having a lot of commits in one pull request / branch makes it not-atomic, so it "does not behave like one single entity, but like a lot".

I believe there are strong reasons for squashing all of your commits into one right before merge.

But let's start the other way round here.

### What's the problem with merging pull requests with a lot of commits?

1. `git-log` is one of the most important git tools and using it properly can make our lifes a lot easier. For instance, with a proper log history you can just type in `git log` and you can immediately see the last significant changes to the source - something that is close to impossible when you merge branches with dozens of commits. If one feature is split across many commits `git-log` is useless because it can’t tell what belongs where. Sure, `git-log` can show you what commits belonged to what branch but this is still useless in most cases because it is just not terse and concise enough for the human brain to operate on.

1. Bugfixing can be significantly easier if you know what commit introduced this very bug because it gives you context, something that is lost in multiple, meaningless commit messages. You also make it easier for git's finest tools like git-bisect.

1. If a single feature is expressed in one single commit you can easily play around with, e.g. deploy features in different combinations to staging servers, `git cherry-pick` and so on.

1. If you realize a feature has gone wrong and you need to remove it from master you can’t do that easily if this feature is spread across multiple commits which are possibly already intertwined with other commits.
Yes, you can revert merge commits completely with *git* but it's a manual process and thus error-prone. Certainly not something you'd want to do at Saturday morning 3 a.m. when you need to revert something that made it live as fast as possible.

### How could we improve this?

The canonical *git* approach to this would be: __Squash it__.

Let's look at an example:

Say you have a feature branch called *fancy-feature* with a corresponding pull request where you want to squash the commits before you merge the PR.

Here's what you'd do:

```
git lp master..fancy-feature  # my alias for log --decorate --pretty=oneline --abbrev-commit

2b2197f (HEAD, fancy-feature) Fix indentation.
5c0ff6b Typo.
1d96940 Fancy Feature.
```

As you can see, we have 3 commits in this branch that are not in master.

Let's squash that:

```
git rebase -i master # -i for "interactive"
```

Now the following edit screen pops up in your favorite editor (whatever `env | grep -i editor` says):

```
1 pick 1d96940 Fancy Feature.¬
2 pick 5c0ff6b Typo.¬
3 pick 2b2197f Fix indentation.
```

Let's transform this to:

```
1 pick 1d96940 Fancy Feature.¬
2 f 5c0ff6b Typo.¬
3 f 2b2197f Fix indentation.
```

I replaced the `pick` in the last 2 lines with `f` for `fixup` which means: "meld into previous commit, but discard it's commit message" (there's a helpful explanation in the rebase view as well).

Save the file and exit it - that's it. Now let's look at this branch again:

```
git lp master..fancy-feature
1e8ae44 (HEAD, fancy-feature) Fancy Feature.
```

__Now it's just one shiny commit.__

The last thing you need to do is update upstream:

```
git push -f origin fancy-feature
```

The `-f` flag is very important here. You changed history in your branch (which is intentional and in my humble opinion acceptable in a topic branch) so *git* requires us to pass this flag to tell *git* that we really want this.

### What's the downside of this?

- You need to get used to doing that at the end of the lifecycle of each branch. However, once you get used to this you'll hardly notice it again.
- force pushing. Potentially dangerous if this happens on master. Using aliases and sticking to workflows mitigate this. E.g. I have this

```
[push]
  default = tracking
```

in my .gitconfig, so for me it's just

```
git push -f
```

without even having to specify the branch name so there's no potential of screwing up the branch name or related errors.

- We lose the commit history within a pull request. However, I have never seen that this was a problem because after a feature is done and approved you'll almost never be like "Why again did I go from this to that?". And even if you were, the *git* history would be the wrong place for me in any case - such a rationale would be better placed in a changelog entry or the commit message.

### Who else is doing it like this?

A lot of popular open-source project are doing this e.g. *Rails* and *Sinatra*.

Check  out the *Rails* log:

```
05ab902 Merge pull request #15042 from arthurnn/revert_dirty_transactions
314cbea just call the method and assert the return value
198d9e3 Reverts "Fix bugs with changed attributes tracking when transaction gets rollback"
0a7beb1 Merge pull request #15041 from arthurnn/update_ruby
e019ffa Use ruby 2.1.2 on travis
ea58684 add tests for path based url_for calls
```

Very clean and readable. You can just read the last 20 - 30 lines and you'll have a good mental image of what happened in the last couple of days.

### Wrapping it up

If you're not doing it like that already, I encourage you to give it a shot.
I believe once you get used to having atomic pull requests, you'll never go back again.
