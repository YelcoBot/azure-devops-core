# azure-devops-core

Centralized library of Azure DevOps YAML templates for building, testing, publishing and deploying .NET and .NET Framework projects.

A consumer pipeline references this repository and calls a single entry point, [main.yml](main.yml), which routes to the right service, command and language.

## Supported matrix

| Service                     | Language                 | Command     | Status                                             |
|-----------------------------|--------------------------|-------------|----------------------------------------------------|
| `iis`                       | `dotnet`, `netframework` | `deploy`    | Build and publish only (no deploy stage yet)       |
| `iis`                       | `dotnet`, `netframework` | `configure` | Not implemented (empty template)                   |
| `nuget`                     | `dotnet`, `netframework` | `deploy`    | Build and publish only (no deploy stage yet)       |
| `visual-studio-marketplace` | `dotnet`, `netframework` | `deploy`    | Build, publish, deploy to Marketplace and Git tag  |
| `github-release`            | `dotnet`, `netframework` | `deploy`    | Build, publish and create a GitHub Release         |
| `code-analysis`             | `dotnet`, `netframework` | `test`      | Build and test                                     |

## Structure

```
main.yml                         Entry point: routes to services/<service>/main.yml
services/<service>/main.yml      Validates the command and routes to commands/<command>.yml
services/<service>/commands/     Stages for each command
jobs/                            Reusable jobs
  build-test.yml                 Checkout, build, test
  build-publish.yml              Checkout, (GitVersion + VSIX version), build, publish, upload artifacts
  download-deploy.yml            Download artifacts, (GitVersion), deploy, (Git tag)
steps/                           Reusable steps
  dotnet/                        build, test, publish (DotNetCoreCLI)
  netframework/                  build, test, publish (MSBuild, VSTest, NuGet pack)
  artifact/                      upload (zip + publish), download (download + unzip)
  gitversion/execute.yml         Sets up and runs GitVersion (output name: GitVersionInfo)
  git/tag.yml                    Creates and pushes the v<MajorMinorPatch> tag
  visual-studio-marketplace/     version (stamps the vsixmanifest), deploy (publishes the VSIX)
  github-release/deploy.yml      Creates the GitHub Release with the published files as assets
```

## Usage

Reference this repository as a resource and extend from `main.yml`:

```yaml
trigger:
  - main

resources:
  repositories:
    - repository: core
      type: github
      name: YelcoBot/azure-devops-core
      endpoint: <github-service-connection>
      ref: refs/tags/v1.0.2

pool:
  vmImage: 'windows-latest'

variables:
  solution: '**/*.sln'
  buildConfiguration: 'Release'
  buildPlatform: 'Any CPU'
  gitVersionVersionSpec: '6.x'

stages:
  - template: main.yml@core
    parameters:
      service: visual-studio-marketplace
      language: netframework
      command: deploy
      projects:
        - name: MyExtension
          type: VSIX
          projectPattern: 'src/MyExtension/MyExtension.csproj'
          outputPath: 'MyExtension'
          vsixFile: 'MyExtension.vsix'
          publisherId: 'MyPublisher'
      artifacts:
        - name: MyExtension
          path: 'MyExtension'
```

Build and test only:

```yaml
stages:
  - template: main.yml@core
    parameters:
      service: code-analysis
      language: dotnet
      command: test
```

## Parameters

| Parameter   | Type   | Required | Description                                                                  |
|-------------|--------|----------|------------------------------------------------------------------------------|
| `service`   | string | Yes      | `iis`, `nuget`, `visual-studio-marketplace`, `github-release` or `code-analysis` |
| `language`  | string | Yes      | `dotnet` or `netframework`                                                   |
| `command`   | string | Yes      | Command of the chosen service (see the matrix above)                         |
| `projects`  | object | No       | Projects to publish. Not passed to `code-analysis`                           |
| `artifacts` | object | No       | Pipeline artifacts to upload and download. Not passed to `code-analysis`     |

| `dependsOn` | object | No       | Stage name or list of stage names the first stage of the service depends on. Pass `[]` to start in parallel; omit it to run after the previous stage |

Stages are named `<service>_<command>_<stage>` with hyphens replaced by underscores, for example `github_release_deploy_build_publish` and `github_release_deploy_deploy`. This lets one pipeline call `main.yml` several times and chain the calls with `dependsOn`.

