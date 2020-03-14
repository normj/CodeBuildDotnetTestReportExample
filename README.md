# AWS CodeBuild Test Reporting with .NET Core

As part of AWS re:Invent 2019 the CodeBuild team announced a new test reporting feature which can help make diagnosing test failures in CodeBuild so much easier. You can read more about [here](https://aws.amazon.com/blogs/devops/test-reports-with-aws-codebuild/).

As of the time of writing this post CodeBuild supports JUnit XML and Cucumber JSON formatted files for creating test reports. I wanted to use this feature for .NET and after a little research I was quickly able to add my .NET tests to my CodeBuild project's reports. Let's take a look at how I made this work.

## Project Setup

For demonstration purpose I have a test project with the following tests. I admit these tests don't really do anything but they will give me 3 passing tests and one failing test for a malformed uri exception.

```csharp
using System;
using System.Net;
using Xunit;

namespace CodeBuildDotnetTestReportExample.Tests
{
    public class ExampleTests
    {
        [Fact]
        public void TestSuccess1()
        {
            
        }
        
        
        [Fact]
        public void TestSuccess2()
        {
            
        }

        [Theory]
        [InlineData("https://www.google.com")]
        [InlineData("fffaaa")]
        public void TestMalformedUri(string uri)
        {
            new Uri(uri);
        }
    }
}
```

Now to run this project CodeBuild I first started with a `buildspec.yml` file that built my project and ran my tests.

```yml
version: 0.2

phases:
    install:
        runtime-versions:
            dotnet: 2.2
    build:
        commands:
            - dotnet build -c Release ./CodeBuildDotnetTestReportExample/CodeBuildDotnetTestReportExample.csproj
            - dotnet test -c Release ./CodeBuildDotnetTestReportExample.Tests/CodeBuildDotnetTestReportExample.Tests.csproj
```

The first step to making my test reports was making sure the `dotnet test` command logged the test run. To do that I need to specify the logger format and where to put the logs. I changed the `dotnet test` command to use the `trx` log format and put the results in the `../testresults` directory.

```yml
            - dotnet test -c Release <project-path> --logger trx --results-directory ../testresults

```

## What to do with a trx file?

As I mentioned before CodeBuild's test reporting supports JUnit XML and Cucumber JSON formatted files. So what are we going to do with trx files which `dotnet test` created? The .NET community has created a .NET Core Global Tool called [trx2junit](https://www.nuget.org/packages/trx2junit/) to convert trx files into JUnit xml files.

What we need to do now in our `buildspec.yml` file is install trx2junit and run it on the trx files created by dotnet test. To do that I updated the **install** phase of my `buildspec.yml` file to install **trx2junit**.

```yml
    install:
        runtime-versions:
            dotnet: 2.2
        commands:
            - dotnet tool install -g trx2junit
```

With **trx2unit** installed I added a **post_build** phase to convert the trx files in the **testresults** directory to JUnit xml files. Depending on your Docker image being used the `~/.dotnet/tools/` directory might not be in the **PATH** environment variable. If its not then just executing **trx2junit** will fail because the executable can't be found. To ensure trx2junit can always be found I executed the tool using full path relative to the home directory and ignore the need for `~/.dotnet/tools/` being in the **PATH** environment variable.

```yml
    post_build:
        commands:
            - ~/.dotnet/tools/trx2junit ./testresults/*
```

After doing these changes to convert the trx files from `dotnet test` into JUnit XML files we can integrate .NET's test logging in CodeBuild's test reporting. The last update I needed to make to my `buildspec.yml` was to tell CodeBuild where to find the test logs using the **reports** section. Below is the full `buildspec.yml` file including the **report** section.

```yml
version: 0.2

phases:
    install:
        runtime-versions:
            dotnet: 2.2
        commands:
            - dotnet tool install -g trx2junit
    build:
        commands:
            - dotnet build -c Release ./CodeBuildDotnetTestReportExample/CodeBuildDotnetTestReportExample.csproj
            - dotnet test -c Release ./CodeBuildDotnetTestReportExample.Tests/CodeBuildDotnetTestReportExample.Tests.csproj --logger trx --results-directory ../testresults
    post_build:
        commands:
            - ~/.dotnet/tools/trx2junit ./testresults/*
reports:
    DotnetTestExamples:
        files:
            - '**/*'
        base-directory: './testresults'          
```

## Setting up the build project

Setting a CodeBuild project has a lot of options. Like what source control provider to use or whether create a stand alone job or part of a pipeline. For this post I'm going to keep it simple and create a standalone CodeBuild project pointing to a GitHub repository. Here are the steps I used to create the project.

* Log onto the CodeBuild console
* Click **Create build project**
* Set a project name
* Select the GitHub repository
* Configure the Environment image
  * Operating System = Ubuntu
  * Runtime = Standard
  * Image = aws/codebuild/standard:2.0
  * Service Role = New service role
* Click **Create build project**

![alt text](./resources/build-setup.gif "CodeBuild project Setup")

Once the project is created we can start a build which will execute the **buildspec.yml** to build the project and run the tests.


## View test report

Now as the builds run CodeBuild will capture the test reports identified in the **reports** section of the **buildspec.yml** file. In the CodeBuild console we can view test reports in the **Report group** section. You can see here my 4 test ran and which one failed. Clicking the test will give details about the failed test.

![alt text](./resources/report-overview.png "Test report")


## Conclusion

Adding test reports to your CodeBuild project will make it easier to diagnose CodeBuild jobs. Hopefully with these few steps it will be easy to add reporting to your .NET builds.