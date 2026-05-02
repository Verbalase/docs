# I'm Lost. How the hell do I do anything?! How does anyone learn this?!

Are you wondering how in the world anyone has learned to code for this game? Have you spent ten hours staring at your screen wondering why you even bothered trying? You're not alone.

### The Problem
Space Station 14 is a very complicated game- as of time of writing (the 18th of January 2025) the entire Goob Station codebase contains (according to [cloc](https://github.com/AlDanial/cloc))
**12,787,465** lines of readable text files across 18,579 files:
11,025,924 non-comment, non-blank lines of YAML
830,842 non-comment, non-blank lines of C#
452,851 non-comment, non-blank lines of JSON
21,015 non-comment, non-blank lines of Localizations (.ftl files)
10,691 non-comment, non-blank lines of XAML
9,447 non-comment, non-blank lines of XML
And tens of thousands of lines of other filetypes.

Adding to this is a daunting task because there is absolutely **no way** you are going to read, understand, and remember every single line of code in the codebase.

But guess what? You don't have to understand the entire codebase. It might feel like it sometimes, but really, it's not that hard coding for this game. There being so much also means there are a **ton** of functions, components, and systems already coded for you to use! The problem is, *you don't know how to find the ones you need and glue them together*.

### The Solution
The solution to this problem is simple, just look at things similar enough to what you're doing. Trying to add a shader overlay? **Copy paste another shader overlay and work from there**. Trying to add support for custom borg names? **Look at a PR adding some character creation Database field, or just look at how existing character creation stuff works!** Searching [wizden PRs](https://github.com/space-wizards/space-station-14/pulls) or [goob PRs](https://github.com/Goob-Station/Goob-Station/pulls) or other codebase PRs are probably the best way to find something similar to what you're doing (or maybe someone's already done it, lol). Having good knowledge on the game is super helpful with this as you can just go "oh well this has x functionality I can copy that" and just take from all around.

### TLDR
Coding from this game is copy pasting just as much as coding in general is :P. Copy and use existing stuff in the game people have already coded for you. I literally copy my own previous code these days.
