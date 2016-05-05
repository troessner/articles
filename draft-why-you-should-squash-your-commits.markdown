
When you look at bigger pull requests, you can see that they often contain dozens and dozens of commits. With a lot of them containing useless commit messages like "Fix.", "Typo", and so on.
This implies that this very pull request / branch is not atomic, meaning that "it does not behave like one single entity, but like a lot".

What's the problem with this?


1.) git-log is one of the most important git tools and using it properly can make all of our lifes easier.




For instance, with a proper git-log you can just type in "git log" and you can immediately see the last significant changes to the source - something that is utterly impossible when you merge branches with hundreds of commits. If one feature is split across many commits git-log is useless because it can’t tell what belongs where (I am aware that git-log can show you what commits belonged to what branch but this is still useless in most cases because it is just not terse and concise enough for the human brain to operate on)




2.) Bugfixing can be significantly easier if you know what commit introduced this very bug because it gives you context (something that is lost in multiple, meaningless commit messages). Not to mention that you can use git's finest tools like git-bisect.




3.) If I realize a feature has gone wrong and I need to remove it from master I can’t do that if this feature is spread across multiple commits which are already intertwined with master without a huge, error-prone effort.




4.) If one feature is one commit I can easily play around with, e.g. deploy features in different combinations to staging servers.




How could we improve this?




The canonical git approach to this would be: Squash it.




To give you an example:




Say you have a feature branch called "fancy-feature" with a corresponding pull request where you want to squash the commits before you merge the PR. Here's what you'd do:




git lp master..fancy-feature  # my alias for log --decorate --pretty=oneline --abbrev-commit


2b2197f (HEAD, fancy-feature) Fix whatever.
5c0ff6b Typo.
1d96940 Fancy Feature.




As you can see, we have 3 commits in this branch that are not in master. Let's squash that:




git rebase -i master # -i for "interactive"




Now the following edit screen pops up in your favorite editor (whatever `env | grep -i editor`) says:




1 pick 1d96940 Fancy Feature.¬
2 pick 5c0ff6b Typo.¬
3 pick 2b2197f Fix whatever.




Let's transform this to:




1 pick 1d96940 Fancy Feature.¬
2 f 5c0ff6b Typo.¬
3 f 2b2197f Fix whatever.




I replaced the "pick" in the last 2 lines with "f" for "fixup" which means: "meld into previous commit, but discard it's commit message" (there's a helpful explanation in the rebase view as well)




Save the file and exit it. That's it. Now let's look at this branch again:




git lp master..fancy-feature
1e8ae44 (HEAD, fancy-feature) Fancy Feature.




Now it's just one shiny commit.




The last thing you need to do is update upstream:




git push -f origin fancy-feature




The "-f" flag is the important thing here. You changed history in your branch (which is intentional and ok in a branch) so git requires us to pass this flag to tell git that we really want this.




What's the downside of this?




- You need to get used to doing that at the end of the lifecycle of each branch. However, once you get used to this you'll hardly notice it again.




- force pushing. Potentially dangerous if this happens on master. Again, using aliases and sticking to workflows mitigate this. E.g. I have this


[push]
  default = tracking




in my .gitconfig, so for me it's just "git push -f" without even having to specify the branch name (so there's no potential of screwing up the branch name or whatever)




- We lose the commit history within a pull request. However, I have never seen that this was a problem because after a feature is done and approved you'll almost never be like "Why again did I go from this to that?". And even if you were, the git history would be the wrong place for me in any case - such a rationale would be better placed in a changelog entry or a source code comment.




Who else is doing it like this?




Basically every popular open-source project (e.g. Rails, Sinatra, and so on).




Check  out the rails log:


05ab902 Merge pull request #15042 from arthurnn/revert_dirty_transactions
314cbea just call the method and assert the return value
198d9e3 Reverts "Fix bugs with changed attributes tracking when transaction gets rollback"
0a7beb1 Merge pull request #15041 from arthurnn/update_ruby
e019ffa Use ruby 2.1.2 on travis
ea58684 add tests for path based url_for calls




Also notice the "Revert" commit in there. Something impossible for us to do now. Besides that, the git log looks flawless and actually helps you to see what's going on since one commit == one meaningful addition to the code.
