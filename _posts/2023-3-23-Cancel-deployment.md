---
layout: post
title: Solving "Deployment request failed due to in progress deployment."
excerpt_separator: <!--more-->
---

Short guide to solve this problem using Github API

<!--more-->


When uploading the previous posts I executed two commits at almost the same time so the deployments failed. From then, all deploys failed:

![img](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/github_fail/0.png)

Checking the details in any of them I found this error:

```
Error: HttpError: Deployment request failed for X due to in progress deployment. Please cancel bcf64b6d972791062f800756412b6795cc892153 first or wait for it to complete.
```

![img](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/github_fail/1.png)

So it seems the problem is the commit "bcf64b6d972791062f800756412b6795cc892153" or "bcf64b6".

Using Github API you can list deployments with:

```
curl -L \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer <TOKEN>"\
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/repos/ricardojoserf/ricardojoserf.github.io/deployments
```

![img](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/github_fail/2.png)

From this I got the deployment id "839873849", so the url is "https://api.github.com/repos/ricardojoserf/ricardojoserf.github.io/deployments/839873849" and deleted it with:

```
curl -L \
  -X DELETE \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer <TOKEN>"\
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/repos/ricardojoserf/ricardojoserf.github.io/deployments/839873849
```

It seems it is solved, changing anything in any file triggers a new successful deploy:

![img](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/github_fail/3.png)

Sources:
- [https://github.com/actions/deploy-pages/issues/22](https://github.com/actions/deploy-pages/issues/22)
- [https://docs.github.com/en/rest/deployments/deployments?apiVersion=2022-11-28#delete-a-deployment](https://docs.github.com/en/rest/deployments/deployments?apiVersion=2022-11-28#delete-a-deployment)

