# Meeting 1

## The Unix model: pipes

With most Unix shells, you can easily chain several commands together, having
each command use the output of the previous one as input:

    $ grep -i '^q' /usr/share/dict/words | grep -iv 'qu' | tr A-Z a-z | uniq
    qasida
    qere
    qeri
    qintar
    qoheleth
    qoph

As far as results go, this is equivalent to running the commands one after
another, leaving intermediate results in temporary files:

    $          grep -i '^q' /usr/share/dict/words > /tmp/1
    $ < /tmp/1 grep -iv 'qu'                      > /tmp/2
    $ < /tmp/2 tr A-Z a-z                         > /tmp/3
    $ < /tmp/3 uniq
    q
    qasida
    qere
    qeri
    qintar
    qoheleth
    qoph

This gives the same result, but Unix pipes are smarter than that: they start
all the processes at once, and have them wait for each other:

    $ perl -E 'say ++$n while 1' | less         # won't block indefinitely

Of course, that only works for programs that can start outputting something
without having all the input first.

    $ perl -E 'say ++$n while 1' | sort         # will block indefinitely

Anyway, here's the important part: Unix tools are developed around the idea
of files being passed around as **streams**, each being made up of **records**
(lines) containing zero or more **fields**. (What constitutes a field varies
with the application and the problem.) Unix pipes help leverage this by
making any command potentially play along with any other command.

## `sed`

With `sed`, one can see the stream philosophy showing. Lines come in on the
assembly line, we exectue various commands on them, and spit them out.

    $ grep ^proo /usr/share/dict/words | sed 's/oo/(O_O)/'
    pr(O_O)
    pr(O_O)emiac
    pr(O_O)emion
    pr(O_O)emium
    pr(O_O)f
    pr(O_O)fer
    pr(O_O)fful
    pr(O_O)fing
    pr(O_O)fless
    pr(O_O)flessly
    pr(O_O)fness
    pr(O_O)fread
    pr(O_O)freader
    pr(O_O)freading
    pr(O_O)froom
    pr(O_O)fy

The `s` means "substitute". You can use any character after the `s`. not just
slashes.

    $ sed 's:foo:bar:'
    $ sed 's!foo!bar!'
    $ sed 's#foo#bar#'

Sed recognizes the quantifier `*` for 0-or-more matches. It doesn't recognize
`+` or `?`.

    $ echo 'f fo foo FOO' | sed 's/fo*/bar/g'
    bar bar bar FOO

Oh, and the `/g` flag makes the search "global", so it's done more than once.
It's only done in a non-overlapping way, though:

    $ echo 'abababa' | sed 's/aba/aCa/g'
    aCabaCa

Otherwise there would be a risk of infinite regress.

Here's an example that shows captures (`\(...\)`), character classes (`[...]`),
and capture references. The `sed` scripts deletes duplicate words from a text.

    $ echo 'going to to feed the the cat' | sed 's!\([a-zA-Z]*\) \1 !\1 !g'
    going to feed the cat

You can make several substitutions inside one `sed` script:

    $ echo 'the Great White North' | sed 's/North/Whale/; s/White/Angry/'
    the Great Angry Whale

Actually, there are a number of other commands in `sed` besides `s`. They
allow the insertion or deletion of lines, or the manipulation of a so-called
"hold buffer". With these commands, the scope of `sed` is actually quite
broad. However, more advanced scripts in `sed` are probably better written
as scripts in more advanced languages. `:-)`

## `awk`

With `awk`, one can find and manipulate fields.

    $ ps
      PID TTY           TIME CMD
     5063 ttys000    0:00.05 -bash
     5301 ttys003    0:00.22 -bash
    
    $ ps | awk '{ print $3 }' # prints the third column
    TIME
    0:00.05
    0:00.22
    0:00.00

    $ ls -l | awk '!/total/ { print $1 }' # prints first column but not 'total'
    -rw-r--r--
    -rw-r--r--
    -rw-r--r--
    drwxr-xr-x@
    drwxr-xr-x
    -rw-r--r--
    -rw-r--r--

The general form of an `awk` program is `'<pattern> { <code> }'`. Patterns
can be regexes within `/.../` (and the regexes can be negated with a `!`, as
in the example above); they can be expressions in general (such as `NR == 10`
to only match line 10); or they can be `BEGIN` or `END`.

    $ ls -l | awk '{ s += $5 }; END { print s }' # sums all the byte sizes
    1572695

The regexes in `awk` are similar to `sed`, though capturing parens (`(...)`)
are no longer backwhacked. As opposed to `sed`, `awk` recognizes the `+` and
`?` quantifiers.

`awk` can also be used in more advanced situations. It has `if` statements,
`while` loops, and `for` loops.

## `perl`

Perl was originally made to occupy much the same niche as `sed` and `awk`.
Here are the previous `sed` and `awk` examples with `perl` instead:

    $ grep ^proo /usr/share/dict/words | perl -wpe 's/oo/(O_O)/'
    pr(O_O)
    pr(O_O)emiac
    pr(O_O)emion
    pr(O_O)emium
    pr(O_O)f
    pr(O_O)fer
    pr(O_O)fful
    pr(O_O)fing
    pr(O_O)fless
    pr(O_O)flessly
    pr(O_O)fness
    pr(O_O)fread
    pr(O_O)freader
    pr(O_O)freading
    pr(O_O)froom
    pr(O_O)fy

    $ echo 'abababa' | perl -wpe 's/aba/aCa/g'
    aCabaCa

    $ ps | perl -wnle 'split; print $_[2]'
    TIME
    0:00.05
    0:00.27
    0:00.00

    $ ls -l | perl -wnle 'if (!/total/) { split; print $_[0] }'
    -rw-r--r--
    -rw-r--r--
    -rw-r--r--
    drwxr-xr-x@
    drwxr-xr-x
    -rw-r--r--
    -rw-r--r--

    $ ls -l | perl -wnle 'split; $s += $_[4]; END { print $s }'
    1572695

Note the use of the `-e` flag for command-line oneliners, and of `-n` and
`-p` to emulate normal `sed` and `awk` streaming behaviour, respectively.
The `-l` flag means "automatically handle line endings"; these are then
removed from the end of each incoming line, and added after each `print`
statement.

The `-w` flag turns on warnings. That's just good style, even for a oneliner.

In Perl, you have to manually `split` the record into fields, which are then
accessed through the variable `@_`. The indices are zero-based, so each
index above is one smaller than the corresponding `awk` field variable.

You have to explicitly use an `if` statement to emulate `awk`s pattern
matching. Just like `awk`, Perl has `BEGIN` and `END`.

## Exercises

* Use `grep` on `/usr/share/dict/words` to find all words with three
  consecutive double letters. An example would be "bookkeeper".

* Write an `awk` script that expects integers in the first two fields, and
  outputs their sub, difference, product, and quotient.

* Use either `perldoc` on the command line or
  [perldoc online](http://perldoc.perl.org/) to find out what the Perl
  equivalent of `awk`'s `NR` variable is. Also find out which arguments,
  if any, `split` accepts.
