---
layout: post
title: "How to: find your first good contribution"
date: 2026-07-23 12:30:00 +0200
categories: kernel kbeginner
---
Finding a first issue can be easier said than done - don't forget that you're
"competing" with hundreds of other people around the world, so sometimes when
you send a patch fixing an issue, the maintainer will then point out that
somebody beat you to the punch and a fix has already been applied (this is the
reason why you should rebase your development tree regularly!). Nevertheless, a
lot of issues which need desperate fixes are sitting in Linus' tree and nobody
is trying to fix them. I'll be using examples from the IIO subsystem in this
post, however the things described here can be applied to other subsystems.
Issues will be sorted from easy to "hard".

## Typos

Probably the simplest change you can do. People make mistakes - sometimes you
make an extra keystroke and nobody notices during the review process. Typos in
the actual code are very rare (but they can happen), usually they appear in
comments. These could be simple grammatical errors, additional characters or
maybe invalid information, such as an incorrect ID value that was misread from a
datasheet. It's an okay starting point - it makes the code look more
professional (and correct), nevertheless keep in mind that this is *low hanging
fruit* - a lot of people send a big patch series fixing typos in entire
subsystems, and maintainers definitely prefer these as opposed to a single patch
fixing "Get's" to "Gets".

## Style issues

Okay, this varies from maintainer to maintainer. The Linux kernel has a defined
coding style [1]. Depending on the state of the subsystem, some maintainers choose
not to accept style changes as other **more serious** issues need fixing first
(a good example of this is `staging/media/atomisp/`, which is dedicated to
adding support for Intel Atom based cameras and is riddled with severe issues).
Subsystems that are more mature will usually want to have their codebase adhere
to the kernel coding style (e.g. IIO accepts style changes).

Examples of bad code style:
- Multi-line function parameters that aren't aligned with the opening parenthesis
- Lines exceeding the 80 (sometimes 100) character count
- Using `if (foo == NULL)` instead of `if (!foo)`
- Trailing whitespace
- Excessive braces around single statements

Of course, these issues can sometimes be painful to spot, this is why the kernel
community made a handy Perl script that finds these issues for you. To use it,
run the following command in the root of your kernel directory:
{% highlight bash %}
scripts/checkpatch.pl -f path/to/file
{% endhighlight %}

An example output can look like the following:
{% highlight text %}
ERROR: space required after that ',' (ctx:VxV)
#32: FILE: drivers/iio/light/al3010.c:32:
+#define AL3010_GAIN_MASK               GENMASK(6,4)
                                                 ^

total: 1 errors, 0 warnings, 247 lines checked
{% endhighlight %}

These are great changes that improve the overall readability and maintenance
of the code.

## Compiler warnings and Sparse

Ideally, the compiler shouldn't produce any warnings when running; nevertheless,
some mistakes slip through the cracks. Running `make` with the `W=1` flag will
make the compiler warn on all issues. Another good flag is `C=1`. This invokes
the Sparse static analyzer (separate install needed). Using this tool can reveal
more serious issues like use-after-frees and incorrect types in assignment.

These are also easy fixes, the compiler and Sparse will tell you (usually)
exactly what's wrong and sometimes even how to fix it.

## Include What You Use

This is a pretty good principle when programming in general - don't add unused
`#include` headers in your files! Not only it makes your list of headers
messier, it also worsens the build time of your kernel/module/etc.! Originally,
the kernel used a `<linux/kernel.h>` header for including fundamental building
blocks, but over time developers started including this header everywhere
instead of including the actual header of the things they use. This creates a
not-so-fun side effect - if an unrelated header in kernel.h is touched and
your driver includes kernel.h, you will have to recompile the header, even if
it's not used in your driver, making your build time longer.

If you're unsure about which headers to remove and which to add, there is a
handy tool for this from Google called "iwyu-tool". It's a bit harder to use
and it requires LLVM, but this is luckily a place where LLMs shine - they're
good at configuring this stuff. Additionally, the tool generates a lot of
unrelated noise; this is prevented by the use of a mapping file. IIO does have
a mapping file that can be used [3].

