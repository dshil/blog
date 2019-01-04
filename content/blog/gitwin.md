---
title: "Git: tuning performance on Windows"
date: 2018-12-30T15:13:34+03:00
draft: false
---

## Preface

Internet was designed in a such way that almost nobody remember the days when
there is no internet. People just got used to this phenomenon, like an air but
also there are some men that remember that days and we should protect them as
the UNESCO area. This feeling that the internet has been at all times shows
that it was very well-designed and later - implemented.

In that days there wasn't such thing as a revision control system. First of
them were implemented [not so long ago](https://en.wikipedia.org/wiki/Version_control). As a result modern developers probably
won't never encounter the difficulties while working with them. By
difficulties I mean: bad design, very slow, not stable. Probably, some of you
remember the story of the Linux Kernel community and BeetKeeper usage. For most
of the developers it was a big pain in the ass. The Kernel was growing very
fast and BeetKeeper was slowing down roughly at the same speed. One day Linus
and [some guys from the Linux Kernel](https://en.wikipedia.org/wiki/Git) built
their own version control system and gave it the great name: git - the stupid
content tracker which is the standard de facto in our days. Git also was
designed in a such way that most of the developers don't remember the days when
there is no Git. Git is treated as a perfect knife that will kill any enemy
without any problems. For most project it actually is the perfect tool and most
of the developers won't encounter any problems with it.

Probably one day you will start working on a very large codebase and realize
that simple `git diff` or `git status` takes more than 30 seconds and usual
instant feedback loop (make some changes - save - see the changes) will be
broken. You will feel angry because your lovely flow feeling will be broken.
Most of the C++ developers are very familiar with it: start the compilation
and go for a lunch. I was firstly bit by this issue during LibreOffice
development for Windows.

## Environment

Before tuning any options it is very important to realize what is the initial
problem. Let's begin with an environment:

### Windows Partition

* OS: Windows 10
* FS: NTFS
* Hard drive: SSD Kingston 480 GB
* POSIX-compatible environment: Cygwin

### Linux Partition

* OS: Fedora 29
* FS: Ext4
* Hard drive: SSD Kingston 480 GB

### NTFS or EXT4

I am not an expert in the file system domain and can't properly explain why
there is zero problems with Git working on EXT4 and there is a huge performance
degradation on NTFS. It doesn't mean that NTFS is worse than EXT4 because it
depends on the operations and environments. I have problems with Git using
Cygwin but probably it is not a case. If you are curious about benchmarking
different FS, the following links may be a good starting point:

* [Why does Windows still use NTFS? Why not ext4?](https://www.quora.com/Why-does-Windows-still-use-NTFS-Why-not-ext4-the-file-system-for-Linux-since-it-actively-prevents-disk-fragmentation)
* [Linux 4.7 - Btrfs vs. EXT4 vs. F2FS vs. XFS vs. NTFS Benchmarks](https://www.phoronix.com/scan.php?page=news_item&px=Linux-4.7-FS-5-Way)
* [Comparison of file systems](https://en.wikipedia.org/wiki/Comparison_of_file_systems)

## Tuning Time

"Git is slow on Windows" is very popular topic in internet and there are a lot
of articles describing how to increase the performance of your Git in the
Windows environment. I will just summarize most important of them.

### Tuning Git Configuration

There are 3 magic options that should be turned on to solve all your problems,
but before tuning any options ensure that you really have any problems:

```sh
git config --global core.preloadindex true
git config --global core.fscache true
git config --global gc.auto 256
```

Let's explore what each option does:

* core.preloadindex enables parallel index preload for operations like `git diff`.
  Defaults to true.
* core.fscache enables file system cache. It is useful only for Windows.
* gc.auto minimizes the number of files under ./git directory.
  Defaults to 6700.

For more info: `man git config` and [1](https://stackoverflow.com/questions/4485059/git-bash-is-extremely-slow-on-windows-7-x64), [2](https://blog.praveen.science/solution-to-git-bash-is-very-slow-in-windows/), [3](https://github.com/msysgit/msysgit/wiki/Diagnosing-why-Git-is-so-slow)

### Cygwin vs Git for Windows

If your project doesn't depend on the Cygwin environment and you just need the
git, I would highly recomment to use [Git For Windows](https://git-scm.com/download/win) .

In comparison to Cygwin, Git for Windows doesn't have any performance issue
for a huge codebase and all git commands like `git status`, `git stash`,
`git commit` work very fast.

### Tuning Development Workflow

When Cygwin is only the choice, you still have some options that will allow
you to speed up your Git workflow. A lot of techniques are described in
[managing large repositories with Git](https://www.atlassian.com/blog/git/handle-big-repositories-git) .
I just picked some of them and adopted my development environment in the
following way:

* No need to clone the repository with the full Git history because it can be
  very huge, e.g., for LibreOffice project, `git rev-list --count HEAD` equals
  to 432192 commits. Specify the number of commits you are interested in
  by `--depth` option during the clone. But remember that shorter the history
  then less chance of getting the value from the `git blame` and `git log`.
* No need to run `git status`, `git diff` for the top level path of your
  repository. You can specify the only changed directory. As a result you
  should think carefully about splitting your changes into several commits that
  reduces the scope of the changes per commit.
* No need to run Git command each time you do some changes (I had this habit,
  probably you too), just slow down your development workflow and keep changes
  as small as possible that allows you to remember the recent activity.
