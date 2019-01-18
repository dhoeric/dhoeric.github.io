---
layout: post
slug: logrotate-crontab
title: "Logrotate with crontab"
date: 2017-06-03 20:32:03 +0800
comments: true
tags: [unix, system, admin, logrotate, crontab]
---
While working with logrotate, it's usually pretty hard to debug while we want to write it in a short period of time.

Here are some basic info of logrotate, as well as crontab, which invokes logrotate regularly in Ubuntu 14.04.

## Test logrotate manually

Directory of logrotate config file, for defining target service's log rotation:
{{< highlight shell >}}
/etc/logrotate.d
{{< / highlight >}}

To test with logrotate manually, you can run
{{< highlight shell >}}
logrotate -d /etc/logrotate.d/nginx
{{< / highlight >}}

To check for last run on logrotate, you can run
{{< highlight shell >}}
cat /var/lib/logrotate/status
{{< / highlight >}}

## Test logrotate with crontab
Logrotate allowing the service to run in hourly, daily, weekly or even monthly manner, which is corporated with crontab.
The specific time is defined in `/etc/crontab`.

{{< highlight shell >}}
$ cat /etc/crontab

57 * * * * root cd / && run-parts —report /etc/cron.hourly
25 6 * * * root test -x /usr/sbin/anacron || ( cd / && run-parts —report /etc/cron.daily )
47 6 * * 7 root test -x /usr/sbin/anacron || ( cd / && run-parts —report /etc/cron.weekly )
52 6 1 * * root test -x /usr/sbin/anacron || ( cd / && run-parts —report /etc/cron.monthly )
{{< / highlight >}}

For allowing logrotate run in hourly manner, you need to put the logrotate script in `/etc/cron.hourly`
{{< highlight shell >}}
mv /etc/cron.daily/logrotate /etc/cron.hourly/logrotate
{{< / highlight >}}



### References
[Digital Ocean: How To Manage Log Files With Logrotate On Ubuntu 12.10](https://www.digitalocean.com/community/tutorials/how-to-manage-log-files-with-logrotate-on-ubuntu-12-10)
