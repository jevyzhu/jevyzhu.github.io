---
layout: post
title: "Bash History Hint"
---

How To Use Bash History Commands and Expansions

Introduction
While working in a server environment, you’ll spend a lot of your time on the command line. Most likely, you’ll be using the bash shell, which is the default of most distributions.

During a terminal session, you’ll likely be repeating common commands often, and typing variations on those commands even more frequently. While typing each command again can be good practice in the beginning, at some point, it crosses the line into being disruptive and an annoyance.

Luckily, the bash shell has some fairly well-developed history functions. Learning how to affectively use and manipulate your bash history will allow you to spend less time typing and more time getting actual work done. Many developers are familiar with the DRY philosophy of Don’t Repeat Yourself. Effective use of bash’s history allows you to operate closer to this principle and will speed up your work flow.

In this guide, we will be checking out these features on an Ubuntu 12.04 VPS instance, but almost all modern Linux distributions should operate in a similar way. Let’s get started!

Setting History Defaults
Before we begin actually using the history, let’s adjust some bash settings to make it more useful.

Bash allows you to adjust the number of previous commands that it stores in history. It actually has two separate options for this: the HISTFILESIZE parameter configures how many commands are kept in the history file, while the HISTSIZE controls the number stored in memory for the current session.

This means you can set a reasonable cap for the size of history in memory for the current session, and have an even larger history saved to disk that you can examine at a later time. By default, bash sets very conservative values for these options, so we’ll expand them to take advantage of a larger history. Some distributions already increase the default bash history settings with slightly more generous values.

Open your ~/.bashrc file with an editor to change these settings:

nano ~/.bashrc
Search for both the HISTSIZE and HISTFILESIZE parameters. If they are set, feel free to modify the values. If these parameters aren’t in your file, add them now. For our purposes, we can easily get away with saving 10000 lines to disk and loading the last 5000 lines into memory. This is a conservative estimate for most systems, but adjust it down if you see a performance impact:

HISTSIZE=5000
HISTFILESIZE=10000
By default, bash writes its history at the end of each session, overwriting the existing file with an updated version. This means that if you are logged in with multiple bash sessions, only the last one to exit will have its history saved.

We can work around this by setting the histappend setting, which will append instead of overwrite the history. This may be set already, but if it is not, you can enable this by adding this line:

shopt -s histappend
If we want to have bash immediately add commands to our history instead of waiting for the end of each session (to enable commands in one terminal to be instantly be available in another), we can also set or append the history -a command to the PROMPT_COMMAND parameter, which contains commands that are executed before each new command prompt.

To do this correctly, we need to do a bit of a hack. We need to append to the history file immediately with history -a, clear the current history in our session with history -c, and then read the history file that we’ve appended to, back into our session history with history -r.

You can do this like so:

export PROMPT_COMMAND="history -a; history -c; history -r; $PROMPT_COMMAND"
When you are finished, save the file and exit.

To implement your changes, either log out and back in again, or source the file by typing:

source ~/.bashrc
Reviewing your Previous Bash History
The way that we review bash history is to use the history command. This will print out our recent commands, one command per line. This should output, at most, the number of lines you selected for the HISTSIZE variable. It will probably be fewer at this point:

history
   . . .
   43  man bash
   44  man fc
   45  man bash
   46  fc -l -10
   47  history
   48  ls -a
   49  vim .bash_history 
   50  history
   51  man history
   52  history 10
   53  history
It also prints the history number for each command. Each command is associated with a number for easy reference. You will see why this is useful in a moment.

We can truncate the output by specifying a number after the command. For instance, if we want to only see the last 5 commands typed, we can type:

history 5
   50  history
   51  man history
   52  history 10
   53  history
   54  history 5
To find all of the history commands that involve a certain string, an easy way of getting an overview is to simply pipe it to grep. For example, we can search for the lines that have cd by typing:

history | grep cd
   33  cd Pictures/
   37  cd ..
   39  cd Desktop/
   61  cd /usr/bin/
   68  cd
   83  cd /etc/
   86  cd resolvconf/
   90  cd resolv.conf.d/
Executing Commands from your Bash History
Printing off our history is nice, but, by itself, it doesn’t really help us access those commands easily, except as a reference. However, we can quickly recall any of the returned output using a special syntax.

We can recall any of our previous history by its number preceded by an exclamation point (!). For instance, if your history looks like mine above, you could see the man page for the history command quickly by typing:

!51
This will immediately recall and execute the command associated with the history number 51.

We can also execute commands relative to our current position. We can do this by using the !-n syntax, where “n” is replaced by the number of commands ago we want to recall.

For instance, if we want to recall and execute a command that we typed before our most recent one, we could type !-2. So if we listed the contents of a long directory path, echoed something and wanted to list again, our session might look like this:

