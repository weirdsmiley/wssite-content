---
title: "My Experience with Linux Kernel Bug Fixing Program"
date: '2024-12-06'
draft: false
summary: 'I participated in the kernel bug fixing program. How did it go? Read
up my experience in this post.'
tags:
  - linux
  - kernel
image: https://upload.wikimedia.org/wikipedia/commons/thumb/f/fd/Linux_Foundation_logo_2013.svg/800px-Linux_Foundation_logo_2013.svg.png
author: Manas
---

I participated in the [Linux Kernel Bug Fixing Mentorship Program][1] in the fall
of 2024. And as the mentorship is concluding, I am putting down my thoughts
about what I have learned from it.

The program began with 10 tasks which included build and running a kernel,
completing [LFD103][2] course, crashing kernel purposefully, writing a kernel
module with various functionalities and so on. These tasks were due to shortlist
candidates for the program. In the program, we had a dedicated Discord server
for communicating with peers and mentors. I should thank [Shuah Khan][10] and
[Anup Sharma][11] who were the mentors for this program. I should also thank 
[Ricardo B. Marliere][12] and [Javier Carrasco][13], who were extremely helpful
in resolving my doubts promptly on the channel. Apart from the Discord folks, I
also interacted with [Miguel Ojeda][14] on [Zulip][3], and a few other folks on
various IRC channels ([`#mm`][4] and [`#kernelnewbies`][5] in particular). In
all, I had an amazing experience and everyone involved was super nice and
extremely supportive.

Coming to the actual program, we were introduced to various tools that a kernel
developer may use in their day to day lives. This included cscope, syzkaller, b4
to name a few. [b4][6] which is a command-line tool for submitting patches (it is
much more than that), was my bread and butter for the past 3 months. It made my
life so much easier. I could write patch series, run checkpatch.pl over it,
archive submissions and do so much more. It was simply lots of fun using it and
it helped me tremendously. [Syzkaller][7] is a dynamic fuzzing tool for Linux
kernel (though its domain-specific language is architecture agnostic and allows
it to be run for any kernel). I used it in two ways. Firstly, I ran my own local
copy of syzkaller in an ubuntu virtual machine. This did gave me a bug or two to
look at (after running it over for more than 4 days). And secondly, I used the
syzkaller's dashboard to visit various reported bugs, and tried to reproduce
some of them on my host machine.  Whenever I could successfully reproduce a
crash, I would then attach a debugger to it to figure out how things were going
wrong. This prompted me to submit a few patches as well whenever I could sense a
fix was appropriate. For this task, I also wrote a hacky script, which could
automate this process. This made my work a lot easier. Everyday, I would pick a
few bugs from the dashboard and do the work of rebuilding, reproducing and
finally submitting a patch. As I was writing my final report for submission, I
checked my browser history to find that I almost looked at over 130+ bugs in the
span of 3 months.

In summary, I contributed a total of [11 patches][8] out of which 7 are either
[in mainline or are going to be soon][9]. I had an awesome experience in this
mentorship and I can definitely see myself becoming a permanent contributor to
the Linux kernel.  And I would definitely encourage aspiring kernel developers
and students to check it out next year. The mentorship opens up for 3 times in
an year and is mostly targeted towards newbies.

_If you have any questions for me, head over to this [discussion page][15]._

<!-- References -->

[1]: https://mentorship.lfx.linuxfoundation.org/project/a1fc17cf-d75e-4834-8eb0-7652a50b7ab1 "Linux Kernel Bug Fixing Program"
[2]: https://training.linuxfoundation.org/training/a-beginners-guide-to-linux-kernel-development-lfd103/ "LFD103"
[3]: https://rust-for-linux.zulipchat.com/ "Rust for Linux - Zulip"
[4]: https://linux-mm.org/ "Linux MM"
[5]: https://kernelnewbies.org/ "Kernelnewbies"

[6]: https://b4.docs.kernel.org/en/latest/ "B4 tool"
[7]: https://syzkaller.appspot.com/upstream "Syzkaller Dashboard"

[8]: https://lore.kernel.org/all/?q=manas18244@iiitd.ac.in "Submissions - Lore"
[9]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?h=v6.13-rc1&qt=author&q=manas18244%40iiitd.ac.in "Upstream commits"

[10]: https://en.wikipedia.org/wiki/Shuah_Khan "Shuah Khan - Wikipedia"
[11]: https://github.com/TwilightTechie "Anup Sharma - Github"
[12]: https://github.com/rbmarliere "Ricardo B. Marliere - Github"
[13]: https://github.com/javiercarrascocruz/ "Javier Carrasco - Github"
[14]: https://github.com/ojeda "Miguel Ojeda - Github"
[15]: https://github.com/weirdsmiley/wssite-content/discussions/2 "Post discussion"

