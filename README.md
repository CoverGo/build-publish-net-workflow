# Overview

Build-publish workflow template generates a GithubCI workflow which converts a repository source code into published Docker images.
The template is mostly intended for .Net backend microservice repositories, however it can also be used for non-.Net services to implement some common logic like image versioning and code scanning.
Template produces hi-quality, production-ready workflow taking care of common aspect like unit, integration and acceptance testing,
nuget publishing, build cache, versioning, docker images tagging, test result collection and so on.
Template is designed for mid-size organisations to generate a workflow with gflows cli local tools using a configuration file.
A repository using this template for workflow generation is called "consumer repository", and the repository owner is called "consumer"

Template supports versioning and enables centralised workflow management across many repositories,
when consumers can update the template version and regenerate the workflow to get new features
instead of manual edits. It also allows consumers to save a lot of time by not learning GitHub actions and inner workings of CI. 
They can rely on simple cli commands to always have up-to date CI required by the company.

Generated workflow interacts with the repository by building and runner some docker images.  
Produced docker image could be used as a result, or could be launched during the CI to extract some files out of it as results. 
Images could be launched in compose environment if needed (for integration tests, for example)

Standard usage scenario for a consumer is: 
1) Perform initial setup:
   a) create folders and config files targeting a particular version of build-publish workflow
   b) create dockerfiles and composefiles if needed
   c) install gflows tool
2) run gflows tool locally to generate GitHub CI workflow 
3) commit the generated workflow and use it via GitHub Actions on new commits
4) When a new template version is published, modify config and repeat steps 2)-3)

# Target repository assumption

Workflow template is designed for a generic .Net microservice, with several assumptions: 

- There is one solution in the repository
- There are several projects in the repository, packable as Docker images 
- There is a main service
- There are zero or more unit tests projects 
- There are zero or more integration test projects
- There are zero or more acceptance test projects requiring Behave.Pro integration
- External dependencies for integration and acceptance tests could be constructed as docker-compose environment 
- Integrate and acceptance test projects contains compose files required to run the tests 
- There are zero or more NuGet packages (for the service client, for example)

# Workflow usage

A consumer needs to provide a valid configuration file to generate workflow from this template. Configuration is required as repositories can differ in project names,
amount, dependencies, presence of tests, nuget and other factors. 
Generation is done by `gflow` tool launched locally with configuration files presented. 
Configuration targets a particular version of build-publish workflow template and describes how exactly workflow should be generated, based on the consumer repository structure. 


### 1. Dockerize repository 
To support this workflow, the repository should contain a multi-stages Dockerfile with targets for different components, listed below.  
It is possible to use separate docker files, but a single, multi-stages docker file is recommended as the best practice.    

#### Mandatory - Domain service

 Produced image should run the service on container run. Target name can be any. 
 The contract is:
> GIVEN dockerfile 'Dockerfile' in consumer repository  
> AND it has target 'service'  
> WHEN CI runner builds the service image using command `docker build -f Dockerfile --target service -t domain_service .`  
> THEN production environment can launch the service using command `docker run domain_service`

**Optional**:
- Unit tests
- Integration tests
- Acceptance tests
- Nuget packages

For all tests, produced docker image should run the tests on container run.  
Contract: 
> GIVEN docker image 'acceptance_tests' built from consumer repository  
> AND configuration set the cucumber results filename as `CucumberResults.json`  
> WHEN CI runner can launches the image using command `docker run acceptance_tests --name ci_acceptance_tests`  
> THEN zero return code means all tests are passed   
> AND non-zero return code means there are some failed tests  
> AND Test results are produced in the container `ci_acceptance_tests` as file `CucumberResults.json` in cucumber format

