# Introduction to macOS - the App sandbox
Continuing with my transition to macOS blogpost series, I'd like to discuss a bit about the macOS App sandbox.  
It is highly recommended to read the [macOS App structure](https://github.com/yo-yo-yo-jbo/macos_app_structure) blogpost first - I will be assuming the reader knows the difference between Apps, processes (tasks), know a little bit about `launchd` and its relation to App launching.

## Sandbox by example
The first time I learned about the macOS sandbox I naively tried to create a malicious [Word Macro](https://support.microsoft.com/en-us/office/create-or-run-a-macro-c6b99036-905c-49a6-818a-dfb98b7c3c9c).  
This (still) is a very common entry vector in the Windows ecosystem, and wanted to see if I could just launch processes and generally wreck havoc.  
Well, things are not so easy on macOS - I was able to run processes, for instance, but it seems they couldn't do much.  
Dropping files always got me cryptic access denied error - what's going on?  
