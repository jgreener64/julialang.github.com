---
layout: post
title:  Shelling Out Sucks
author: <a href="http://karpinski.org/">Stefan Karpinski</a>
---

Spawning a pipeline of connected programs via an intermediate shell — a.k.a. "shelling out" — is both idiomatic and highly convenient for many tasks in "glue languages" like Perl, Python, and Ruby.
However, shelling out is also a common source of bugs, security holes, unnecessary overhead, and silent failures.
Here are the three reasons why shelling out is problematic:

1. *[Metacharacter brittleness.](#Metacharacter+Brittleness)*
When commands are constructed programmatically, the resulting code is almost always brittle:
if a variable used to construct the command contains any shell metacharacters, including spaces, the command will likely break and do something very different than what was intended — potentially something quite dangerous.
3. *[Indirection and inefficiency.](#Indirection+and+Inefficiency)*
When shelling out, the main program forks and execs a shell process just so that shell can, in turn, fork and exec a series of commands with their inputs and outputs appropriately connected.
Not only is starting a shell an unnecessary step, but since the main program is not the parent of the pipeline commands, it cannot be notified when they terminate — it can only wait for the pipeline to finish and hope the shell indicates what happened.
2. *[Lack of error handling.](#Lack+of+Error+Handling)*
Errors in shelled out commands don't automatically become exceptions in most languages.
This typically leads to code that fails silently when shelled out commands don't work.
Worse still, because of the indirection problem, there are many cases where the failure of a process in a spawned pipeline cannot be detected by the parent process, even when errors are fastidiously checked for.

In the rest of this post, we'll go over examples demonstrating each of these problems.
At [the end](#Summary+and+Remedy), I'll present how I think programming languages should deal with these issues better.
In a [followup post], I'll talk about how this improved approach is implemented in Julia and all the shell-free pipeline goodness that ensues.
Examples are given in [Ruby] which shells out to [Bash], but I could just as easily pick on [Python], [Perl], other [UNIX shells](http://en.wikipedia.org/wiki/Unix_shell), or just about any language you can think of.

[Bash]:     http://www.gnu.org/software/bash/
[Perl]:     http://www.perl.org/
[Python]:   http://python.org/
[Ruby]:     http://www.ruby-lang.org/
[OS X]:     http://en.wikipedia.org/wiki/Mac_OS_X

## Metacharacter Brittleness

Let's start with a simple example of shelling out from Ruby.
Suppose you want to count the number of lines containing the string "foo" in all the files under a directory given as an argument.
One option is to write Ruby code that reads the contents of the given directory, finds all the files, opens them and iterates through them looking for the string "foo".
However, that's a lot of work and it's going to be much slower than using a pipeline of standard UNIX commands, which are written in C and heavily optimized.
The most natural and convenient thing to do is to shell out.
Here's how you would do it in Ruby:

    `find #{dir} -type f -print0 | xargs -0 grep foo | wc -l`.to_i

This expression interpolates the `dir` variable into a command, spawns a Bash shell to execute the resulting command, captures the output into a string, and then converts that string to an integer.
The command uses the `-print0` and `-0` options to correctly handle strange characters in file names piped from `find` to `xargs` (these cause file names to be delimited by NUL bytes instead of whitespace).
Even with extra-careful options, this shelling out code is simple and clear.
Here it is in action:

    irb(main):001:0> dir="src"
    => "src"
    irb(main):002:0> `find #{dir} -type f -print0 | xargs -0 grep foo | wc -l`.to_i
    => 5    

Great.
However, this only works as expected if the directory name `dir` doesn't contain any characters that the shell considers special.
For example, the shell decides what constitutes a single argument to a command using whitespace.
Thus, if the value of `dir` is a directory name containing a space, this will fail:

    irb(main):003:0> dir="source code"
    => "source code"
    irb(main):004:0> `find #{dir} -type f -print0 | xargs -0 grep foo | wc -l`.to_i
    find: `source': No such file or directory
    find: `code': No such file or directory
    => 0

The simple solution to the problem of spaces is to surround the interpolated directory name in quotes, telling the shell to treat spaces inside as normal characters:

    irb(main):003:0> `find '#{dir}' -type f -print0 | xargs -0 grep foo | wc -l`.to_i
    => 5

Excellent.
So what's the problem?
While this solution addresses the issue of file names with spaces in them, it is still brittle with respect to other shell metacharacters.
What if a file name has a quote character in it?
Let's try it.
First, let's create a very weirdly named directory:

    bash-3.2$ mkdir "foo'bar"
    bash-3.2$ echo foo > "foo'bar"/test.txt
    bash-3.2$ ls -ld foo*bar
    drwxr-xr-x 3 stefan staff 102 Feb  3 16:17 foo'bar/

That is an admittedly strange name, but it's a perfectly legal as a UNIX directory name.
Now back to Ruby:

    irb(main):003:0> dir="foo'bar"
    => "foo'bar"
    irb(main):004:0> `find '#{dir}' -type f -print0  | xargs -0 grep foo | wc -l`.to_i
    sh: -c: line 0: unexpected EOF while looking for matching `''
    sh: -c: line 1: syntax error: unexpected end of file
    => 0

Doh.
Although this may seem like an unlikely corner case that one needn't realistically worry about, there are serious security ramifications.
Suppose the name of the directory came from an untrusted source — like a web submission, or an argument to a setuid program from an untrusted user?
Suppose an attacker could arrange for any value of `dir` they wanted:

    irb(main):005:0> dir="foo'; echo MALICIOUS ATTACK 1>&2; echo '"
    => "foo'; echo MALICIOUS ATTACK 1>&2; echo '"
    irb(main):006:0> `find '#{dir}' -type f -print0  | xargs -0 grep foo | wc -l`.to_i
    find: `foo': No such file or directory
    MALICIOUS ATTACK
    grep:  -type f -print0
    : No such file or directory
    => 0

Your box is now owned.
Of course, you could sanitize the value of the `dir` variable, but there's a fundamental tug-of-war between security (as limited as possible) and flexibility (as unlimited as possible).
The ideal behavior is to allow any directory name, no matter how bizarre, but "defang" all shell metacharacters.

The only two way to fully protect against these sorts of metacharacter attacks — whether malicious or accidental — while still using an external shell to construct the pipeline, is to do full shell metacharacter escaping:

    irb(main):005:0> require 'shellwords'
    => true
    irb(main):006:0> `find #{Shellwords.shellescape(dir)} -type f -print0  | xargs -0 grep foo | wc -l`.to_i
    find: `foo\'; echo MALICIOUS ATTACK 1>&2; echo \'': No such file or directory
    => 0

With shell escaping, this safely attempts to search a very oddly named directory instead of executing the malicious attack.
Although shell escaping does work (assuming that there aren't any mistakes in the shell escaping implementation), realistically, no one actually bothers — it's too much trouble.
Instead programs that shell out with interpolated variables are typically riddled with potential bugs in the best case and massive security holes in the worst case.

## Indirection and Inefficiency

If we were using the above Ruby code to count the number of lines with the string "foo" in a directory, we would want to check to see if everything worked and respond appropriately if something went wrong.
In Ruby, you can check if a shell command run using backticks or the similar `system` command was successful using the bizarrely named `$?.success?` indicator:

    irb(main):008:0> dir="src"                                                              
    => "src"
    irb(main):009:0> `find #{Shellwords.shellescape(dir)} -type f -print0  | xargs -0 grep foo | wc -l`.to_i
    => 5
    irb(main):010:0> $?.success?                                                                
    => true

Ok, that correctly indicates success.
Let's make sure that it can detect failure:

    irb(main):011:0> dir="nonexistent"                                                              
    => "nonexistent"
    irb(main):012:0> `find #{Shellwords.shellescape(dir)} -type f -print0  | xargs -0 grep foo | wc -l`.to_i
    find: `nonexistent': No such file or directory
    => 0
    irb(main):013:0> $?.success?=> true

Wait. What?!
That wasn't successful.
What's going on?

The heart of the problem is that when Ruby (or Python or Perl) shells out, the commands in the pipeline are not immediate children of the main program, but rather its grandchildren:
the program spawns a shell, which makes a bunch of UNIX pipes, connects their inputs and outputs appropriately using the [`dup2` system call](https://developer.apple.com/library/IOs/#documentation/System/Conceptual/ManPages_iPhoneOS/man2/dup2.2.html), and then forks and execs the appropriate commands.
As a result, your main program is not the parent of the commands in the pipeline, but rather, their grandparent.
Therefore, it doesn't know their process IDs, nor can it wait on them or get their exit statuses when they terminate.
The shell process, which is their parent, has to do all of that.
Your program can only wait for the shell to finish and see if *that* was successful.
If the shell is only executing a single command, this is fine:

    irb(main):014:0> `true`
    => ""
    irb(main):015:0> $?.success?
    => true
    irb(main):016:0> `false`
    => ""
    irb(main):017:0> $?.success?
    => false

Unfortunately, the shell is by default quite lenient about what it considers to be a successful pipeline:

    irb(main):018:0> `perl -e "die" | cat`
    Died at -e line 1.
    => ""
    irb(main):019:0> $?.success?
    => true

As long as the last command in a pipeline succeeds, the entire pipeline is considered a success.
Thus, even when one or more of the earlier programs in a pipeline fails spectacularly, the last command may not, leading the shell to consider the entire pipeline to be successful.
This is likely not what you meant by success, however.

Bash's notion of pipeline success can, fortunately, be made stricter with the `pipefail` option:

    irb(main):020:0> `set -o pipefail; perl -e "die" | cat`
    Died at -e line 1.
    => ""
    irb(main):021:0> $?.success?
    => false

Since shelling out from Ruby (or Perl or Python) spawns a new shell every time, this option has to be set for every pipeline in order to find out the true success status when shelling out pipelines of several commands.
Of course, just like shell escaping interpolated variables, setting `pipefail` at the start of every command in backticks is simply something that no one actually does.
As a result, even if you are a very careful programmer and check the return codes every time you shell out, unless you are *also* prefix every pipeline with `set -o pipefail`, you are still be a potential of undetectable pipeline failures.

Given the other problems of indirection, it's seems like a barely relevant afterthought to mention that the indirection of spawning a shell process just to spawn a bunch of other processes is inefficient.
However, it is a real source of unnecessary overhead:
the main process could just do the work the shell does itself.
Asking the kernel to fork a process and exec a new program is a non-trivial amount of work.
The only reason to have the shell do this work instead is that it's complicated and hard to get right.
The shell makes it easy.
So programming languages have traditionally relied on the shell to setup pipelines for them, regardless of the additional overhead and indirection.

## Lack of Error Handling

Let's return to out example of shelling out to count "foo" lines.
Here's the total expression we need to use in order to shell out without being susceptible to metacharacter breakage and so that we can check if the entire pipeline succeeded:

    `set -o pipefail; find #{Shellwords.shellescape(dir)} -type f -print0  | xargs -0 grep foo | wc -l`.to_i

Ruby doesn't automatically raise an error when a shelled out command fails (nor do Python or Perl).
Therefore, to avoid silent errors, we need to check `$?.success?` after every time we shell out and raise an exception if it indicates failure.
Of course, doing this manually is tedious, and as a result, it largely isn't done.
For example, to make our "foo"-counting example really well-behaved, we could wrap it in a function like so:

    def foo_count(dir)
      n = `set -o pipefail; find #{Shellwords.shellescape(dir)} -type f -print0  | xargs -0 grep foo | wc -l`.to_i
      if not $?.success?
        raise("pipeline failed")
      end
      return n
    end

This function behaves the way we would like it to:

    irb(main):022:0> foo_count("src")
    => 5
    irb(main):023:0> foo_count("nonexistent")
    find: `nonexistent': No such file or directory
    RuntimeError: pipeline failed
        from (irb):35:in `foo_count'
        from (irb):40
        from :0
    irb(main):024:0> foo_count("foo'; echo MALICIOUS ATTACK; echo '")
    find: `foo\'; echo MALICIOUS ATTACK; echo \'': No such file or directory
    RuntimeError: pipeline failed
        from (irb):35:in `foo_count'
        from (irb):41
        from :0

However, this 7-line, 200-character function is a far cry from the simplicity and ease of the simple expression we started with:

    `find #{dir} -type f -print0 | xargs -0 grep foo | wc -l`.to_i

## Summary and Remedy

To sum it up, shelling out is great, but making code that shells out bug-free, secure, and not prone to silent failures requires three things that typically aren't done:

1. Shell-escaping all variables interpolated into commands
2. Prefixing each multi-command pipeline with `set -o pipefail`
3. Explicitly checking for failure after each shelled out pipeline.

The trouble is that after doing all of these things, shelling out is no longer terribly convenient, and the code becomes rather verbose.
In short, shelling out responsibly kind of sucks.

As is so often the case, the root of all of these problems stems from relying on a middle man rather than doing things yourself.
If a language constructs and executes pipelines itself, it remains in control of all the subprocesses, and can determine their individual exit conditions, and automatically handle errors appropriately.
Moreover, when there is no shell interpreting commands, there is also no shell to treat metacharacters specially, and therefore no danger of metacharacter brittleness.
The challenge is to make constructing and executing pipelines in a high-level language as easy and expressive as it is in the shell.
In my [followup post], I describe how Julia implements command execution and pipeline construction without a shell, thereby avoiding the traditional pitfalls of shelling out, and increasing the power and flexibility of programmatically constructing and executing external commands.

[followup post]: /drafts/2012/02/put-this-in-your-pipe/