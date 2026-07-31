# su26-ai301-contribution

# 🚀 AI301 Open Source Contribution Journal

This repository documents my open-source contributions for CodePath AI301.

## 📚 Contributions

- [Contribution #1 – LinkFileList generation needs to use quoting](#contribution-1-linkfilelist-generation-needs-to-use-quoting)
- [Contribution #2 – Display proper URLs when initializing](#contribution-2-display-proper-urls-when-initializing)

| # | Project | Issue | Status |
|---|---------|-------|--------| 
| 1 | Swift Build | LinkFileList generation needs to use quoting (#13)| ✅ Phase IV Complete - Iterating|
| 2 | Firebase Tools | Display proper URLs when initializing (#3728) | 🟡 Phase III In Progress |

---

<details open>
<summary><h1>🚀 Contribution #1 – Swift Build</h1></summary>


# Contribution #1: LinkFileList generation needs to use quoting

**Contribution Number:** 1  
**Student:** Aanya Bharti  
**GitHub Username:** aanyabharti101  
**Issue:** https://github.com/swiftlang/swift-build/issues/13  
**Project Fork Link:** https://github.com/aanyabharti101/swift-build  
**Status:** Phase IV Complete  

---

## Why I Chose This Issue

I chose this issue because it was one of the first open-source issues I found where I could clearly follow the problem from the bug report all the way down to the build system. Most of my previous experience has been with application development and research projects, where I have developed strong problem-solving and debugging skills. This issue gave me the opportunity to apply those skills while learning more about what happens during the compilation and linking stages of software development rather than focusing only on application-level code. Since I develop on macOS, I was also excited by the opportunity to explore the Swift ecosystem and contribute to a project involved in building and linking software behind the scenes.

What stood out to me about this issue was how specific and unexpected the bug was. A project can fail simply because its path contains spaces and different linkers interpret file lists differently. I found it interesting that the fix wasn't about adding a new feature, but about understanding subtle differences in tool behavior across platforms. Through this contribution, I hope to gain experience working in a large codebase, communicating with maintainers, and solving the kinds of real-world edge cases that aren't usually covered in coursework.

---

## Understanding the Issue

### Problem Description
This issue involves how Swift Build generates LinkFileList files when paths contain spaces. Apple's ld64 linker interprets file list entries differently than many non-Apple linkers, which can cause paths with spaces to be interpreted incorrectly. As a result, builds may fail when projects are located in directories whose names contain spaces. I chose this issue because it combines debugging, build-system internals, and cross-platform compatibility, all areas I want to learn more about through open-source contributions.


### Expected Behavior

When Swift Build generates a LinkFileList for a non-Apple linker, file paths containing spaces should be written using a quoted response-file format so the linker interprets the entire path as a single argument.
For example, if an object file is located at:

/tmp/Test/a Project/Object.o

the generated LinkFileList should contain a properly quoted entry such as:

'/tmp/Test/a Project/Object.o'
(or the equivalent quoting format expected by the linker).

This ensures that directories containing spaces, such as "a Project", are treated as part of the file path rather than being split into multiple arguments during linking.
As a result, builds should succeed regardless of whether project paths contain spaces.



### Current Behavior

When a project path contains spaces, Swift Build generates LinkFileList entries without quoting.

During reproduction, the generated LinkFileList contained:

/tmp/Test/a Project/build/a Project.build/Debug/StaticLib1.build/Objects-normal/arm64/File.o
/tmp/Test/a Project/Object.o

Because these paths are emitted as raw unquoted strings, non-Apple linkers may incorrectly interpret the spaces as argument separators instead of part of the file path. This can cause linking failures when projects are located in directories whose names contain spaces.




### Affected Components
Primary files investigated:

- Sources/SWBCore/SpecImplementations/LinkerSpec.swift
- Sources/SWBUtil/ResponseFiles.swift
- Sources/SWBUniversalPlatform/Specs/Ld.xcspec
- Sources/SWBGenericUnixPlatform/Specs/UnixLd.xcspec
- Sources/SWBWindowsPlatform/Specs/WindowsLd.xcspec
- Tests/SWBTaskConstructionTests/TaskConstructionTests.swift

Relevant function:

- LinkerSpec.inputFileListContents()

Relevant build setting:

- LINKER_FILE_LIST_FORMAT

---

## Reproduction Process

### Environment Setup

Environment:

- macOS (Apple Silicon)
- Swift 6.2.4
- swift-driver 1.127.15

Setup Process:

1. Forked swiftlang/swift-build
2. Cloned repository locally
3. Built project using:

   swift build

Challenges Encountered:

- Several test commands produced Homebrew Boost search-path warnings.
- QNX platform tests could not be executed locally because the required QNX SDK is not installed.

Resolution:

- The build completed successfully despite the warnings.
- For investigation, I focused on task-construction tests and code inspection rather than QNX execution.
  

### Steps to Reproduce

1. Open:

   Tests/SWBTaskConstructionTests/TaskConstructionTests.swift

2. Locate the staticLibraries() test.

3. Change the test project name from:

   "aProject"

   to:

   "a Project"

4. Add temporary debugging output inside the LinkFileList validation block:

   print("SRCROOT:", SRCROOT)
   print("LINK FILE LIST CONTENTS:", contents.asString)

5. Run:

   swift test --filter TaskConstructionTests/staticLibraries

6. Observe the generated LinkFileList contents.

Expected:

Paths containing spaces should be quoted when using a non-Apple linker response-file format.

Actual:

The generated LinkFileList contains unquoted paths such as:

/tmp/Test/a Project/Object.o

demonstrating that paths containing spaces are written without quoting.


### Reproduction Evidence

- **Commit showing reproduction:** Reproduction was performed locally using temporary test modifications and debugging output. No reproduction-specific commit was created.
- **Screenshots/logs:** [If applicable]
  <img width="1400" height="1330" alt="image" src="https://github.com/user-attachments/assets/4ec64f9d-81cc-44e4-a1be-5f4b0b884fb1" />

- **My findings:** Paths containing spaces are emitted without quoting.
---

## Solution Approach

### Analysis
The issue does not appear to be caused by ResponseFiles.swift itself because the file already supports multiple quoting formats, including:

- unixShellQuotedNewlineSeparated
- unixShellQuotedSpaceSeparated
- windowsShellQuotedNewlineSeparated

Investigation showed that LinkFileList generation ultimately depends on:
LINKER_FILE_LIST_FORMAT

which is evaluated in:
Sources/SWBCore/SpecImplementations/LinkerSpec.swift

The likely root cause is that certain linker configurations use an unquoted response-file format even when the underlying linker requires shell-quoted paths.


### Proposed Solution
Determine how the linker type is detected during task construction and ensure that non-Apple linkers use a quoted LinkFileList format.

Potential approach:

- Detect whether the linker is Apple's ld64 or a non-Apple linker.
- Preserve current behavior for ld64.
- Override LINKER_FILE_LIST_FORMAT for non-Apple linkers to use:

  unixShellQuotedNewlineSeparated

- Add tests that verify paths containing spaces are correctly written to LinkFileList files.
  

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]
Swift Build generates LinkFileList files containing object file paths. Non-Apple linkers require quoted paths when those paths contain spaces. Currently, unquoted paths may be emitted, causing linker failures.

**Match:** [What similar patterns/solutions exist in the codebase?]
Existing response-file handling already supports quoted formats in:
Sources/SWBUtil/ResponseFiles.swift
Platform-specific linker specifications also define different LINKER_FILE_LIST_FORMAT values.


**Plan:** [Step-by-step implementation plan]
1. Trace how LINKER_FILE_LIST_FORMAT is selected.
2. Identify where linker type is detected.
3. Modify logic so non-Apple linkers use unixShellQuotedNewlineSeparated.
4. Add or update tests covering paths containing spaces.
5. Verify existing linker tests continue to pass.

**Implement:** [Link to your branch/commits as you work] https://github.com/aanyabharti101/swift-build/tree/issue-13-linkfilelist-quoting


**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]
- Verify contribution follows project conventions.
- Run relevant tests before creating a PR.
- Ensure Apple linker behavior remains unchanged.

**Evaluate:** [How will you verify it works?]
Success criteria:

- Paths containing spaces are quoted for non-Apple linkers.
- Existing tests continue to pass.
- New regression test passes.

---

## Testing Strategy

### Unit Tests

- [x] Test case 1: Added regression test linkFileListQuotingForNonAppleLinker() in Tests/SWBTaskConstructionTests/LinkerTaskConstructionTests.swift. The test creates a project path containing a space and verifies that generated LinkFileList entries are quoted for non-Apple linkers.
- [x] Test case 2: Reviewed existing tests in LinkerTaskConstructionTests.swift and followed the same testing structure and assertions used elsewhere in the file.
- [x] Test case 3: Reviewed the implementation logic to confirm that the format override is only applied when the discovered linker is not ld64.
- [x] Test case 4: Added `responseFileContentUnixNewlineQuotedPaths()` in `Tests/SWBUtilTests/ResponseFileTests.swift` to verify that `unixShellQuotedNewlineSeparated` produces quoted newline-separated entries for paths containing spaces.

### Integration Tests

- [x] Integration scenario 1: Reviewed how the new logic integrates with the existing linker task construction workflow in LinkerTools.swift.
- [x] Integration scenario 2: Verified that the implementation reuses the existing response-file formatting infrastructure rather than introducing a separate quoting mechanism.

### Manual Testing

Reproduced the issue by changing the test project name from aProject to a Project.
Added temporary debugging output to inspect generated LinkFileList contents.
Confirmed paths containing spaces were emitted without quoting before the fix.
Traced execution through LinkerTools.swift, LinkerSpec.swift, and ResponseFiles.swift to verify the fix targets the correct code path.

- Ran `swift test --filter linkFileListQuotingForNonAppleLinker`. The regression test compiled successfully and was discovered by the test runner, but execution was skipped because the test requires a Linux host and development was performed on macOS.

---

## Implementation Notes

### Week [3] Progress
Investigated Issue #13 and traced how LinkFileList files are generated during linker task construction.
Reproduced the issue using a project path containing spaces.
Identified LINKER_FILE_LIST_FORMAT as the setting controlling LinkFileList formatting.
Modified Sources/SWBCore/SpecImplementations/Tools/LinkerTools.swift to determine linker type before generating LinkFileList contents.
Added logic to use unixShellQuotedNewlineSeparated for non-Apple linkers while preserving existing ld64 behavior.
Added regression test linkFileListQuotingForNonAppleLinker() in Tests/SWBTaskConstructionTests/LinkerTaskConstructionTests.swift.
Strengthened the regression test to verify the expected quoted object-file path rather than only checking for a quoted source-root prefix.
Added a macOS-runnable ResponseFiles unit test to verify that `unixShellQuotedNewlineSeparated` quotes paths containing spaces.

### Week [4] Progress
Completed implementation and testing for Issue #13. Added ResponseFiles unit-test coverage for quoted paths, 
strengthened the LinkFileList regression test, rebased against the latest upstream changes, and opened Pull 
Request #1477 against the Swift Build repository. Requested review from project code owners and maintainers.


### Week [5] Progress
Submitted Pull Request #1477 to the upstream Swift Build repository. Requested review from project code owners and completed the Phase IV documentation. 
While waiting for review, began a second contribution cycle by selecting Firebase Tools Issue #3728 ("Display proper URLs when initializing"), 
setting up the repository locally, and completing the Phase I investigation and project planning.

### Week [6] Progress
Received maintainer engagement after a follow-up comment on the pull request. The maintainer triggered the project's full cross-platform CI using
`@swift-ci test`, which uncovered a Linux-specific failure in the new regression test. Investigated the CI logs and determined that the implementation was 
functioning correctly, but the regression test expected the macOS build directory (`Debug`) while Linux generates `Debug-linux-x86_64`. Updated the regression 
test to account for the Linux-specific build configuration, committed the fix, and pushed the follow-up changes to continue the review process.

Current status: Pull request is under active review and awaiting another round of CI and maintainer feedback.



### Code Changes

- **Files modified:**
Sources/SWBCore/SpecImplementations/Tools/LinkerTools.swift
Tests/SWBTaskConstructionTests/LinkerTaskConstructionTests.swift
Tests/SWBUtilTests/ResponseFileTests.swift

- **Key commits:** 
  * 690816c - Quote LinkFileList paths for non-Apple linkers [link: https://github.com/aanyabharti101/swift-build/commit/690816c1a6431f1cf297ccf8fae2636100f8591b]
  * 095a3ada - Strengthen LinkFileList quoting regression test [link: https://github.com/aanyabharti101/swift-build/commit/095a3ada52a17f621c5a697ba40cbcab18e2e102]
  * 9c36402 - Add ResponseFiles coverage for quoted paths [link: https://github.com/aanyabharti101/swift-build/commit/9c36402]
  
- **Approach decisions:** 
* Reused the existing response-file formatting infrastructure instead of introducing a new quoting mechanism.
* Limited behavior changes to non-Apple linkers to avoid altering existing ld64 behavior.
* Added a regression test using existing TaskConstruction test helpers so the fix follows established project testing patterns.

- **Branch Link:**
https://github.com/aanyabharti101/swift-build/tree/issue-13-linkfilelist-quoting

## Pull Request

**PR Link:** https://github.com/swiftlang/swift-build/pull/1477

**PR Description:** This Final PR fixes Issue #13 by ensuring that LinkFileList entries are generated using a quoted response-file format for non-Apple linkers while preserving existing behavior for Apple's ld64 linker.

**Maintainer Feedback:**
- 2026-06-23: Opened PR #1477 and requested review from project code owners.
- 2026-06-23: Awaiting maintainer review.
- 2026-06-30: Maintainer responded by triggering the project's full CI (@swift-ci test) to validate the implementation. Awaiting test results and any further review comments.
- 2026-07-01 to 2026-07-07: Investigated Linux CI failures and identified that the regression test expected a macOS-style build directory (`Debug`) while Linux generates `Debug-linux-x86_64`.
- 2026-07-08: Updated the regression test to account for the Linux-specific build directory and pushed a follow-up commit. Awaiting another round of review and CI.

**Status:** Iterating

---

## Learnings & Reflections

### Technical Skills Gained
Through this contribution I learned how Swift Build constructs linker tasks and generates response files during the build process. 
I gained experience tracing build-system behavior across multiple layers of a large codebase rather than focusing on application-level code.
I also learned how different linkers handle file-list parsing and why quoting requirements vary across platforms.
In addition, I became more comfortable reading existing test infrastructure and creating regression tests that follow established project patterns.

### Challenges Overcome
The original issue referenced non-Apple linker behavior, but development was performed on macOS.
QNX-specific tests could not be executed locally because the required SDK was not installed.
To work around this, I reproduced the issue through task-construction tests, code tracing, and inspection of generated LinkFileList contents rather than relying on platform-specific execution.
Another challenge was the new regression test is Linux-only and cannot execute on my macOS development machine.
To address this, I verified that the test is discovered and compiled successfully using swift test --filter linkFileListQuotingForNonAppleLinker and relied on code inspection plus manual reproduction steps to validate the implementation locally.

### What I'd Do Differently Next Time
Next time I would spend more time exploring the existing test suite before beginning implementation. 
Reading related tests earlier would have helped me identify the best location for regression coverage more quickly.
I would also create a small investigation document while debugging so that I can track findings, dead ends, and relevant source files more systematically throughout the contribution process.

---


## Resources Used

- Swift Build Issue #13: https://github.com/swiftlang/swift-build/issues/13
- Swift Build repository: https://github.com/swiftlang/swift-build
- GitHub Pull Request documentation: https://docs.github.com/en/pull-requests
- Swift Build source files:
  - Sources/SWBCore/SpecImplementations/Tools/LinkerTools.swift
  - Tests/SWBTaskConstructionTests/LinkerTaskConstructionTests.swift
  - Tests/SWBUtilTests/ResponseFileTests.swift
 



</details>

---






<details open>
<summary><h1>🚀 Contribution #2 – Firebase Tools</h1></summary>


# Contribution [#2]: [Display proper URLs when initializing.


**Contribution Number:** 2  
**Student:** Aanya Bharti  
**Issue:** https://github.com/firebase/firebase-tools/issues/3728  
**Status:** Phase III Complete  

---

## Why I Chose This Issue

I chose this issue because it is a focused developer-experience improvement in a widely used open-source project. 
The current behavior displays a Firebase Functions Emulator URL using 0.0.0.0, which is correct as a bind address but inconvenient for developers because it is not directly usable in a browser. 
The issue has clear maintainer guidance pointing to the relevant source file, making it a good opportunity to contribute to a large TypeScript codebase while improving the developer experience.


---

## Understanding the Issue

### Problem Description

When the Firebase Functions Emulator starts, it logs the URL for each HTTP function using 0.0.0.0 as the host (for example, http://0.0.0.0:5001/...). 
While 0.0.0.0 is the correct address for binding the emulator to all network interfaces, it is not the most useful address for developers to open in a browser. 
The issue requests that the displayed URL use 'localhost' instead, making it easier to click or copy the link during local development. 


### Expected Behavior
When the Functions Emulator is configured to bind to `0.0.0.0`, it should continue listening on all network interfaces, but the user-facing initialization message should display a browser-friendly URL such as:

`http://localhost:5001/demo-url-repro/us-central1/testHTTPSFunction`


### Current Behavior

Initial source-code investigation indicates that the original behavior may have been partially addressed since the issue was opened. 
Local reproduction confirmed that the initialization message displayed `127.0.0.1` instead of the requested `localhost`.


### Affected Components

- `src/emulator/functionsEmulator.ts` — prints the HTTP function initialization message.
- `src/emulator/functionsEmulatorUtils.ts` — contains the new display-formatting helper.
- `src/emulator/functionsEmulatorUtils.spec.ts` — contains regression tests for the helper.
- `src/emulator/registry.ts` — constructs connectable emulator URLs using the registered host and port.
- `src/utils.ts` — contains `connectableHostname()`, which converts wildcard addresses to connectable loopback addresses.


---

## Reproduction Process

### Environment Setup

I cloned my Firebase Tools fork, configured the upstream Firebase repository, synchronized my `main` branch, and created the working branch `fix-issue-3728`.
My initial environment used Node.js 25.2.1, which was not the stable version I wanted to use for the project. 
I installed `nvm` 0.40.6 and switched to Node.js 22.23.1 with npm 10.9.8. 
I then installed the project’s 1,829 packages, successfully built the TypeScript codebase and MCP applications, 
and used `npm link` to connect the `firebase` command to my local source checkout. 
The linked CLI reports Firebase Tools version 15.24.0.

The dependency installation modified `npm-shrinkwrap.json`, even though I had not intentionally changed the 
dependency configuration. I restored that unrelated generated change to keep my working branch clean.

**Working branch:** [fix-issue-3728](https://github.com/aanyabharti101/firebase-tools/tree/fix-issue-3728)


### Steps to Reproduce

1. Build the local Firebase Tools source:

   ```bash
   npm run build
   ```

2. Install the dependencies for the existing Functions Emulator test fixture:

   ```bash
   cd scripts/triggers-end-to-end-tests/v1
   npm ci
   cd ../../..
   ```

3. Start the Functions Emulator using the locally built CLI:

   ```bash
   node lib/bin/firebase.js emulators:start \
     --config scripts/triggers-end-to-end-tests/firebase.json \
     --only functions:v1 \
     --project demo-test
   ```

4. Examine the messages printed when the HTTP functions initialize.

5. Compare the displayed hostname with the requested `localhost` behavior.

### Reproduction Evidence
Local reproduction confirmed that the HTTP function initialization message displayed `127.0.0.1` instead of `localhost`, while the emulator retained `0.0.0.0` as its wildcard bind address.

- **Commit showing reproduction:** [fix-issue-3728](https://github.com/aanyabharti101/firebase-tools/tree/fix-issue-3728)  
- **Screenshots/logs:** 
#### Before Fix Reproduction: Function Initialization URL Displays `127.0.0.1`
<img width="1159" height="414" alt="Screenshot 2026-07-22 at 10 20 00 PM" src="https://github.com/user-attachments/assets/4d2dde14-3732-4c6c-b526-7492f3b83e21" />
*Before the fix, the HTTP function initialization message displayed `http://127.0.0.1:5001/...` instead of `http://localhost:5001/...`. The host/port table separately displayed `0.0.0.0:5001`, confirming that the emulator retained its wildcard bind address.*



- **My findings:**  The emulator correctly used `0.0.0.0` as its bind address, but the user-facing function initialization URL was converted to `127.0.0.1`. Therefore, the issue still existed in a modified form: the displayed URL used a numeric loopback address instead of the requested `localhost`.
The `127.0.0.1` addresses displayed in the Emulator UI and host/port summary are separate from the function-initialization messages and are outside the scope of issue #3728.
---

## Solution Approach

### Analysis

The networking behavior itself was not the root cause. Firebase still needs to retain its configured bind or connectable address internally. 
The problem occurred in the presentation layer because `functionsEmulator.ts` inserted the internally constructed URL directly into the user-facing `function initialized (...)` message.
Changing the emulator’s internal hostname could affect networking behavior. Therefore, the safer approach was to create display-specific formatting that changes only the URL shown in the terminal.


### Proposed Solution

Add a display-only helper that parses a function URL and replaces the IPv4 wildcard or loopback hostname with `localhost`. 
Apply the helper only when constructing the function-initialization message and preserve non-loopback hostnames without modification.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The Functions Emulator displays a numeric local hostname in its initialization message, while developers should see a clearer `localhost` URL. Internal networking behavior must remain unchanged.

**Match:** The codebase already contains hostname-normalization logic in `connectableHostname()`. The new solution follows a similar pattern but is scoped specifically to user-facing function URLs. Regression tests were placed alongside the existing Functions Emulator utility tests.


**Plan:** 
1. Add `formatFunctionUrlForDisplay()` to `functionsEmulatorUtils.ts`.
2. Convert `0.0.0.0` and `127.0.0.1` to `localhost`.
3. Preserve URLs containing non-loopback hostnames.
4. Add regression tests for all three cases.
5. Apply the helper to the initialization message in `functionsEmulator.ts`.
6. Build the local CLI and launch the Functions Emulator.
7. Confirm that the actual initialization messages display `localhost`.
8. Run broader related tests and repository checks.

**Implement:** 
- [Added the formatting helper and regression tests](https://github.com/aanyabharti101/firebase-tools/commit/c6a7794)
- [Applied the formatted URL to the initialization message](https://github.com/aanyabharti101/firebase-tools/commit/ce3abac)

**Review:** 

- [x] Ran Prettier on the modified files.
- [x] Ran `git diff --check`; no whitespace problems were reported.
- [x] Restored the unrelated `npm-shrinkwrap.json` modification.
- [x] Separated the implementation into two focused commits.
- [x] Rebased the branch onto the latest `upstream/main`.
- [x] Confirmed that only three issue-related files were modified.
- [x] Confirmed that the working tree was clean.
- [x] Confirmed that the branch contained only the two contribution commits.

**Evaluate:** 

- [x] Ran the three targeted regression tests.
- [x] Ran the complete related utility test file.
- [x] Built the complete local Firebase Tools CLI.
- [x] Launched the Functions Emulator using an existing test fixture.
- [x] Confirmed that initialization messages display `localhost`.
- [x] Ran the required lint and style checks.
- [x] Confirmed that existing related tests pass without regressions.
- [x] Documented the original pre-fix behavior.


---

## Testing Strategy

### Unit Tests

The following regression tests were added to `functionsEmulatorUtils.spec.ts`:

- [x] Converts the IPv4 wildcard address `0.0.0.0` to `localhost`.
- [x] Converts the IPv4 loopback address `127.0.0.1` to `localhost`.
- [x] Preserves a non-loopback host such as `192.168.1.10`.
- [x] Ran the complete `functionsEmulatorUtils.spec.ts` test file.
- [x] Confirmed that all 28 related utility tests passed with exit code `0`.

Targeted test command:

```bash
npm run mocha -- --grep "formatFunctionUrlForDisplay"
```

Result:

```text
3 passing
```

### Lint and Style Checks

- [x] Ran `npm run lint:changed-files`.
- [x] Completed with `0 errors`. The command reported repository warnings unrelated to this contribution.

### Integration Tests

- [x] Successfully built the local Firebase Tools CLI with `npm run build`.
- [x] Started the Functions Emulator using the existing trigger end-to-end test fixture.
- [x] Confirmed that all emulators reached the ready state.
- [x] Confirmed that the formatting change appeared in the actual initialization output.




### Manual Testing

I launched the Functions Emulator using the locally built Firebase Tools CLI and examined the terminal output produced when its HTTP functions initialized.

The initialization messages displayed URLs beginning with:

```text
http://localhost:9002/demo-test/us-central1/...
```

This confirmed that the fix works in the running emulator and displays `localhost` as requested.

The Emulator UI and host/port summary continued to display `127.0.0.1`. Those values are unrelated to the function-initialization message changed by this contribution and are outside the scope of the issue.


---



## Implementation Notes

### Week [7] Progress

I prepared the Firebase Tools development environment by cloning my fork, configuring the upstream repository, creating the `fix-issue-3728` branch, and installing the project dependencies.

I investigated the source files involved in constructing and displaying Functions Emulator URLs. I discovered that the original `0.0.0.0` behavior had been partially addressed by converting the hostname to `127.0.0.1`, but the requested `localhost` display behavior was still missing.

I also resolved a Node.js version mismatch by installing nvm and switching from Node.js 25.2.1 to Node.js 22.23.1. After installation modified `npm-shrinkwrap.json`, I restored the unrelated generated change to keep the branch clean.



### Week [8] Progress

This week, I traced how Firebase constructs and displays HTTP function URLs and determined that the issue should be fixed only in the user-facing initialization message.

I added `formatFunctionUrlForDisplay()`, which changes the local numeric hostnames `0.0.0.0` and `127.0.0.1` to `localhost` while preserving non-loopback hosts. I also added three focused regression tests and connected the helper to the `function initialized (...)` message.

The targeted tests completed with `3 passing`. I then built the local CLI and launched the Functions Emulator using an existing test fixture. The resulting initialization messages displayed `http://localhost:9002/...`, confirming that the implementation fixes the requested behavior in the running emulator.


### Week [9] Progress

This week, I synchronized my contribution branch with the latest upstream Firebase Tools repository by rebasing it onto `upstream/main` and updating my fork with `--force-with-lease`.

I then repeated the project validation after the rebase. The three focused regression tests passed, and the complete related utility test file completed with `28 passing` and exit code `0`. The changed-file lint check completed with `0 errors`, and the full Firebase Tools build completed twice with exit code `0`.

The complete Mocha suite reported `4,785 passing` and `10 pending` with no test failures before the local process was terminated with exit code `137`. I documented this limitation and relied on the separately completed related test file for a clean local result.

Finally, I launched the Functions Emulator again and confirmed that its HTTP function initialization messages displayed `http://localhost:9002/...`. A final Git review confirmed a clean working tree, two focused commits, and only the three intended files changed.





### Code Changes
- **Files modified:** 
- `src/emulator/functionsEmulatorUtils.ts`
  - Added `formatFunctionUrlForDisplay()`.

- `src/emulator/functionsEmulatorUtils.spec.ts`
  - Added three regression tests covering wildcard, loopback, and non-loopback hosts.

- `src/emulator/functionsEmulator.ts`
  - Applied the helper to the user-facing function-initialization message.
  
- **Key commits:** 
- [`c6a779431` — Format local function URLs for display](https://github.com/aanyabharti101/firebase-tools/commit/c6a779431)
- [`ce3abacab` — Display the formatted URL during initialization](https://github.com/aanyabharti101/firebase-tools/commit/ce3abacab)

- **Approach decisions:** 
I used a separate display-formatting helper instead of changing the emulator’s internal hostname. This keeps the networking behavior intact and limits the change to the terminal output requested by the issue.

I tested both `0.0.0.0` and `127.0.0.1` because the original issue referenced the wildcard address, while the current codebase can first convert that address to the numeric loopback address.

I also included a non-loopback test case to confirm that the helper does not incorrectly replace valid network hostnames.

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]


### Challenges Overcome

- I switched to Node.js 22.23.1 to use a stable version compatible with the repository.
- Installing dependencies modified `npm-shrinkwrap.json`, so I restored the unrelated change to keep my contribution focused.
- The complete test suite finished without reporting test failures, but macOS terminated the process afterward because of a local resource limitation. I therefore ran the directly related utility test file separately and confirmed that it passed with exit code `0`.
  

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Firebase Tools issue #3728](https://github.com/firebase/firebase-tools/issues/3728)
- [Firebase Tools contributing guide](https://github.com/firebase/firebase-tools/blob/main/CONTRIBUTING.md)
- [Firebase Tools source repository](https://github.com/firebase/firebase-tools)



</details>

---



