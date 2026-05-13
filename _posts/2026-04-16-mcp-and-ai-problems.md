---
layout: post
title: MCP and AI Problems
date: '2026-04-16T16:47:00.000+00:00'
author: Luke Briner
---

## AI with MCP
Model Context Protocol (MCP) is a standard way of exposing functionality to AI tools. If you have a database, instead of just requiring the AI tool to directly access and query everything, you put an MCP server in front of it. Now the database vendor can create a single MCP interface and any AI tool can use it.

That is the theory but making it happen in practice has taken me several days.

## The setup
* I have an existing Clickhouse database and it works fine
* I have a Claude account, which also works fine (at the moment)
* I found the official clickhouse MCP server which can run in Kubernetes and is based on FastMCP
* I have Azure Entra ID for authorisation

Sounds like a 30 minute setup job but it has taken me 3 days so far and this is because:

* The clickhouse MCP server is out of date on DockerHub but since it only has the tag "latest", this is not easy to track
* Azure have recently started enforcing an OAuth specification issue that makes the handshake break if you haven't updated your code properly
* Claude docs are mostly useless for setting this up
* Claude AI doesn't know much more, which is not surprising because it is trained from the docs but it takes a long time to tell you things that are not true because it doesn't know
* Claude Desktop doesn't always know that it is Claude desktop (some Electron shit no-doubt) so it causes frustration when it tells you that, "you have to do this in the desktop app instead"
* The ClickHouse MCP docs are mainly just about ClickHouse mcp and not about fastmcp, where some of the functionality lives. For example, it tells you about bearer auth but not about OAuth, which is required by Claude
* Claude talks about "local" servers when it is really talking about locally *defined* servers because some of these are on a URL and are remote
* MCP servers don't work in Claude for Windows by default because they don't have spaces in paths (like "Program Files") so you have to work around it by using cmd to call npx and even that doesn't seem to work
* Almost none of the errors that occurred have any helpful information about why that error might be happening
* Very few of the FastMCP environment variables are directly described in their docs although they *are* present in some of the docs in the Github repo.
* The OAuth flow is very confusing and again, Claude AI doesn't really understand it, probably because Anthropic haven't invested much of their billions of investor money into documentation to teach their own AI models!

## Clickhouse
This is just the database setup in a cluster but I add a specific user for MCP that just has SELECT and only on specific tables. I have it access to the system.* tables too but I cannot test yet whether it needs that to answer more general questions/find out what tables exist.

## Clickhouse MCP server
I wanted this to just work but it didn't really, partly because it had to work in a way that Claude Desktop expected and secondly due to the lack of clear documentation about the configuration.

1. The container image by default does not have HOME specified. For reasons that are best left to FastMCP, it defaults to / which means that when it tries to create cache directories under /.fastmcp, it fails. You don't know why, you just see the stack trace that the folder is either not there and/or not accessible. This is related to the client storage that is needed for the OAuth functionality but it took me a while to find this error before working out that I could just inject FASTMCP_HOME=/tmp/.fastmcp for a directory that would be writable
2. The clickhouse environment variables are mostly self-explanatory but I quoted them in Octopus deploy, even though Octopus deploy will quote them again so when reading in e.g. CLICKHOUSE_CONNECT_TIMEOUT="10", parseInt falls over and prints another call-stack. Nothing helpful about what it was doing when it fell over to help me work that out.
3. Then I had the whole brainache about http vs sse. SSE was an older transport that is now mostly being replaced by streamable http and FastMCP supports both but you have to select one. I kept trying different things because Claude was telling me ambiguous things like "Local servers have to use SSE" and "Claude might not work with http properly" and the rest of it and before it works, you don't know which of the knobs to change to try something else. It DOES work with http although confusingly on the /mcp path!
4. I then had to configure the Azure OAuth proxy, which is part of FastMCP but is another area where the documentation mentions the code configuration but not the environment variables so you play another game with AI who might find them or might not. FastMCP has this thing that parameters in any module can be supplied by environment variable with a certain prefix. For example, all settings for the Azure provider are under a prefix FASTMCP_SERVER_AUTH_AZURE_ but since these don't appear in the docs, AI can't find them. They do appear in the source code but only if you know to go and look there.
5. I also haven't been able to find out how to use an AKS PVC for the shared cache which means I have reduced the number of replicas to 1 for now so it can just use a local directory. That is my fault but just another hurdle in the long journey.
6. The final problem was the lack of updated Docker image on Dockerhub which was using an older version of FastMCP. I am unsure why this isn't an automated part of the MCP Clickhouse project, I guess they are too busy scaling to set it up. The upshot is that this version no longer works with some OAuth providers who do not permit the *resource* parameter to be passed to the authorize endpoint any more. This, again, manifests in a weird error that wouldn't be there if it was updated. The latest versions of FastMCP remove this parameter. Of course I could build the latest mcp code but it seems like I shouldn't have to if they add the build pipeline to the official image.

