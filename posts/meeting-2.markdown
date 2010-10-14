# Meeting 2

## List assignment

How can one switch values between two variables? It's not enough to just assign
first one to the other and then vice versa, because that'll overwrite one of
the values in the process.

    perl -wle '$a = 42; $b = 43;
    >          $a = $b; $b = $a;
    >          print $a; print $b'
    43
    43

One common solution is the **temporary variable idiom**; using a third
variable to store one of the values while it is being overwritten.

    $ perl -wle '$a = 42; $b = 43;
    >            $t = $a; $a = $b; $b = $t;
    >            print $a; print $b'
    43
    42

In Perl, there's a way to do **list assignment**, making a third variable
unnecessary. All the variables in a list assignment are assigned to
simultaneously. Note the parentheses on both sides of the `=` assignment
operator.

    $ perl -wle '$a = 42; $b = 43;
    >            ($a, $b) = ($b, $a);
    >            print $a; print $b'
    43
    42

There's nothing to prevent us from doing the initialization in the same way.

    $ perl -wle '($a, $b) = (42, 43);
    >            ($a, $b) = ($b, $a);
    >            print $a; print $b'
    43
    42

Let's see what happens if we forget the parentheses on the left-hand side of a
list assignment:

    $ perl -wle '$a, $b = ("OH", "HAI");
    >            print $a; print $b'
    Useless use of a constant in void context at -e line 1.
    Useless use of a variable in void context at -e line 1.
    Use of uninitialized value $a in print at -e line 1.

    HAI

