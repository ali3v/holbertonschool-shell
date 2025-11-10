What Happens When You Type ls *.c and Press Enter?

You probably use commands like:

ls *.c


all the time to list your C source files.

On the surface, it feels simple: you type the command, hit Enter, and filenames appear. But behind that tiny moment, a lot of things happen: your terminal talks to your shell, your shell rewrites your command, the operating system creates a new process, and eventually the ls program prints something back.

This post walks through that journey step by step, in plain English, for someone who’s just starting to learn the shell.

1. From Keyboard to Shell

You start in a terminal emulator (GNOME Terminal, iTerm2, Windows Terminal, etc.). Inside that terminal, a shell process is running – usually bash, zsh, or similar.

When you type:

ls *.c


here’s what happens first:

Each key you press (l, s, space, *, ., c) is sent from your keyboard to the terminal.

The terminal displays those characters on the screen so you can see what you’re typing.

When you press Enter, the terminal sends the whole line to the shell, typically ending with a newline character (\n).

You can think of it as:

Keyboard → Terminal → Shell

The terminal is just the “window” and I/O handler; the shell is the actual program that understands and runs your commands.

2. The Shell Reads the Command Line

The shell receives a line of text:

ls *.c


It does not immediately run a program called ls. First, it parses the line according to shell syntax rules.

Internally, the shell needs to figure out:

Where the command name ends (here: ls),

Where the arguments begin (here: *.c),

Which characters are “special” (like *, ?, $, |, >, <, etc.),

Whether there are variables to expand ($HOME, $PATH, etc.).

For our simple example, there are just two pieces:

Command: ls

One argument: *.c

3. Splitting Into Words

The shell first splits the line into words (often called “tokens”), usually separated by spaces (unless quotes or other syntax rules override that).

For:

ls *.c


the shell ends up with:

Word 1: ls

Word 2: *.c

If you typed:

ls main.c test.c


then the tokens would be:

ls

main.c

test.c

At this stage, *.c is still literally the characters *, ., and c. The shell hasn’t expanded it yet.

4. Wildcard Expansion: Turning *.c Into Real Filenames

The * character is a wildcard. In the shell, using patterns like *.c is called globbing (or pathname expansion).

The important point:

ls itself does not understand *.
The shell expands *.c into a list of filenames before ls runs.

Here’s what the shell does with *.c:

It looks at the current directory.

It finds all files whose names end with .c.

It replaces the single word *.c with a list of matching filenames.

Imagine your directory contains:

hello.c
main.c
util.c
notes.txt
README.md


The pattern *.c matches:

hello.c

main.c

util.c

So your original command:

ls *.c


is transformed by the shell into:

ls hello.c main.c util.c


That transformed form is what ls will actually see as its arguments.

What if nothing matches?

If there are no .c files in the current directory, different shells behave slightly differently. Common behavior in bash (with default settings) is:

*.c stays as *.c

So ls is called like:

ls '*.c'


ls then tries to find a literal file named *.c and prints an error if it doesn’t exist.

5. Building the Argument List (argv)

The shell now has a final list of words to pass to the program:

ls hello.c main.c util.c


Conceptually, it prepares something like this:

Program: ls

Arguments:

Index	Value
0	ls
1	hello.c
2	main.c
3	util.c

If you’ve seen a C main function like:

int main(int argc, char *argv[])


this is what argc and argv represent:

argc would be 4,

argv[0] = "ls",

argv[1] = "hello.c", etc.

The shell sets that up and then goes on to find the program to run.

6. Finding the ls Program With $PATH

You didn’t type /bin/ls. You just typed ls. So how does the shell know where the ls program lives?

It uses an environment variable called PATH, which is a colon-separated list of directories, for example:

/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin


When you run ls, the shell searches for an executable file named ls in each directory in PATH, in order:

Check /usr/local/sbin/ls

If not there, check /usr/local/bin/ls

Then /usr/sbin/ls

Then /usr/bin/ls

Then /sbin/ls

Then /bin/ls

On most systems, it will find ls in /bin/ls or /usr/bin/ls.

If it can’t find ls in any of those directories, you get:

bash: ls: command not found


In our case, it finds it successfully, so the shell now knows the full path to the program it should start, e.g. /bin/ls.

7. Creating a New Process: fork() and exec()

Now the shell needs to run /bin/ls with the arguments hello.c main.c util.c.

On Unix-like systems, this usually happens in two main steps:

fork() – The shell asks the operating system to create a new process that is a copy of the shell.

The original process is the parent (the shell).

The new process is the child, which initially has the same code and memory as the shell.

exec() – In the child process, the shell makes a system call (execve or similar) to replace itself with the ls program.

After a successful exec, the child is no longer running the shell’s code.

It is now running the ls program’s code, with the argv and environment variables provided by the shell.

Meanwhile, the parent shell usually waits for the child process to finish before giving you a new prompt. That’s why you don’t see a new prompt until ls is done printing.

8. Standard Input, Output, and Error

Every program, including ls, starts with three standard “streams” already set up:

stdin (standard input) – file descriptor 0

stdout (standard output) – file descriptor 1

stderr (standard error) – file descriptor 2

In a normal interactive shell:

stdin is connected to your keyboard,

stdout is connected to the terminal window,

stderr is also connected to the terminal window.

When ls writes its output (the list of filenames), it writes to stdout. The terminal displays whatever comes out of stdout as text.

If ls needs to print an error, it uses stderr instead.

9. What ls Actually Does

Now we are inside the ls program.

It has:

argv[0] = "ls"

argv[1] = "hello.c"

argv[2] = "main.c"

argv[3] = "util.c"

Roughly speaking, here’s what ls does next:

Process options (flags)
If you had written ls -l *.c, ls would parse -l as an option meaning “long listing format.” In our case, there are no flags, so it sticks with default behavior.

Handle each path argument
For each argument (e.g. hello.c), ls asks the kernel for information about the file:

Does it exist?

Is it a regular file, directory, symlink, etc.?

What are its permissions, owner, size, and modification time?

Sort and format the results
ls usually sorts filenames (alphabetically by default), then decides how to format them: in multiple columns, one per line, long format with permissions, etc., depending on the options and terminal width.

Write the result to stdout
Finally, ls writes text to standard output. For example, it might print:

hello.c  main.c  util.c


The kernel passes that text back to your terminal emulator, and the terminal draws those characters on your screen.

10. The Child Exits, the Shell Returns

Once ls has finished its work:

It calls exit() with an exit status (0 means “success”, non-zero means some kind of error).

The kernel marks the process as finished and notifies the parent process (the shell).

The shell wakes up, collects the exit status, and then prints a new prompt, like:

dorian@machine:~/project$


At that point, the shell is ready for the next command you type.

All of this happens so fast that it feels instantaneous.

11. A Quick End-to-End Summary

Let’s put the whole story together in one short list:

You type ls *.c and press Enter.

The terminal sends the line to the shell.

The shell splits it into words: ls and *.c.

The shell expands the wildcard *.c into real filenames, e.g. hello.c main.c util.c.

The shell searches PATH to find the full path to ls (e.g. /bin/ls).

The shell calls fork() to create a new process.

In the child process, the shell calls exec() to replace itself with the ls program.

ls receives the filenames as arguments, gathers information about each, and prints them to stdout.

The terminal displays the output.

ls exits; the shell gets control again and prints a new prompt.

That’s what really happens in that tiny moment between pressing Enter and seeing your .c files appear.
