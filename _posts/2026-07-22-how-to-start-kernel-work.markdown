---
layout: post
title: "How to: choose a subsystem to work on"
date: 2026-07-22 16:00:00 +0200
categories: kernel kbeginner
---
## Documentation? 

A common piece of advice to people who want to dip their toes in kernel work is
to start writing documentation, and initially it sounds like a good idea - no
code, just writing *highly technical* text describing code you *don't*
understand... see where I'm going with this? How can you write documentation for
something you don't (and can't) understand? Sure, finding typos is fine, but
typing up brand new text files describing a driver you're looking at for the
first time sounds like a very bad idea. Even harder if English isn't your first
language! In this post I'll describe how to choose a subsystem to start your
kernel journey in and not burn out before your first patch gets accepted.

## What do I choose then?

When you first look at the [kernel source tree](https://github.com/torvalds/linux)
(since you're new, you're probably looking at the Github mirror), the structure
looks daunting. arch? fs? mm? What even is that? Unless you're
extremely gifted at understanding code for the first time, or using an LLM,
you should definitely avoid these directories! These are just an example of the
core subsystems of the kernel (`arch/` being computer architecture specific code,
`fs/` is filesystems and `mm/` is memory management) which are extremely hard to
contribute to, even for more experienced developers. If you want to contribute,
look at the `drivers/` subsystem - this is where the majority of changes are
made. New devices are constantly being added, old devices are (sometimes)
removed and a lot of fixes and improvements are made.

Okay, now that we've established drivers as our point of interest, let's actually
go through the driver subsystems. Again, some of these will be very hard to
contribute to (for example `gpu/drm/`, the direct rendering manager, which is
essentially what makes your graphics card work). Instead, we should choose a
subsystem that
- has abstractions that are relatively easy to understand
- allows easy testing due to cheap hardware
- has a friendly community that welcome new contributors (however this is true
  for the majority of subsystems)

## Examples of good subsystems

Call me biased, but the Industrial I/O (IIO for short) subsystem sounds like a
great fit! The abstractions are quite simple, hardware is cheap (Aliexpress
always helps!) and the community is very helpful! The IIO subsystem adds support
for various sensors that work via the I2C or SPI protocol, however a lot of the
drivers have style issues, missing error checks etc. Most of these don't even
require testing on the actual device, which is a relief.

Another good subsystem (which is closely associated with IIO) is hwmon. This
adds support to hardware monitoring sensors (temperature), fan controllers and
water cooling systems in PCs. You can also find a lot of style issues and other
things which need cleaning up.

There are a lot of subsystems in the kernel and virtually all of them are
friendly, nevertheless they differ in actual difficulty understanding the
underlying architecture. Ideally you should pick something that you're already
doing in "userspace" - if you work with sensors and Arduino, IIO is for you.
If you work on networking stuff, `net/` is ideal. If you're not sure, that's
fine, you have plenty of options to find something you like!

## What about staging?

A lot of guides you find online will recommend the `staging/` subsystem as a
good starting point. My experience? 50-50. Staging drivers have **a lot** of
cruft that needs cleaning - from code not conforming to modern ABI, spaghetti
code that needs a refactor to typos and general style changes. It depends on
the type of driver you're working on, but a lot of maintainers won't accept
simple style changes (such as checkpatch.pl fixes) as they are very low priority
and create noise in the process. These changes would get accepted after the
very critical issues are resolved within the driver, therefore submitting
patches for staging drivers tends to be a hit or miss. It is a good place where
to analyze kernel code antipatterns though!

## Mentoring

A lot of people will find a subsystem to work on when applying for the Linux
Foundation Mentorship [1] program. Here, maintainers can post various projects
that need undertaking in their subsystem and anyone can apply. You will get very
valuable advice and guidance all while actually working on the kernel!

## To end this long article

At the end of the day, it's up to you to choose which subsystem you want to
contribute. It will look scary in the beginning, but once you start reading
the dedicated mailing list and see what others have to say, the initial shock will
start to wear off and you will feel more comfortable sending patches. After all,
we've all been there, and we still come back to contribute!

This is part 1 of a series on how to contribute to the Linux kernel.

[1] https://mentorship.lfx.linuxfoundation.org/#projects_all