Without the parentheses on the left-hand side, the assignment becomes an
ordinary scalar assignment. Only the last element of the right-hand side list
is ever used in the assignment. We get the first warning because `"OH"` is
never used; the second because `$a` is never used ("void context" means "the
value is never used and just falls out into space"); and the third warning is
from trying to print the undefined `$a`.

In short, what makes an assignment a list assignment is the presence of
parentheses on the left-hand side. Remember that; it's important. `:-)`

Let's see what happens if we leave out the parentheses on the right-hand side.

    $ perl -wle '($a, $b) = "OH", "HAI"; print $a; print $b'
    Useless use of a constant in void context at -e line 1.
    OH
    Use of uninitialized value $b in print at -e line 1.
    
    $

(The final dollar sign (`$`) is not part of the output; just used here to
indicate that the last line of output is empty.)

In this case, we *do* get a list assignment, but due to the lack of
parentheses, the assigned list of values is only one element long. We get a
warning for not doing anything with `"HAI"`, and then (after `"OH"` has been
printed) another warning for trying to print the undefined `$b`.

All these warnings, by the way, reach us because we have the `warnings` pragma
turned on through the -w flag. They're often very useful in understanding
misbehaving code, or in alerting the programmer to "thinkos"; always have
`warnings` turned on. This is how the above program looks without the
`warnings` pragma:

    $ perl -le '($a, $b) = "OH", "HAI"; print $a; print $b'
    OH

    $

(Again, final `$` isn't part of the output.)

The parentheses on the left-hand side make the assignment a list assignment.
Using an array variable on the left-hand side also makes it a list assignment.

    $ perl -wle 'my @a = 3..5; print for @a'
    3
    4
    5

The parentheses on the right-hand side are only necessary if you're actually
assigning a literal list of things. If you're assigning an array variable, you
don't need them.

    $ perl -wle 'my @a = 1..10;
    >            my ($a, $b) = @a;
    >            print $a; print $b'
    1
    2

(Also note that despite `@a` having eight more values here that are not
assigned to anything, we don't get a warning in this case. We only get a
warning when we have an overflow in a literal list of items. `@a` is not a
literal list, it's an array variable.)

    my ($a, $b, $c) = (1, 2, 3);    # literal list, parens needed
    my (#a, $b, $c) = @a;           # array variable, no parens needed

Similarly, if assigning from a subroutine returning a list, there's no need to
use parentheses on the right-hand side. (To illustrate this, the `localtime`
function is used. We'll come back to this function later.)

    $ perl -wle 'my ($second, $minute, $hour) = localtime;
    >            print $second; print $minute; print $hour'
    34
    9
    14

In a list assignment, you can mix scalars and arrays.

    $ perl -wle 'my ($a, @b) = "a".."d"; print $a; print @b'
    a
    bcd

Note that once an array is in the list on the left-hand side, it slurps up the
rest of the list on the right-hand side.

    $ perl -wle 'my (@a, $b) = "a".."d"; print @a; print $b'
    abcd
    Use of uninitialized value $b in print at -e line 1.

So it doesn't make much sense to have an array variable anywhere but last in
the list of things to assign to.

## Flattening

What happens if one tries to put an array in the list on the right-hand side of
a list assignment?

    $ perl -wle 'my @a = 2..4; my @b = (1, @a, 5); print for @b'
    1
    2
    3
    4
    5

Rather than keeping its identity as an array, `@a` is interpreted as a list and
flattened into `@b`, so that `@b` ends up containing five elements, not
three.

## Scalar context and list context

There's a built-in function called `localtime`. When assigning its return value
to a scalar, it returns a nicely-formatted string telling the (local) time:

    $ perl -wle 'my $t = localtime; print $t'
    Thu Sep 30 16:39:40 2010

However, when assigning the return value to a list, we get a bunch of values:

    $ perl -wle 'my @t = localtime; print $_ for @t'
    40
    39
    16
    30
    8
    110
    4
    272
    1

(Those happen to be second, minute, hour, day, (zero-based) month, (1900-based)
year, day-of-week, day-of-year, and DST flag, respectively.)

In other words, `localtime` returns a scalar in a scalar assignment, and a list
in list assignment. A number of Perl built-ins lead a similar double life, and
the things returned can be quite different.

    $ perl -wle '$s = reverse "OH HAI"; print $s'
    IAH HO
    
    $ perl -wle '@a = reverse "OH", "HAI"; print for @s'
    HAI
    OH

The thing that determines what the function returns is called (amount) context.
Scalar assignments induce **scalar context**, and list assignments induce
**list context**. The **void context** is a third kind, and occurs when a value
isn't used for anything.

## A couple of one-liners

In this section, the following file will be used for all examples:

    $ cat example 
    The quick
    
    brown fox
    
    jumped over the lazy dog.

### Print only non-empty lines in a file:

    $ perl -wnle 'print if $_ ne ""' example 
    The quick
    brown fox
    jumped over the lazy dog.

The `ne` operator checks whether two strings are **n**ot **e**qual.

    $ perl -wnle 'print if /./' example 
    The quick
    brown fox
    jumped over the lazy dog.

`.` matches any character in a regex. The only thing that doesn't match any
character is a zero-character string. (The `-l` flag has taken care of the
final newline on each line for us.)

### Number all lines in a file:

    $ perl -wnle '++$a; print "$a. $_"' example 
    1. The quick
    2. 
    3. brown fox
    4. 
    5. jumped over the lazy dog.

**Variable interpolation** works in double-quoted strings, and replaces all
variable names with their string values.

    $ perl -wnle 'print "$.. $_"' example 
    1. The quick
    2. 
    3. brown fox
    4. 
    5. jumped over the lazy dog.

`$.` already counts lines for us, so we don't really need to do it ourselves
with `$a`. (More exactly, `$.` is the number of lines we have read from the
"last filehandle accessed", which is `STDIN` in our case.) Perl knows not to
interpolate the second dot in `$..`, because special-character variables only
ever consist of the sigil and the special-character, nothing more.

### Print the number of lines in a file:

    $ perl -wnle 'END { print $. }' example 
    5

The `END` block, as you'll recall, only executes its payload code after the
script has finished. At this point, `$.` equals the number of lines read
during the run of the whole script.

    $ perl -wnle '}{ print $.' example 
    5

This is the [eskimo
operator](https://www.socialtext.net/perl5/index.cgi?eskimo_operator), a
"secret Perl operator".

### Print the number of non-empty lines in a file:

    $ perl -wnle '++$a if /./; END { print $a }' example 
    3

All things we've seen before.

    $ perl -wnle '++$a if /./; }{ print $a' example 
    3

Eskimo operator again.

### Convert all text to uppercase:

    $ perl -wnle 'print uc $_' example 
    THE QUICK
    
    BROWN FOX
    
    JUMPED OVER THE LAZY DOG.

`$_` contains the current line. `uc` uppercases a string and returns the
result.

    $ perl -wnle 'print uc' example 
    THE QUICK
    
    BROWN FOX
    
    JUMPED OVER THE LAZY DOG.

`uc` defaults to `$_`, so no need to mention it explicitly.

    $ perl -wple '$_ = uc' example 
    THE QUICK
    
    BROWN FOX
    
    JUMPED OVER THE LAZY DOG.

With the `-p` flag, we don't need to `print` explicitly either.

## Exercises

* Write a Perl one-liner that numbers and prints only non-empty lines in a
 file.

* Write a Perl one-liner that numbers and prints all lines, but only prints the
  numbers on the non-empty lines.

* Find and learn about at least one more secret Perl operator.

* Read as much of `perldoc perlsyn` or [the online version of the same
  document](http://perldoc.perl.org/perlsyn.html) that you feel you need to
  grok the use of `if`, `unless`, `while`, `until`, and `for`.
