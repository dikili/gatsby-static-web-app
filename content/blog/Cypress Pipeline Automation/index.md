---
title: Cypress Test Pipeline Automation Considerations
date: 2021-05-29 22:59:00 +0300
description: How to configure Cypress & Azure Devops CI/CD process # Add post description (optional)
img: ./cypress-cover.jpg # Add image post (optional)
tags: ['cypress', 'automation', 'ci/cd', 'pipeline'] # add tag
---

### Cypress Test Pipeline Automation

In a single test automation pipeline we can run cypress automation tests however doing this would violate the best practice CI/CD automation process because first of all what would this be categorised as , would this be a build pipeline or a release pipeline ? if this would be a build pipeline then running tests on the build pipeline does more than what build pipeline meant to do and on top of this, you would be running tests only on the master or main branch (this config could be changed but then this is extra work) Also as you know cypress has many files but to run the tests we only need few of those files only; namely any file under the cypress folder and any file with .json extension and if linting is used, need to get the lint files as well.

Another caveat is that; in a multiple environments (all environments would have multiple environments anyways), how do we run these tests against multiple environments if we use the build pipeline, this make it even more trickier.

Thus in ideal scenerio we should not run tests on the build pipeline but more like use it to satisfy pre-requisites and move files. These tasks would be Installs Node.js, runs npm ci, linting and once all good copy over the files for the test execution as below. To achieve even greater performance, suggestion would be to archive these files and then extract them before the test run step to help eliminate the pipeline agent heavy lifting with full sized files.

```
- task: CopyFiles@2
  displayName: 'copying only the files needed to run the cypress tests'
  timeoutInMinutes: 1
  inputs:
    SourceFolder: '$(cypress-test-folder)'
    Contents: |
      cypress/**
      *.json
      .eslint*
    TargetFolder: '$(cypress-test-folder)/files'
    CleanTargetFolder: true

```

### Build Pipeline Tasks

cypress-test.yml

```
 trigger:
 branches:
   include:
   - '*'

schedules:
 - cron: 05 9 * * 1-5
   displayName: 'Daily build (weekdays only)'
   always: true
   branches:
     include:
     - 'main'

pool:
 name: aws-linux

variables:
 cypress-test-folder: '.'
 cypress-test-archive-artifact: '$(Build.ArtifactStagingDirectory)/$(Build.Repository.Name).zip'

steps:
- task: NodeTool@0
 displayName: 'Install Node.js'
 timeoutInMinutes: 1
 inputs:
   versionSpec: '12.x'

- task: Npm@1
 displayName: 'npm ci'
 timeoutInMinutes: 1
 inputs:
   command: 'ci'
   workingDir: $(cypress-test-folder)

- task: Npm@1
 displayName: 'npm run lint'
 timeoutInMinutes: 1
 inputs:
   command: 'custom'
   workingDir: '$(cypress-test-folder)'
   customCommand: 'run lint -s --quiet'

- task: CopyFiles@2
 displayName: 'copying only the files needed to run the cypress tests'
 timeoutInMinutes: 1
 inputs:
   SourceFolder: '$(cypress-test-folder)'
   Contents: |
     cypress/**
     *.json
     .eslint*
   TargetFolder: '$(cypress-test-folder)/files'
   CleanTargetFolder: true

- task: ArchiveFiles@2
 displayName: 'zipping artifact files'
 inputs:
   rootFolderOrFile: '$(cypress-test-folder)/files'
   includeRootFolder: false
   archiveType: 'zip'
   archiveFile: '$(cypress-test-archive-artifact)'
   replaceExistingArchive: true

- task: PublishBuildArtifacts@1
 timeoutInMinutes: 1
 inputs:
   PathtoPublish: '$(cypress-test-archive-artifact)'
   ArtifactName: 'drop'
   publishLocation: 'Container'
```

And from this point on , after the installation and all these checkes, the files would sit under \$(cypress-test-folder) this defined folder and Release pipeline can pick the the files from this location and just operate on them.

