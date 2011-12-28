---
layout: post
title: Make It More Readable
---
# Make It More Readable

[The pertinent 
challenge](http://www.vimgolf.com/challenges/4ef209ef78702b0001000019).

Say I've got a file (see below) that's one long uninterrupted wad of code, 
interspersed with the occasional comment. It's all jammed together with no white 
space, and I'd like to ... well, make it more readable. Specifically, by 
inserting two blank lines before every comment.

<pre>
#Set the global prefix key to C-q (default is C-b)
set-option -g prefix C-q
bind-key C-q last-window
# Remove default binding since we’re replacing
unbind %
bind | split-window -h
bind - split-window -v
# Set status bar
set -g status-bg black
set -g status-fg white
set -g status-left '#[fg=green]#H'
# Highlight active window
set-window-option -g window-status-current-bg red
set -g status-right '#[fg=yellow]#(uptime | cut -d "," -f 2-)'
# Set window notifications
setw -g monitor-activity on
set -g visual-activity on
# Automatically set window title
setw -g automatic-rename
</pre>

(In the solutions below, I omit the `ZZ` at the end, which saves and quits the 
file—saving two characters over the equivalenet `:wq<CR>`. Hey, every keystroke 
matters in Vim golf!)

## Finding and Replacing: The Substitute Command

I decided on a simple, straightforward substitute command. There are a couple of 
caveats about the target file to be aware of:

1. The first line is a comment, but I don't want to insert lines before it.
2. Some lines have hash characters in the middle of the line, not just the 
   beginning.

With those caveats in mind, here's my first attempt:

    :2,$s/^#/\r\r#<CR>

This is just shy of a global replace; the `2,$` at the beginning of the command 
specifies a _range_ from the second line of the file to the last. This addresses 
the first caveat.

Ranges, like so much else about Vim, are flexible, powerful ... and cryptic. I 
can specify line numbers explicitly, such as `2`. I also have access to a bunch 
of line-specifying shorthand, such as the `$`, which translates to "whatever the 
last line of the file is." (Remember this meaning of `$`; it will show up 
again.)

The search string, `^#`, only finds hashes at the beginning of a line, 
addressing the second caveat.

### Work Smarter

The smarter Vim golfers noticed something: _All the comments I'm targeting have 
a hash followed by a space character_. Thus searching for `# ` lets me do a 
straight-up global substitute via the `%` range shorthand:

    :%s/# /\r\r&<CR>

I've long thought of `%s` as an atomic unit: global find and replace. That could 
be because I learned about it early on in my Vim days, when I was just learning 
stuff by rote rather than delving into the vast inner workings of Vim.

But that's the wrong way of looking at it. Really, `s/foo/bar/`, 
`2,$s/foo/bar/`, and `%s/foo/bar/` are all the same underlying `substitute` 
command—it's just that the latter two cases explicity specify a range. There's 
always a range involved, in fact, but when it's not explicity specified, it 
defaults to `.`, shorthand for the current line.

Oh, and hey, the ampersand in the replace portion of the command may require 
some explanation. It gets replaced by the entire match (in this case `# `).

### An Interesting Variation

This doesn't save any characters, but it's cool and uses an extra dash of Vim 
voodoo:

    :%s/# /\r&<CR>@:

It looks almost exactly like the last solution, however:

- Only one new line gets inserted by the substitute command
- There's that funky `@:` at the end

If you've used Vim's awesome macro-recording feature, you're familiar with the 
`@`-letter convention for executing a previously recorded macro. Well, Vim 
provides some special macros in addition to those you record yourself.

So the `@:` "macro" is a lot like `.`. You know how `.` repeats whatever you 
just did, and it's awesome? `@:` is like that, but re-executes the last 
Ex command. (As a mnemonic, it may help to remember `:` is the key 
press required to enter command mode.)

In this specific case, it re-executes the `%s/# /\r&` substitute command.  I end 
up with the same result, only one inserted newline at a time.

## The Global Command

The `substitute` command I've been using thus far is actually a special case of 
the `global` command. With the `global` command, you specify a search pattern, 
but not for the sole purpose of replacing the found text. Nope, the search 
pattern you give to `:g` just grabs every line in the file that matches the 
pattern. The second part of the global command is where you tell it what you 
want to do to all the matching lines.

### Lame Example

As an example, I'll re-implement one of the previous solutions as a global 
command:

    :g/# /s/# /\r\r&

Okay, that's admittedly pretty lame; I'm effectively embedding the exact same 
`s` command in there. Heck, even the search pattern is repeated. Ugh! (Although 
repeating the search pattern is not necessary; `:g/# /s//\r\r&` would work just 
as well.) But here's the takeaway: Everything after the second `/` is the 
Ex command to execute on each line. So what if I did something 
interesting?

### Awesome

Here's the slick `global`-command solution several folks came up with:

    :g/# /$t-<CR>@:

Ignoring the `@:` at the end I'm left with:

    :g/# /$t-

So `:g/# /` fetches all the lines matching `# `. Then,
_everything after the second `/`_ is the command to execute against each matched 
line. In this case, `$t-`.

The `t` command is a briefer synonym of the `co[py]` command. It copies lines 
from the given range (the stuff before the `t`) to the line beneath the given 
address (the stuff after the `t`).

The _from_ range here is `$`, which, you may recall from my original attempt, is 
equivalent to the last line of the file. It just so happens that the last line 
of the file is a blank line (that's hard to tell from the copy/pasted input file 
above, but it becomes readily apparent when you open the file with `vimgolf`).

That means the trailing `-` is the _to_ address. But the `-` itself is not 
actually an address; it's an address _modifier_. It subtracts one from the 
actual address—which is not explicitly specified. However, similar to the 
`substitute` command, when no address is specified, it falls back to the current 
line. In the context of a `global` command, the current line changes to each 
matched line as it is processed.

The modifier is necessary because `t` copies its lines _beneath_ the target 
address; I'd end up with blank lines underneath the comments instead of above 
them.

With three magical characters, I copy the blank line from the bottom of the 
file, and paste it underneath the lines immediately preceding the comments.
Another way of saying that is I'm pasting a blank line _before_ the comments. 
Follow that up with the handy `@:` command, and I've got myself some nicely 
separated chunks of comment-prefixed code.

## Further Reading

As always, Vim's extensive documentation is useful for finding out more about 
this. Here's some online resources for finding out more:

- [Ranges](http://vimhelp.appspot.com/cmdline.txt.html#cmdline-ranges)
- [The Global Command](http://vimhelp.appspot.com/repeat.txt.html#multi-repeat)
- ["Power of g"](http://vim.wikia.com/wiki/Power_of_g), by Arun Esai
