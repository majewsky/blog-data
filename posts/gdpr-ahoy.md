# GDPR ahoy!

The [General Data Protection Regulation][gdpr] comes into full force today. I
took the opportunity to add a data privacy statement to the blog. Here is it
in its full glory:

> No system under my control records any personal data of users of this website.

This is true because I disabled the nginx access log a long time ago, and
because I restricted the nginx error log to not report 404 errors. So the logs
are basically empty now (except for alert messages from nginx). [These are the
relevant lines in my nginx config.][link]

No idea why any blogger would be panicking about GDPR.

Side-note: Even though I choose not to wiretap my users' browsers, [I still
have analytics][monitoring].

[gdpr]: https://en.wikipedia.org/wiki/General_Data_Protection_Regulation
[link]: https://github.com/majewsky/system-configuration/blob/da7f81133dee981f771dafd3c066ad28e7a09a8f/hologram-nginx.pkg.toml#L102-L103
[monitoring]: https://blog.bethselamin.de/posts/latency-matters-aftermath.html
