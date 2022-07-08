---
title: "Over the Wire Bandit 1 4"
date: 2022-06-23T16:38:23-04:00
draft: false
tags = []
categories = []
---

## Over The Wire Intro

I'm starting a series on [Over the Wire Wargames](https://overthewire.org "Over The Wire Wargames"). Each wargame has different levels you progress through to beat it. It provides a fun way to learn security concepts from the very basics to the advanced. It gives you roadmap to take you through each wargame so that they build on what you learned from the others. All you have to do is ssh into the room and start figuring out how to get the password for the next level. The beginner one is called Bandit. SSH will get you into a terminal of the room where you can type commands to find the password. They give you two commands to help when you get stuck: man and help. The man command is used by typing ```man command_name``` and will bring up the manual for the command. The help command is used the same way, but is used for a shell built-in command. If one doesn't work try the other. They give you clues as to what commands may be useful or different things to search for. I'm starting my way through Bandit and will be posting how I solved each level.

## [Level 0](https://overthewire.org/wargames/bandit/bandit0.html "Level 0")

This level will teach you how to ssh into the room. The useful command is ssh. On Windows, you will need a program called [Putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html "Putty") which allows to use ssh. When you have Putty open, fill in the host and port. You can then type a name, maybe bandit, under Saved Sessions and click save. Then you just need to double click on bandit to open it next time. On Linux or Mac, open the terminal and type ```ssh bandit0@bandit.labs.overthewire.org -p 2220```. Once we find a password, we need to close that ssh session and reopen another one using the new username and password.

## [Level 0 -> Level 1](https://overthewire.org/wargames/bandit/bandit1.html "Level 0 -> Level 1")

For this level, you need to figure out how to open a file called readme. The useful commands are ls, cd, cat, file, du, and find. First we need to figure out what these commands do by using man. If we ```man ls```, we find out that it will list a directory's contents. That sounds useful. Let's run ```ls``` and see what it returns. It tells us there is a file called readme in this directory. From the Level Goal, we know readme is supposed to be in the home directory so we must start in the home directory when we first ssh into the room. That could be useful information later on. Now we just need to open the file so let's see if any of the other commands are going to help. ```man cd``` says there isn't a manual for it so we need to run ```help cd```. That tells us it's used for changing to another directory. We're in the correct directory so that's not useful. ```man cat``` tells us it concatenates file and prints to the standard output. That sounds confusing. Basically it will print the contents of a file or files which sounds like what we need. If we try ```cat readme```, it returns the password for the next level!

## [Level 1 -> Level 2](https://overthewire.org/wargames/bandit/bandit2.html "Level 1 -> Level 2")

For this level we need to open a file called -. The useful commands are ls, cd, cat, file, du, and find. We know cat will open a file so let's try ```cat -```. Seems like it didn't do anything so let's press ctrl+c to stop it. Reading the manual again, we see that a - reads from the standard input. So whatever we type, it will print out on the screen. How can we read from a file called - then? It's time to get some help from the Internet. After searching, we find there are a couple ways to do it. The first way, ```cat ./-```, the ./ represents the current working directory. The second way, ```cat < -```, the < character tells cat to use the file - as an input. Either way gets the password for the next level.

## [Level 2 -> Level 3](https://overthewire.org/wargames/bandit/bandit3.html "Level 2 -> Level 3")

For this level we need to open a file called spaces in this filename. The useful commands are ls, cd, cat, file, du, and find. Let's try to cat it. It says:
```
    cat: spaces: No such file or directory
    cat: in: No such file or directory
    cat: this: No such file or directory
    cat: filename: No such file or directory
```
It's treating each word as a filename. Reading the man page again doesn't give any useful information. The Helpful Reading Material says to search for spaces in filename so let's try that. There's a couple ways to accomplish this. The first is the use quotes, cat ```"spaces in this filename"```, to tell cat that it's all one filename. The second is to escape the spaces to tell cat that the spaces are part of the filename, ```cat spaces\ in\ this\ filename```. Either way gets the password for the next level.

## [Level 3 -> Level 4](https://overthewire.org/wargames/bandit/bandit4.html "Level 3 -> Level 4")

For this level we need to open a hidden in the inhere directory. The useful commands are ls, cd, cat, file, du, and find. First need to change to the inhere directory. Let's run ```ls``` to see if it's in our home directory. It is so let's change to it using cd which we know changes directories from earlier, ```cd inhere```. Now we can ls again to see if we can find the hidden file. Nothing's there. Let's do some research on hidden files in Linux. We find that hidden files start with a . so let's go the ls manual and see how to list them. The first option, -a, tells ls not to ignore entries starting with . so let's run ```ls -a```. We see the .hidden file now so let's cat that to get the next password.

This seems like a good place to stop for now. The next level starts getting into different commands. It'll make things easier to read and follow along by grouping them by similar commands. I hope you enjoyed the first few levels of Bandit. I'll be posting the new group soon.
