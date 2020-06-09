# TL;DR

If you want to speedup your Drupal and have a big 'url_alias' table (more than 10K rows?), you can try this:

``` SQL
CREATE EXTENSION pg_trgm;
CREATE INDEX url_alias__alias_trgm_gist_idx ON url_alias USING gist (alias gist_trgm_ops);
CREATE INDEX url_alias__source_trgm_gist_idx ON url_alias USING gist (source gist_trgm_ops);
ANALYZE url_alias;
```

---\



I was tasked with improving the performance of a somewhat large customized Drupal site. We have about 110K comments and 278K nodes. Our Drupal instance was unbearably slow. It took about 30 seconds just to open the content admin interface. At some moment, the Drupal caches were disabled due to problems with our custom code, and as the Drupal instance grew the performance declined.

> The general opinion was that our features and generalized network problems were to blame.

Before touching anything, I started to measure execution times. The load in our servers is quite variable: we have over 450K users and other services like Wordpress or Moodle running there.

I scripted Firefox using [Watir](http://watir.com) to have some average measures over the day and changed the [log format in our Nginx](http://nginx.org/en/docs/http/ngx_http_log_module.html) load balancer to show the time spent on our application servers.

The Watir script adds a UUID for each URL request. This allowed me to relate the time to load a page with the time it takes to generate it (we have [Piwik/Matomo](https://matomo.org) installed but it measures the full network trip).

> With this timings I discarded network problems: almost all the time was spent on our web servers.

The web servers didn't show evident problems: almost no CPU use, nor enough disk IO to cause delays. So I moved to our Postgres servers. I've installed [pgBadger](http://dalibo.github.io/pgbadger) long ago and configured the logs to show any query that lasts more than 100ms. pgBadger is an incredible tool to improve Postgres performance in the long term. If you're worried about the performance penalty of logging, think that the alternative is to wait until you have a slow server. It will be worst by then.

In the Postgres logs I saw some slow queries that lasted for a few seconds, but on the slowest Drupal pages, there were a lot of queries that lasted between 200 to 300ms. These small queries were blamed too in pgBadger as the ones that used more CPU time due to the high number of executions.

They were very simple queries, just like:

{% highlight sql %}
SELECT URL_ALIAS.SOURCE, LANGCODE, PID FROM URL_ALIAS
WHERE (ALIAS ILIKE '/SOME/URL/PATH/HERE')
      AND (LANGCODE IN ('ES', 'UND'))
ORDER BY LANGCODE ASC NULLS FIRST, PID DESC NULLS LAST;
{% endhighlight %}

No complex joins, nor subselect queries.

> The real problem was that the ILIKE forced Postgres to do a sequential scan over the URL_ALIAS table: 145K rows for every URL in a Drupal page.

As you can see, there are no '%' in the ILIKE filter, it is there only to do a case-insensitive comparison. The origin of the problem was that MySQL is case insensitive, but Postgres and SQLite no. There is an issue since 2012 about "[Normalize how case sensitivity is handled across database engines](https://www.drupal.org/project/drupal/issues/1518506)", but It's still open.

I could have changed the Drupal code to use "lower(ALIAS)" with an expression index instead of ILIKE, but I would have to maintain this code on every Drupal update.

> Thankfully, Postgres has the pg_trgm extension for creating indexes that ILIKE (and other textual searches) can use.

You should read about it [here](https://www.postgresql.org/docs/current/static/pgtrgm.html) (did I told you that Postgres documentation is gorgeous!).

Finally, the solution was:

{% highlight sql %}
CREATE EXTENSION pg_trgm;
CREATE INDEX url_alias__alias_trgm_gist_idx ON url_alias USING gist (alias gist_trgm_ops);
CREATE INDEX url_alias__source_trgm_gist_idx ON url_alias USING gist (source gist_trgm_ops);
ANALYZE url_alias;
{% endhighlight %}

After this small change, the ILIKE selects dropped from 200-300ms to 20-30ms. As a mate said it was like magic. The Drupal instance became what you expect from a customized web app. Not as fast as Google, but almost all pages load under a second, and the slower ones are usually under 2.5s.

I would like to have some more time to try to make our app more performant (calls to external web services and trying to enable caches mainly), but we have other projects in the queue so it will have to wait.

I was very happy with how we solved the problem: some days measuring and then just one small change with a big impact. I felt like a performance sniper (sometimes you can hear the Gatling guns when some groups tackle performance problems :-).
