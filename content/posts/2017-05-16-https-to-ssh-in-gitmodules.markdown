---
layout: post
slug: https-to-ssh-in-gitmodules
title: "One-liner to replace HTTPS into SSH url in .gitmodules"
date: 2017-05-16 23:57:00 +0800
comments: true
tags: [unix, perl, sed, git, submodules]
---
While pulling [submodule](https://git-scm.com/docs/git-submodule) repositories in CI servers like Jenkins/CircleCI, repositories with HTTPS submodule url will ask for credentials, which is hard for automation.

_Demo of `.gitmodules`_
{{< highlight shell >}}
$ cat .gitmodules
[submodule "common"]
    path = common
    url = https://github.com/dhoeric/common.git
{{< / highlight >}}

After you trying to pull submodule repo, you will be asked for password...
{{< highlight shell >}}
$ git submodule sync
$ git submodule update --init
Submodule 'common' (https://github.com/dhoeric/common.git) registered for path 'common'
Cloning into '~/r/hello-submodule/common'...
Username for 'https://github.com':
{{< / highlight >}}

**To change all HTTPS url into SSH, run the following command**
{{< highlight shell >}}
perl -i -p -e 's|https://(.*?)/|git@\1:|g' .gitmodules
{{< / highlight >}}


The output of `.gitmodules`
{{< highlight shell >}}
$ cat .gitmodules
[submodule "common"]
    path = common
    url = git@github.com:dhoeric/common.git
{{< / highlight >}}
