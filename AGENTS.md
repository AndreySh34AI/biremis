# Project instructions

This repository is a fork of Telegram-iOS.

The project implements a Restricted Mode in which different PIN codes
select different observable Telegram states.

## Primary rules

1. Never modify code outside the scope explicitly requested by the task.

2. Do not perform unrelated refactoring.

3. Before changing Telegram-iOS architecture, inspect the actual current
   implementation.

4. Documentation under `restricted_mode/` defines project requirements.

5. `restricted_mode/SPEC.md`,
   `restricted_mode/THREAT_MODEL.md`, and
   `restricted_mode/SECURITY_INVARIANTS.md`
   are authoritative.

6. Agents must not silently weaken requirements in those documents.

7. If implementation conflicts with a security invariant, stop and
   document the conflict instead of bypassing the invariant.

8. Full Mode must preserve normal upstream Telegram behavior.

9. Restricted Mode must not be implemented solely as UI filtering.

10. Always consider leakage through:
    - Postbox
    - search
    - notifications
    - recent peers
    - shared media
    - caches
    - extensions
    - background processing
    - Spotlight
    - Siri / App Intents
    - logs

11. Treat the current source code as the source of truth for the upstream
    Telegram-iOS implementation.

12. Before creating new infrastructure, search the repository for an
    existing Telegram-iOS mechanism that can be safely reused or extended.

13. Keep Restricted Mode changes isolated from upstream Telegram-iOS code
    where reasonably possible, to simplify future upstream merges.


## Development environment

Development platform:
- macOS
- Apple Silicon
- Xcode 26.6
- Python 3
- Bazel-based Telegram-iOS build system

Project root:

    /Users/andrey/Work/Telegram-iOS

Bazel cache:

    ~/telegram-bazel-cache

Development configuration:

    local-config/development.json


## Project generation

Generate the Xcode project with:

    python3 build-system/Make/Make.py \
      --overrideXcodeVersion \
      --cacheDir="$HOME/telegram-bazel-cache" \
      generateProject \
      --configurationPath=local-config/development.json \
      --xcodeManagedCodesigning \
      --disableProvisioningProfiles

The installed Xcode version is 26.6 while the upstream repository may
expect Xcode 26.2.

Do not remove `--overrideXcodeVersion` merely to resolve this version
mismatch.


## Autonomous build workflow

When a task involves source-code changes, Codex should perform the
development/build/debug cycle itself whenever the required commands are
available.

The normal workflow is:

1. Read the relevant Restricted Mode requirements.

2. Inspect the actual current Telegram-iOS implementation.

3. Identify affected subsystems and files.

4. Make the smallest change that satisfies the task and security
   requirements.

5. Build the affected project or target.

6. Read and analyze the build output.

7. If the build fails:
   - identify the first meaningful error;
   - determine its root cause;
   - inspect the relevant source/configuration;
   - fix errors caused by the implementation;
   - build again.

8. Repeat the build/fix cycle until:
   - the build succeeds; or
   - progress requires information, credentials, permissions, or a
     design/security decision from the user.

Do not ask the user to manually run build commands and paste their output
when the agent can execute those commands itself.

Do not consider a source-code task complete until the affected code has
been built successfully, unless building is impossible for a clearly
reported reason.


## Build failure analysis

When a build fails, do not assume that the final Bazel/Xcode error line
is the root cause.

Find the first meaningful compiler or build-system error and inspect its
context.

Distinguish between failures caused by:

- Swift / Objective-C compilation
- Bazel configuration
- Xcode/toolchain configuration
- project generation
- dependencies
- code signing
- provisioning
- simulator/device configuration
- generated files
- environment mismatch

Fix the root cause rather than suppressing the symptom.

Do not weaken security requirements merely to make the project compile.


## Configuration safety

Treat `local-config/development.json` as sensitive local configuration.

Do not change:

- api_id
- api_hash
- team_id
- bundle_id
- signing configuration

unless explicitly requested.

Never print or copy credentials into reports, documentation, source files,
or commits.


## Git safety

The working tree may contain user changes.

Before substantial modifications, inspect `git status`.

Never discard unrelated user changes.

Do not run destructive commands such as:

    git reset --hard
    git clean -fd
    git checkout -- .
    git restore .

unless explicitly requested.

Do not commit, push, force-push, or rebase unless explicitly requested.


## Development workflow

Before implementation:
- read the relevant documents under `restricted_mode/`;
- inspect relevant source;
- document intended changes;
- identify affected subsystems;
- identify relevant security invariants.

During implementation:
- prefer minimal and localized changes;
- preserve existing Telegram-iOS architecture where possible;
- avoid unrelated cleanup and refactoring;
- periodically build meaningful intermediate changes when useful.

After implementation:
- build;
- run relevant tests;
- inspect `git diff`;
- document modified files;
- report unresolved security assumptions;
- report anything that could not be verified.

Do not commit unless explicitly requested.


## Completion criteria

A task is complete only when:

1. The requested behavior is implemented.

2. Relevant security invariants remain satisfied.

3. Full Mode behavior remains unchanged unless the task explicitly
   requires otherwise.

4. The affected project/target builds successfully.

5. Relevant tests have been run when available.

6. The final diff has been inspected for accidental unrelated changes.

7. Remaining assumptions or unverified behavior are explicitly reported.


## Final report

At the end of an implementation task, report concisely:

- what was changed;
- which important files were modified;
- what was built;
- build result;
- tests performed;
- unresolved issues;
- unresolved security assumptions.

If the build succeeds, explicitly state that it succeeded.

If something could not be verified, state exactly what was not verified
and why.