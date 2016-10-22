# Hello World

I had known for a long time that I need a new blog. I had one years ago in the cloud
([it's still live](https://majewsky.wordpress.com)), but I definitely wanted something self-hosted this time. I had a
brief look at static website generators, and quickly decided that (as usual) I want a custom-tailored solution.

The first iteration is an nginx serving static files rendered by a
[tiny Go program](https://github.com/majewsky/blog-generator). Content comes from a
[GitHub repo](https://github.com/majewsky/blog-data) and is pulled every few minutes. Good enough for a first shot.
I might change the cronjob to be triggered by a GitHub webhook later on, but only if the delay until the next cronjob
run annoys me enough.
