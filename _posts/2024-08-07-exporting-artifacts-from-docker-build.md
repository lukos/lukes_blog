---
layout: post
title: Exporting artifacts from Docker build
date: '2024-08-07T19:07:00.000+01:00'
author: Luke Briner
---

## Why export artifacts from inside a Dockerfile?
In it simplest form, a Dockerfile (or equivalent) is a build description for a self-contained image which is most likely to either form the base of other Dockerfiles or more often to create a containerised application/service.

The idea of isolation inside the container is where the main strength of containerised applications come from. All dependencies internally and therefore extremely portable across devices and OS's. However, since the applications are often built as part of continuous integration (CI), there are additional issues that affect the ability to run multiple builds and create other artifacts very quickly.

One of these that comes up frequently is the cache for npm, Composer, Nuget etc. where the frequent updates to different versions and the amount of data you have to pull very easily breaks the Docker build cache leading to the rebuilding of more layers than you would like after making small code changes. Docker provides [Cache mounts](https://docs.docker.com/build/guide/mounts/#add-a-cache-mount) for this purpose. Effectively you can use a host location for build cache so that instead of a clean image build needing to do a massive e.g. npm install, the packages will already be available from previous builds reducing the latency introduced when running on multiple build-agents or if you were the change the base image.

The other issue for me was exporting build artifacts that are not part of the final image build. In my case, the generation of OpenAPI docs which then need pushing to an public web site but are not needed in the actual application they are generated from.

## How might we export from the Dockerfile?
A trivial approach would simply be to build the image, then run it up as a separate step, curl the swagger endpoint to get the document and then kill of the container. A little bit of hassle but possibly doable. A bit of a fiddle if you have to do it manually on your local machine though.

What about Volumes? This question comes up a lot on forums: "Can I use volumes during the *build* stage of docker?" and the answer that surprises many people is, "No, you cannot". Volumes are designed for the runtime environment, not for building. Why? Well the answer I got was mostly related to portability. If the build process requires a volume then anyone using the image wouldn't simply be able to "pull and build", which is a big plus point for Docker. People would likely misuse the feature for all kinds of evils so fair-enough!

## Using docker's --output
I never knew [this](https://docs.docker.com/reference/cli/docker/buildx/build/#output) existed until today. It allows you to output the result of a build (or a target within the build!) to somewhere other than the local build cache/local image store.

There are a few drivers that support e.g. OCI, tarballs and other registries but I am just interested in the default `local` driver which simply outputs the build result to the local file system. Once this is done, you can inspect the files for debugging purposes but in this case we can do something cleverer to gain access to build artifacts that are otherwise lost during the build process.

All I want is a single file: the `swagger.json` file produced by an `npm` command line tool that can extract the Swagger data from the dotnet core assembly directly without having to run it up.

In my docker file, to produce the file, it looks something like this:

```dockerfile
FROM build AS publish
# ...Everything needed beforehand...
RUN dotnet publish "./Public.API.csproj" -c Release -o /app/publish

RUN dotnet new tool-manifest && dotnet tool install Swashbuckle.AspNetCore.Cli
# (v2 is the name of the swagger file I need to fetch)
RUN dotnet swagger tofile --output /src/swagger.json bin/Release/net8.0/Public.API.csproj v2
```
Now that I have done this, I can run a biuld command like this to get the contents of the publish target:
`docker build --target publish --output swagger/ .`

Which is great except now I have a swagger directory with a lot of extra stuff in it that I don't need. Not the end of the world but a few hundred MB of disk space wasted on the build agent. How can I only output the one `swagger.json` file?

## Multi-stage builds
This is something that has been around for a while and allows us to use fat images (with e.g. SDKs) to build the app but then to use a much smaller runtime image for the final product. When you pass a target to `docker build` it works out which stages to build to get that far. Since you can have all kinds of branches, for example, the swagger stage might not need the tests to be carried out (although it could).

So what?

We add a new stage that only copies the `swagger.json` and is based on `scratch`, the empty base docker image. Like this:
```dockerfile
FROM scratch AS export
COPY --from=publish /src/swagger.json .
```
Simple as that. When I *now* run `docker build --target export --output swagger/ .` I *only* get the swagger.json, all the previous stages just stayed in the build cache. What is nice about this is that when I then run my final build for deployment, which is based on the `publish` stage, most of it is already built and the last stage is to get the runtime build, copy the publish output and tag it - takes a few seconds.

In my CI build, the next step is to upload the `swagger.json` I just generated using `rdme` but that is just some basic cli stuff.

> Don't forget though: You need to add your output location to `.dockerignore` so that when you run the final build, it doesn't see the docker context changed by the new files and invalidate previous stages!