## Claude and Connectors
We all know that Antropic are scaling like crazy and can't be bothered with unimportant things like documentation and useful error messages. The irony is that AI is only as good as its training materials. If you don't document something properly, not even your flagship AI Chatbot can help people. It felt weird asking Claude AI how to setup Claude Desktop and it didn't know.

### Local Servers
Firstly, I just wanted to test that the MCP server worked. Had I setup auth and everything correctly. I used bearer token initially and this is theoretically supported by Claude but only for the badly named "local" servers which require manually editing the `claude_desktop_config.json` file. Of course, the format for "mcpServers" is not documented anywhere so you have to rely on Google/AI to make stuff up. Use "url", don't use "url"; "sse" is supported, "sse" is not supported. "mcp" is supported, how do I tell it to use mcp instead of sse? You can't use mcp.

Every time you change it, you have to restart Claude Desktop and hope to see some kind of error, although of course, you can't really tell what the errors mean. At one point, it seemed to work with my development mcp instance so I changed it to use my production one, which was basically the same, and then it stopped working. At one point it was erroring that all these random emojis were not "valid json", which they were not. What was this? It is reading stdout as well as the response body and getting confused. Right!? So you need to change the command to use &2>NUL but that only works in a single command with npx and not with the array syntax. You shitting me?

This was when I understood that even if I could get it to work, I could not add a shared connector using Bearer Token auth, that very unusual and rarely used method! If that was the case, I might as well just get OAuth working I suppose, at least I know once I get it working, I can let others in the company use it, which they will need to. So Local Servers was a shitshow that I left with little regret other than wasted time and Claude Desktop having an identity crisis.

Once I did get OAuth setup (or so I thought) on the MCP server, I had to ask an admin of our Claude account to add the connector and again, the url is not really checked for validity since the endpoint is actually <url>/mcp it accepted it without the mcp, auth worked but then it errored trying to request mcp data at /. I also really struggled to understand why Claude was asking for OAuth credentials. If I have already setup the MCP server as an OAuth client, what has that got to do with Claude? Time for another rabbit hole and another round of Claude Desktop explaining what was going on really badly and clearly not understanding.

### OAuth Attempt 1 - JWTVerifier
Setup the MCP server just as a token verifier and give Claude the OAuth creds because that's what Claude told me. Failed. Some weird error. Claude then corrected me not to use JWTVerifier but RemoteVerifier. That was also not true but fortunately I have done a lot of work with OAuth and smelled a rat.

### OAuth Attempt 2 - AzureProvider
This was making MCP server back into an OAuth Server which I eventually worked out meant that Claude is not doing the authorization like Claude told me it was. That was fine, or so I thought. Another error - invalid client id. What? Had to get the Admin to check our Entra ID but it was correct, which was then when I remembered that Claude also had the OAuth creds.

### OAuth Attempt 3 - AzureProvider but no Claude Creds
Get the admin to remove the OAuth creds from Claude and just use the MCP server for authorization. It turns out that you ONLY give Claude the creds when your provider supports auto-registration, which most providers don't because you want to control whic apps can register and which can't. OK so this goes a bit further but then shows a weird Authorization error even though the MCP server shows no errors. Claude tells me that the error message is misleading and using the logs, realises that since the connector points at the root url and not the `/mcp` path, it isn't working, I need to register it with the `/mcp` path. No problem.

### OAuth Attempt 4 - Azure Provider and registered with /mcp
Now I get another weird error that, "The resource parameter provided in the request doesn't match with the requested scopes.", which is unexpected although fortunately, some quick Googling found a range of similar reports which detailed that this was caused by an updated to v2 of OAuth at Azure and other providers that no longer accepts a `resource` parameter, which was being passed. I am not sure why this didn't happen before but anyway the next mystery was why this wasn't fixed.

## ClickHouse MCP again
I had the source code for both fastmcp and clickhouse mcp and found that the resource parameter was specifically being removed for this purpose so why wasn't it working? Lightbulb moment: What version of FastMCP is clickhouse using? Well the code says it is using the latest version of v2, which has the fix but...wait for it... the version on DockerHub is 6 months old and eventually I worked out was based on a much older v2 that does not have the fix so this is why it is still broken.

Someone has already raised an [issue](https://github.com/ClickHouse/mcp-clickhouse/issues/161) 2 weeks ago asking for the Dockerhub release to be updated and I have added another comment. I don't want to be ungrateful for free software but I also think if you run an official build of something, you should invest the time to keep it up to date and at least reply to issues like this to either say that "I can't because..." or maybe set an ETA. I can build it myself but then 100s of us around the world are doing the same thing with all the overhead of me managing it. I might have to because my time is running out to get this working.