# GitHub workflow for creating a NuGet package

In this guide, we’ll cover how to make a .NET NuGet package and host it on GitHub’s NuGet Package Registry using GitHub Actions. Our focus will be on a package within the `Greetings.Nuget.Demo` namespace. This package contains a `Greetings` class that provides methods for displaying 'Hello' and 'Hi' messages, enabling easy integration of these greetings from the `Greetings.Nuget.Demo Package` into your .NET applications.

### Create project

```
mkdir Greetings.Nuget.Demo
cd .\Greetings.Nuget.Demo\
git init --initial-branch=main
dotnet new classlib -o src/Greetings.Nuget.Demo -n Greetings.Nuget.Demo
rm .\src\Greetings.Nuget.Demo\Class1.cs
dotnet new gitignore
dotnet new sln
dotnet sln add (ls -r **/*.csproj)
rm
```

### Add logic

- Add `src\Greetings.Nuget.Demo\Greetings.cs`

```csharp
namespace Greetings.Nuget.Demo;

public class Greetings
{
    public void SayHello() => Console.WriteLine("Hello From Greetings.Nuget.Demo Package");

    public void SayHi() => Console.WriteLine("Hi From Greetings.Nuget.Demo Package");

    public void SayGoodMorning() => Console.WriteLine("Good Morning From Greetings.Nuget.Demo Package");

    public void SayGoodNight() => Console.WriteLine("Good Night From Greetings.Nuget.Demo Package");

    public void SayGoodAfternoon() => Console.WriteLine("Good Afternoon From Greetings.Nuget.Demo Package");

    public void SayGreetings() => Console.WriteLine("Greetings From Greetings.Nuget.Demo Package");
}
```

### Update NuGet Package metadata

- Add the following to ``src\Greetings.Nuget.Demo\Greetings.Nuget.Demo.csproj`

```xml
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <PackageId>Greetings.Nuget.Demo</PackageId>
    <Company>[Add a Company if needed]</Company>
    <Authors>[Add a Authors]</Authors>
    <Description>[Add a description]</Description>
    <RepositoryUrl>https://github.com/[UsernameOrOrganizationName]/Greetings.Nuget.Demo.git</RepositoryUrl>
    <PackageTags>Demo;NuGet</PackageTags>
    <GeneratePackageOnBuild>False</GeneratePackageOnBuild>
  </PropertyGroup>
```

> Note you need to update the metadata to suit your github account.

Since actions are running inside a virtual machine we should authenticate that action with our Github account in order to publish this package to the Github package registry.

For that, we will use the secret GITHUB_TOKEN

> `secrets.github_token` is the syntax referring to the GITHUB_TOKEN secret that GitHub automatically creates to use in your workflow. You can use the GITHUB_TOKEN to authenticate in a workflow run, [see source](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication#about-the-github_token-secret).

> NOTE If the secret GITHUB token fails to push the package then create a personal access token(classic) in GitHub, with the `write:packages` permission. Use the generated token and add a repository secret (or organisation secret if you have access) with that value. Use that in the `main.yml` work flow.

### Create a GitHub workflow

- Create `.github\workflows\main.yml` with the following content:

```yml
name: Build & Publish NuGet to GitHub Registry

on:
  push:
    branches: [master]

env:
  user: [user]
  organization: [org]
  project: "src/Greetings.Nuget.Demo/"

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      Version: ${{ steps.gitversion.outputs.SemVer }}
      CommitsSinceVersionSource: ${{ steps.gitversion.outputs.CommitsSinceVersionSource }}

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 #fetch-depth is needed for GitVersion

      #Install and calculate the new version with GitVersion
      - name: Install GitVersion
        uses: gittools/actions/gitversion/setup@v0.10.2
        with:
          versionSpec: 5.x

      - name: Determine Version
        uses: gittools/actions/gitversion/execute@v0.10.2
        id: gitversion # step id used as reference for output values

      - name: Display GitVersion outputs
        run: |
          echo "Version: ${{ steps.gitversion.outputs.SemVer }}"
          echo "CommitsSinceVersionSource: ${{ steps.gitversion.outputs.CommitsSinceVersionSource }}"

      # Build/pack the project
      - name: Setup .NET
        uses: actions/setup-dotnet@v3.2.0
        with:
          dotnet-version: 8.0.x

      - name: Build and Pack NuGet package with versioning
        run: dotnet pack ${{ env.project }} -p:Version='${{ steps.gitversion.outputs.SemVer }}' -c Release

      - name: Upload NuGet package to GitHub
        uses: actions/upload-artifact@v3.1.3
        with:
          name: nugetPackage
          path: ${{ env.project }}bin/Release/*.nupkg

  release:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/master' # only run job if on the master branch

    steps:
      #Push NuGet package to GitHub packages
      - name: Download nuget package artifact
        uses: actions/download-artifact@v3.0.2
        with:
          name: nugetPackage
          path: nugetPackage

      - name: Prep packages
        run: dotnet nuget add source --username ${{ env.user }} --password ${{ secrets.GITHUB_TOKEN }} --store-password-in-clear-text --name github "https://nuget.pkg.github.com/${{ env.organization }}/index.json"

      - name: Push package to GitHub packages
        if: needs.build.outputs.CommitsSinceVersionSource > 0 #Only release if there has been a commit/version change
        run: dotnet nuget push nugetPackage/*.nupkg --api-key ${{ secrets.GITHUB_TOKEN }} --source "github"

      #Create release
      - name: Create Release
        if: needs.build.outputs.CommitsSinceVersionSource > 0 #Only release if there has been a commit/version change
        uses: ncipollo/release-action@v1.13.0
        with:
          tag: ${{ needs.build.outputs.Version }}
          name: Release ${{ needs.build.outputs.Version }}
          artifacts: "nugetPackage/*"
          token: ${{ secrets.GITHUB_TOKEN }}
```

> Note you need to update the environment varaibles `user` and `organization`.

## Add GitVersion

We will use a `GitVersion.yml` file, [See GitVersion for detail instructions](GitVersion.md) :

```
dotnet-gitversion init
```

Configure it using `GitFlow` workflow and `Semantic Versioning`:

```
GitVersion init will guide you through setting GitVersion up to work for you

Which would you like to change?

0) Save changes and exit
1) Exit without saving

2) Run getting started wizard

3) Set next version number
4) Branch specific configuration
5) Branch Increment mode (per commit/after tag) (Current: )
6) Assembly versioning scheme (Current: )
7) Setup build scripts
```

Select the option 2:

```
The way you will use GitVersion will change a lot based on your branching strategy. What branching strategy will you be using:

1) GitFlow (or similar)
2) GitHubFlow
3) Unsure, tell me more
```

Select the option 1:

```
By default GitVersion will only increment the version of the 'develop' branch every commit, all other branches will increment when tagged

What do you want the default increment mode to be (can be overriden per branch):

1) Follow SemVer and only increment when a release has been tagged (continuous delivery mode)
2) Increment based on branch config every commit (continuous deployment mode)
3) Each merged branch against main will increment the version (mainline mode)
4) Skip
```

Again option 1.

This was it, select option 0 (Save changes and exit).

## Commit and Push

```
git add .
git commit -m 'Initial Setup'
git push
```
