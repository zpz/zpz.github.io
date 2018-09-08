---
layout: post
title: "Scoping and Visibility of Environment Variables"
---


I was considering using environment variables to do some simple
configuration management. To this end I experimented with ways to set environment variables
and access them in a (Python) program.
In the following I list some basic observations.
(Some terminologies may be inaccurate.)
I use Bash shell.

## Subshell within a shell script

Within a shell script, a block of statements enclosed in a pair of parentheses
run in a **subshell**. Variables defined in the parent shell are visible in the subshell,
but not the other way around.

Bash shell script
```sh
A=38

(
echo in subshell
echo A: $A
A=40
B=50
echo A: $A
echo B: $B
)

echo out of subshell
echo A: $A
echo B: $B

export A

echo export A in parent shell

(
echo in subshell
echo A: $A
A=40
B=50
echo A: $A
echo B: $B
export B
echo export B in subshell
)


echo out of subshell
echo A: $A
echo B: $B
```

Output:
```sh
in subshell
A: 38
A: 40
B: 50
out of subshell
A: 38
B:
export A in parent shell
in subshell
A: 38
A: 40
B: 50
export B in subshell
out of subshell
A: 38
B:
```

Comments:

1. Subshell can read variables defined in the parent shell.
   This does not require the variables to have been `export`ed.
2. Variables defined in a subshell are not visible in the parent shell
   after exiting from the subshell.
3. If subshell modifies a parent-shell variable, it essentially creates a new variable.
   The new value is not visible in the parent shell.
   The old value persists in the parent shell.
4. Doing `export` will not make a subshell variable visible in the parent shell.
   Basically, it seems there is no way to put a variable in the parent environment
   from within a child environment.


## `bash script.sh` vs `source script.sh`

`vars.sh` defines variables:
```sh
# vars.sh

A=a
export B=b
```

`caller.sh` 'calls' this script:
```sh
# caller.sh

bash ./vars.sh
echo A: $A
echo B: $B
```

Output:
```sh
A:
B:
```

Use `source` instead of `bash` to execute a script:
```sh
# caller2.sh

source ./vars.sh
echo A: $A
echo B: $B
```

Output:
```sh
A: a
B: b
```

Comments:

1. `bash ./vars.sh` runs a separate program (here, another shell script) in a subprocess.
   Variables defined in this other program are not visible in the calling (i.e. parent)
   environment. Using `export` in the subprocess does not help.
2. `source ./vars.sh` (or equivalently, `. ./vars.sh`) does *not* create a subprocess or subshell.
   It is as if writing the content of the 'sourced' script in-place.
   Variables defined in the sourced script are visible; `export` is not needed.

By the way, putting a 'shebang' in 'vars.sh' and making the script an executable program
is equivalent to keeping it a plain text file and `bash`-executing it.
I prefer to use `bash` for the explicitness.


## Accessing variables from a called program

Now we know how to define a bunch of variables in a shell script:
either define them in-place, or `source` them in from another shell script.
Next we want to access these variables from a program that is called in this cript.
For now, the other 'program' is a shell script.

'Driver' script:
```sh
# sh1.sh

A=a
B=b

bash ./sh2.sh
```

'Callee' program:
```sh
# sh2.sh

echo A: $A
echo B: $B
```

Output of `bash ./sh1.sh`:
```sh
A:
B:
```

(If we replace `bash ./sh2.sh` in `sh1.sh` by `source ./sh2.sh`,
the output will be different. But that is not general usage---that is an option
only when the callee is also a shell script. We'll downplay this usage.)



Revised 'driver' script which `export`s the variables:
```sh
# sh3.sh

export A=a
export B=b

bash ./sh2.sh
```

Output of `bash ./sh3.sh`:
```sh
A: a
B: b
```

Comments:

1. Variables defined in the calling script are not visible in the callee program
   (which runs in a subprocess)
2. **unless** they are `export`ed.


By the way,
```sh
export A=a
export B=b
```
is equivalent to
```sh
A=a
B=b
export A B
```
and
```sh
export A B
A=a
B=b
```

## Calling a Python program from a shell script

This situation is not different from calling another shell script using `bash`.

Driver script:
```sh
# sh1.sh
A=a
export B=b
python sh.py
```

Callee program:
```sh
# sh.py
import os

print('A: ' + os.environ.get('A', ''))
print('B: ' + os.environ.get('B', ''))
```

Output of `bash ./sh1.sh`:
```sh
A:
B: b
```

## Simple configuration management based on environment variables

Based on these observations, simple configurations can be managed this way:

1. Define variables in a shell script, and store this config script outside of source control.
   Make sure the variables that need to be accessed in the main program must be `export`ed.
2. In a launch script (which should be source controlled), `source` the config script,
   and then launch the main program.
3. In the main program, query environment variables for configurations.

