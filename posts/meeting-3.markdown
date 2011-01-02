# Meeting 3

## Value context

Last week we talked about **amount context**, the thing that causes assignment
and function calls to behave differently depending on their surroundings.

                     List context
                   (several values)
    
                         ---
    
                    Scalar context
                   (a single value)
    
                         ---
    
                     Void context
             (a value that's never "used")

Turns out that within the scalar context, there is a further subdivision:

            Str   |      Num      |     Int
         a string | a real number | an integer
                  |               |
         ---------+---------------+-----------
      
                        Bool
                   a boolean value

We call this **value context**, and understanding it is the key to
understanding what a scalar value is in Perl. A scalar value is something that
can take the form of all of the above types &mdash; string, number, or boolean
&mdash; depending on the context. A few examples will help highlight this:

    my $string = "3 little mice";
    say $string;          # "3 little mice"       Str used as Str
    say $string / 9;      # 0.333333333333333         used as Num
    say $string + 2;      # 5                         used as Int
    if ($string) { ... }  # (True)                    used as Bool
    
    my $real = 1 / 3;
    say scalar reverse $real;
                          # 333333333333333.0     Num used as Str
    say $real * 4;        # 1.333333333333333         used as Num
    my @a = "a".."c";
    say @a[$real];        # "a"                       used as Int
    if ($real) { ... }    # (True)                    used as Bool
    
    my $integer = 42;
    say $integer x 4;     # "42424242"            Int used as Str
    say $integer * 0.1;   # 4.2                       used as Num
    say $integer * 2;     # 84                        used as Int
    if ($integer) { ... } # (True)                    used as Bool
    
    my $boolean = !!$integer;
    say $boolean;         # "1"                  Bool used as Str
    say $boolean / 10;    # 0.1                       used as Num
    say @a[$boolean];     # "a"                       used as Int
    if ($boolean) { ... } # (True)                    used as Bool

Even in showing all these "conversions", the line between the different types
blurs terribly. We're best off in simply viewing a scalar as something that
can take on the four different value types (`Str`, `Num`, `Int`, `Bool`)
depending on its value context.

(From a type perspective, one could perhaps view `Int` as a subtype of `Num`,
and `Num` as a subtype of `Str`... but that view doesn't serve us in any
particular way in Perl. In fact, the internals of a scalar treat the three
types as being on equal footing. The `Bool` type is always calculated from
one of the other types.)

As a final instructive point, consider what happens when an array is used
in boolean context:

    if (@array) {
        ...
    }

The reduction takes place in two steps:

* First, since the array is used in value context, it is de-promoted from
  being a set of values to being a scalar. For arrays, this happens to be the
  length of the array, i.e. an integer.

* Then, since the integer is used in boolean context, it is de-promoted from
  being an Int to being a Bool. It becomes a true value if (and only if) the
  integer is non-zero.

In other words, writing `if (@array) { ... }` means "if `@array` has any
elements...". It is plasticity of this kind that risks confusing newcomers,
but that in the long run makes Perl a comfortable and flexible tool.

## Loop control

Perhaps the simplest loop imaginable is the infinite loop:

    while (1) {
        ...
    }

This can of course be refined by giving the loop a more specific condition
than just `1` &mdash; but there are also tools for interrupting an iteration
that's already underway; `next` and `last`:

   next         immediately go to the next iteration
   last         immediately exit the loop

There's also `redo`, which doesn't add any new behavior to `next` in a `while`
loop, but which comes in handy sometimes in `for` loops:

   redo         immediately re-start the current iteration

It's good to get into the habit of thinking in terms of `next` rather than
complex C-style `for` loops:

    # You can do this:
    $ perl -wle 'for (my $i = 0; $i < 50; $i += 10) { print $i }'
    0
    10
    20
    30
    40
    
    # But strive to do this instead:
    $ perl -wle 'for my $i (0..49) { next if $i % 10; print $i }'
    0
    10
    20
    30
    40

The reason this is preferable is that many `next` calls can be put after each
other on separate lines, keeping separate conditions separate. In contrast,
a single C-style `for` loop header has to capture *all* of those conditions
in a single expression. `next` simply scales better with increasing
complexity.

Often one is faced with nested loops, and `next`/`last`/`redo` then quite
naturally act on the innermost surrounding loop. In the presence of nested
loops, it is almost always a good idea to give the loops **labels**, and
then to supply those when doing `next`/`last`/`redo`:

     OUTER:
     while (1) {
         next OUTER if ...;
         last OUTER if ...;
     
         INNER:
         for (@values) {
             next INNER if ...;
             redo INNER if ...;
             last INNER if ...;

             next OUTER if ...;
             last OUTER if ...;
         }
     }

Using labels in this way not only gives a large amount of control; with wisely
chosen names for the labels, it might make things much easier to read and
understand.

## Nim

Nim is a game where players alternate taking a non-zero number of beads
from one of three piles, and the winner is the one taking the very last
bead.

Running the game looks like this:

    $ ./nim.pl 
    1 ###
    2 ####
    3 #####
    
    > 4 from 3
    1 ###
    2 ####
    3 #
    
    >

And here's how we implement it in 41 lines of Perl 5:

    #! /usr/bin/env perl -w
    use 5.010;
    use strict;

The first line is actually for the shell, telling it how the script is to be
executed. Relying on `/usr/bin/env` here is good practice, because then the
`perl` executable is found no matter where it has been installed, which might
vary from computer to computer. The `-w` flag corresponds exactly to `use
warnings`.

`use 5.010` makes older Perl versions refuse to run this script. That's good,
because we're using some of the new features from Perl 5.10. You might wonder
why version 5.10 is written as `5.010`... that's a general convention where
the *minor* version number (the 10, in this case) is written out as multiples
of `0.001`. Had we requested Perl 5.10.1, we'd have written it as `5.010_001`.
(The underscore is not essential, it's just there for grouping.)

Because we `use strict`, among other things, we now always have to declare
variables. Lexical variables (the most common form) are declared with the
`my` keyword, as shown here:

    my @piles = (3, 4, 5);

With this line, we initialize our "game" data, filling up the piles with
different amounts of stones.

    MOVE:
    while (1) {

An infinite loop. This loop goes around once for every new move in the game.
The `MOVE` label will come in handy later.

        my $row = 1;
        for my $beads (@piles) {
            say $row, ' ', '#' x $beads;
            ++$row;
        }
        say '';

Here we use the convenient `x` operator to indicate that we want as many
[octothorpes](http://en.wikipedia.org/wiki/Number_sign) as the number of
`$beads` in that particular pile. Speaking of which, the `my $beads`
declaration may look out of place after the `for` keyword, but it's a way
to specify your own `for`-loop placeholder, to be used in each iteration
instead of the default `$_`.

        INPUT:
        while (1) {

Another infinite loop. Goes around once for every input request made to the
user. This one is only *seemingly* infinite, because we `last` out of it when
we get the correct input.

            print "> ";
            my $command = <> // last MOVE;

We print a prompt, and then read input from `STDIN`. (The `<>` operator can
read from any filehandle, but left empty, it reads from `STDIN`.)

The `// last MOVE` part could have been written serparately as `if (!defined
$command) { last MOVE; }`, but the shorter way is not so bad either. The `//`
operator itself is known as "defined-or", and chooses its left-hand side if
that side is defined, otherwise its right-hand side.

            chomp $command;
            next INPUT
                unless $command;

If the user just pressed Enter, we'll want to show another prompt, so we do
`next INPUT` to get back to the beginning of the inner loop in case we find
nothing in `$command` (`unless $command`).

Problem is, there's always a newline in something read through `<>`, since
any line has a newline at the end. So we start by removing that newline with
`chomp`.

            if ($command =~ /^(\d+) from (\d+)$/) {
                my ($beads, $pile) = ($1, $2);

The regex matches some numbers, the string `" from "`, and some numbers. It
captures the numbers into the variables `$1` and `$2`, which are then assigned
to variables with descriptive names.

The characters `^` and `$` help anchor the regex to beginning-of-string and
end-of-string, respectively. We don't want to match just part of the string.

                if ($beads > $piles[$pile - 1]) {
                    warn "There are only ", $piles[$pile - 1],
                         " beads in that pile.\n";
                    next INPUT;
                }

On the off chance that the user tries to remove more beads than there are in
the pile, give a warning and then do another round through the inner loop.
`warn` prints things to `STDERR` rather than `STDOUT`; it doesn't make any
difference here, but it comes in handy sometimes to have warnings be separated
from ordinary output.

The reason we put in a `\n` at the end of the printout is a bit curious.
Without it, we get line-and-file information added to the message, something
that we do not need at all in this case. Putting in the `\n` gives us just
the message.

                $piles[$pile - 1] -= $beads;
                last INPUT;
            }

Here we do the actual removal of beads, and then exit the inner loop.

            else {
                warn "Unrecognized command.\n";
            }
        }
    }

The `else` goes together with `if ($command =~ /^(\d+) from (\d+)$/) {`
(something which is more easily seen without a lot of comments interspersed
in the code), so this is the message written out to a user that types
something not on the form `2 from 3`.

We could have added a `next INPUT` after outputting the warning. But, since
we're at the end of the inner loop, that is what will happen anyway.

Here's the whole script:

    #! /usr/bin/env perl -w
    use 5.010;
    use strict;
    
    my @piles = (3, 4, 5);
    
    MOVE:
    while (1) {
        my $row = 1;
        for my $beads (@piles) {
            say $row, ' ', '#' x $beads;
            ++$row;
        }
        say '';
    
        INPUT:
        while (1) {
            print "> ";
            my $command = <> // last MOVE;
            chomp $command;
            next INPUT
                unless $command;
            if ($command =~ /^(\d+) from (\d+)$/) {
                my ($beads, $pile) = ($1, $2);
                if ($pile > @piles) {
                    warn "There are only ", scalar(@piles), " piles.\n";
                    next INPUT;
                }
                if ($beads > $piles[$pile - 1]) {
                    warn "There are only ", $piles[$pile - 1],
                         " beads in that pile.\n";
                    next INPUT;
                }
                $piles[$pile - 1] -= $beads;
                last INPUT;
            }
            else {
                warn "Unrecognized command.\n";
            }
        }
    }

## Exercises

* Use the output from `perl -MO=Deparse -ne ''` to explain to yourself why
  `^D` (control-D) ends a `perl -n` script automatically, whereas in the
  game we had to insert a manual check `// or last MOVE`.

* There's one bit of error handling missing from the game. Make sure that a
  warning is printed (and a fresh prompt re-issued) if the player tries too
  take beads from a pile that doesn't exist.

* Make the warning `There are only 0 beads in that pile` instead read `There
  are no beads left in that pile` when there are no beads left in a pile.

* Instead of `>`, make the prompt say `player 1>` or `player 2>`, depending
  on whose turn it is.

* Look up the `grep` built-in. Use it to make the outer loop keep going only
  as long as there is still at least one bead in at least one pile. Finish
  the game by printing who won.

### Bonus exercises

(Non-trivial, but rewarding.)

Here's a subroutine to make the computer play a competitive game of Nim:

    sub computer_move {
        my @piles = @_;
    
        my $excess = 0;
        for my $beads (@piles) {
            $excess ^= $beads;
        }
    
        my $pile = 1;
        if ($excess) {
            for my $beads (@piles) {
                return($excess, $pile) if ($beads & $excess) == $excess;
                ++$pile;
            }
        }
    
        do {
            $pile = int(rand @piles) + 1;
        } until $piles[$pile - 1];
    
        return(1, $pile);
    }

* Put it in your program, and make the appropriate changes so that the computer
  now plays as player 2. Make sure you win against it at least once.

* Read `perldoc perlop` or [the online equivalent](http://perldoc.perl.org/perlop.html)
  and find out what the `^` and `&` operators do.

* Explain to yourself how the computer algorithm works and how to beat it.
