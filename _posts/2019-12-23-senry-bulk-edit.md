---
layout: post
categories: bash
title: "Edit all sentry projects with curl, jq and duct tape"
description: "Some memo about bulk interactions with Sentry API"
keywords: "bash, sentry"
---
## Script

```bash
#!/bin/bash

# get token from Sentry
TOKEN="XXXXXXXXXXXXXXXXXXXXXXXXXX"

SENTRY_ROOT=sentry.example.com
SENTRY_ORG=myorg

# easiest way to get payload - look into browser development tools and copy endpoints and jsons
DATA='{"allowedDomains":["*.example.work", "*.example.net", "*.example.io"]}'

# get all projects
APPS=$(curl -sb -H "Accept: application/json"  -H "Authorization: Bearer ${TOKEN}" https://${SENTRY_ROOT}/api/0/organizations/${SENTRY_ORG}/projects/ | jq -r '.[].slug')

# put settings to all projects
for APP in ${APPS}; do
  curl "https://${SENTRY_ROOT}/api/0/projects/${SENTRY_ORG}/${APP}/" -X PUT -H 'Content-Type: application/json' -H 'Pragma: no-cache' -H 'Cache-Control: no-cache' -H "Authorization: Bearer ${TOKEN}" --data ${DATA}
done
```
