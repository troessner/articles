For quite some time, a lot of companies have been on the "yes, we do pair programming" bandwagon and still are. Some of them even to the (ridiculous) extent that pair programming was or is mandatory a la "at least 2 days a week".

I've been through that and those were certainly some bad times in my software development career.

I have always been sceptical of pair programming to begin with.

First of all, it's synchronous. You have to pay attention that very same time your partner pays attention. But people and their rythm are different and if you do this all day, this gets exhausting very quick.

Second, regardless of what technology you're using, there will always be artefacts you have to generate and update, tests to run, a network connection you wait for and so on. If you use a language or framework that is more machine-friendly than developer-friendly this might amount to a lot. But even with something developer-friendly like Ruby, Python or Elixir this will steal a lot of your time.

And during that time one partner takes care of those mundane tasks the other drifts off - there's just nothing you can do about it, this is how most humans work. And if one partner looses focus, he takes the other one down with him ususally.

So in a nutshell, there are quite a lot of situations where __pair programming sucks hard__.

### So what are the alternatives?

[Pull requests](https://help.github.com/articles/using-pull-requests/).

They share a lot of benefits with pair programming but don't have a lot of the disadvantages.

Most importantly: They are asynchronous by design so you can dive into problems when you feel like. Your time, your speed. Very important.

Besides that it is still collaborative. You can still have a conversation in that pull request that is (almost) as precise as if you would be sitting right next to somebody.

The biggest advantage is the biggest downside as well: Due to the asynchronous nature there might be some delay in communications. This is something though that is fixable with a good development culture.

Of course there are a couple of exceptions though where pair programming makes not only a lot of sense, but offers significant benefits:

- Learning workflows from your partner. Sure you can read up on git and friends, but nothing makes it easier for you than to see those git workflows in action. Also, seeing how your partner executes those workflows takes the fear away to fuck up things on the remote end when you have to do it yourself: You have not only seen how it's done, you have seen the result of it. To stick with the git example, a lot of git novices are afraid "cause damage on the remote git repository" - seeing how easy it is to juggle multiple branches and pushing to remote lowers their mental barrier.
- Learning how to use tools from your partner. That's basically how I learned vim.
- Onboarding (kind of obvious).
- Learning a new language in a [polyglot environment](https://en.wikipedia.org/wiki/Polyglot_(computing)).

### Wrapping it up

I guess with pair programming it's like with everything in life: Don't make up rules just for the sake of having them and follow them without questioning them.

Use pair programming judiciously - quite often you can achieve the same thing better with pull requests.