### Release Pipeline Configuration

A new release pipeline needs to be created via 'New' in the release pipelines section. Add artifact should be the second stage then artifact config would look like this;

Sourcetype should be selected as Build.

Project -> {workspacename}
Source(build pipeline) -> {nameofthebuildpipeline} e.g search-team\test-recruiter-search-features

Default version -> Latest
Source alias -> \_cypress-tests

After artifact section is complete then Stages can be configured; Click 'Add Stage' and select empty option, then give it an environment name e.g 'Dev' and save.

After this say ok to the default path option and click the link that says 1 job, 0 task. 'Agent job' section needs to be configured, giving it any custom name, selecting the agent pool it belongs to should suffice here.

Task group should be added here but before it can me chosen following task group should be created in the task groups section on azure from the left pane.

This can be imported in the task groups section.

```

{"tasks":[{"environment":{},"displayName":"Are the files in an archive?","alwaysRun":false,"continueOnError":false,"condition":"succeeded()","enabled":true,"timeoutInMinutes":0,"inputs":{"targetType":"inline","filePath":"","arguments":"","script":"# Write your commands here\n\necho tests-are-in-archive? $(tests-are-in-archive)","workingDirectory":"","failOnStderr":"false","noProfile":"true","noRc":"true"},"task":{"id":"6c731c3c-3c68-459a-a5c9-bde6e6595b5b","versionSpec":"3.*","definitionType":"task"}},{"environment":{},"displayName":"Extract files ","alwaysRun":false,"continueOnError":false,"condition":"eq(variables['tests-are-in-archive'], true)","enabled":true,"timeoutInMinutes":0,"inputs":{"archiveFilePatterns":"**/*.zip","destinationFolder":"$(cypress-tests-folder)","cleanDestinationFolder":"false","overwriteExistingFiles":"false","pathToSevenZipTool":""},"task":{"id":"5e1e3830-fbfb-11e5-aab1-090c92bc4988","versionSpec":"1.*","definitionType":"task"}},{"environment":{},"displayName":"Use Node 12.x","alwaysRun":false,"continueOnError":false,"condition":"succeeded()","enabled":true,"timeoutInMinutes":0,"inputs":{"versionSpec":"12.x","checkLatest":"false","force32bit":"false"},"task":{"id":"31c75bbb-bcdf-4706-8d7c-4da6a1959bc2","versionSpec":"0.*","definitionType":"task"}},{"environment":{},"displayName":"Install Cypress Dependencies","alwaysRun":false,"continueOnError":false,"condition":"succeeded()","enabled":true,"timeoutInMinutes":0,"inputs":{"targetType":"inline","filePath":"","arguments":"","script":"echo\napt-get update\n\necho \"Installing missing cypress dependencies ...\"\napt-get install -y --no-install-recommends libgtk2.0-0 libgtk-3-0 libgbm-dev libnotify-dev libgconf-2-4 libnss3 libxss1 libasound2 libxtst6 xauth xvfb","workingDirectory":"","failOnStderr":"false","noProfile":"true","noRc":"true"},"task":{"id":"6c731c3c-3c68-459a-a5c9-bde6e6595b5b","versionSpec":"3.*","definitionType":"task"}},{"environment":{},"displayName":"npm ci","alwaysRun":false,"continueOnError":false,"condition":"succeeded()","enabled":true,"timeoutInMinutes":6,"inputs":{"command":"ci","workingDir":"$(cypress-tests-folder)","verbose":"false","customCommand":"","customRegistry":"useNpmrc","customFeed":"","customEndpoint":"","publishRegistry":"useExternalRegistry","publishFeed":"","publishPackageMetadata":"true","publishEndpoint":""},"task":{"id":"fe47e961-9fa8-4106-8639-368c022d43ad","versionSpec":"1.*","definitionType":"task"}},{"environment":{},"displayName":"Clean NPM cache on failure","alwaysRun":false,"continueOnError":false,"condition":"failed()","enabled":true,"timeoutInMinutes":0,"inputs":{"targetType":"inline","filePath":"","arguments":"","script":"echo\necho \"Clean NPM cache ...\"\nnpm cache clean --force\n\necho \necho \"Remove node modules ...\"\nrm -rf node_modules \n\necho \"##[warning]Cleaned NPM cache because of a previous failure\"\n","workingDirectory":"$(cypress-tests-folder)","failOnStderr":"false","noProfile":"true","noRc":"true"},"task":{"id":"6c731c3c-3c68-459a-a5c9-bde6e6595b5b","versionSpec":"3.*","definitionType":"task"}},{"environment":{},"displayName":"Cypress info","alwaysRun":false,"continueOnError":false,"condition":"succeeded()","enabled":true,"timeoutInMinutes":0,"inputs":{"command":"custom","workingDir":"$(cypress-tests-folder)","verbose":"false","customCommand":"run cy:info","customRegistry":"useNpmrc","customFeed":"","customEndpoint":"","publishRegistry":"useExternalRegistry","publishFeed":"","publishPackageMetadata":"true","publishEndpoint":""},"task":{"id":"fe47e961-9fa8-4106-8639-368c022d43ad","versionSpec":"1.*","definitionType":"task"}},{"environment":{},"displayName":"Swap variables","alwaysRun":false,"continueOnError":false,"condition":"succeeded()","enabled":true,"timeoutInMinutes":0,"inputs":{"sourcePath":"","filePattern":"$(cypress-tests-folder)/cypress.deployment.json","warningsAsErrors":"false","tokenRegex":"__(.{1,})__","secretTokens":""},"task":{"id":"9240b5c1-a1b2-4799-9325-e071c63236fb","versionSpec":"1.*","definitionType":"task"}},{"environment":{},"displayName":"Cypress verify","alwaysRun":false,"continueOnError":false,"condition":"succeeded()","enabled":true,"timeoutInMinutes":0,"inputs":{"command":"custom","workingDir":"$(cypress-tests-folder)","verbose":"false","customCommand":"run cy:verify","customRegistry":"useNpmrc","customFeed":"","customEndpoint":"","publishRegistry":"useExternalRegistry","publishFeed":"","publishPackageMetadata":"true","publishEndpoint":""},"task":{"id":"fe47e961-9fa8-4106-8639-368c022d43ad","versionSpec":"1.*","definitionType":"task"}},{"environment":{},"displayName":"run cypress tests","alwaysRun":false,"continueOnError":true,"condition":"succeeded()","enabled":true,"timeoutInMinutes":0,"inputs":{"command":"custom","workingDir":"$(cypress-tests-folder)","verbose":"false","customCommand":"run cy:run -- --config-file=cypress.deployment.json --config numTestsKeptInMemory=0   --config userAgent=\\\"Mozilla/5.0 (Macintosh; Intel Mac OS X x.y; rv:42.0) Gecko/20100101 Firefox/42.0\\\"","customRegistry":"useNpmrc","customFeed":"","customEndpoint":"","publishRegistry":"useExternalRegistry","publishFeed":"","publishPackageMetadata":"true","publishEndpoint":""},"task":{"id":"fe47e961-9fa8-4106-8639-368c022d43ad","versionSpec":"1.*","definitionType":"task"}},{"environment":{},"displayName":"Publish Test Results **/test-output-*.xml","alwaysRun":false,"continueOnError":false,"condition":"succeededOrFailed()","enabled":true,"timeoutInMinutes":0,"inputs":{"testRunner":"JUnit","testResultsFiles":"**/test-output-*.xml","searchFolder":"$(System.DefaultWorkingDirectory)","mergeTestResults":"true","failTaskOnFailedTests":"false","testRunTitle":"$(test-run-title)","platform":"","configuration":"","publishRunAttachments":"true"},"task":{"id":"0b0f01ed-7dde-43ff-9cbb-e48954daf9b1","versionSpec":"2.*","definitionType":"task"}}],"runsOn":["Agent","DeploymentGroup"],"revision":67,"createdBy":{"displayName":"Dikili Dikili","id":"26e2df61-fcea-6bd5-b74a-02d2c4098504","uniqueName":"Dikili.Dikili@companyonline.co.uk"},"createdOn":"2020-05-19T14:21:48.890Z","modifiedBy":{"displayName":"Dikili Dikili","id":"26e2df61-fcea-6bd5-b74a-02d2c4098504","uniqueName":"Dikili.Dikili@companyonline.co.uk"},"modifiedOn":"2021-05-20T15:15:54.380Z","comment":"added more information about the archive flag","id":"b231d178-99be-4814-bef4-63bcc4cc52bf","name":"run-cypress-tests","version":{"major":1,"minor":0,"patch":0,"isTest":false},"iconUrl":"https://cdn.vsassets.io/v/M168_20200513.4/_content/icon-meta-task.png","friendlyName":"run-cypress-tests","description":"Run cypress tests\n\nRequirements: cypress.deployment.json in the repo and junit reporter configured in `cypress.deployment.json` so you can see the results in deployment output.","category":"Test","definitionType":"metaTask","author":"Dikili Dikili","demands":[],"groups":[],"inputs":[{"aliases":[],"options":{},"properties":{},"name":"cypress-tests-folder","label":"cypress-tests-folder","defaultValue":"","required":true,"type":"string","helpMarkDown":"where the root of your cypress tests are","groupName":""},{"aliases":[],"options":{},"properties":{},"name":"test-run-title","label":"test-run-title","defaultValue":"$(test-run-title) - $(Release.EnvironmentName)","required":true,"type":"string","helpMarkDown":"Provide a name for the Test Run.","groupName":""},{"aliases":[],"options":{},"properties":{},"name":"tests-are-in-archive","label":"tests-are-in-archive","defaultValue":"false","required":true,"type":"string","helpMarkDown":"Are the cypress tests in a zip archive? set true to extract files","groupName":""}],"satisfies":[],"sourceDefinitions":[],"dataSourceBindings":[],"instanceNameFormat":"Task group: run-cypress-tests $(cypress-tests-folder)","preJobExecution":{},"execution":{},"postJobExecution":{}}

```

