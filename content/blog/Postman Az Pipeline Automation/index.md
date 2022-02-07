---
title: Automating Postman in Az pipeline
date: 2022-02-07 19:40:00 +0300
description: # Add post description (optional)
img: ./postman.png # Add image post (optional)
tags: [] # add tag
---

Following steps are required to be able to automation postman tests;

1) Ensure there are tests inside a specific postman collection!
2) Create a release pipeline with following steps;

### Clean NPM Cache
This is a Bash scripting step to clean the cache;

```
//clean cache
npm cache clean --force
//remove node modules
rm -rf node_modules
//you can alternatively delete package-lock.json
rm -rf package-lock.json
```
### Install Newman
This is the useful tool that I have referred to in my initial statement

Add an NPM package runner
Display name: Install Newman
Command: custom
Command and arguments : install -g newman

### Install Newman Reporter (newman-reporter-htmlextra)

Add an NPM package runner

Display name: same as the title of this heading
Command: custom
Command and arguments: install -g newman-reporter-htmlextra

### Run Postman tests via Newman
task: carlowahlstedt.NewmanPostman.NewmanPostman@4
Display name: same as the title of this heading
Collection source type: https://api.getpostman.com/collections/$(postman_collection_id)?apikey=$(postman-api-key)

Environment Url: https://api.getpostman.com/environments/$(postman_environment)?apikey=$(postman-api-key)

Reporters:cli, htmlextra, junit
Enable HTML extra dark theme
Enable HTML extra console logs

Html Extra Report Title: Postman e2e tests

### Publish test results

Add a publish test results step

Display name: Publish test results
Test result format: JUnit
Test result files: **/newman-*.xml
Search folder: $(System.DefaultWorkingDirectory)
