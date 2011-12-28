---
layout: post
title: Word Blender
---
# Word Blender

One of the coolest things to pop up recently is [VimGolf](http://vimgolf.com),
in which participants compete to solve "challenges." That is, they modify some
given text to match specific target output in the fewest number of
keystrokes&mdash;using the [Vim editor](http://www.vim.org/), of course. I
consider myself reasonably proficient with Vim; VimGolf is a humbling reminder
of how much I still have to learn.

VimGolf's best feature allows you to look at other solutions once you've
submitted your own, which provides a trove of great ideas for shaving
superfluous keystrokes from your editing. I quickly learned, for instance, that
saving a file with `:wq` is a two-character waste compared to the much more
svelte `ZZ`.

## The Challenge

The solution to one challenge, however, really blew my mind with all the nifty
Vim awesomeness I picked up from analyzing it. The challenge is called ["Word
Blender"][1] (submitted by VimGolf user [Tim Chase](http://vimgolf.com/gumnos)).
The basic idea is that all of the words in the sample text have had their
interiors reversed: for example, _talking_ becomes _tniklag_. The first and last
characters stay the same; only the characters in the middle have their positions
moved&mdash;thus words of fewer than four characters remain the same. To solve
the challenge, you need to fix all the thusly rearranged words.

[1]: http://vimgolf.com/challenges/4d2482950947c63e260000b1

## The Old College Try

My own [solution][2] is fairly pedestrian, a simple global search-and-replace:

[2]: http://vimgolf.com/challenges/4d2482950947c63e260000b1#4d24f6420947c63e26000181

`:%s/\v(\w)(\w+)(\w)/\=submatch(1).join(reverse(split(submatch(2),'\zs')),'').submatch(3)/gZZ`

At 93 characters, it's nowhere near the best (at the time of this writing, a
mind-boggling _30_ keystrokes), but I still had to dig through Vim's expansive
help to come up with it. I'd never used the `\=` construct in a replace command
before, so all the function calls in there were fresh Vim learning.

The `\v`, at the beginning of the search regular expression, was another thing
I've picked up since taking up VimGolf. By default, Vim requires that a lot of
the "magic" characters in regular expressions be escaped (generally, those with
anything beyond their literal interpretations, like parens or pipes). The `\v`
flag triggers "very magic" mode, relaxing the escaping rules. So in my solution,
that saved a net of four characters, since each of the parens in `(\w)(\w+)(\w)`
would have required a backlash prefix! (Beyond padding a VimGolf score, this is
useful for making complex regular expressions more readable.)

## Awesome Sauce

The real inspiration for this blog post, however, is [Matthew
Draper](http://vimgolf.com/_matthewd)'s solution to the same challenge. Here it
is in its entirety:

`0qx"RXq/\v&lt;\w{4}qq"rxlyee@=len(@")@x"rPnq119@qZZ`

Now I'll break down what's going on in this compact bundle of win:

1. **`0`**. Move to the first character of the line.
2. **`qx`**. Start recording a macro, saving it in the _x_ buffer.
3. **`"RX`**. `X` deletes the character to the _left_ of the cursor; the `"R` 
   preceding it stuffs that deleted character into the `r` register.  
   And&mdash;extra nifty&mdash;the fact that it's a _capital R_ means it 
   _appends_ to the `r` register instead of just replacing its contents.
4. **`q`**. Finish recording the _x_ macro.
5. **`/\v&lt;\w{4}`**. Search for the next chunk of four word characters.  
   Remember, words of fewer than four characters don't need to be changed.  Note 
   the use of `\v` to trigger "very magic" mode in the search expression.  It 
   actually doesn't save anything (`/\&lt;\w\{4}` is an equivalent search 
   expression with the same number of characters), but it arguably looks 
   cleaner.
6. **`qq`**. Start recording a second macro, this one saved in the _q_ buffer.
7. **`"rx`**. The `x` deletes the character the cursor is on; the `"r` stuffs it 
   into the _r_ register. Thanks to the preceding search, we're at the beginning 
   of a target word, so we've just taken the first character of that word and 
   deleted it and stuffed it into a register for later retrieval. Note that it's 
   a _lowercase r_ this time, so the register's contents are _replaced_ with 
   just that single letter. Later, this will effectively reset the _r_ register 
   when another word is visited.
8. **`l`**. Move one character to the right.
9. **`ye`**. Copy all the characters through the end of the word.
10. **`e`**. Move to the end of the word.
11. **`@=`**. `@` is normally used to execute a macro; for instance, `@x` would 
    execute the macro recorded in steps 2&ndash;4.  `@=` is a special case: You 
    are moved to Vim's command line, where you can enter an expression whose 
    results will be executed. We'll see the expression Matthew uses in the next 
    step.
12. **`len(@")`**. This is the expression (followed by the enter key) provided 
    to the `@=` command initiated in the preceding step. It calls the len 
    function on `@"`. In the context of Vim's command line, the `@` lets you 
    access the contents of a register. The `"` is a way to refer to the 
    so-called unnamed register, which contains whatever was last yanked. The 
    last thing we yanked was the current word, less two characters (we already 
    chopped off the first letter (step 7), then moved one character to the right 
    (step 8) before yanking rest of it (step 9). So the result of this 
    expression is the length of the current word less two. You know how most 
    commands in Vim can be prefaced with the number of times you'd like it to 
    execute? These last two steps have set that up, dumping us back into normal 
    mode as though we'd just typed that calculated length value in manually; 
    Vim's now waiting for us to tell it what command to execute that number of 
    times. Marvelous!
13. **`@x`**. Execute the macro stored in buffer _x_, recorded in steps 
    2&ndash;4, which deletes the character to the left of the cursor and 
    _appends_ it to the _r_ register. Thanks to steps 11 and 12, we're executing 
    this macro a number of times equal to the length of the current word less 
    two. We already moved to the last character of the word (step 10), so this 
    effectively backspaces across the word, adding each character deleted to the 
    buffer. Since these interior characters are in reverse order, the _r_ 
    register is being built up in the _correct_ order. Ingenious!
14. **`"rP`**. As a result of step 13, there's only one character left in the 
    current word, its last letter&mdash;the rest has all been stashed in the _r_ 
    register. The `P` puts its source contents immediately _before_ the cursor 
    position; the `"r` tells Vim that _what_ we want to put is the contents of 
    the _r_ register: the rest of the current word, now all in correct order. 
    Brilliant!
15. **`n`**. Move to the next match to our search criteria (`\w{4}`, the next 
    word of at least four characters).
16. **`q`**.  Stop recording the macro.
17. **`119@q`**. Execute the macro recorded in the _q_ buffer (steps 6 – 16) 119 
    times. It goes through the rest of the sample text, fixing all the words 
    that need it. It's not as elegant as the rest of Matthew's Vim fu, but after 
    the magic offered up elsewhere in this snippet, I'm more than willing to let 
    this slide.
18. **`ZZ`**. Save the file and quit Vim.

## Final Thoughts

I love Vim. Every time I feel like I've climbed to a certain level of mastery, I
glance up and get a glimpse of how much more of the mountain remains to be
climbed. (This actually permeates just about every aspect of my career in
computer science.) I find that exciting: There is _always_ more to learn.

It's incredible that I was able to learn all this from fifty (or so) twitches of
Matthew Draper's deft fingers. Perhaps it's even more incredible that I've
devoted a substantially larger number of my own keystrokes to documenting it. I
think writing all this helped me understand it better, as well as cementing it
in my mind. Hopefully, it will help somebody else along their way to Vim
enlightenment.

If you're a Vim user, you really ought to check out VimGolf. It's a really fun
way to hone your skills. I do have a criticism, though.

Currently, you can only see the solutions that are a little bit above your own
score. It would be _much_ better if, once you've submitted a solution, you could
see _all_ of the other solutions. VimGolf suggests that you submit multiple
entries, incrementally improving your score and thereby unlocking higher and
higher scores. This is good in theory, but in practice your immediate betters
are usually so similar to your entry that you don't learn much, if anything.
This is especially true for challenges that get a lot of entries&mdash;you might
not see anything that's even one keystroke better than your score. You end up
stuck in a forest of nearly identical trees.  Sure, I could just copy the
best-scoring entry I have access to&mdash;assuming it's better than my own
score&mdash;and climb the ladder that way, but that just feels like cheating.
I'd much rather be stuck with my initial score, and then learn from others'
approaches so that I can do better on similar challenges in the future. After
all, VimGolf is a game, a fun way to play with Vim and incidentally learn more
about the editor. The sense of competition drives you to become a better Vimmer,
but ultimately what I want from VimGolf is a better command of Vim that I can
use in real-world coding.
