---
layout: post
title: Running Playwright tests inside docker-compose
date: '2025-04-16T16:48:00.000+00:00'
author: Luke Briner
---

## UI Tests using Playwright
[Playwright](https://playwright.dev/dotnet/) is one of the best libraries I have ever used. Honestly. Some of you will have used Selenium and understand about how flaky UI tests can be for all kinds of weird reasons but most tests under Selenium were littered with `Thread.Sleep()` or whatever just to do basic stuff. You had to install browsers and although we theoretically wanted to run UI tests on Firefox, Internet Explorer and Opera as well as Chrome, we never really got this to work under Selenium.

Enter Playwright and a simple nuget package and some basic config and you get rock-solid UI tests that run immaculately headless or in the browser with really nice Locator and Assertion syntax to make testing pages a breeze.

## Running tests locally
If you try and run the tests first time, you are likely to see an error in the test output that basically says you need to run a command to download the browsers into your app, although fortunately, you don't have to do this often. Run it a second time and it just works, assuming you have setup the correct URLs to visit for your tests.

## Running Playwright on a build server
This is where it starts to get interesting. For a start, you can't run the tests in a non-headless browser since these will run non-interactively but also if you just replicate the local way of testing, you will also need a step to check for the latest browsers being installed. Other than that, it should mostly work the same way.

## Running Playwright against Docker Containers
If you are just using `docker run` and mapping a port to the host, you could run it the same way but if you are running up, say, a database, app and an api at the same time, it is much more likely you will use docker-compose. Again, you could expose ports to the host and use Playwright traditionally but you can also run the tests from a Playwright docker image/container! Why might you do that? It completely avoids the browser check because the image already has the browsers installed. You can also easily peg the version of your Playwright nuget package to the version of the Playwright image, which means you could mix versions that need to build on the same build agent. In Team City, you can choose to run a step inside a container but which will have access to the default docker compose network created by a previous "Run everything up" step so it can connect but because it is running logically on the host, all the output gets fed back to Team City automatically, test attachments are displayed correctly etc.

This is how we have run most of our UI tests for containerised applications.

## And then we needed custom domains!
I didn't think much about it. I could access my "surveys" application via the DNS name "http://surveys/" but I also wanted to access it via e.g. "http://testdomain.lukebriner.net" and point to the same application. How do you add aliases? This is the rub in Docker Compose: You can only add aliases to a custom network, not to the default network (according to ChatGPT anyway!), so I had to add this to my docker compose file:
```yaml
services:
  surveys:
    ports:
      - "8080:80"
    networks:
      test-network:
        aliases:
          - testdomain.lukebriner.net
```
OK, so? Well, this breaks the behaviour of Team City, which allows you to "run build step in container", which will only attach to the default docker compose network. If I want to use the custom domain name, it won't resolve over the docker network so it fails. Ah. So how do we get tests running from a Playwright container inside a custom network? It sounded easy!

## Running Playwright tests inside Docker Compose
Obviously, instead of running the build as a step "in a container" to point to the existing docker compose services, we could add a tests service to the same docker compose configuration (or an override) and then just run it via `docker-compose run` easy peasy. Ah nope.

First problem: Now I am starting the tests at the same time as the apps-under-test, I need to make the tests wait for the application to be running. ChatGPT suggested [wait-for-it.sh](https://github.com/vishnubob/wait-for-it), or in my case a cut-down version that didn't need to do too much. Added that script. Also needed to modify the Dockerfile to install netcat, which is used by the script and make the script executable.

```dockerfile
FROM mcr.microsoft.com/playwright/dotnet:v1.50.0-jammy
RUN apt-get update && apt-get install -y netcat

# Build the app here

COPY Surveys.Core.Tests.UI/wait-for-it.sh ./
RUN chmod +x ./wait-for-it.sh

# etc...
```

However, since we only want this script to run on the build agent, I also had to override the default `dotnet test ...` in the tests Dockerfile with the correct command for Team City and this needed to be done in the `docker-compose-build.yml` file. Note that we are waiting for surveys:80 to be ready before running `dotnet test`:
```yaml
services:
  tests:
    build:
      context: .
      dockerfile: Surveys.Core.Tests.UI/Dockerfile
    networks:
      test-network:
    ipc: host
    depends_on:
      - surveys
      - database
    entrypoint: ""
    command: >
      bash -c "./wait-for-it.sh surveys 80 -- dotnet test --no-build"
```

So now it runs but it doesn't report anything to Team City. The tests might succeed or fail but there is nothing reported to the UI, just console logs. Hmmm. The problem is that if you run the build step "in a container", Team City makes sure it reports the stdout to the Team City console and it just works. When *you* run it in a container yourself, this doesn't happen. The solution? The Team City logger format.

## Make the Team City logger work in Docker
This was harder than it needed to be and involved lots of conversations with ChatGPT. This is partly due to the terse documentation [here](https://github.com/JetBrains/TeamCity.VSTest.TestAdapter) and also you don't always get great error messages. For example, the docs say that you have to set the test adaptor path but just says to set it to ".", which seems like it shouldn't be needed if the default value is ".". After playing with the various settings, I managed to get it to work with:

```yaml
services:
  tests:
    build:
      context: .
      dockerfile: Surveys.Core.Tests.UI/Dockerfile
    networks:
      test-network:
    ipc: host
    depends_on:
      - surveys
      - database
    entrypoint: ""
    command: >
      bash -c "./wait-for-it.sh surveys 80 -- dotnet test --no-build --test-adapter-path:./bin/Debug/net8.0 --logger:teamcity --results-directory /app/TestResults"
```
The results-directory was not so important, I was using that at one point when I was collecting the results using the Team City XML Report Processing feature. So then it started to write output to the log and Team City started reporting tests running as it was moving along, finally showing them as "Tests passed" (or failed). I then had a long and complicated fight with the attachments!

Again, when you run this as a step "in a container", the test attachments, which we generate for failed tests, automatically get picked up by Team City and shown as inline images and zip links for traces. Team City copies these since we use the same filename each time, these get copied as the tests are run.

## Make the attachments work
This involved two steps (one of which exposed all the horrible variation with how environment variables work across docker, docker-compose up, docker-compose run, Dockerfiles etc).

The first is that we need a path for the attachment that is the same on the host agent as it is inside the tests container. Why? The path is used to write the file to the container but that path is also reported to the host as the source for the attachments. Also, this path needs to be under the checkout directory. In short, I needed to map the checkout directory location e.g. `/opt/teamcity/work/12345678` to the same path inside the container. When the container puts a file there and says to the log that it is in this folder, Team City can also see it in the same location on the host as long as the volume maps correctly. How would the container know this location? Envronment variable. I basically did this in the Team City step:
```bash
export CHECKOUT_DIR=%teamcity.build.checkoutDir%
docker-compose -f docker-compose.yml -f docker-compose.override.yml -f docker-compose.override.build.yml build tests
docker-compose -f docker-compose.yml -f docker-compose.override.yml -f docker-compose.override.build.yml run --rm tests
```

And then inside the `docker-compose.override.build.yml` file, I added these volumes and environment variable copy from the host. Again, the TestResults volume was for the basic test results output. It works except it doesn't do the attachments and it only updates the test results in one hit at the end so you can't gauge progress.
```yaml
services:
  tests:
    build:
      context: .
      dockerfile: Surveys.Core.Tests.UI/Dockerfile
    networks:
      test-network:
    depends_on:
      - surveys
      - database
    entrypoint: ""
    command: >
      bash -c "./wait-for-it.sh surveys 80 -- dotnet test --no-build --test-adapter-path:./bin/Debug/net8.0 --logger:teamcity --results-directory /app/TestResults"
    volumes:
      - ./TestResults:/app/TestResults
      - "${CHECKOUT_DIR}:${CHECKOUT_DIR}"
    environment:
      - CHECKOUT_DIR
```
The line `- "${CHECKOUT_DIR}:${CHECKOUT_DIR}"` is the part that makes sure the same path is mapped inside and outside the container. I also had to change my C# tests code to write the files to the correct location, which was fairly easy to do:

```c#
[TearDown]
public async Task TearDown()
{
    string tracePath = null;
    if (TestContext.CurrentContext.Result.Outcome == ResultState.Error)
    {
        var sharedLocation = Environment.GetEnvironmentVariable("CHECKOUT_DIR") ?? "/app";

        await Page.ScreenshotAsync(new()
        {
            Path = Path.Combine(sharedLocation,"screenshot.png"),
            FullPage = true,
        });

        TestContext.AddTestAttachment(Path.Combine(sharedLocation, "screenshot.png"), "test screenshot");

        tracePath = Path.Combine(
            sharedLocation,      // Matches the path on the host
            "playwright-traces",
            $"{TestContext.CurrentContext.Test.FullName}.zip");
    }

    await Context.Tracing.StopAsync(new()
    {
        Path = tracePath
    });

    if (tracePath != null)
    {
        TestContext.AddTestAttachment(tracePath, "test traces");
    }
}
```
I put the whole method for reference but it is basically reading the `CHECKOUT_DIR` environment variables and saving things there.

I ran another build but although the path was now correct, it was not correctly parsing the test attachments, even though the path was valid. Instead it would show something like this:
```
------- Stdout: -------
Attachment "test screenshot": "file:///opt/teamcity/work/7c43d83aeaf3f388/screenshot.png
```

Which was not clickable even though the path was correct since, of course, I was clicking this on my machine where that path would be invalid. Why isn't it working?

## You need the Team City metadata in the log
Another very long back-and-forth with ChatGPT about what was going on here. The Team City logger was now working, I could see it "initialized" in the build log but ChatGPT said that I should see special markers like `##teamcity[publishArtifacts]` etc. although these were definitely not in my builds but they were in other builds that were running the step "in a container" so it seemed like they should be there. This was the missing sauce: why wasn't my build adding this from the Team City logger like they should? The answer? The `TEAMCITY_VERSION` environment variable! In a previous conversation with ChatGPT, it had told me to set this to "Local", which did make it work. What wasn't made clear at the time is that if you set it to "Local", it won't emit any of the Team City meta tags: why would it if you are not running it in Team City? In the name of making it work both locally and on Team City, I left the default value in the Dockerfile to be "Local" and then added an override in the `docker-compose.yml` to set it to the version of Team City we are using e.g.

```yaml
services:
  tests:
    # SNIP
    environment:
      - TEAMCITY_VERSION=2025.03
```

Hey presto! The tags appeared and the attachments started working.

I'm annoyed it took this long to work out, which was a combination of the number of variables (docker, docker-compose, .Net, Playwright, dotnet test, Team City), but also documentation that was not great and messages from utilities not really explaining what was going on. Imagine if the team city logger instead of just saying "Initialized" would say, "Logger set to "Local" mode, not emmitting special tags" or even just emit them anyway, who cares? If you use the special logger locally, you would kind of expect it to be the same as it would be on Team City.

Anyway, it works now. These are the complete files I ended up with:

`docker-compose.yml`
```yaml
services:
  surveys:
    image: surveys-core:latest
    build:
      context: .
      dockerfile: Surveys.Core/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://+:80
      - AppConfiguration__ApplicationUrls__Surveys=http://localhost:8080/
      - ConnectionStrings__conn=Server=database;Database=SurveysDB;User Id=sa;Password=Password123!;
      - WAIT_HOSTS=database:1433
      - WAIT_TIMEOUT=50
    depends_on:
      - database

  database:
    image: ${DOCKER_REGISTRY-}database:latest
```

`docker-compose.override.yml`
```yaml
services:
  surveys:
    ports:
      - "8080:80"   # Expose to localhost on port 8080
    networks:
      test-network:
        aliases:
          - testdomain.lukebriner.net

  database:
    ports:
      - "5433:1433"   # Expose to localhost on port 5433
    networks:
      test-network:

networks:
  test-network:   
```

`docker-compose.override.build.yml` (used on Team City)
```yml
services:
  tests:
    build:
      context: .
      dockerfile: Surveys.Core.Tests.UI/Dockerfile
    networks:
      test-network:
    ipc: host
    depends_on:
      - surveys
      - database
    working_dir: /app/Surveys.Core.Tests.UI
    entrypoint: ""
    command: >
      bash -c "./wait-for-it.sh surveys 80 -- dotnet test --no-build --test-adapter-path:./bin/Debug/net8.0 --logger:teamcity --results-directory /app/TestResults"
    volumes:
      - ./TestResults:/app/TestResults
      - "${CHECKOUT_DIR}:${CHECKOUT_DIR}"
    environment:
      - CHECKOUT_DIR
      - TEAMCITY_VERSION=2025.03

networks:
  test-network:
  
```

`Dockerfile` (for tests container)
```dockerfile
# Use official Playwright for .NET image as base
FROM mcr.microsoft.com/playwright/dotnet:v1.50.0-jammy
RUN apt-get update && apt-get install -y netcat

WORKDIR /app

# Copy your solution and projects
COPY *.sln ./
COPY Surveys.Core/ ./Surveys.Core/
COPY SharedLibraries/ ./SharedLibraries/
COPY Surveys.Core.Tests.UI/ ./Surveys.Core.Tests.UI/

# Restore, build, and run tests
WORKDIR /app/Surveys.Core.Tests.UI

COPY Surveys.Core.Tests.UI/wait-for-it.sh ./
RUN chmod +x ./wait-for-it.sh

RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore

RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet build --no-restore -c Debug

ENV ASPNETCORE_ENVIRONMENT=Build
ENV TEAMCITY_PROJECT_NAME='Surveys Core'
ENV TEAMCITY_VERSION=Local
ENV PW_CHROMIUM_ARGS='--no-sandbox'

ENTRYPOINT ["dotnet", "test", "--no-build", "--logger:trx"]
```