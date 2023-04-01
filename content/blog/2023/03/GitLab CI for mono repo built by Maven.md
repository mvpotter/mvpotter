+++
title = "GitLab CI for mono repo built by Maven"
date = "2023-03-12T14:10:03+07:00"
tags = ["GitLab", "CI", "Maven", "Monorepo"]
categories = []
description = ""
menu = ""
banner = ""
images = []
draft = false
+++

A couple of years ago we have decomposed one of our projects to a number of Microservices. For the sake of simplicity we continued to keep sources in a single repository. On every commit appropriate maven goal was launched for parent project and it delegated the task to its children. At the very beginning services were quite small and did not required much time to compile and execute tests. However, through the time more logic and tests were implemented. Some services started to require more time to be built and the question was raised how to reduce the build time to get faster feedback from CI?

At first we thought about moving each service to a dedicated repository, however we did not wanted to loose advantages of monorepo. Thus, we started to investigate GitLab CI features to figure out what can help us. Fortunately, GitLab CI allows to launch pipelines only for dedicated modules, allowing to build service only if changes were made to its sources. With the help of GitLab CI [include](https://docs.gitlab.com/ee/ci/yaml/#include), [rules](https://docs.gitlab.com/ee/ci/yaml/#rules) and [variables](https://docs.gitlab.com/ee/ci/variables/) we managed to improve out pipelines. And I wanted to share the configuration we came to.

There is a [sample project](https://gitlab.com/mvpotter/gitlab-ci-maven-monorepo) with everything I am going to mention in the article. So you can check the code right away.

![GitLab CI Pipeline](https://miro.medium.com/v2/resize:fit:1400/1*cQ9Bt8cru89vADvF2D_ySg.png)

Let’s start with some basic rules for building a microservice (*/gitlab/actions/.gitlab-ci-basic-rules.yml*):

```yaml
.basic-rules:  
  rules:  
    - if: $CI_PIPELINE_SOURCE != "push"  
      when: never  
    - if: $CI_COMMIT_BRANCH  
      changes:  
        - $MODULE/**/*.{$EXTENSIONS_TO_CHECK}  
        - "pom.xml"  
      when: always  
    - when: manual  
      allow_failure: true
```

We launch jobs on every push to the repository. Thus, if [*$CI_PIPELINE_SOURCE*](https://docs.gitlab.com/ee/ci/jobs/job_control.html#common-if-clauses-for-rules) has different value, we just skip the job.

The next condition is *\$CI_COMMIT_BRANCH* check. If variable is set, means the push was made to a branch. It is possible to add condition for specific branch, e.g. *\$CI_COMMIT_BRANCH == “develop”*. Or to define custom behaviour on a tag creation with *\$CI_COMMIT_TAG*.

Moreover, we define that we want to launch the job only in case one of the following changed:

-   Files in path *\$MODULE/&ast;&ast;/&ast;.{\$EXTENSIONS_TO_CHECK}*. *\$MODULE* is a variable with service module name and *EXTENSIONS_TO_CHECK* is a list of files extensions that should be considered. E.g. most probably there is no need to rebuild a module if README.md only was changed
-   Parent *pom.xml*. We have common libraries declarations parent pom. Thus, if something changes there, it is necessary to rebuild all the services. But, it is project specific thing. Service modules might be completely independent from each other and ignore changes in the parent pom.

The last condition states that if a job is not launched automatically, it would still be displayed on the pipeline and could be lunched manually if necessary. *allow_failure: true* used not to block dependent job execution. E.g. it there were no changes in services files, but we still want to be able to deploy them from the pipeline without launching build manually.

Then we can define build action (*/gitlab/actions/.gitlab-ci-build.yml*):

```yaml
include:  
  - local: /gitlab/actions/.gitlab-ci-basic-rules.yml  
  
.build-action:  
  extends: .basic-rules  
  stage: build  
  image: maven:3.9.0-eclipse-temurin-17-alpine  
  script:  
    - mvn $MAVEN_OPTS -T 1C -f $MODULE clean install
```

We reuse basic rules by extending appropriate block. And as a script we just run *mvn clean inslall -f* option launches specific maven module and all its submodules.

For each service we create own configuration w required actions, e.g. for service **service-one** we have the following config (*/gitlab/modules/.gitlab-ci-service-one.yml*):

```yaml
include:  
  - local: /gitlab/actions/.gitlab-ci-build.yml  
  
.service-one-module:  
  variables:  
    MODULE: "service-one"  
  
build-service-one:  
  extends:  
    - .build-action  
    - .service-one-module
```

Here we define *MODULE* variable used in the build action script and extend both blocks for *build-service-one* job. So we have a job that would build our module automatically in case any file of appropriate extension was changed there would be an option to launch i manually otherwise.

We have deployment script that updates only services that were changed from the last deployment. Thus, we configure jobs that allows to deploy solution to the target platform. A simple example of such job can be found in */gitlab/environments/.gitlab-ci-dev-env.yml*:

```yaml
include:  
  - local: /gitlab/actions/.gitlab-ci-deploy.yml  
  
.dev-env:  
  before_script:  
    - export PROJECT_TAG=$(grep -m 1 '<version>' pom.xml | awk -F '>' '{ print $2 }' | awk -F '<' '{ print $1 }')  
    - export DOCKER_TAG=$(echo "$CI_COMMIT_REF_NAME"|tr "/" "_")-$PROJECT_TAG  
  variables:  
    PROFILE: "dev"  
  environment:  
    name: dev  
    url: https://dev.mvpotter.com  
  
deploy-dev:  
  extends:  
    - .deploy-action  
    - .dev-env  
  only:  
    - develop  
  when: manual
```

Deploy script is dummy for the sake of simplicity. Here, we extract project version from a *pom.xml* to calculate appropriate docker image tag. And define *deploy-dev* job that is available for *develop* branch only.

And the root *.gitlab-ci.yml* script looks the following way:

```yaml
image: ruby:alpine3.17  
  
variables:  
  EXTENSIONS_TO_CHECK: xml,java,yml,sql,js,json,html,png,css,scss  
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"  
  
cache:  
  paths:  
    - .m2/repository/  
  
stages:  
  - build  
  - deploy  
  
include:  
  - local: /gitlab/modules/.gitlab-ci-service-one.yml  
  - local: /gitlab/modules/.gitlab-ci-service-two.yml  
  - local: /gitlab/environments/.gitlab-ci-dev-env.yml  
  - local: /gitlab/environments/.gitlab-ci-prod-env.yml
```

Here we just define *EXTENSIONS_TO_CHECK* variable. It can be overriden for each service if necessary. And include all our models and environments configurations.

It is great that GitLab CI has so flexible configuration. However, there are a number of limitations for the moment.

1.  Every service is rebuilt if you create a new branch. It is possible to define [*compare_to*](https://docs.gitlab.com/ee/ci/jobs/job_control.html#skip-job-if-the-branch-is-empty) property. Unfortunately, it does not support variables for the moment. In our project we have a number of base branches that can be used for features, so we cannot define just a single one there.
2.  No transparent integration with GitLab pages. For example, we had a page where test coverage report was deployed. It worked when we built the whole project. However, now if only subset of services is rebuilt you cannot longer see results for the other ones that were build earlier as they are rewritten.

Hope that the article helped you. If you have ideas how to improve the configuration or how to deal with current limitations please feel free to share them in comments.