---
layout: page
title: About
permalink: /about/
---
Hi, I'm Josh, .NET developer by day and Linux kernel developer by night. I
started this blog in the hopes of helping out others that would like to start
contributing to the kernel and don't know where to start.

I primarily work in the IIO subsystem, but occasionally I branch out to other
subsystems (counter, hwmon etc.). Additionally, I *try* to fix bugs reported by
syzkaller.

I am a maintainer of the following in the kernel:
- [Vishay VEML3328 RGBCIR Light Sensor](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iio/light/veml3328.c)
- [Microchip MCP47A1 6-bit Digital-to-Analog Converter](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iio/dac/mcp47a1.c)

If you wish to see my kernel commits in mainline, you can do so [here](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?qt=author&q=Joshua+Crofts).

Thanks for reading!

{% highlight c %}
static int __init welcome(void)
{
    printk(KERN_INFO "Welcome to KBlog!\n");
    return 0;
}

module_init(welcome);
{% endhighlight %}