### `projects` items

| Field                   | Used by                              | Description                                                        |
|-------------------------|--------------------------------------|--------------------------------------------------------------------|
| `name`                  | all                                  | Display name of the project                                        |
| `type`                  | `netframework`, VSIX flow            | `WEB`, `VSIX`, `APP`, `CLI` or `NUGET`                             |
| `projectPattern`        | all                                  | Path or pattern of the project file                                |
| `outputPath`            | all                                  | Folder under `$(Build.ArtifactStagingDirectory)` for the output    |
| `publishReadyToRun`     | `dotnet`                             | Value for `/p:PublishReadyToRun`                                   |
| `publishSingleFile`     | `dotnet`                             | Value for `/p:PublishSingleFile`                                   |
| `publishSelfContained`  | `dotnet`                             | Value for `/p:SelfContained`                                       |
| `publishPlatform`       | `dotnet`                             | Value for `/p:RuntimeIdentifier`                                   |
| `vsixFile`              | `VSIX`                               | Name of the `.vsix` file to publish                                |
| `publisherId`           | `VSIX`                               | Visual Studio Marketplace publisher id                             |

How each `type` is published with `netframework`:

| Type           | Result                                                                              |
|----------------|-------------------------------------------------------------------------------------|
| `WEB`          | MSBuild `Build;_CopyWebApplication` into `outputPath`                               |
| `VSIX`         | MSBuild build, then copies `*.vsix`, `publishManifest.json` and `overview.md`       |
| `APP`, `CLI`   | MSBuild build with `OutDir` set to `outputPath`                                     |
| `NUGET`        | `nuget pack` into `outputPath`                                                      |

With `dotnet`, every project is published with `dotnet publish` regardless of `type`.

### `artifacts` items

| Field  | Description                                                                                             |
|--------|---------------------------------------------------------------------------------------------------------|
| `name` | Pipeline artifact name (also the name of the zip)                                                       |
| `path` | Folder under `$(Build.ArtifactStagingDirectory)` to zip; extracted to `$(Pipeline.Workspace)/<path>`    |

Set `path` to the `outputPath` of the project it carries, so the deploy stage finds the files where it expects them.

## Requirements

Variables the consumer pipeline must define:

| Variable                | Used for                                             |
|-------------------------|------------------------------------------------------|
| `solution`              | NuGet restore, build and `dotnet test`               |
| `buildConfiguration`    | Build, test and publish                              |
| `buildPlatform`         | Build, test and publish                              |
| `gitVersionVersionSpec` | GitVersion version to install (VSIX and deploy flows)|

For the `visual-studio-marketplace` service:

- [GitTools](https://marketplace.visualstudio.com/items?itemName=gittools.gittools) extension (`gitversion-setup@4`, `gitversion-execute@4`).
- [Azure DevOps Extension Tasks](https://marketplace.visualstudio.com/items?itemName=ms-devlabs.vsts-developer-tools-build-tasks) extension (`PublishVisualStudioExtension@5`).
- A service connection named `visual-studio-marketplace`.
- A `GitVersion.yml` file at the root of the consumer repository.
- A `source.extension.vsixmanifest` next to the VSIX project file; its version is overwritten with `MajorMinorPatch`.
- `publishManifest.json` and `overview.md` copied to the build output of the VSIX project.
- Permission for the pipeline identity to push tags: after a successful deploy the tag `v<MajorMinorPatch>` is created and pushed (skipped if it already exists).

For the `github-release` service (use it for extensions the Marketplace does not accept, such as SSMS):

- A GitHub service connection (OAuth or PAT, not the Azure Pipelines app) with permission to create releases, and its name in the `gitHubConnection` variable. Define the variable in the pipeline YAML, not in a variable group: service connections are resolved at compile time.
- The consumer repository hosted on GitHub (`$(Build.Repository.Name)` is used as the target repository).
- A `GitVersion.yml` file at the root of the consumer repository.
- The release and its tag are both named `v<MajorMinorPatch>`; the task creates the tag on the built commit.
- Assets: `*.vsix` for `VSIX` projects, every top-level file of `outputPath` for the other types.

A Windows agent is required for `netframework` (MSBuild, VSTest) and for the Windows PowerShell steps.
