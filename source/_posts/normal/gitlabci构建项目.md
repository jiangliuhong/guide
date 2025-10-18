---
title: gitlabci项目集成
date: 2019-03-26 00:17:00:00
categories: 随笔
tags:
   - git
---

# gitlabci项目集成

```
Running with gitlab-runner 11.3.1 (0aa5179e)
  on docker-ci 0f9fe2c4
Using Docker executor with image node:latest ...
ERROR: Preparation failed: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running? (executor_docker.go:1150:0s)
```

```
sudo docker run -d --name gitlab-runner --restart always \
 -v ~/srv/gitlab-runner/config:/etc/gitlab-runner \
 -v ~/var/run/docker.sock:/var/run/docker.sock \
 gitlab/gitlab-runner:latest
```

