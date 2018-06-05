---
layout: post
title:  "HPKit released"
date:   2015-01-11 20:27:36 +0200
categories: hp hpkit
author: giulio
---
HPKit release 0.1, a set of simple **shell linux** tools to communicate to test equipment, Hewlett Packard/Agilent and others, via GPIB/HPIB interface.

You can download it from GitHub:

[https://github.com/rapgenic/hpkit.git](https://github.com/rapgenic/hpkit.git)

It consists of three programs:

- **hplot**, emulates an HP plotter to print or save a screenshot of the instrument's display; it can be used with hp2xx ([https://www.gnu.org/software/hp2xx/](https://www.gnu.org/software/hp2xx/));
- **hplisten**, similar to hplot, though it doesn't emulate a plotter, but only listens for generic data;
- **hptalk**, used to send commands to the instrument and read the answer.

The code is released under GNU GPLv3 license, so feel free to download, edit and redistribute it.

For further information see [HPKit]({{ site.baseurl }}{% link pages/01-hpkit.md %}) page.
