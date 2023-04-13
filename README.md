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
			
...
```

This is a lot to unpack, so let's take some high-level notes:
- First, our command-line used `-dv` flag, which stands for `display` and `verbose`. Then, `--entitlements` presents *entitlements* associated with the App or binary (yes, `codesign` can work on both). We will dive into entitlements in a different blogpost, but for now let's say they reflect capabilities of the App, and one of them states that the App is sandboxed (`com.apple.security.app-sandbox` has a Boolean value of `True`).
- The first few lines of output are general information about the binary, its hash and many other interesting things. They're out-of-scope for the purpose of this blogpost, but are quite interesting!
- Next we get a big dictionary. Those of you who remember me ranting about `plists` (again, in my [macOS App Structure blogpost]) might suspect that the key-value dictionary is a representation of some property list, and they will be correct.
- The sandbox rules are states in some of the dictionary keys. For example, `com.apple.security.temporary-exception.files.absolute-path.read-only` mention an array of absolute paths the App is permitted to read from.
- The (in)famous regular expression that lives under `com.apple.security.temporary-exception.sbpl` is also here - that's to create those notorious `~$whatever.docx` temporary files that Word is so fond of.
- Of course, it goes without saying that creating new child processes inherit the sandbox rules, otherwise there is no much point to it.
Note how powerful those sandbox rules are!

## Sandbox escape with launchd
In MDSec's blogpost from 2018 that I mentioned earlier, the `deny file-write*` part under `com.apple.security.temporary-exception.sbpl` did not exist, which allowed macros to create files with arbitrary contents such as `/Library/LaunchAgents/~$evil.plist`. Why does that escape the sandbox?  
`LaunchAgents` and `LaunchDaemons` are a well known (legitimate) persistence mechanism in macOS. I've mentioned them before, but you can think of them as Services (if you come from the Windows world) - `LaunchDaemons` start when the OS boots (and therefore live outside of a user's session), while `LaunchAgents` start when a user logs in.  
Interestingly, both are described in simple `plist` files. Here's one example for my OneDrive updater:
```shell
jbo@McJbo ~ % plutil -p /Library/LaunchAgents/com.microsoft.OneDriveStandaloneUpdater.plist
{
  "Label" => "com.microsoft.OneDriveStandaloneUpdater"
  "Program" => "/Applications/OneDrive.app/Contents/StandaloneUpdater.app/Contents/MacOS/OneDriveStandaloneUpdater"
  "ProgramArguments" => [
  ]
  "RunAtLoad" => 1
  "StartInterval" => 86400
}
```

Those LaunchAgents and LaunchDaemons get launched by `launchd` (remember that process?) and therefore it escapes the sandbox, as `launchd` had no knowledge of whether the `plist` was dropped from a sandboxed process or not (and even if it did - how would it know what sandbox rules to apply?).

This concept of using `launchd` to escape the macOS sandbox was used extensively, and in fact, [I have used that](https://www.microsoft.com/en-us/security/blog/2022/07/13/uncovering-a-macos-app-sandbox-escape-vulnerability-a-deep-dive-into-cve-2022-26706/) in the past.  
Saving you a couple of clicks - here's the idea:
- As you might recall, `launchd` launches macOS Apps. Those Apps might be launched by double-clicking them, or by other means - for example, clicking on a `zip` file will use the `Archive Utility` since it's associated to `zip` files.
- An alternative way to launch Apps through `launchd` is with the `open` command.
- The `open` command is rich - you can use some of its interesting features such as selecting the App, selecting the filename to open or even give full command-line arguments.
- I specifically used the builtin `Python` App (which no longer exists on new macOS vanilla devices) to launch Python with an `stdin` argument that essentially redirects the standard input from a file I dropped (that file was `~$evil.py` due to Word's constaints).
- In our case, `launchd` ran an unsandboxed `Python` App instance which started reading from `~$evil.py` which had arbitrary Python commands, essentially escaping the sandbox.

There have been similar ideas in other disclosures (one nice example lives [here](https://desi-jarvis.medium.com/office365-macos-sandbox-escape-fcce4fa4123c)) but the idea stays the same. I am pretty sure there are plenty more in plain sight!

## The fixes and the quarantine xattr
The one issue found by MDSec was specific to Office - and was fixed with more strict rules.  
The ones abusing `LaunchServices` (which is the framework name for launching apps with `launchd`) are more generic - and hence Apple had to fix them.  
One of the things I've noticed was files dropped by Word are now created with the `com.apple.quarantine` extended attribute, yes the same one I mentioned in my [introduction to Gatekeeper](https://github.com/yo-yo-yo-jbo/macos_gatekeeper/) blogpost.  
As it turns out, that quarantine attribute is some hardening against certain attacks - for example, the `Terminal` App refused to launch shell scripts created with that attribute. That's the reason, by the way, I had to use the `--stdin` option for `Python`.

## Summary
We've briefly discussed another macOS technology - the sandbox. We've seen how powerful and configurable it is, and how it could get broken.  
We've also tied some things together - how apps work with sandbox rules, how `launchd` launching apps breaks more than just the process trees and how `plist` files can be used for good or for evil - this time with persistence (`LaunchAgents` and `LaunchDaemons`).  
Luckily, we've even tied the `com.apple.quarantine` extended attribute from the [introduction to Gatekeeper](https://github.com/yo-yo-yo-jbo/macos_gatekeeper/) blogpost and explained how it might be used as an extra hardening against sandbox escapes. Not too bad!  
In the next couple of blogposts, we will explore more security mechanisms in macOS and might talk about strategies of breaking them.

Stay tuned!

Jonathan Bar Or
