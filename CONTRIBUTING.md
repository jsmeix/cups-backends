# Make yourself understood

Things are written in Bash, a language that can be used in various styles.

The primary intent is to make it easier for everybody to understand the code
and subsequently to contribute fixes and enhancements.

So make yourself understood to enable others to fix and enhance
your code contributions properly as needed.

From this overall idea the following coding hints are derived.

For the fun of it an extreme example what coding style should be avoided:
```
#!/bin/bash
for i in `seq 1 2 $((2*$1-1))`;do echo $((j+=i));done
```
Try to find out what that code is about - it does a useful thing.

## Code must be easy to read

* Variables and functions must have names that explain what they do, even if it makes them longer.
  Avoid too short names, in particular do not use one-letter-names
  (like a variable named `i` - just try to 'grep' for it over the whole code to find code that is related to `i`).
  Use different names for different things so that others can 'grep' over the whole code
  and get a correct overview what actually belongs to a specific kind of thing.
  In general names should consist of a generic part plus one or more specific parts to make them meaningful.
  For example `boot_dev` would be mostly meaningless.
  Better use names like `boot_partition` versus `efi_system_partition`
  versus `bios_boot_partition` versus `bootloader_install_device`
  to make it clear and unambiguous what each thingy actually is about.
  
* Introduce intermediate variables with meaningful names to tell what is going on.<br>
  For example instead of running commands with obfuscated arguments like<br>
  `rm -f $( ls ... | sed ... | grep ... | awk ... )`<br>
  which looks scaring (what the heck gets deleted here?) better use <pre>
  foo_dirs="..."
  foo_files=$( ls $foo_dirs | sed ... | grep ... )
  obsolete_foo_files=$( echo $foo_files | awk ... )
  rm -f $obsolete_foo_files </pre> that tells the intent behind
  (regardless whether or not that code is the best way to do it - but now others can easily improve it).

* Use spaces when possible to aid readability like<br>
  `output=( $( COMMAND1 OPTION1 | COMMAND2 OPTION2 ) )`<br>
  instead of  `output=($(COMMAND1 OPTION1|COMMAND2 OPTION2))`<br>
  In particular avoid bash arithmetic evaluation and expansion<br>
  without spaces as in `result=$(((foo-bar)*baz))`<br>
  but prefer readability over compression when possible<br>
  `result=$(( ( foo - bar ) * baz ))`

## Code must be easy to understand

Do not only tell **what** the code does (i.e. the implementation details)
but also explain what the intent behind is (i.e. **why**) to make the code maintainable,
in particular to keep special adaptions and enhancements up to date in the future
also by others who did not originally make them.

* Provide comprehensive comments that tell what the computer should do
  and also explain why it should do it so that others understand the intent behind
  so that they can properly fix issues or adapt and enhance it as needed at any time later
  (even if all is totally obvious for you, others who do not know about your particular use case
  or do not have your particular environment may understand nothing at all about your code).
  
* If there is a GitHub issue or another URL available for a particular piece of code
  provide a comment with the GitHub issue or any other URL that tells about the reasoning
  behind current implementation details.

Here the initial example so that one can understand what it is about:
<pre>
#!/bin/bash
# output the first N square numbers
# by summing up the first N odd numbers 1 3 ... 2*N-1
# where each nth partial sum is the nth square number
# see https://en.wikipedia.org/wiki/Square_number#Properties
# this way it is a little bit faster for big N compared to
# calculating each square number on its own via multiplication
N=$1
if ! [[ $N =~ ^[0-9]+$ ]] ; then
    echo "Input must be non-negative integer." 1>&2
    exit 1
fi
square_number=0
for odd_number in $( seq 1 2 $(( 2 * N - 1 )) ) ; do
    (( square_number += odd_number )) && echo $square_number
done
</pre>
Now the intent behind is clear and now others can easily decide
if that code is really the best way to do it and easily improve it if needed.

## Try hard to care about possible errors

By default bash proceeds with the next command when something failed.
Do not let your code blindly proceed in case of errors because that could
make it hard for others to find out that the root cause of a failure
is in your code when it errors out somewhere later at an unrelated place
with a weird error message which could lead to false fixes that
cure only a particular symptom but not the root cause.

