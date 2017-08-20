# Does the HN crowd show up in monitoring?

So [my previous post](/posts/latency-matters.html) made [the frontpage on Hacker News](https://news.ycombinator.com/item?id=15059795).
Awesome! I have seen many websites collapse under the load of a HN crowd, so this is the perfect time to look at the monitoring.

![Munin charts](/assets/post-images/munin1.png)

1 megabit per second? That's lower than I anticipated, even with all the size optimizations that I implemented (or let's
just say, all the bloat that I purposefully did not add). Same goes for CPU usage: I've heard so many people on the
internet complain about how expensive TLS handshakes are, yet my virtual server handles all of this with less than six
percent of a single core. It's barely even visible on the CPU usage chart, and completely drowned out by noise on the
load chart.
