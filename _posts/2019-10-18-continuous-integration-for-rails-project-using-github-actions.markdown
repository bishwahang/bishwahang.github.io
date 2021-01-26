---
layout: post
title:  "Continuous Integration for Rails project using Github Actions"
date:   2019-10-18 11:00:00 +0100
author: bishwa
categories: programming
---

## Intro

With the release of [Github Actions](https://github.com/features/actions), we experimented with it to replace our current Continuous Integration (CI) process.
This post describes the steps we took in order to do so.

Our CI includes of 3 checks:

1. specs
2. rubocop check
3. linter check for swagger docs

These checks run every time a developer pushes a commit or creates a Pull Requests (PR). The aim of this post is to give detailed description of the process to perform those jobs with Github Actions.

## Defining a Workflow
Let's start with creating a new workflow in Github Actions that will perform those tasks.
In your root rails project:
```bash
mkdir -p .github/workflows
touch .github/workflows/main.yml
```
The `main.yml` file is where we define our workflow for CI. We name our workflow `CI`, and then list the name of events which will trigger our workflow.
```yaml
name: CI
on: [push, pull_request]
```
We then describe the jobs that we want to run in the workflow. We can describe more than one job and each of them runs in parallel. Here we describe 3 different jobs for the aforementioned 3 checks.
```yaml
name: CI
on: [push, pull_request]
jobs:
  specs:
  rubocop:
  swagger:
```


## Defining a Job

Every job needs an instance of virtual host machine to run on. This is specified by statement `runs-on`.
We can then define the sequential tasks that we want to perform inside this machine with `steps` statement.

```yaml
name: CI
on: [push, pull_request]
jobs:
  specs:
    name: specs
    runs-on: ubuntu-latest
    steps:
    - name Install Libraries
      run: |
        sudo apt-get update
        sudo apt-get install -y postgresql-client libpq-dev
```

In the above workflow we defined a job named `specs`, that runs on `ubuntu-latest` instance. Inside the instance it installs package `postgresql-client` and `libpq-dev`. This is required for configuring the postgres database required for specs.

## Services

`Services` can be used to create additional containers for a job or steps. In our case, we are using it to spin off `postgres` service.
```yaml
name: CI
on: [push, pull_request]
jobs:
  specs:
    name: specs
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:10.8
        ports:
        - 5432:5432
    steps:
    - name Install Libraries
      run: |
        sudo apt-get update
        sudo apt-get install -y postgresql-client libpq-dev
```
Once we have our `postgres` service up and running, and `postgres` client installed, we will create test database.
```yaml
name: CI
on: [push, pull_request]
jobs:
  specs:
    name: specs
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:10.8
        ports:
        - 5432:5432
    steps:
    - name Install Libraries
      run: |
        sudo apt-get update
        sudo apt-get install -y postgresql-client libpq-dev
    - name: Configure databases
      run: |
        echo "Postgres"
        psql -h localhost -c 'create database "test-database";' -U postgres
```

## Actions by Github

Next thing we will do is checkout our rails app code.
Github provides many official, ready to use actions, under its verified [account](https://github.com/actions), and one of those is `actions/checkout`. We can use this actions directly with the statement `uses`.
Similarly, another actions `actions/setup-ruby` is used to checkout the ruby version that we want to use for our rails app.

```yaml
name: CI
on: [push, pull_request]
jobs:
  specs:
    name: specs
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:10.8
        ports:
        - 5432:5432
    steps:
    - name Install Libraries
      run: |
        sudo apt-get update
        sudo apt-get install -y postgresql-client libpq-dev
    - name: Configure databases
      run: |
        echo "Postgres"
        psql -h localhost -c 'create database "test-database";' -U postgres
    - name: Checkout code
      uses: actions/checkout@v1

    - name: Set up Ruby
      uses: actions/setup-ruby@v1
      with:
        ruby-version: 2.6.3
```

Once the code has been checked out and correct ruby version is setup, we install the gems using `bundler` and then run the `specs`.

```yaml
name: CI
on: [push, pull_request]
jobs:
  specs:
    name: specs
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:10.8
        ports:
        - 5432:5432
        env:
          POSTGRES_PASSWORD: ""
    steps:
    - name: Install libraries
      run: |
        sudo apt-get update
        sudo apt-get install -y postgresql-client libpq-dev
    - name: Configure databases
      run: |
        echo "Postgres"
        psql -h localhost -c 'create database "test-database";' -U postgres
    - uses: actions/checkout@v1

    - name: Set up Ruby
      uses: actions/setup-ruby@v1
      with:
        ruby-version: 2.6.5

    - name: Install bundler and gems
      run: |
        gem install bundler --no-document
        bundle config GITHUB__COM $GITHUB_ACCESS_TOKEN
        bundle install --jobs 4 --retry 3
      env:
        GITHUB_ACCESS_TOKEN: {% raw %}${{ secrets.GITHUB_ACCESS_TOKEN }} {% endraw %}

    - name: Run tests
      run: |
        pg_config --version
        bin/rails db:schema:load RAILS_ENV=test
        bin/rspec
```

## Secrets and Environment Variables

One interesting feature in the above `yaml` is the use of `secrets`. We can define the secrets in the settings section of our Github repository. More about it can be read [here](https://help.github.com/en/articles/virtual-environments-for-github-actions#creating-and-using-secrets-encrypted-variables).
The secrets defined in the Github repo, in our case, `GITHUB_ACCESS_TOKEN`, can be accessed inside our job instance. We used it to setup environment variable, which is passed into the `step` container to install gems.
```yaml
env:
  GITHUB_ACCESS_TOKEN: {% raw %}${{ secrets.GITHUB_ACCESS_TOKEN }} {% endraw %}
```

Environment variables can be set for each individual steps. Github also sets many environmental variables by default, which are accessible in every step. See [here](https://help.github.com/en/articles/virtual-environments-for-github-actions#environment-variables).

We follow the similar structure to run our other two jobs `rubocop` and `swagger`.

## Slack Notification

There was one last function missing, slack notification. We wanted to be notified in the slack if our CI passed or failed.
In order to notify when all the CI checks were success, we made use of workflow syntax `needs`. This statement defines the list of jobs that needs to be completed successfully before the described job will run.
We used action [*rtCamp/actions-slack-notify*](https://github.com/rtCamp/action-slack-notify/), in order to notify the slack webhook.
```yaml
slack_success:
    name: slack
    runs-on: ubuntu-latest
    needs: [swagger, rubocop, specs]
    steps:
      - uses: actions/checkout@v1
      - name: Success Notify
        uses: rtCamp/action-slack-notify@master
        env:
          SLACK_USERNAME: "Action Bot"
          SLACK_ICON: "https://github.com/freeletics.png"
          SLACK_TITLE: "Success"
          SLACK_MESSAGE: "All checks passed :white_check_mark:"
          SLACK_WEBHOOK: {% raw %}${{ secrets.SLACK_WEBHOOK }} {% endraw %}
```

This job now notified us whenever all the CI checks passed. The remaining part for this section was notification if any one of the 3 CI checks fails.
We used `if` syntax from the workflow to achieve this action. The `if` statement has higher precedence than `needs` statement. The *slack_failure* job is made dependent on the *slack_success* job. And, the *slack_failure* job will only started if the `slack_success` job had failed.
The failure of a job or previous step action can be checked using statements `failed()` or `cancelled()` expression.
```yaml
slack_failure:
    name: slack
    runs-on: ubuntu-latest
    needs: [slack_success]
    if: failure() || cancelled()
    steps:
      - uses: actions/checkout@v1
      - name: Success Notify
        uses: rtCamp/action-slack-notify@master
        env:
          SLACK_USERNAME: "Action Bot"
          SLACK_ICON: "https://github.com/freeletics.png"
          SLACK_COLOR: "#FF0000"
          SLACK_TITLE: "Failure"
          SLACK_MESSAGE: "Some checks failed :x:"
          SLACK_WEBHOOK: {% raw %}${{ secrets.SLACK_WEBHOOK }} {% endraw %}
```


## Outro

The complete workflow code for the CI is shown at the bottom of [post](#complete-workflow-yaml)

With this we could replicate our current CI implementation completely using Github Actions. However, we haven't yet started using it for our daily development.
Some of the reasons are:
1. The support for latest ruby is late. We are already using **ruby 2.6.5** for our apps, but `actions/setup-ruby` is yet to support it. There has been discussion about open sourcing the process of adding newer versions of ruby. [Comment](https://github.com/actions/setup-ruby/issues/8#issuecomment-522084301).
2. The gems are not cached between builds. For every build, `bundler` needs to install the gems fresh, and it takes quite a time. There has also been discussion about it and confirmation that Github Actions is working on adding caching. [Comment](https://github.com/rails/rails/pull/36893#discussion_r326103547)


### *References:*

1. [Workflow syntax for GitHub Actions](https://help.github.com/en/articles/workflow-syntax-for-github-actions)
2. [Contexts and expression syntax for GitHub](https://help.github.com/en/articles/contexts-and-expression-syntax-for-github-actions)


### Complete Workflow Yaml

```yaml
name: CI
on: [push, pull_request]
jobs:
  specs:
    name: specs
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:10.8
        ports:
        - 5432:5432
        env:
          POSTGRES_PASSWORD: ""
    steps:
    - name: Install libraries
      run: |
        sudo apt-get update
        sudo apt-get install -y postgresql-client libpq-dev

    - name: Configure databases
      run: |
        echo "Postgres"
        psql -h localhost -c 'create database "test-database";' -U postgres

    - uses: actions/checkout@v1

    - name: Set up Ruby
      uses: actions/setup-ruby@v1
      with:
        ruby-version: 2.6.5

    - name: Install bundler and gems
      run: |
        gem install bundler --no-document
        bundle config GITHUB__COM $GITHUB_ACCESS_TOKEN
        bundle install --jobs 4 --retry 3
      env:
        GITHUB_ACCESS_TOKEN: ${{ secrets.GITHUB_ACCESS_TOKEN}}

    - name: Run tests
      run: |
        pg_config --version
        bin/rails db:schema:load RAILS_ENV=test
        bin/rspec
  rubocop:
    name: rubocop
    runs-on: ubuntu-latest
    steps:
      - name: Install libraries
        run: |
          sudo apt-get update
          sudo apt-get install -y libpq-dev
      - name: Set up Ruby
        uses: actions/setup-ruby@v1
        with:
          ruby-version: 2.6.5

      - uses: actions/checkout@v1

      - name: Install bundler and rubocop
        run: |
          gem install bundler --no-document
          bundle config GITHUB__COM $GITHUB_ACCESS_TOKEN
          bundle install --jobs 4 --retry 3 --with=test
        env:
          GITHUB_ACCESS_TOKEN: ${{ secrets.GITHUB_ACCESS_TOKEN}}

      - name: Run rubocop checks
        run: bundle exec rubocop -D -c .rubocop.yml

  swagger:
    name: swagger
    runs-on: ubuntu-latest
    steps:
      - name: Install Node
        uses: actions/setup-node@v1
        with:
          node-version: 10.9.0

      - name: Install swagger cli
        run: |
          npm install -g swagger-cli

      - uses: actions/checkout@v1

      - name: Run swagger linter check
        run: |
          for f in doc/freeletics_api_v{1,2,3}.yml; do swagger-cli validate $f || break 0; done

  slack_success:
    name: slack
    runs-on: ubuntu-latest
    needs: [swagger, rubocop, specs]
    steps:
      - uses: actions/checkout@v1
      - name: Success Notify
        uses: rtCamp/action-slack-notify@master
        env:
          SLACK_USERNAME: "Action Bot"
          SLACK_ICON: "https://github.com/freeletics.png"
          SLACK_TITLE: "Success"
          SLACK_MESSAGE: "All checks passed :white_check_mark:"
          SLACK_WEBHOOK: {% raw %}${{ secrets.SLACK_WEBHOOK }} {% endraw %}

  slack_failure:
    name: slack
    runs-on: ubuntu-latest
    needs: [slack_success]
    if: failure() || cancelled()
    steps:
      - uses: actions/checkout@v1
      - name: Success Notify
        uses: rtCamp/action-slack-notify@master
        env:
          SLACK_USERNAME: "Action Bot"
          SLACK_ICON: "https://github.com/freeletics.png"
          SLACK_COLOR: "#FF0000"
          SLACK_TITLE: "Failure"
          SLACK_MESSAGE: "Some checks failed :x:"
          SLACK_WEBHOOK: {% raw %}${{ secrets.SLACK_WEBHOOK }} {% endraw %}
```