## Replacing sprintf/snprintf with sysfs_emit()

Previously, if you wanted to pass a string to display in userspace via a `_show()` function, `sprintf()` or `snprintf()` were used to write text into the output buffer. These functions, however, aren't protected against page buffer overflows (`PAGE_SIZE`). On the other hand, `sysfs_emit()` is safe, purpose-built for sysfs `_show()` callbacks, and its usage is recommended in the kernel documentation.

The diff for this change is fairly simple:

{% highlight diff %}
 static ssize_t val_show(struct device *dev, struct device_attribute *attr, char *buf)
 {
 	struct my_data *data = dev_get_drvdata(dev);

-	return sprintf(buf, "%d\n", data->val);
+	return sysfs_emit(buf, "%d\n", data->val);
 }
{% endhighlight %}

## mutex_lock/unlock() vs. guard()(mutex)

A very fun feature of the GNU extensions of the C language that the kernel uses
is the `<linux/cleanup.h>` header, allowing automatic cleanup of objects once
the program exits from their scope. This can be applied to allocated memory (no
need to call free), power management and **locking**. Consider the following code:

{% highlight C %}
static int m62332_set_value(struct iio_dev *indio_dev, u8 val, int channel)
{
	struct m62332_data *data = iio_priv(indio_dev);
	struct i2c_client *client = data->client;
	u8 outbuf[2];
	int res;

	if (val == data->raw[channel])
		return 0;

	outbuf[0] = channel;
	outbuf[1] = val;

	mutex_lock(&data->mutex);

	if (val) {
		res = regulator_enable(data->vcc);
		if (res)
			goto out;
	}

	res = i2c_master_send(client, outbuf, ARRAY_SIZE(outbuf));
	if (res >= 0 && res != ARRAY_SIZE(outbuf))
		res = -EIO;
	if (res < 0)
		goto out;

	data->raw[channel] = val;

	if (!val)
		regulator_disable(data->vcc);

	mutex_unlock(&data->mutex);

	return 0;

out:
	mutex_unlock(&data->mutex);

	return res;
}
{% endhighlight %}

See the mutex_lock() and mutex_unlock() calls? And the fact that we have to use
a goto to jump to a label to unlock? Although not as severe as one would expect,
this still makes the code harder to read and could cause issues if one accidentally
forgets to unlock the mutex. This is where `guard(mutex)` comes into play. With it,
we can rewrite the function to look like this:

{% highlight C %}
static int m62332_set_value(struct iio_dev *indio_dev, u8 val, int channel)
{
	struct m62332_data *data = iio_priv(indio_dev);
	struct i2c_client *client = data->client;
	u8 outbuf[2];
	int res;

	if (val == data->raw[channel])
		return 0;

	outbuf[0] = channel;
	outbuf[1] = val;

	guard(mutex)(&data->mutex);
 
 	if (val) {
 		res = regulator_enable(data->vcc);
 		if (res)
			return res;
 	}
 
 	res = i2c_master_send(client, outbuf, ARRAY_SIZE(outbuf));
 	if (res < 0)
		return res;
	if (res != ARRAY_SIZE(outbuf))
		return -EIO;
 
 	data->raw[channel] = val;
 
 	if (!val)
 		regulator_disable(data->vcc);
 
 	return 0;
 }
{% endhighlight %}

We removed the goto and the label, ensured that the mutex will be unlocked on
any exit from the function and saved lines!

## TL;DR

Contributing to the kernel can be easier than you expect! Nobody is expecting a
1000-line patch changing critical code as your first contribution. Start small
so that you get comfortable with the workflow and review process and over time
you'll find yourself submitting brand new drivers and doing major refactors.

This is part 2 of a series on how to contribute to the Linux kernel.

[1] https://www.kernel.org/doc/html/latest/process/coding-style.html\
[2] https://elixir.bootlin.com/linux/v7.2-rc4/source/include/linux/kernel.h\
[3] https://lore.kernel.org/all/20260512073505.1310-1-joshua.crofts1@gmail.com/
