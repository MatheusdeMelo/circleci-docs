---
layout: classic-docs
title: "Pipeline Variables"
short-title: "Pipeline Variables"
description: "Detailed information about Pipeline variables, parameters and values"
categories: [getting-started]
order: 1
version:
- Cloud
---

Pipeline variables can be used to create reusable pipeline configurations. To use pipeline variables you must have [pipelines enabled]({{ site.baseurl }}/2.0/build-processing) and use configuration [version]({{ site.baseurl }}/2.0/configuration-reference/#version) `2.1` or higher.

There are two types of pipeline variables:

* **Pipeline values** represent pipeline metadata that can be used throughout the configuration.
* **Pipeline parameters** are typed pipeline variables that are declared in the `parameters` key at the top level of a configuration. Users can pass `parameters` into their pipelines when triggering a new run of a pipeline through the API.

## Pipeline values
{: #pipeline-values }

Pipeline values are available to all pipeline configurations and can be used without previous declaration.

Value                       | Description
----------------------------|--------------------------------------------------------
pipeline.id                 | A [globally unique id](https://en.wikipedia.org/wiki/Universally_unique_identifier) representing for the pipeline
pipeline.number             | A project unique integer id for the pipeline
pipeline.project.git_url    | The URL where the current project is hosted. E.g. https://github.com/circleci/circleci-docs
pipeline.project.type       | The lower-case name of the VCS provider, E.g. “github”, “bitbucket”.
pipeline.git.tag            | The name of the git tag that was pushed to trigger the pipeline. If the pipeline was not triggered by a tag, then this is the empty string.
pipeline.git.branch         | The name of the git branch that was pushed to trigger the pipeline.
pipeline.git.revision       | The long (40-character) git SHA that is being built.
pipeline.git.base_revision  | The long (40-character) git SHA of the build prior to the one being built. <br/><br/> **Note:** While in most cases `pipeline.git.base_revision` will be the SHA of the pipeline that ran before your currently running pipeline, there are some caveats. When the build is the first build for a branch, the variable will not be present. In addition, if the build was triggered via the API, the variable will not be present.
pipeline.in_setup           | True if the pipeline is in the setup phase, i.e. running a [setup workflow]({{ site.baseurl }}/2.0/dynamic-config/).
pipeline.trigger_source     | The source that triggers the pipeline, current values are `webhook`, `api`, `scheduled_pipeline`
pipeline.schedule.name      | The name of the schedule if it is a scheduled pipeline. Value will be empty string if the pipeline is triggerd by other sources
pipeline.schedule.id        | The unique id of the schedule if it is a scheduled pipeline. Value will be empty string if the pipeline is triggerd by other sources
{: class="table table-striped"}

For example:

```yaml
version: 2.1

jobs:
  build:
    docker:
      - image: circleci/node:latest
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context / project UI env-var reference
    environment:
      CIRCLE_COMPARE_URL: << pipeline.project.git_url >>/compare/<< pipeline.git.base_revision >>..<<pipeline.git.revision>>
    working_directory: ~/main
    steps:
      - run: echo "This is pipeline ID << pipeline.id >>"
      - run: echo $CIRCLE_COMPARE_URL
```

**Note:** When using the above method to set the variables in the `environment` key if the pipeline variable is empty it will set the variable to `<nil>`. If you need an empty string instead, [set the variable in a shell command]({{ site.baseurl }}/2.0/env-vars/#setting-an-environment-variable-in-a-shell-command).

## Pipeline parameters in configuration
{: #pipeline-parameters-in-configuration }

Pipeline parameters are declared using the `parameters` key at the top level of a `.circleci/config.yml` file.

Pipeline parameters support the following types:
* string
* boolean
* integer
* enum

See [Parameter Syntax]({{ site.baseurl }}/2.0/reusing-config/#parameter-syntax) for usage details.

Pipeline parameters can be referenced by value and used as a config variable under the scope `pipeline.parameters`.

The example below shows a configuration with two pipeline parameters (`image-tag` and `workingdir`) defined at the top of the config, and then subsequently referenced in the `build` job:

```yaml
version: 2.1
parameters:
  image-tag:
    type: string
    default: "latest"
  workingdir:
    type: string
    default: "~/main"

jobs:
  build:
    docker:
      - image: circleci/node:<< pipeline.parameters.image-tag >>
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context / project UI env-var reference
    environment:
      IMAGETAG: << pipeline.parameters.image-tag >>
    working_directory: << pipeline.parameters.workingdir >>
    steps:
      - run: echo "Image tag used was ${IMAGETAG}"
      - run: echo "$(pwd) == << pipeline.parameters.workingdir >>"
```

### Passing parameters when triggering pipelines via the API
{: #passing-parameters-when-triggering-pipelines-via-the-api }

A pipeline can be triggered with specific `parameter` values using the API v2 endpoint to [trigger a pipeline](https://circleci.com/docs/api/v2/#trigger-a-new-pipeline). This can be done by passing a `parameters` key in the JSON packet of the `POST` body.

**Note:** Please note that the `parameters` key passed in this `POST` request is **NOT** secret.

The example below triggers a pipeline with the parameters described in the above config example (NOTE: To pass a parameter when triggering a pipeline via the API the parameter must be declared in the configuration file.).

```
curl -u ${CIRCLECI_TOKEN}: -X POST --header "Content-Type: application/json" -d '{
  "parameters": {
    "workingdir": "./myspecialdir",
    "image-tag": "4.8.2"
  }
}' https://circleci.com/api/v2/project/:project_slug/pipeline
```

## The scope of pipeline parameters
{: #the-scope-of-pipeline-parameters }

Pipeline parameters can only be resolved in the `.circleci/config.yml` file in which they are declared. Pipeline parameters are not available in orbs, including orbs declared locally in your config.yml file. This was done because access to the pipeline scope in orbs would break encapsulation and create a hard dependency between the orb and the calling config, potentially jeopardizing determinism and creating a surface area of vulnerability.


## Config processing stages and parameter scopes
{: #config-processing-stages-and-parameter-scopes }

### Processing stages
{: #processing-stages }

Configuration processing happens in the following stages:

- Pipeline parameters are resolved and type-checked
- Pipeline parameters are replaced in the orb statement
- Orbs are imported

The remaining configuration is processed, element parameters are resolved, type-checked, and substituted.

## Element parameter scope
{: #element-parameter-scope }

Element parameters use lexical scoping, so parameters are in scope within the element they are defined in, e.g. a job, a command, or an executor. If an element with parameters calls another element with parameters, like in the example below, the inner element does not inherit the scope of the calling element.

```yaml
version: 2.1

commands:
  print:
    parameters:
      message:
        type: string
    steps:
      - run: echo << parameters.message >>

jobs:
  cat-file:
    parameters:
      file:
        type: string
    steps:
      - print:
          message: Printing << parameters.file >>
      - run: cat << parameters.file >>

workflows:
  my-workflow:
    jobs:
      - cat-file:
          file: test.txt
```

Even though the `print` command is called from the cat-file job, the file parameter would not be in scope inside the print. This ensures that all parameters are always bound to a valid value, and the set of available parameters is always known.

## Pipeline value scope
{: #pipeline-value-scope }

Pipeline values, the pipeline-wide values that are provided by CircleCI (e.g. `<< pipeline.number >>`) are always in scope.

### Pipeline parameter scope
{: #pipeline-parameter-scope }

Pipeline parameters which are defined in configuration are always in scope, with two exceptions:

- Pipeline parameters are not in scope for the definition of other pipeline parameters, so they cannot depend on one another.
- Pipeline parameters are not in scope in the body of orbs, even inline orbs, to prevent data leaks.

## Conditional workflows
{: #conditional-workflows }

Use the [`when` clause](https://circleci.com/docs/2.0/configuration-reference/#using-when-in-workflows) (or the inverse clause `unless`) under a workflow declaration, along with a [logic statement](https://circleci.com/docs/2.0/configuration-reference/#logic-statements), to decide whether or not to run that workflow. Logic statements in a `when` or `unless` clause should evaluate to a truthy or falsy value.

The most common use of this construct is to use a pipeline parameter as the value, allowing an API trigger to pass that parameter to determine which workflows to run.

Below is an example configuration using the pipeline parameter `run_integration_tests` to drive whether the workflow `integration_tests` will run.

```yaml
version: 2.1

parameters:
  run_integration_tests:
    type: boolean
    default: false

workflows:
  version: 2
  integration_tests:
    when: << pipeline.parameters.run_integration_tests >>
    jobs:
      - mytestjob

jobs:
  mytestjob:
    steps:
      - checkout
      - ... # job steps
```

The example shown above prevents the workflow `integration_tests` from being triggered unless it is explicitly invoked when the pipeline is triggered with the following in the `POST` body:

```json
{
    "parameters": {
        "run_integration_tests": true
    }
}
```

The `when` key accepts any truthy or falsy value, not just pipeline parameters, though pipeline parameters will be the primary use of this feature until more are implemented. `when` also has an inverse clause `unless`, which inverts truthiness of the condition.
