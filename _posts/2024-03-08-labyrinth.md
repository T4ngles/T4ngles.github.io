---
layout: post
title: labyrinth
tags: symbolic hard link file system directory honeypot 
categories: ideas
---

_Creating a recursive file directory trap_

I am somewhat of a wild dreamer in that I daydream about random things that could work or may come about in the future or just weird things like how the `Prior Incantato` spell was just pressing up on a bash terminal and a weaker version of `history`. _Side note: I've recently been using `history | grep what I did that one time` which probably means I should be clearing my history_

Anyways another thing I dreamed of was whether I could create some sort of self pointing recursive file/folder structure that would cause any bad actors to be trapped inside of. The obvious idea was just to make shortcuts in folders that reference their parent folder which would allow you to keep clicking in a window forever but that just makes it look like it's not doing anything. Furthermore shortcuts show up as `.lnk` files which are a dead giveaway, no one would get caught in that. 

## Symbolic Links

Then I learnt about symbolic links, hard links and junctions. This [page](https://www.2brightsparks.com/resources/articles/ntfs-hard-links-junctions-and-symbolic-links.html) on 2BrightSparks explains it well as does this short [writeup](https://lettieri.iet.unipi.it/hacking/symlinks.pdf) by Lettieri.

Making a hard link on windows: `mklink /H dirA dirB`
Making a junction on windows: `mklink /J dirA dirB`
Making a symbolic link on windows: `mklink /D dirA dirB`

With learning about these I was able to explore my idea a bit more and I actually found that there were more than a few exploits using symbolic and hard links which was cool. I tried a few recursive folder structures using each type of link and found one that kinda did what I was hoping for.

It uses junctions and it goes like this:

![Minotaur Jail](/assets/labyrinth.png)

_Note: You need to make the junction links seperately with mklink. You can't copy the first one in the Parent Folder into the Child Folder._

So the idea is you traverse the Labyrinth folder then go through the junction folder which takes you back to the parent or you go down the child folder finding the Minotaur.txt then you hit another junction which takes you back to the parent. This would keep looping endlessesly trapping the process that wanted to enumerate files and folders. You could also just keep clicking it manually too, you'll see the path bar get longer and longer.

To test this I wrote a simple python script to list all the files recursively from a given filepath.

```
for root, subdirs, files in os.walk(labyrinth_filepath):
	for filename in files:
		print(filename)
```

Running this on the Labyrinth folder filepath would return endless Minotaur.txt lines showing that the walk got caught in this recursive trap. Of course you could make use of the os.path.islink function to check whether each directory was a link or not and ignore them if they were but some file scanning and enumeration scripts would hopefully be caught in this. 

I actually forgot I was experimenting with these folder structures when I was working on my [file_integrity scripts]({% post_url 2024-03-09-file-integrity %}) and my file scanning which uses os.walk went on for half a day and wrote a huge log of repeating folders it had walked through. After adjusting these scripts to accept paths to ignore I then fired up sourcetree to commit and push the changes when I noticed one of my repo tabs was frozen. Lo and behold sourcetree got caught in this recursive trap too since I had this Labyrinth folder in one of my repos. I moved it out and that tab loaded just fine. 

Further testing of this Labyrinth shows:

| Works | Doesn't Work |
|---|---|
|Python os.walk 							| Sophos File Scanner
|Sourcetree 								| MalwareBytes
|Windows Defender (Custom Scan) 			|
|WinRar (I dare you to add it to a .rar) 	|

Ok so if you also want to do some testing feel free. I've included a `.bat` file in my [cyberTools](https://github.com/T4ngles/cyberTools/tree/master/labyrinth) repo which creates the labyrinth structure at the script location but it's also here if you are impatient and don't trust `.bat` files on the internet.

```
mkdir labyrinth
cd labyrinth
mkdir ParentFolder
cd ParentFolder
mkdir ChildFolder
mklink /J Junction1 %~dp0labyrinth\ParentFolder
cd ChildFolder
mklink /J Junction2 %~dp0labyrinth\ParentFolder
echo "" > Minotaur.txt
```

The naive intent behind this trap is to place the labyrinth in every important directory likely to be the target of enumeration or ransomware and if it can even buy just a little time it might be enough for the SOC team to carry out containment.