it would basically create a set of tasks once imported that enables tests to run but before that to extract archived files as well as to swapping of variables from the cyress.deployment.json file that needs to be created for all the environments and all variables will need to follow **variable1** syntax in order to be swapped with that stage's value.

When we come to the Task group's configuration, it should look like below;

Display name -> Task group: run-cypress-tests \$(System.DefaultWorkingDirectory)/\_cypress-tests/drop

cypress-test-folder -> \$(System.DefaultWorkingDirectory)/\_cypress-tests/drop

test-run-title -> $(test-run-title) - $(Release.EnvironmentName)

tests-are-in-archive -> true

In the pipeline variables section, test-run-title variable should be defined with the name of the repository.

Defining variables that will be potentially used by other pipelines can go into as 'Variable group' . Variable groups are created in the library section of the Azure devops on the left pane section under Pipelines heading. And the config would look like below;

Variable group name -> test-recruiter-search-features DEV

Description -> give it a sensible description

Variables

Name -> variable1 (without the \_\_ as these will be swapped with this value now)
Value -> e.g 536

Meaning variable1 will be populated with 536 for the DEV stage.

Next thing to do is to go back to where the release pipeline stage is created and associating the variable group with the release pipeline's variable groups section and then assiging that variable group a scope ,this scope will match to the name of the stage created when we first created the release pipeline, thus will know that when executing the tasks for that stage , this is the specific variable group it needs to use.

This way we completed both of the Build pipeline and the Release pipeline section of CI/CD process and our cypress test should be automatically triggered from the defined branch.
