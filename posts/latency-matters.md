# Latency matters

A month ago, danluu wrote about [terminal and shell performance](https://danluu.com/term-latency/). In that post, he
measured the latency between a key being pressed and the corresponding character appearing in the terminal. Across
terminals, median latencies ranged between 5 and 45 milliseconds, with the 99.9th percentile going as high as 110 ms for
some terminals. Now I can see that more than 100 milliseconds is going to be noticeable, but I was certainly left
wondering: Can I really perceive a difference between 5 ms latency and 45 ms latency?

Turns out that I can.

Like probably all shell configurations, mine has a prompt, that is: It displays some contextual data when I can enter
the next command. Most prompts include the username, hostname, and the current working directory. Mine can also include the
current Git branch and commit, the current kubectl context, and which OpenStack credentials are currently loaded. It
knows some quite unique tricks. For example, if the current working directory is not accessible anymore, it highlights
the inaccessible path elements. Or, inside a Git repository, it highlights the path elements inside the repo.

![prettyprompt screenshot](https://raw.githubusercontent.com/majewsky/gofu/master/screenshot-prettyprompt.png)

Since that's quite a lot of logic, I delegated the rendering of the entire prompt to a custom program. That program used
to be a [Python 2
script](https://github.com/majewsky/devenv/blob/643a55f49b13401e6333fbb3a1413cd7dc59907f/bin/prettyprompt.py), but since
I'm not nearly as fluent in Python as I used to be, I didn't dare touch it anymore. I therefore decided to rewrite it
into a [Go program](https://github.com/majewsky/gofu). The feature set didn't change, but here's a thing that *did*
change:

```bash
$ time ./bin/prettyprompt.py > /dev/null; time prettyprompt > /dev/null
./bin/prettyprompt.py > /dev/null  0,05s user 0,00s system 98% cpu 0,057 total
prettyprompt > /dev/null  0,00s user 0,00s system 92% cpu 0,003 total
```

The new prompt renders in 3 ms whereas the old one took 57 ms. That's nothing to do with sloppy programming, that's just
how long it takes to start up the Python interpreter, as can be easily observed by running a Python interpreter without
actually doing anything:

```bash
$ time python2 -c 0 > /dev/null
python2 -c 0 > /dev/null  0,02s user 0,01s system 52% cpu 0,050 total
```

(By the way, all these measurements are on hot caches. The first execution of `python2` takes more than double as long.)

The Go program, on the other hand, does not need to start a runtime. It's probably short-lived enough to never even
garbage-collect.

I did these measurements while I was still writing the Go program, just for fun. But only after switching to the new
prompt did I realize how much snappier my terminal feels just because of this change. There was always this short gap
between seeing the output of one command and being able to enter the next command, but I did never really understand
that this was due to my slow prompt renderer, and not due to the behavior of the shell, the terminal, the OS scheduler,
or any other entity involved in the process.

When I press Enter in my shell now, the next prompt just appears instantaneously, without any perceivable latency
between the keypress and the prompt being rendered. This feels so magical, I cannot even put it into words. It's as if
this were a new notebook, not the same one that I've been using for five years now.

The point is, latency really *is* important for how an application or a system feels. I will definitely care more about
responsiveness of my programs in the future after this experience.