ls /usr/share/doc/manpages
echo hello
!-2             # lists the contents again
To re-execute the previous command, bash provides a shortcut that we can use instead of !-1. The shortcut is !!, which will substitute the most recent command and execute:

!!
Many people use this when they type a command that they realize they needed sudo privileges to execute. Typing sudo !! will re-execute the command with sudo in front of it. The session might look like this:

touch /etc/hello
touch: cannot touch `/etc/hello': Permission denied
sudo !!
sudo touch /etc/hello
[sudo] password for demouser:
This demonstrates another property of this syntax. They are pure substitutions, and can be incorporated within other commands at will.

Scrolling through Bash History
There are a few ways that we can scroll through our bash history, putting each successive command on the command line to edit.

The most common way of doing this is to press the up arrow key at the command prompt.

Each additional press of the up arrow key will take you further back in your command line history. If you need to go the other direction, the down arrow key traverses the history in the opposite direction, finally bringing you back to your current prompt.

If moving your hand all the way over to the arrow keys seems like a big hassle, you can move backwards in your command history using the CTRL-p combination, and use the CTRL-n combination to move forward in history again.

If you want to jump back to the current command prompt, you can do so by typing Meta->. In most cases, the “meta” key and the “>” will mean typing ALT-Shift-.. This is useful if you find yourself far back in your history and want to get back to your current command.

You can go to the first line of your command history by doing the opposite maneuver and typing Meta-<. This typically means pressing ALT-Shift-,.

To summarize, these are some keys to scroll through the history and jump to either end:

UP arrow key: Scroll backwards in history

CTRL-p: Scroll backwards in history

DOWN arrow key: Scroll forwards in history

CTRL-n: Scroll forwards in history

ALT-Shift-.: Jump to the end of the history (most recent)

ALT-Shift-,: Jump to the beginning of the history (most distant)

Searching through Bash History
Although piping the history command through grep is definitely the easiest way of accomplishing some procedures, it isn’t ideal in many situations.

Bash includes search functionality for its history. The typical way of utilizing this is through searching backwards in history (most recent results returned first) using the CTRL-r key combination.

For instance, you can type CTRL-r, and begin typing part of the previous command. You only have to type out part of the command. If it matches an unwanted command instead, you can press CTRL-r again to see the next result.

If you accidentally pass the command you wanted, you can move in the opposite direction by typing CTRL-s. This also can be useful if you’ve moved to a different point in your history using the keys in the last section and wish to search forward.

Note: In many terminals, the CTRL-s is actually mapped to suspend the terminal session. This will intercept any attempts to pass CTRL-s to bash, and will “freeze” your terminal. To unfreeze, simply type CTRL-q to unsuspend the session.

This suspend and resume feature is not needed in most modern terminals, and we can turn it off without any problem by typing:

stty -ixon
We should add this to our ~/.bashrc file to make this change permanent as well.

If you try again, it should work as expected to allow you to search forwards.

Searching after You’ve Typed Part of the Command
A common scenario to find yourself in is to type in part of your command, and then realize that you have executed it previously and can search the history for it.

The correct way of searching using what is already on your command line is to move your cursor to the beginning of the line with CTRL-a, call the reverse history with CTRL-r, paste the current line into the search with CTRL-y, and then using the CTRL-r again to search in reverse.

For instance, suppose we want to update our package cache on an Ubuntu system. We’ve already typed this out recently, but we didn’t think about that until after we’ve typed the sudo in again:

sudo
At this point, we realize that this is an operation we’ve definitely done in the past day or so. We can hit:

CTRL-a
This moves our cursor to the beginning of the line.

CTRL-r
We call our reverse incremental history search. This has a side effect of copying all of the content on the command line that was after our cursor position. It puts this into a clipboard.

CTRL-y
We paste the command segments that we’d just copied from the command line into the search.

CTRL-r
We move backwards in our history, searching for commands containing the content we’ve just pasted.

This might seem like a huge pain in the neck, but it’s actually not too bad when you get used to it. It is extremely helpful when you find yourself in that awkward position where you’ve typed out half of a complex command and know you’re going to need the history to finish the rest.

To make it easier, you can think of this as a simpler, compound command:

CTRL-aryr
Getting Familiar with More Advanced History Expansion
We’ve already touched on some of the most basic history expansion techniques that bash provides. Some of the ones we’ve covered so far are:

!!: Expand to the last command
!n: Expand to command with history number “n”.
!-n: Expand to command that was “n” number of commands before the current command in history.
Event Designators
The above three examples are instances of event designators. These generally are ways of recalling previous history commands using certain criteria. They are the selection portion of our available operations.

We can execute the last “ssh” command by typing something like:

!ssh
This searches for lines beginning with “ssh”. If we want to search for a string that doesn’t happen at the beginning of the command, we can surround it with “?” characters. For instance, to repeat our least apt-cache search command, we could probably get away with typing:

!?search?
Another trick that you can try is a variation on the !! last history command. You can do a quick search and replace by typing:

<pre>
^<span class=“highlight”>original</span>^<span class=“highlight”>replacement</span>^
</pre>

This will recall the previous command (just like “!!”), search for an instance of “original” within the command string, and replace it with “replacement”. It will then execute the command.

This is useful for dealing with things like misspellings. For instance:

cat /etc/hosst
cat: /etc/hosst: No such file or directory
^hosst^hosts^
Word Designators
After event designators, we can add a colon (:) and add on a word desginator to select a portion of the matched command.

It does this by dividing the command into “words”, which are defined as any chunk separated by whitespace. This allows us some interesting opportunities to interact with our command parameters.

The word numbering starts at the initial command as “0”, the first argument as “1”, and continues on from there.

For instance, we could list the contents of a directory, and then decide we want to change to it, like this:

ls /usr/share/doc/manpages
cd !!:1
In cases where we are operating on the last command, we can actually compress this by removing the second “!"and the colon, like this:

cd !1
This will operate in the same way.

We can refer to the first argument as ”^“ and the final argument as ”$“ if that makes sense for our purposes. These are more helpful when we use ranges instead of specific numbers. For instance, we have three ways we can get all of the arguments from a previous command into a new command:

!!:1*
!!:1-$
!!:*
The lone ”“ expands to all portions of the command being recalled other than the initial command. Similarly, we can use a number followed by ”“ to mean that everything after the specified word should be included.

Modifiers
The last thing that we can do to augment the behavior of the history line we are recalling is to modify the behavior of the recall to manipulate the text itself. Modifiers are added after an additional colon (:) character at the end of the expansion.

For instance, we can chop off the path leading up to a file by using the "h” modifier (it stands for “head”), which removes the path up until the final slash (/) character. Be aware that this won’t work the way you want it to if you are using this to truncate a directory path and the path ends with a trailing slash.

A common use-case for this is if we are modifying a file and realize we’d like to change to the file’s directory to do operations on related files.

For instance, we could read the copyright information of a package:

cat /usr/share/doc/manpages/copyright
After being satisfied that we can use the package for our needs, we may want to change to the directory. We can do this by calling the cd command on the argument chain and chopping off the filename at the end:

cd !!:$:h
pwd
/usr/share/doc/manpages
After we’re there, we may want to open that copyright file again to double check, this time in a pager.

We can do the reverse manipulation, chopping off the path and using only the filename with the “t” modifier for “tail”. We can search for our last cat operation, and use the “t” flag to pass only the file name:

less !cat:$:t
You could just as easily keep the full absolute path name and this command would work correctly in this instance. However, there are other times where this isn’t true. We could be looking at a file nested within a few sub directories below our current directory using a relative path, change to the subdirectory using the “h” modifier, and then we wouldn’t be able to rely on the relative path name to reach the file any more.

Another extremely helpful modifier is the “r” modifier, which strips the trailing extension. This could be useful if you are using tar to decompress a file and want to change into the directory afterwards. Assuming the directory produced is the same name as the file, you could do something like:

tar xzvf long-project-name.tgz
cd !!:$:r
If your tarball uses the tar.gz extension instead of tgz, you can just pass the modifier twice:

tar xzvf long-project-name.tar.gz
cd !!:$:r:r
A similar modifier, “e”, removes everything besides the trailing extension.

If you do not want to execute the command that you are recalling, and simply want to find it, you can use the “p” modifier to have bash echo the command instead of executing it.

This is useful if you are unsure of if you’re selecting the correct piece. This not only prints it, but also puts it into your history for further editing if you’re unhappy with it.

For instance, imagine you ran a find command on your home directory and then realize that you want to run it from the root (/) directory. You could check that you’re making the correct substitutions like this (assuming the original command is #119):

find ~ -name "file1"    # original command
!119:0:p / !119:2*:p
find / -name "file1"
If the outputted command we cobbled together is correct, we can execute it easily with:

CTRL-p
This would probably be easier if we could just make substitutions in our command easily. We can do that using the s/original/new/ syntax.

For instance, we could have accomplished that by typing:

!119:s/~/\//
This will substitute the first instance of the search pattern. We can substitute every match by also passing the “g” flag with the “s”. For instance if we want to create files named file1, file2, and file3, and then want to create directories called dir1, dir2, dir3, we could do this:

touch file1 file2 file3
mkdir !!:*:gs/file/dir/
Conclusion
You should now have a good idea of how you can leverage the history operations available to you. Some of these will probably be more useful than others, but it is good to know that bash has these capabilities in case you find yourself in a position where it would be helpful to dig them up.

If nothing else, the history command alone, the reverse search, and the simple history expansions should help you speed up your work flow.
