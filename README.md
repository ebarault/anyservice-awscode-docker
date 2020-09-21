# anyservice-awscode

Integrate AWS CodeBuild and Pipelines with git services

## Compatible Providers

* BitBucket
* GitLab

**Image**: [Docker Hub][dockerhub-url]

## AWS CodeBuild

This image will create a zip archive of the project, send it to a `CODEBUILD_S3_BUCKET`/`CODEBUILD_S3_KEY` location and start a new AWS CodeBuild build process.  The build process will be tied to the version of the archive in S3.

Bitbucket|GitLab pipelines will wait for AWS CodeBuild to finish and return success or failure based on the outcome of the build.

This image will download the codebuild artifacts and download it to .codebuild_artifacts

## CI environment variables export
CI-specific environment variables are exported to the AWS Codebuild runner:

### Bitbucket pipeline
All environment variables prefixed by `BITBUCKET_` are exported.
The full list of environment variables set by Gitlab pipeline is available at https://confluence.atlassian.com/bitbucket/environment-variables-794502608.html

The user can export extra environment variables by prefixing them by `BITBUCKET_`.

### GITLAB-CI
All environment variables prefixed by `CI_|GITLAB_` are exported.
The full list of environment variables set by Gitlab pipeline is available at https://docs.gitlab.com/ce/ci/variables/README.html

The user can export extra environment variables by prefixing them by `CI_` or `GITLAB_`, or by using the `CI_ENV_PATTERN` environment variable to provide extra prefixes to export.

### Bitbucket pipeline integration (`bitbucket-pipelines.yml`)
```
image: ebarault/codebuild-git-integration:latest
pipelines:
  default:
    - step:
        script:
            - |
              AWS_DEFAULT_REGION=eu-west-1 \
              CODEBUILD_PROJECT_NAME=my-project-name \
              CODEBUILD_S3_BUCKET=my-s3-bucket-name \
              CODEBUILD_S3_ARCHIVE_KEY=codebuild/project-name/code.zip \
              CODEBUILD_S3_RESULT_PATH=codebuild/project-name/codebuild-results \
              CODEBUILD_START_JSON_FILE=cicd/buildspec/start_build.json \
              start-build

```

### GitLab-CI integration (`.gitlab-ci.yml`)
```
image: ebarault/codebuild-git-integration:latest
stages:
    - build

codebuild_start:
    tags:
        - docker
    stage: build
    script:
        - |
            AWS_DEFAULT_REGION=eu-west-1 \
            CODEBUILD_PROJECT_NAME=my-project-name \
            CODEBUILD_S3_BUCKET=my-s3-bucket-name \
            CODEBUILD_S3_ARCHIVE_KEY=codebuild/project-name/code.zip \
            CODEBUILD_S3_RESULT_PATH=codebuild/project-name/codebuild-results \
            CODEBUILD_START_JSON_FILE=cicd/buildspec/start_build.json \
            start-build
```

## AWS Pipeline

Once a successful build has completed an AWS codepipeline can be executed as a Bitbucket Custom Pipeline from the respective commit. The artifact(s) produced by AWS CodeBuild will be fetched.  If the artifacts are not already in a zip archive they will be put in one and uploaded to the pipeline bucket and key path.

The pipeline needs to be configured to run automatically when new files are loaded on the selected codepipeline S3 bucket.

### Bitbucket pipeline integration (`bitbucket-pipelines.yml`)
```
image: ebarault/codebuild-git-integration:latest
pipelines:
  custom:
    pipeline_release:
      - step:
          script:
            - |
              AWS_DEFAULT_REGION=us-east-1 \
              CODEBUILD_S3_BUCKET=my-s3-bucket-name \
              CODEBUILD_S3_RESULT_PATH=codebuild/project-name/codebuild-results \
              CODEPIPELINE_S3_BUCKET=my-pipeline-s3-bucket \
              CODEPIPELINE_S3_ARCHIVE_KEY=codepipeline/project-name/pipeline.zip \
              push-to-pipeline
```

### GitLab-CI integration (`.gitlab-ci.yml`)
```yaml
image: ebarault/codebuild-git-integration:latest
stages:
    - deploy

pipeline_release:
    tags:
        - docker
    stage: deploy
    script:
        - |
          AWS_DEFAULT_REGION=us-east-1 \
          CODEBUILD_S3_BUCKET=my-s3-bucket-name \
          CODEBUILD_S3_RESULT_PATH=codebuild/project-name/codebuild-results \
          CODEPIPELINE_S3_BUCKET=my-pipeline-s3-bucket \
          CODEPIPELINE_S3_ARCHIVE_KEY=codepipeline/project-name/pipeline.zip \
          push-to-pipeline
```

**_Note_**: Additional cli options can be passed to `aws codebuild start-build` command by providing them to the docker's image `start-build` script, as in:

