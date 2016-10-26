---
layout: post
slug: useful-docker-cleanup
title: Some useful Docker cleanup one-liners
date: 2016-10-27
---

Here's a couple of simple one liners for Docker housekeeping:

Stop all running docker containers:

```bash
docker ps | awk 'NR > 1 {print "docker stop " $1}' | xargs -0 bash -c
```

Remove all stopped containers:

```bash
docker ps -a | awk 'NR > 1 {print "docker rm " $1}' | xargs -0 bash -c
```


