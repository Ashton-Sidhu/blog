---
description: How to solve field [backend_url] not present as part of path [traefik.access.backend_url] error with Elastic Traefik integration
title: Using Elastic Traefik Integration with Traefik 2.x
toc: true 
categories: [traefik, elasticsearch, integration, homelab, backend_url, traefik.access.backend_url, ingest, pipeline]
image: ""
author: Ashton Sidhu
date: 2022-09-28
draft: true
---

As of writing this post, the Elastic Traefik integration currently only supports Traefik versions `1.x`. This post goes over the changes that need to be made to get the integration to work with Traefik versions `2.x`.

## Problem

The Traefik access log schema changed from version 1 to version 2. The 2 fields we care about to make the Traefik integration work are: `FrontendName` & `BackendAddr`. `FrontendName` is now `serviceName` and `BackendName` is now `serviceAddr` in Traefik `2.x`. When you try use the Elastic Traefik integration out of the box, you will get an error `field [backend_url] not present as part of path [traefik.access.backend_url]` in the transformed log.

## Solution

1. Navigate to the ingest pipelines page in Kibana.

2. Type in `logs-traefik.access` and select `logs-traefik.access-1.2.0-format-json`.

    a. Change the rename processor `Renames "temp.FrontendName" to "traefik.access.frontend_name"`.
         - Change the value in the `Field` box from `temp.FrontEndName` -> `temp.ServiceName`
    
    b. Change the rename processor `Renames "temp.BackendAddr" to "traefik.access.backend_url"`.
        - Change the value in the `Field` box from `temp.BackendAddr` to `temp.ServiceAddr`

3. Click `Save pipeline`

## What if you already ingested data?

When you make this change, any new logs will be processed by the updated ingest pipeline but the old logs already ingested will not.

I solved this by doing the following:

1. Delete the `logs-traefik.access-default` data stream from the Index Management page
2. Make a copy of your `access.log` so all the previous data gets reprocessed `cp access.log access.log.before_final_processor`

Since `access.log.before_final_processor` is considered a new log file, Elastic will reindex all the data in it with the updated processor and any new data will still flow in from `access.log`.

> Why not use the reindex API

I did try reindexing the data using the updated processor, but it didn't work. Reindexing the data takes the documents from one index and copies it to another. The problem that arises for us is that we want to reindex data from the log file, not from the source index which already contains transformed data. In our case, we would be sending already processed data through the updated pipeline that will throw errors in this case.
