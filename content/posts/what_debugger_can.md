
---
title: "Things you can do with a debugger but not with print debugging"
date: 2025-06-21T15:38:10+05:30
draft: false
type: post
tags: [Developer Tools]
showTableOfContents: true
---

People do or do not use debuggers for a variety of reasons. For one thing, they are hard to setup in many codebases. Second, you can't use them when your application is running on remote environments (such as kuberenetes). So, anecdotally, I have seen way more people using Print/`log.Debug` compared to a debugger.

Which is a shame, because while debug logging is convenient at times, debuggers can do some things which you can't easily simulate via debug logging.

## Debuggers let you See all the way up the call stack

In most debuggers you can see all the callers and inspect the state there. So if you don't know how we got to some stage, you can select the parent frame in the debugger menu and check the variables, or evaluate an expression there.

## Debuggers can evaluate expressions dynamically

Most debuggers for high-level languages let you evaluate expressions involving function calls and even modify the state of the running program.

This doubles as an REPL with access to all your program state.

## Debuggers can catch exceptions at the source

All debuggers have exception breakpoints which stop at the point where exception _is thrown_. This is super handy to inspect the exact state and figure out why exception happened. In almost all debuggers, you can also limit this functionality to _uncaught exceptions only_.

## You can alter the course of execution without modifying code
Let's say I am debugging a network issue, and I need to point a URL to another test endpoint and see if the issue still persists. I can stop just before the network call is made, evaluate an expression which assigns the new URL to the variable, and let the code run. This is better than making code changes, because there's no accidental risk of committing code changes.

## Using a debug configuration standardizes the project setup

If your team is using VSCode or IntelliJ IDEs, it's possible to check in debug configuration files so that everyone can have a standardized local development flow. These config files can specify `.env` files (for credentials), environment variables (common ones such as python / go / java options), as well as CLI arguments.

Make it a convention to have a debug configuration for every entry-point of the application (such as the server or the CLI), this way a new contributor can have a decent head start with the codebase.

## Conclusion
I bet you've already heard enough about conditional breakpoints. Some would've also heard about time travel debuggers (TTD) which let you step back in time. But most languages do not have a mature TTD implementation. So I am not writing about that.

You might also like another one of my recent posts, [On how to debug arbitrary function calls from an IPython REPL in VSCode debugger](/posts/vscode-ipython-debugging/).
