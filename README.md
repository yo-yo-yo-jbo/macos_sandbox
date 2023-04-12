# Introduction to macOS - the App sandbox
Continuing with my transition to macOS blogpost series, I'd like to discuss a bit about the macOS App sandbox.  
It is highly recommended to read the [macOS App structure](https://github.com/yo-yo-yo-jbo/macos_app_structure) blogpost first - I will be assuming the reader knows the difference between Apps, processes (tasks), know a little bit about `launchd` and its relation to App launching.

## Sandbox by example
The first time I learned about the macOS sandbox I naively tried to create a malicious [Word Macro](https://support.microsoft.com/en-us/office/create-or-run-a-macro-c6b99036-905c-49a6-818a-dfb98b7c3c9c).  
This (still) is a very common entry vector in the Windows ecosystem, so I wanted to see if I could just launch processes and generally wreck havoc.  
Well, things are not so easy on macOS - I was able to run processes, for instance, but it seems they couldn't do much.  
Dropping files always got me cryptic `Operation not permitted` error - what's going on?  
I started reading a bit about macOS and Word and came upon [this excellent blogpost](https://www.mdsec.co.uk/2018/08/escaping-the-sandbox-microsoft-office-on-macos/) by [Adam Chester](https://twitter.com/_xpn_) (works at MDSec). I highly recommend reading the blogpost, but I'll summarize the findings here:
- Apps may use a technology called [the App Sandbox](https://developer.apple.com/documentation/security/app_sandbox).
- Once set up, the OS enforces *configurable rules* on the App, such as what filenames it could create, whether it can use network capabilities and so on.

MacOS used to have a working utility called `sandbox-exec` that would execute commands in a sandbox. While deprecated, it could highlight quite a lot. You see in the manual page for it that it gets a `profile`, so we can conclude sandbox rules are maintained in profiles. Those profiles can come in various shapes and forms - files, pre-defined names or even as literal strings.  
The manual pages also state developers should be using the App Sandbox feature. Reading more about it I understood that the sandbox rules are *baked into the binary*, in our case, living under `/Application/Microsoft Word.app/Contents/MacOS/Microsoft Word` (if this seems alien to you, check out my [macOS App Structure blogpost](https://github.com/yo-yo-yo-jbo/macos_app_structure/)).  
While you can easily extract them by hand, it's best to use a tool: `codesign`:

```shell
```jbo@McJbo ~ % codesign -dv --entitlements - /Applications/Microsoft\ Word.app/Contents/MacOS/Microsoft\ Word
Executable=/Applications/Microsoft Word.app/Contents/MacOS/Microsoft Word
Identifier=com.microsoft.Word
Format=app bundle with Mach-O universal (x86_64 arm64)
CodeDirectory v=20500 size=351454 flags=0x10000(runtime) hashes=10972+7 location=embedded
Signature size=8980
Timestamp=Apr 10, 2023 at 8:09:50 AM
Info.plist entries=52
TeamIdentifier=UBF8T346G9
Runtime Version=13.1.0
Sealed Resources version=2 rules=13 files=28766
Internal requirements count=1 size=180
[Dict]
	[Key] com.apple.application-identifier
	[Value]
		[String] UBF8T346G9.com.microsoft.Word
	[Key] com.apple.developer.aps-environment
	[Value]
		[String] production
	[Key] com.apple.developer.team-identifier
	[Value]
		[String] UBF8T346G9
	[Key] com.apple.security.app-sandbox
	[Value]
		[Bool] true
    
...

	[Key] com.apple.security.temporary-exception.files.absolute-path.read-only
	[Value]
		[Array]
			[String] /Library/Preferences/com.microsoft.office.licensingV2.plist
			[String] /Library/Application Support/Microsoft/
      
...

	[Key] com.apple.security.temporary-exception.sbpl
	[Value]
		[Array]
			[String] (allow file-read* file-write* (require-all (vnode-type REGULAR-FILE) (regex #"(^|/)~\$[^/]+$")) )
			[String] (deny file-write* (subpath (string-append (param "_HOME") "/Library/Application Scripts")) (subpath (string-append (param "_HOME") "/Library/LaunchAgents")) )
	[Key] com.apple.security.temporary-exception.shared-preference.read-only
	[Value]
		[Array]
			[String] com.ThomsonResearchSoft.EndNote
```

This is a lot to unpack.
