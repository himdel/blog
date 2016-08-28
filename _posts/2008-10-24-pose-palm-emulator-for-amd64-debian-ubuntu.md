---
layout: post
title: "pose (Palm Emulator) for amd64 debian / ubuntu"
date: 2008-10-24
categories: amd64 debian package padict palm pose pose32 ubuntu
---

Some time ago, I moved from 32bit debian to 64bit debian on my notebook. So far, having configured flash and installed firefox from ubuntu, I haven't run into any problems. Until I tried to run Padict (Japanese dictionary for palms) under pose .. there is no pose package for amd64! And it's not so easy to compile either.

So [now there is](http://himdel.mine.nu/pose32_3.5_amd64.deb). The creation was relatively straightforward, first I downloaded i386 versions of pose and recursively all its dependencies that aren't architecture-independent from packages.debian.org. That's: <code>pose (3.5-9.1), libdrm2 (2.3.1-1), libexpat1 (2.0.1-4), libfltk1.1 (1.1.9-6), libfontconfig1 (2.6.0-1), libfreetype6 (2.3.7-2), libgcc1 (1:4.3.2-1), libgl1-mesa-glx (7.0.3-6), libjpeg62 (6b-14), libpng12-0 (1.2.27-2), libstdc++6 (4.3.2-1), libx11-6 (2:1.1.5-2), libxau6 (1:1.0.3-3), libxcb-xlib0 (1.1-1.1), libxcb1 (1.1-1.1), libxdamage1 (1:1.1.1-4), libxdmcp6 (1:1.0.2-3), libxext6 (2:1.0.4-1), libxfixes3 (1:4.0.3-2), libxft2 (2.1.12-3), libxinerama1 (2:1.0.3-2), libxrender1 (1:0.9.4-2), libxxf86vm1 (1:1.0.2-1), zlib1g (1:1.2.3.3.dfsg-12)</code>.
And then I only extracted them (<code>for foo in *deb; do dpkg-deb -X $foo .; done</code>), renamed lib and usr/lib to lib32 and usr/lib32 respectively, removed crud from usr/share, renamed the binary, manpage and menu entry to pose32, created a Makefile, ran <code>dh_make -n -c artistic -s -p pose32</code>, edited debian/control a bit and finaly typed debuild and waited :).

You still need pose-skins and a rom image.
No, this version works, I see no reason to continually update it.
