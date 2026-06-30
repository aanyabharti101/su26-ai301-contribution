# su26-ai301-contribution

# 🚀 AI301 Open Source Contribution Journal

This repository documents my open-source contributions for CodePath AI301.

## 📚 Contributions

- [Contribution #1 – LinkFileList generation needs to use quoting](#contribution-1-linkfilelist-generation-needs-to-use-quoting)
- [Contribution #2 – Display proper URLs when initializing](#contribution-2-display-proper-urls-when-initializing)

| # | Project | Issue | Status |
|---|---------|-------|--------|
| 1 | Swift Build | LinkFileList generation needs to use quoting | ✅ Phase IV Complete |
| 2 | Firebase Tools | Display proper URLs when initializing (#3728) | 🟡 Phase I |

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

**Status:** Awaiting Review

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
]

**Contribution Number:** 2  
**Student:** [Your Name]  
**Issue:** [GitHub issue link]  
**Status:** [Phase I / Phase II / Phase III / Phase IV] [In Progress / Complete]

---

## Why I Chose This Issue

[1-2 paragraphs explaining why this issue interests you, how it matches your skills/learning goals, what you hope to learn]

---

## Understanding the Issue

### Problem Description

[In your own words, what's broken or missing?]

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

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

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]



</details>

---