[Example of a docker file with service target](https://github.com/CoverGo/OpenDomain/blob/a96d5e79652bfb7e0db5224b815599d4fd1381db/Dockerfile#L39)

Test images can produce additional results as json file in cucumber format, to support BDD.  
> GIVEN dockerfile 'Dockerfile' in consumer repository  
> AND it has target 'unit_test'
> AND configuration set the results folder as `TestResults`  
> WHEN CI runner builds the service image using command `docker build -f Dockerfile --target unit_test -t unit_test_run .`  
> THEN CI runner can launch unit tests using command `docker run unit_test_run --name ci_unit_tests`  
> AND zero return code means all tests are passed   
> AND non-zero return code means there are some failed tests  
> AND Test results are produced as xml file with JUnit format in the container `ci_unit_tests` folder `TestResults`
> 
Example of a docker file with [unit](https://github.com/CoverGo/OpenDomain/blob/a96d5e79652bfb7e0db5224b815599d4fd1381db/Dockerfile#L76), [integration](https://github.com/CoverGo/OpenDomain/blob/a96d5e79652bfb7e0db5224b815599d4fd1381db/Dockerfile#L92) and [acceptance](https://github.com/CoverGo/OpenDomain/blob/a96d5e79652bfb7e0db5224b815599d4fd1381db/Dockerfile#L126) tests targets

Workflow comes with support for Behave.Pro plugin for Jira and can upload such cucumber results to it

Nuget packages should be produced as a part of a nuget docker image.  
Contract:
> GIVEN docker image 'acceptance_tests' built from consumer repository  
> AND configuration set the cucumber results filename as `CucumberResults.json`  
> WHEN CI runner launches the image using command `docker run acceptance_tests --name ci_acceptance_tests`  
> THEN zero return code means all tests are passed   
> AND non-zero return code means there are some failed tests  
> AND Test results are produced in the container `ci_acceptance_tests` as file `CucumberResults.json` in cucumber format

Example of dockerfile producing [nuget package](https://github.com/CoverGo/OpenDomain/blob/a96d5e79652bfb7e0db5224b815599d4fd1381db/Dockerfile#L152)


### 1.1 Create compose files 

Template assumes integration and acceptance tests require additional services for run. To create an environment for these test workflow supports docker-compose. 
There should be a single compose file per test project. Compose file should contain a service for the tests and its dependencies. 
Contract:   

> GIVEN compose file 'integration-tests.yml' for integration tests
> AND integration test service in it named 'tests-integration'
> WHEN CI runner start compose as `docker-compose -f integration-tests.yml up`
> AND waits for tests finish as `docker wait tests-integration`
> THEN zero return code means all tests are passed   
> AND non-zero return code means there are some failed tests  
> AND Test results are produced in the container `tests-integration` 

Example of compose for for [integration](https://github.com/CoverGo/OpenDomain/blob/a96d5e79652bfb7e0db5224b815599d4fd1381db/docker-compose-tests-integration.yml#L35) and [acceptance](https://github.com/CoverGo/OpenDomain/blob/a96d5e79652bfb7e0db5224b815599d4fd1381db/docker-compose-tests-acceptance.yml#L33) tests.

## 2. Install gflows tooling 

 Follow instructions from [official website](https://github.com/jbrunton/gflows#installing)  
 For Mac OS you can find a [detailed guide](#MacOS installation guide) in appendix

## 3. Create configuration folders

 Workflow generation relies on conventions to locate configuration files and it is time to create these files.
 All paths are relative to repository root. 

**gflows config targeting build-publish workflow**

` ./gflows/config.yml ` with content

```yml
# Config file for GFlows.
# See https://github.com/jbrunton/gflows/wiki/Configuration for options.
githubDir: .github # the folder with your GitHub workflow, .github is the default one 
templates:
  engine: ytt  # engine used to generate GitHubWorkflow, ytt is the only option supported  
  defaults:
    libs:
      - workflow-configuration  # folder for repository settings file 
    dependencies:
      - https://raw.githubusercontent.com/CoverGo/build-publish-net-workflow/v2.0/.gflows # reference to build-publish workflow
      - https://raw.githubusercontent.com/CoverGo/ci-workflow-libraries/v2.2/.gflows # reference to some common libraries build-publish workflow uses
     

```
**workflow settings file**

` ./gflows/workflow-configuration/build-publish/build-publish.settings.yml ` example content: 

```
#@data/values
---
service:
  name: Service
  slug: auth-service
  dockerfile: Dockerfile
  docker_target: runtime
  image_name: covergo/opendomain-admin
```
Example of a [production-used configuration file](https://raw.githubusercontent.com/CoverGo/OpenDomain/master/.gflows/workflow-configuration/build-publish/build-publish.settings.yml).
This file is the main one to edit for a customer.


## 4. Generate GitHub workflow

From repository root, run command 

`gflows update` 

it should produce output like 
```
andrey@Andreys-MacBook build-publish-net-workflow % gflows update  
  create .github/workflows/build-publish.yml (from .gflows/workflows/build-publish)
```

Commit and push the created file `.github/workflows/build-publish.yml` then check github actions tab to see the generated workflow 

# Basic settings file structure

The Build-publish workflow template has a single yml settings file with several sections.
Sections can come in any order in the file

- **service** for the domain service itself, the purpose of the whole repository is to deliver this component (mandatory single section)  
- **nuget** for the project building nuget packeges like service typed clients or redistributable packages (optional single section)  
- **unit_test** for the repository unit tests projects (optional section with many items, one per project)  
- **integration_test** for the repository integration and acceptance tests projects (optional section with many items, one per projec)  
- **registry** for the Docker registry used to publish produced images (mandatory)  

[Example](https://github.com/CoverGo/build-publish-net-workflow/blob/main/.gflows/workflow-configuration/build-publish/settings.yml) of a config file with many sections

Each section us a part of yml, describing different workflow aspects in a common order like:

``` yml
 <component description> (mandatory)
 <docker image build configuration>  (mandatory)
 <docker image run results acquisition configuration> (optional)
 <docker compose environment configuration> (optional)
 <additional configuration> (optional)
```
Example of the settings file:
```yml

service:
  name: Service
  slug: auth-service
  dockerfile: Dockerfile
  docker_target: runtime
  image_name: covergo/opendomain-admin

nuget:
  name: GraphQL client
  slug: opendomain-client
  dockerfile: Dockerfile
  docker_target: pack_client
  image_name: covergo/opendomain-admin-client
  cache_from:
    - covergo/opendomain-admin
  container_result_path: nupkg
 
unit_test:
  - name: Unit tests
    slug: opendomain-tests-unit
    dockerfile: Dockerfile
    docker_target: run_tests_unit
    image_name: covergo/opendomain-tests-unit
    cache_from:
      - covergo/opendomain-admin
    container_result_path: app/TestResults

integration_test:
  - name: Integration tests
    slug: opendomain-tests-integration
    dockerfile: Dockerfile
    docker_target: run_tests_integration
    image_name: covergo/opendomain-tests-integration
    cache_from:
      - covergo/opendomain-admin
      - covergo/opendomain-tests-unit
    container_result_path: app/TestResults
    compose_file: docker-compose-tests-integration.yml
    diagnostic_filter: opendomain*

cache_registry: 
  url: ghcr.io
  name: GitHub Container Registry
  user: ${{ github.repository_owner }}
  password: ${{ secrets.CR_PAT_FULL }}
```
Note that `unit_test`, `integration_test` is an array of generic sections, one per project.   
Setting file above will produce a workflow in GitHubUI:
![Workflow with sections](https://user-images.githubusercontent.com/73324662/142962904-ff938c1c-4f6d-4642-a28b-07e003f893f7.png)

## Section reference
```yml
section:
#component description
  name: Auth #name for the UI 
  slug: auth-service #used for job dependencies references 
#docker image build configuration
  # name of the Dockerfile to build. If presented, job will build image, mandatory 
  dockerfile: Dockerfile
  #  target in the dockerfile to build, optional
  docker_target: build_service
  # name of image to push to the registry, as a result of build, mandatory
  image_name: covergo/auth
  # docker cache configuration, optional. Can be a map or an array
  docker_cache:
    #Array:  
    # non-empty list containing names of the images to use as build cache.
    # Should match "image_name" field of some sections in the same configuration file 
    # order matters, as cache will be used in the same order as declaration
    # cache order should be the same as the Dockerfile targets dependencies on each other
    # base targets should come first
      - covergo/auth-test-unit
      - covergo/auth
  
  #Map: 
    #optional, type of docker cache to use. Possible values: "gha" and "registry"
    #gha will use GitHub Cache API via docker gha cache exporter. Experimental, fast, and limited to 5GB per repo 
    #registry will create a special "cache" images containing all layers and used only for caching, branch-specific, and store it in ghcr.io
    type: registry #[gha,registry]
    # optional, dependencies on external docker images, produced by other jobs,
    # used as an additional cache during build process.
    # Can be used only for "registry" cache type
    from_images:
      # non-empty list containing names of the images to use as build cache.
      # Should match "image_name" field of some sections in the same configuration file 
      # order matters, as cache will be used in the same order as declaration
      # cache order should be the same as the Dockerfile targets dependencies on each other
      # base targets should come first
      - covergo/auth
      - covergo/auth-test-unit

#docker image run results acquisition configuration
  #optional, if presented means this job will run docker image as stand-alone or inside compose
  #represents the path to get some artifacts from the image (compose service) as a result of the run 
  #artifacts will be treated based on other factor, e.g. it could be test results in some format, nuget package, or smth else 
  #requires image_name to locate the target image 
  container_result_path: app/TestResults
  
#docker compose environment configuration
  # name of the compose file to use. if presented, means job will run docker image inside a compose environment
  compose_file: docker-compose.yml
  #service name in compose file to use as a result-producing service
  #results will be taken from this service container, based on 'container_result_path' setting
  compose_service_name: cases-test-integration
  #optional, password to protect sensitive diagnostics information about compose environment
  diagnostic_password: 123
  #optional, custom filter to identify images in compose environment to collect diagnostics
  diagnostic_filter: filter*
```


## Technical design

Build-publish workflow template is a central template for CoverGo .Net backend microservices, covering most of the needs to produce the images and packages. 
It uses some gflows packages for reused code and CoverGo-specific GitHub actions. More information [here](https://miro.com/app/board/o9J_lHoje38=/?moveToWidget=3074457357845402105&cot=14). 

## Migration from v1 to v2: 
 
+ Combine existing docker files into a single [multi-staged](https://docs.docker.com/develop/develop-images/multistage-build/) file 
+ in each config section: 
  + Change `dockerfile` field to the single dockerfile name
  + Add `docker_target` field to sections, based on the combined dockerfile targets   
  + Add `docker_cache` [array](#Section reference) in sections `nuget`,`unit_test`, `integration_test` , as v2 does not support convention-based dependencies between jobs (e.g. run integration tests only after unit tests) 
  + Remove `enabled` field as it is not used anymore. If you need to disable a section, remove it from config
+ if you use `integration_tests_legacy` section, change format of `environment.service_under_test.image` field to match on of the `image_name` fields from other sections.  
For example given `ghcr.io/covergo/auth:${{ needs.version.outputs.issue_id_slug }}` should be converted to `covergo/auth`
- If you don't need a specific filter for git branches to trigger CI, remove `git` section. It will trigger CI on push to any branch
- build-publish v2 uses common libraries v2.2, in case if a repository is using other workflows dependent on the common libraries as well, 
 they need to be updated to a version supporting common libraries 2.2:  
 *workflow-check*: v1.9.2  
 *scan-code-net*: v2.3.1  
 *deploy-tenant*: v1.1.1
## gflows MacOS installation guide 

If you don’t like install gflows using go, here is alternative way:

1.Download binaries from

`curl -L https://github.com/jbrunton/gflows/releases/download/0.5.0/gflows-darwin-amd64 > gflows`

2. Grant execution permission

`sudo chmod +rwx gflows`

3. Check it works in local folder

`./gflows`

You should see output like

```
andrey@Andreys-MacBook Downloads % ./gflows
Generate GitHub workflows from jsonnet templates

Usage:
gflows [command]

Available Commands:
check       Check workflow files are up to date
help        Help about any command
import      Import existing workflows
init        Setup config and templates for first time use using the given template engine
ls          List workflows
update      Updates workflow files
version     Print version
watch       Alias for check --watch --show-diffs

Flags:
-c, --config string    Location of config file
-d, --debug            Print debug information
--disable-colors   Disable colors in output
-h, --help             help for gflows
```
Use "gflows [command] --help" for more information about a command.

4. Copy file to to /usr/local/bin/gflows and

`sudo cp gflows /usr/local/bin`

5. Restart terminal or open a new window to access gflows, check gflows availability

`gflows`

You should see output like
```
andrey@Andreys-MacBook ~ % gflows
Generate GitHub workflows from jsonnet templates

Usage:
gflows [command]

Available Commands:
check       Check workflow files are up to date
help        Help about any command
import      Import existing workflows
init        Setup config and templates for first time use using the given template engine
ls          List workflows
update      Updates workflow files
version     Print version
watch       Alias for check --watch --show-diffs

Flags:
-c, --config string    Location of config file
-d, --debug            Print debug information
--disable-colors   Disable colors in output
-h, --help             help for gflows

Use "gflows [command] --help" for more information about a command.
```
