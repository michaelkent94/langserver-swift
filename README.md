# Swift Language Server

![macOS](https://img.shields.io/badge/os-macOS-green.svg?style=flat)
![Linux](https://img.shields.io/badge/os-linux-green.svg?style=flat)
![Apache 2](https://img.shields.io/badge/license-Apache2-blue.svg?style=flat)
![](https://img.shields.io/badge/Swift-3.1.1-orange.svg?style=flat)
[![Build Status](https://travis-ci.org/RLovelett/langserver-swift.svg?branch=master)](https://travis-ci.org/RLovelett/langserver-swift)
[![Join the chat at https://gitter.im/langserver-swift](https://badges.gitter.im/langserver-swift/Lobby.svg)](https://gitter.im/langserver-swift?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

# Overview

A Swift implementation of the open [Language Server Protocol](https://github.com/Microsoft/language-server-protocol). The Language Server protocol is used between a tool (the client) and a language smartness provider (the server) to integrate features like auto complete, goto definition, find all references and alike into the tool.

Currently this implementation is used by [Swift for Visual Studio Code](https://github.com/RLovelett/vscode-swift).

# Prerequisites

## Swift

* Swift version 3.1.1
* The toolchain that comes with Xcode 8.3.2 (`Apple Swift version 3.1 (swiftlang-802.0.53 clang-802.0.42)`)

## macOS

* macOS 10.12 (*Sierra*) or higher
* Xcode Version 8.3.2 (8E2002) or higher using one of the above toolchains (*Recommended*)

## Linux

* **Coming Soon**

# Build

```
% cd <path-to-clone>
% make debug
```

or with Xcode

```
% cd <path-to-clone>
% make xcodeproj
```

# Test

```
% cd <path-to-clone>
% make test
```
# Debug and Development

The language server itself relies on a language server client to interact with it. [This server has been developed to work with Visual Studio Code](https://github.com/RLovelett/vscode-swift). Though it should be noted that any client that implements the protocol _should_ work and is thusly supported.

An example workflow for interactively debugging the language server while using it with the Visual Stuio Code client is provided in this section. The instructions are devided into two sections. The first section explains how to generate and configure an Xcode project for debugging. The second section explains how to configure the Visual Studio Code plugin to use the debug executable.

## Xcode (e.g., [langserver-swift](https://github.com/RLovelett/langserver-swift))

In the directory containing the clone of this repository use SwiftPM to generate an Xcode project.

```
% git clone https://github.com/RLovelett/langserver-swift.git
% cd langserver-swift
% make xcodeproj
```

Since the language server client, e.g., VSCode, will actually launch the language server LLDB needs to be told to wait for the application to launch. This can be configured in Xcode after opening the generated project in Xcode. See the screenshot below.

<img width="997" alt="screen shot 2017-02-22 at 8 55 57 am" src="https://cloud.githubusercontent.com/assets/335572/23214552/1b0afce2-f8dd-11e6-8812-370ad148ee73.png">

The next step is to build the executable and launch LLDB. Both of these steps can be performed by going to "Product > Run" or the keyboard shortcut ⌘R. After building completes, Xcode should report something like "Waiting to attach to LanguageServer : LanguageServer".

<img width="844" alt="screen shot 2017-02-22 at 9 40 33 am" src="https://cloud.githubusercontent.com/assets/335572/23216177/0b0fc6a0-f8e3-11e6-9f0c-a5d71a01933a.png">

One final step is to determine the `TARGET_BUILD_DIR`. This is used to tell the VSCode extension in the next section where the debug language server is located.

From a terminal whose current working directory contains the Xcode project previously generated by SwiftPM you can get this information from `xcodebuild`.

```
% xcodebuild -project langserver-swift.xcodeproj -target "LanguageServer" -showBuildSettings | grep "TARGET_BUILD_DIR"
   TARGET_BUILD_DIR = /Users/ryan/Library/Developer/Xcode/DerivedData/langserver-swift-gellhgzzpradfqbgjnbtkvzjqymv/Build/Products/Debug
```

Or using `make`:

```
% make print_target_build_dir
```

Take note of this value it will be used later.

# VSCode (e.g., [vscode-swift](https://github.com/RLovelett/vscode-swift))

Open the directory containing the clone of the Visual Studio Code extension in Visual Studio Code.

```
% git clone https://github.com/RLovelett/vscode-swift.git
% code .
```

Start the TypeScript compiler or the build task (e.g., ⇧⌘B or Tasks: Run Build Task).

Now open `src/extension.ts` and provide the value of `TARGET_BUILD_DIR` for the debug executable. The change should be similar to the patch that follows.

```
diff --git a/src/extension.ts b/src/extension.ts
index b5ad751..7970ae1 100644
--- a/src/extension.ts
+++ b/src/extension.ts
@@ -13,7 +13,7 @@ export function activate(context: ExtensionContext) {
         .get("languageServerPath", "/usr/local/bin/LanguageServer");

     let run: Executable = { command: executableCommand };
-    let debug: Executable = run;
+    let debug: Executable = { command: "${TARGET_BUILD_DIR}/LanguageServer" };
     let serverOptions: ServerOptions = {
         run: run,
         debug: debug
```

**NOTE:** Make sure the `${TARGET_BUILD_DIR}` is populated with the value you generated in the Xcode section. It is not an environment variable so that will not be evaluated.

Once this is complete you should be able to open the VSCode debugger and and select `Launch Extension`. This should start both the language server (Xcode/Swift) and the extension (VScode/TypeScript) in debug mode.

# Caveats

1. As noted above you might not be able to capture all the commands upon the language server initially starting up. The current hypothesis is that it takes a little bit of time for LLDB (the Swift debugger) to actually attach to the running process so a few instructions are missed.

One recommendation is to put a break-point in [`handle.swift`](https://github.com/RLovelett/langserver-swift/blob/251641da96ac1e0ae90f0ead3aa2f210fcb2c599/Sources/LanguageServer/Functions/handle.swift#L17) as this is likely where the server is getting into to trouble.

2. Messages are logged to the `Console.app` using the `me.lovelett.langserver-swift` sub-system. One place to look the raw language server JSON-RPC messages is there.