* In case of errors better abort than to blindly proceed.

* At least test mandatory conditions before proceeding.
  If a mandatory condition is not fulfilled abort with a meaningful error message.

Preferably during development of new scripts or when scripts are much overhauled
and while testing new code use `set -ue` to die from unset variables and unhandled errors
and use `set -o pipefail` to better notice failures in a pipeline.

Using `set -eu -o pipefail` also during runtime is not recommended
because it is a double-edged sword which can cause more problems in practice
than it intends to solve in theory.
I.e. problems for users when things fail for them only because of `set -eu -o pipefail`
while actually the code would work fail-safe without `set -eu -o pipefail` like
<pre>
for file in "${FILES[@]}" ; do
    grep SOMETHING "$file" >>something.found
done
</pre>
which lets `grep` intentionally fail for empty or blank elements in the FILES array
to skip such elements (with the `grep` error message as information)
and proceed with the next element in the FILES array
cf. the section "Beware of the emptiness" below.

## Maintain backward compatibility

Implement adaptions and enhancements in a backward compatible way
so that your changes do not cause regressions for others.

* One same code must work on various different systems.
  On older systems as well as on newest systems and on various different Linux distributions.

* Preferably use simple generic functionality that works on any Linux system.
  Better very simple code than oversophisticated (possibly fragile) constructs.
  In particular avoid special bash version 4 features.
  The code should also work with bash version 3.

## Character encoding

Use only traditional (7-bit) ASCII charactes.
In particular do not use UTF-8 encoded multi-byte characters.

* Non-ASCII characters in scripts may cause arbitrary unexpected failures
  on systems that do not support other locales than POSIX/C.
  In the POSIX/C locale non-ASCII characters are invalid in scripts.
  
* English documentation texts do not need non-ASCII characters.
  Using non-ASCII characters in documentation texts makes it needlessly hard
  to display the documentation correctly for any user on any system.
  When non-ASCII characters are used but the user does not have the exact right
  matching locale set on his system arbitrary nonsense can happen, cf.
  https://en.opensuse.org/SDB:Plain_Text_versus_Locale

* The plain UTF-8 character encoding is compatible with ASCII but
  setting LANG to en_US.UTF-8 is not ASCII compatible, see this mail
  https://lists.opensuse.org/opensuse-packaging/2017-11/msg00006.html
  that reads (excerpt): <pre>
  Setting LANG to en_US.UTF-8 is a horrible idea for scripts
  ...
  collating order and ctypes get in the way
  as it's not ASCII compatible </pre>

## Text layout

* Indentation with blanks, no tabs.

* No backticks for command substitution, only `$( COMMAND )`

* Curly braces only if really needed: `$VAR` instead of `${VAR}`

### test, [, [[, ((

* Use `[[` where it is required (e.g. for pattern matching or complex conditionals)
  and `[` or `test` everywhere else.

* `((` is the preferred way for numeric comparison,
  variables don't need to be prefixed with `$` there.

### Paired parentheses

Use paired parentheses for `case` patterns so that editor commands (like '%' in 'vi')
that check for matching opening and closing parentheses work everywhere in the code.

Example:
<pre>
case WORD in
    (PATTERN1)
        COMMAND1
        ;;
    (PATTERN2)
        COMMAND2
        ;;
    (*)
        COMMAND3
        ;;
esac
</pre>

## Beware of the emptiness

Be prepared for possibly empty or empty looking (i.e. blank) values.

For example code like
<pre>
for file in "${FILES[@]}" ; do
    if grep -q SOMETHING $file ; then
        ...
    fi
done
</pre>
hangs up in `grep SOMETHING` which searches for SOMETHING in the standard input (i.e. on stdin)
when the FILES array contains an empty or blank element
so in this case proper quoting results fail-safe code
<pre>
for file in "${FILES[@]}" ; do
    if grep -q SOMETHING "$file" ; then
        ...
    fi
done
</pre>
while in other cases one must explicitly test the value like
<pre>
for user in "${USERS[@]}" ; do
    # Ensure $user is a single non empty and non blank word
    # (no quoting because test " " returns zero exit code):
    test $user || continue
    echo $user >>valid_users
done
</pre>
when empty or blank lines are wrong in the valid_users file.