```yaml
    ...
    script:
        - start-build --buildspec-override cicd/my-other-buildspec.yml
```

## Environment Variables

It is recommended to keep all configuration in `bitbucket-pipelines.yml` | `.gitlab-ci.yml` files except for `AWS_ACCESS_KEY_ID` and the `AWS_SECRET_ACCESS_KEY`. That will ensure a commit is always associated with the corresponding S3 buckets and paths.

#### `AWS_ACCESS_KEY_ID`
**required**

The AWS Access Key for the User who will start the build

#### `AWS_SECRET_ACCESS_KEY`
**required**

The AWS Secret Access key for the User who will start the build

#### `AWS_DEFAULT_REGION`
**required**

The region AWS CodeBuild will be executed in

#### `AWS_ASSUME_ROLE`
**optional**

The arn of an AWS IAM Role to assume when running codebuild/codepipeline. This will generate an inject STS temporary session credentials into the shell environment

#### `AWS_ASSUME_ROLE_DURATION`
**optional**

The maximum validity period requested for the STS session credentials generated via the AWS_ASSUME_ROLE option. Defaults to 3600 seconds (one hour)

#### `VERBOSE`

When set to `true`, this set `set-x` in shell scripts to carbon copy all executed commands, this is helpful when debugging but can reveal secrets unwillingly

#### `CODEBUILD_S3_BUCKET`
**required**

The S3 bucket AWS CodeBuild will use to pull the code archive

#### `CODEBUILD_S3_ARCHIVE_KEY`
**required**

The S3 key AWS CodeBuild will use to pull the code archive

**Example**
`codebuild/my-project/my-code.zip`

#### `CODEBUILD_START_JSON_FILE`
**optional**

Full path to a JSON file within the project that will be merged with options provided in environment variables and added to the `start-build` CodeBuild command.

**Example**
`cicd/buildspec/start_build.json`

The expected format for the `start_build.json` file matches the cli skeleton generated by the AWS cli for the codebuild `start-build` command
See AWS documentation:
- https://docs.aws.amazon.com/cli/latest/reference/codebuild/start-build.html
- https://docs.aws.amazon.com/codebuild/latest/APIReference/API_StartBuild.html

```sh
aws codebuild start-build --generate-cli-skeleton
```
```json
{
    "projectName": "",
    "sourceVersion": "",
    "artifactsOverride": {
        "type": "CODEPIPELINE",
        "location": "",
        "path": "",
        "namespaceType": "NONE",
        "name": "",
        "packaging": "NONE"
    },
    "environmentVariablesOverride": [
        {
            "name": "",
            "value": "",
            "type": "PARAMETER_STORE"
        }
    ],
    "buildspecOverride": "",
    "timeoutInMinutesOverride": 0
}
```

#### `CODEBUILD_PROJECT_NAME`
**optional**

The name of the AWS CodeBuild Project. If not set here, this must be defined in `CODEBUILD_START_JSON_FILE`

#### `CODEBUILD_S3_RESULT_PATH`
**optional**

If set and the build is a success this will store a file for every git commit built, containing the most recent AWS CodeBuild ID.

Useful for custom pipelines.

**Example**
`codebuild/my-project/codebuild-results`

**Example Bucket Structures**
```
- codebuild
-- my-project
--- codebuild-results
---- BITBUCKET_REPO_OWNER-REPONAME-COMMIT_SHA.json
---- ...
```

```
- codebuild
-- my-project
--- codebuild-results
---- GITLAB_PROJECT_NAMESPACE-COMMIT_REF-COMMIT_SHA.json
---- ...
```

#### `WAIT_FOR_CODEBUILD=[true]`
If `true` then Bitbucket|GitLab CI Pipeline will wait for AWS CodeBuild to
finish. If the build fails then so will the pipeline in Bitbucket|GitLab CI.

#### `CODEPIPELINE_S3_BUCKET`
**required**

The source S3 bucket configured in AWS CodePipeline, set to automatically run when new files are loaded

#### `CODEPIPELINE_S3_ARCHIVE_KEY`
**required**

The source S3 key configured in AWS CodePipeline, set to automatically run when new files are loaded

#### `CODEBUILD_CHROOT`
**optional**

The relative path of the inner folder to zip and send to codebuild through S3. Defaults to `./` which archives the root folder

#### `CI_ENV_PATTERN`
**optional**

Add additionals environment variables patterns to pass vars to codebuild when matching a given prefix. Each pattern separated by a pipe character `|`. Defaults to `CI_|GITLAB_` when using Gitlab-CI and `BITBUCKET_` when using Bitbucket

#### `ARTIFACTS_PACKAGING`
**optional**

The artifacts packaging value set in your codebuild project. Value are ZIP or NONE, default is ZIP.

---



[dockerhub-url]: https://hub.docker.com/r/ebarault/codebuild-git-integration/
