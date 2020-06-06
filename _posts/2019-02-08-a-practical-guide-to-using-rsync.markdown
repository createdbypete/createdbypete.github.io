---
layout: post
title: A Practical Guide to Using rsync 🤩
date: '2019-02-08 00:42:00'
tags:
- tips
---

A free and open-source command-line utility available on most platforms `rsync` has been around for quite a while now and chances are you might have &nbsp;used it already. One of the things I use it for most is to _sync_ directories on my local machine (for creating ad-hoc backups to an external drive in my case) or you can sync to a remote connection just as easily.

## Installation

Most machines will have `rsync` available, you can check by running `which rsync` in your terminal to show you where it is located. If you don’t get a response then you will need to install it

MacOS already has `rsync` available so you don’t need to worry about installing it again although a more recent version is available through [Homebrew](https://brew.sh).

Windows users I am sorry but I don’t believe the `rsync` is directly available to you however it appears [a question on StackExchange](https://superuser.com/questions/69514/windows-alternative-to-rsync) may have some options for you to consider.

## Usage overview

There are a few options that you will use the most, so we’ll focus on these in this practical guide. You can mix and match these depending on what you would like to achieve too.

    rsync [options] [source] [destination]

- `-v`, `--verbose`, this will increase the verbosity, showing the path for every file copied. Can be helpful to see what’s going on.
- `-a`, `--archive`, archive mode is very useful for backups, it will preserve everything about the file such as timestamps and permissions. The only thing it does not preserve is hard-links, because finding multiply-linked files can be expensive.
- `-z`, `--compress`, compress file data during the transfer to reduce bandwidth, this is particularly useful when working with a remote server.
- `-n`, `--dry-run`, perform a trial run with no changes made, very useful to do this before combining with the `--delete` option below.
- `--delete`, this will delete the files in the destination not present in the source.
- `-b`, `--backup`, you might also wish to keep and files that might be overwritten or deleted.
- `--help`, probably the most useful one of them all, show the help for `rsync`.

## You said Practical Guide?

My most common use case for `rsync` has been for synchronising folders on my local machine with an external hard-drive. Occasionally for managing files on remote servers at times too but I have less need for that these days.

The `rsync` command can also be used more creatively and I intend to demonstrate a few of those use cases below along side the basics:

### 1. Copy/sync a directory on your local machine

A classic, simply creates an exact (recursive) replica of another directory. Make the destination a path to your external storage device and you’ve got an efficient backup tool. You can always use the `--dry-run` option to check what’s going to change for the nervous.

**Please note:** the trailing `/` on the source is important to copy the contents of the directory and not the directory itself.

    rsync -azvh --delete /my/src/directory/ /my/dest/directory

You might want any symlinks in your directory to be resolved into real files. In this case add `-L` to the list of options.

- `-L`, `--copy-links`, transforms a symlink into the real file or directory it targets.
- `--safe-links`, consider this option if you want to ignore symlinks that point outside the directory you’re copying.

### 2. Copy/sync a directory to a remote server

Another classic use case. The main difference here is we are specifying the username and host(or IP) of the machine we wish to connect to, the remote server will also need to have `rsync` installed.

I would also opt for using `rsync` over ssh as it will encrypt your data as it travels over the internet, if you don’t want this remove the `-e ssh` option. Using SSH assumes you can already connect to the remote server this way without `rsync`.

    rsync -azvh -e ssh /my/src/directory/ root@123.123.123.123:/my/dest/directory

Add the `--delete` option if you want to remove any files that don’t in the source directory.

- `-e`, `--rsh=COMMAND`, allows you to specify the remote shell to use. In our case `ssh`.

### 3. Sync files but make backups of any deleted files

If the idea of the `--delete` option makes you nervous that is completely understandable, `rsync` provides no method of recovering the deleted files. However, you can pass in the `--backup` option, this will make copies of any files due to be deleted or updated.

The `--backup` option needs another option to work best, introducing `--backup-dir`. These options together allow you to specify the location of the backups and a string to add to the end of the filename, such as the date.

    rsync -avz --delete --backup --backup-dir="backup_$(date +\\%Y-\\%m-\\%d)" /src/path/ /dest/path

By using `$(date +\\%Y-\\%m-\\%d)` I’m telling it to use today’s date in the folder name, assuming you are using `bash` for your shell. If not you may need to adjust this to suit.

- `-b`, `--backup`, with this option, preexisting destination files are renamed as each file is transferred or deleted.
- `--backup-dir=DIR`, this tells `rsync` to store all backups in the specified directory on the receiving side. There are many more options provided by `rsync` but for this short guide I’ll leave that as an exercise for you the reader to explore with `rsync -h` or `man rsync` now you’ve seen some examples.

I’d be interested to hear about other useful ways `rsync` can be used, [send me a tweet @createdbypete](https://twitter.com/createdbypete) if you feel like sharing.
