<!-- markdownlint-disable MD007 -- Unordered list indentation -->
<!-- markdownlint-disable MD010 -- No hard tabs -->
<!-- markdownlint-disable MD033 -- No inline html -->
<!-- markdownlint-disable MD055 -- Table pipe style [Expected: leading_and_trailing; Actual: leading_only; Missing trailing pipe] -->
<!-- markdownlint-disable MD041 -- First line in a file should be a top-level heading -->
# Requirements

This is a product backlog just for pre-v1.0.0 release. After that, bugs, features, and enhancements will be managed in Github Issues.

<!-- TOC ignore:true -->
## Table of contents

<!-- TOC -->

- [Issues](#issues)
- [Template](#template)

<!-- /TOC -->

## Issues

Sorted per the notes in [Template](#template).

- The planned container cleanup prunes every project's docker build cache
	- ID: 2026110311291919
	- Type: Bug
	- Status: Queued
	- Priority|Severity: High
	- Opened: 20261103-112919
	- Opened by: JC
	- Related IDs: 2026102506554101, common 2026110311291900
	- Incorrect behavior: Item 2026102506554101 plans `docker builder prune -f --filter until=168h` at the end of a run, from common's `Cleanup after a run` rule. That prunes docker's default builder, which every project shares, so it would drop any other project's cache once it's a week unused.
	- Expected behavior: Leave the prune out of the cleanup. Docker trims the cache itself on b23, through `builder.gc` in `/etc/docker/daemon.json`. Cache unused for a week goes first, then the rest is cut to 120 GB, or further when the disk is under 10% free, but never below 20 GB. The rest of the cleanup item stands.
	- Decisions:
		- Don't move to a buildx builder of the project's own instead. shcl tried that on 2026-11-02. A cold build of a 2 GB image took 299 s against 266 s on the default builder, and the builder kept a second 3 GB copy of the layers for a week. shcl is going back to the default builder. (JC, me)
	- Estimated effort: Low

- Pre-next-cut: Updates to CICD script flags (do next)
	- ID: 2026101911150300
	- Type: Enhancement
	- Status: Queued
	- Priority|Severity: High
	- Opened: 20261019-111500
	- Opened by: t00mietum
	- Related IDs: 2026101911150304, 2026102506554108
	- Requirements:
		- Sweeping override, 2026-10-24: this supersedes any earlier decision on this project that conflicts with it.
		- 2026-10-25: this is now the reset's standard flags item, and the shared flags replace the list below where they differ.
			- None for the default run under about an hour, `--quick`, `--long` with `--slow` as another name, `--leakcheck`, `--no-leakcheck`, `--backlog-run-quick [<area>...]`, `--backlog-run-full`, `--merge-gate`, `--no-container`, `--no-remote`, `--no-sync`, `--keep-temp`, `--list`, `--help`, `--quiet`, and `--message`.
			- `--ci` becomes `--merge-gate`, not `--gate`. `--gate` and `--ci` stay as other names for `--backlog-run-quick` and `--merge-gate`.
			- The backlog and merge gate runs never dogfood, publish, commit, push, install, sync, rewrite a tracked file, or prompt.
			- A pre-push hook runs `--merge-gate` on a push to `main` only. A push to `dev` or a feature branch isn't gated, and a docs-only push to `main` skips it, with the docs paths in one place the hook reads.
		- Rename `--ci` to `--gate`.
		- `--quick`:
			- Runs whatever linting, validation, and testing steps are very quick.
			- Cross-compiles whatever is fast. E.g. maybe skip slow arm64.
			- Skips anything requiring an external host.
		- No flags by default does everything.
		- The other existing flags can stay.

- CICD reset: containers
	- ID: 2026102506554100
	- Type: Task
	- Status: Queued
	- Priority|Severity: High
	- Opened: 20261025-065541
	- Opened by: t00mietum
	- Related IDs: 2026102506554108
	- Requirements:
		- Sweeping override, 2026-10-24: this supersedes any earlier decision on this project that conflicts with it.
		- There's no code or pipeline yet, so this goes in with the first crate.
		- Build and test in a pinned image from a recipe in the repo, with the Rust toolchain and the X11 and Wayland dev libraries pinned.
		- The headless clipboard tests can run in it, on a private X server and a headless Wayland compositor. Checks against a real panel's tray run on the host.
		- A missing docker falls back to the host with one line, and never fails a run. `--no-container` runs on the host on purpose.

- CICD reset: container cleanup
	- ID: 2026102506554101
	- Type: Task
	- Status: Queued
	- Priority|Severity: High
	- Opened: 20261025-065541
	- Opened by: t00mietum
	- Prereq IDs: 2026102506554100
	- Related IDs: 2026101911150302
	- Requirements:
		- Sweeping override, 2026-10-24: this supersedes any earlier decision on this project that conflicts with it.
		- Every container, image, and volume gets `dev.project=tukzclip` and `dev.run=<run folder name>`, and containers are named `tukzclip-<run folder name>-<step>`.
		- Containers run with `--rm`, unless `--keep-temp` was passed. The run's containers are removed by label at its end, on failure and Ctrl+C too.
		- The project's dangling images go, and older tags once a newer one has passed, keeping the current tag and the one before it. Then `docker builder prune -f --filter until=168h`.
		- Nothing without the project's label is touched. The end of the run says what it removed and about what that freed.

- CICD reset: CPU and priority
	- ID: 2026102506554102
	- Type: Task
	- Status: Queued
	- Priority|Severity: High
	- Opened: 20261025-065541
	- Opened by: t00mietum
	- Requirements:
		- Sweeping override, 2026-10-24: this supersedes any earlier decision on this project that conflicts with it.
		- Half the logical CPUs, or a quarter when the 1-minute load is over half the CPU count, checked at the start and before each step. The heavy steps, such as the leak checks and the soak test, use a quarter, busy or not.
		- The counts are worked out once and passed to cargo's `-j` and `--test-threads`, and to anything else that runs in parallel.
		- Everything runs under `nice -n 19`, plus `ionice -c 3`. Containers get `--cpus` and `--cpu-shares 256`, and `nice` inside.

- CICD reset: step output
	- ID: 2026102506554103
	- Type: Task
	- Status: Queued
	- Priority|Severity: High
	- Opened: 20261025-065541
	- Opened by: t00mietum
	- Requirements:
		- Sweeping override, 2026-10-24: this supersedes any earlier decision on this project that conflicts with it.
		- Each step prints the shared lines: a 79 character rule, `[ YYYYmmDD-HHMMSS.NN: Step 1 of 9; Starting Format ]`, its output, then `PASS Format` or `FAIL Format` in the same form.
		- Steps turned off aren't counted, and the first section lists them with the flag or reason.

- CICD reset: a test for every Done item
	- ID: 2026102506554104
	- Type: Task
	- Status: Queued
	- Priority|Severity: High
	- Opened: 20261025-065541
	- Opened by: t00mietum
	- Requirements:
		- Sweeping override, 2026-10-24: this supersedes any earlier decision on this project that conflicts with it.
		- No item is Done yet, so nothing is missing a test today.
		- Every Done feature and bug from now on names its test's ID in `Test case`, or says in a few words why it has none.

- CICD reset: test IDs
	- ID: 2026102506554105
	- Type: Task
	- Status: Queued
	- Priority|Severity: High
	- Opened: 20261025-065541
	- Opened by: t00mietum
	- Requirements:
		- Sweeping override, 2026-10-24: this supersedes any earlier decision on this project that conflicts with it.
		- Every test gets a unique ID, from when it was first committed: whole minutes since 2000-01-01 UTC, in lower-case Crockford base 32. 2 that come out the same take the next free minute, and an ID never changes.
		- The run prints one line per test, such as `PASS decp8 text copy outlives its app`, or `FAIL` or `SKIP`.
		- `Test case` rows in the backlog name the IDs.

- Pre-next-cut: Builds clean up stale artifacts and side effects
	- ID: 2026101911150302
	- Type: Enhancement
	- Status: Queued
	- Priority|Severity: Avg
	- Opened: 20261019-111500
	- Opened by: t00mietum
	- Related IDs: 2026102506554101, 2026102506554106
	- Requirements:
		- Sweeping override, 2026-10-24: this supersedes any earlier decision on this project that conflicts with it.
		- 2026-10-25: containers are now 2026102506554101, and temp files 2026102506554106. This item keeps the rest.
			- Build caches older than the current toolchain and lockfile are pruned. A warm cache for the current ones stays.
			- Old artifacts beyond what GFS rotation keeps go.
			- One line at the end says what was removed and roughly what it freed.
			- Checked by running the pipeline twice, killing one partway, and seeing nothing new pile up.
		- Verify or ensure that builds clean up *stale* artifacts and side effects when they're done: caches, container cruft, tmp files, etc.

- Pre-next-cut: CICD `--leakcheck` and `--no-leakcheck`
	- ID: 2026101911150304
	- Type: Enhancement
	- Status: Queued
	- Priority|Severity: Avg
	- Opened: 20261019-111500
	- Opened by: t00mietum
	- Parent ID: 2026101714414461
	- Related IDs: 2026101911150300
	- Requirements:
		- Sweeping override, 2026-10-24: this supersedes any earlier decision on this project that conflicts with it.
		- 2026-10-25: under the reset, `--leakcheck` adds every leak check and `--no-leakcheck` takes every one out, fast ones included. The rest of the flags are in 2026101911150300.
		- CICD has both `--leakcheck` and `--no-leakcheck`.
			- `--leakcheck` is useful to add back in for `--quick`.
			- `--no-leakcheck` is useful to exclude from the default, or with `--quiet`.

- tukzclip: our own clipboard program. Keeps the clipboard after the app that copied closes, and bridges the Wayland and X11 clipboards.
	- ID: 2026101509021629
	- Type: Feature
	- Status: Queued
	- Opened: 20261015-090216
	- Opened by: t00mietum
	- Requirements:
		- A separate project under the tukzedo umbrella.
		- Named 'tukzclip'.
		- A program that shows up as an icon in the system tray.
		- By default, it does two things:
			- Keeps the clipboard contents in memory for a predefined length of time (default 60m).
			- Bridges Wayland and X11 clipboard.
		- Additional optional features, off by default:
			- Convert the contents to (UTF-8) text-only.
			- Run a command-line with the clipboard contents as the argument.
			- Keep a history a user-adjustable X slots long, and for at most Y minutes each.
			- Persist history between runs. (With a warning about secrets being on storage.)
	- Note: we need a clipboard bridge anyway, even if we weren't using Wayfire, because X11 has a "feature" (bug), where if you copy contents from say a text editor, then close that editor before pasting it somewhere, the clipboard contents disappear. This is "surprising", to say the least. Since a clipboard is "global" to a user's session and works between apps, you'd expect that closing the program doesn't clear it. I get that it might be due to "security concerns", but that is a total half-measure.
	- Note: Wayland does the same. The clipboard belongs to the app that set it, and is gone when that app exits.
	- Note: moved here from the lxqtukz backlog, where the paste bug (2026101417551800) led to it.

- Leak, memory, and static checks in CI/CD, for all code
	- ID: 2026101714414461
	- Type: Task
	- Status: Queued
	- Priority|Severity: Avg
	- Opened: 20261017-144144
	- Opened by: t00mietum
	- Requirements:
		- The checks cover first-party and third-party code. Vendored code is included.
			- On third-party code, run the leak, memory, and correctness checks named here, not style lints. Style findings in someone else's crate or module aren't ours to fix.
			- CodeQL reports only on source inside the database's root. For its run, vendor the dependencies into the tree first (`cargo vendor`, `go mod vendor`, unpacked sdists for Python).
		- Any report fails the test that made it.
		- Any leak is fixed. If the fix has to wait, it gets a backlog item, and the test that shows it turns the check off with that item's ID beside it.
		- A `--leakcheck` flag adds the slow checks:
			- CodeQL, from a pinned CLI run locally. No hosted service.
			- A soak test. It loops through the program's main actions headless, and fails if the heap at round 2N is larger than at round N by more than a set margin. The heap report is kept as a pipeline artifact.
		- Every tool is pinned and runs in the build container or lint image, never installed on the host.
		- Each new gate is watched to fail before it's trusted, by planting the leak it's meant to catch and seeing it go red.
		- Known-advisory scans run on every full run: `cargo deny check advisories` or `cargo audit`, `govulncheck`, and `pip-audit`. They cover third-party code only, which is the point.

- Leak, memory, and static checks in CI/CD, for Rust
	- ID: 2026101714414463
	- Type: Task
	- Status: Queued
	- Priority|Severity: Avg
	- Opened: 20261017-144144
	- Opened by: t00mietum
	- Parent ID: 2026101714414461
	- Requirements:
		- Safe Rust leaks only through `Rc` or `Arc` cycles, `mem::forget`, `Box::leak`, and collections that grow forever. Real leaks and memory errors come from `unsafe` and FFI, in our crates or in dependencies.
		- Every full run builds the whole test suite with AddressSanitizer and LeakSanitizer, in a build dir of its own.
			- `RUSTFLAGS=-Zsanitizer=address cargo +NIGHTLY test -Zbuild-std --target x86_64-unknown-linux-gnu`.
			- That needs a dated nightly, pinned beside the stable toolchain. Sanitizers aren't stable.
			- Every crate built from source is instrumented, dependencies included.
		- If the code starts threads, every full run also builds the tests with ThreadSanitizer (`-Zsanitizer=thread`), in another build dir.
		- Every full run runs the test suite under Miri: `cargo +NIGHTLY miri test`. It catches undefined behavior in `unsafe` code, ours and dependencies', and fails on memory still allocated at exit.
			- It moves behind `--leakcheck` only if a full run measures too slow.
		- A heap-growth test with `dhat` in testing mode reads `dhat::HeapStats` at round N and 2N in-process. It's the soak test for a library with no main program.
		- clippy runs with `-D warnings`, plus `clippy::mem_forget` and `clippy::undocumented_unsafe_blocks`.
		- Every `unsafe` block has a `// SAFETY:` comment.
		- Traps:
			- Without `-Zbuild-std`, std isn't instrumented. ASan mostly copes, but TSan gives false reports.
			- Without an explicit `--target`, the sanitizer flags reach build scripts and proc macros, and the build fails in odd places.
			- LSan reports at process exit, so a leak fails the whole test binary, not one test. Rerun with `--test-threads=1` and a test-name filter to find which.
			- Miri can't call FFI and has limited file and network access. A test that needs them gets `#[cfg_attr(miri, ignore)]` with a one-line reason. It's not silently left out.
			- Miri doesn't count memory reachable from a `static` as a leak, so a cache in a static that grows forever passes. The soak test catches that.
			- heaptrack and dhat both see only the global allocator. A crate that maps memory itself, or a custom allocator, hides it.
			- ASan and TSan can't share a build, and heaptrack can't share one with either. The soak test uses a plain build.
	- Note: tukzclip is planned in Rust, per `design.md`. No code yet, so the gates go in with the first crate.

- CICD reset: temp files and safe removal
	- ID: 2026102506554106
	- Type: Task
	- Status: Queued
	- Priority|Severity: Avg
	- Opened: 20261025-065541
	- Opened by: t00mietum
	- Related IDs: 2026101911150302
	- Requirements:
		- Sweeping override, 2026-10-24: this supersedes any earlier decision on this project that conflicts with it.
		- Each run's folder is `<HOST_TEMP>/tukzclip/<YYYYmmDD-HHMMSS>_<NNN>/`, made with a plain `mkdir` and a counter, with `TMPDIR`, `TEMP`, and `TMP` pointed into it. Unit tests' scratch folders and any saved history file go there too.
		- A trap removes it at the end, on failure and Ctrl+C too, through the shared guard. No bare `rm -rf`, and also no unguarded removal anywhere else. `--keep-temp` keeps it and prints its path.
		- Each run writes `run.pid` there, and removes the project's own folders older than 2 days whose process is gone.
		- A test that starts an X server, a compositor, a session bus, or a clipboard owner stops it, on failure too. Tests never touch the desktop session's own clipboard.

- CICD reset: version line
	- ID: 2026102506554107
	- Type: Task
	- Status: Queued
	- Priority|Severity: Avg
	- Opened: 20261025-065541
	- Opened by: t00mietum
	- Requirements:
		- Sweeping override, 2026-10-24: this supersedes any earlier decision on this project that conflicts with it.
		- `tukzclip --version` and the tray menu's About print `tukzclip v<version> build <build>`, then the copyright line. The build number is the commit's time in minutes since 2000, in lower-case Crockford base 32.
		- A non-release build adds its date after the build number, from that same time, in the viewer's time zone.
		- The version comes from one place, read by both the tag and the build stamp.

- CICD reset: CICD process gaps
	- ID: 2026102506554108
	- Type: Task
	- Status: Queued
	- Priority|Severity: Avg
	- Opened: 20261025-065541
	- Opened by: t00mietum
	- Related IDs: 2026101911150300, 2026101714414461
	- Requirements:
		- Sweeping override, 2026-10-24: this supersedes any earlier decision on this project that conflicts with it.
		- A `cicd.bash` per the shared CICD rules, with the first crate: remote sync first, format, build, lint, tests, fuzz, release build, dogfood, and cleanup.
		- `rust-toolchain.toml` pins the toolchain, and lint and audit tools are pinned in one place.
		- Lint output rotated into `cicd/artifacts/lint/`, with a `--check` that prints only what's new.
		- Fuzzing of what tukzclip reads from other apps: odd MIME types, huge and empty offers, and offers that vanish mid-read.
		- A pre-push hook for the merge gate, per 2026101911150300.
		- `.deb` and `.rpm` packages, and a profiler stage.
		- The leak checks are already filed, under 2026101714414461 and 2026101714414463.

**Stop here for a next release**

## Template

- Notes:

	- Only use rows that you actually need or expect will be filled in. Always fill in the title, ID, Type, Status, Opened and Created by.

	- The ID is the local time to the hundredth of a second. Opened is when it was written down, which may differ. (Use a keyboard macro and possibly something like project 'zuid' to generate.)

	- Status values meaning: Testing means the fix is in and checks are running or still to run. Waiting on signoff means automated testing passed. Moot means something else changed that made it irrelevant. Canceled means it still applies but was decided against. Waiting for testing means the fix is in and waits on a long CI run or an outside test host. Can't reproduce means a real attempt to reproduce it failed.
		- An item with nothing left for a person to judge skips Waiting on signoff and goes straight to Done. `Clean up the backlog` in the directives says when.

	- As issues are worked, and statuses change, place them in correct sorting order within the list:
		- First by status: Waiting for answers, Waiting on signoff, Testing, Waiting for testing, Can't reproduce, Stalled, Started, Queued, Done, Deferred, Canceled, Moot
		- Then by severity|priority: Critical, High, Avg, Low
		- Then by type: Bugs, [not bugs together]

	- Rows marked [Bug] are for bugs only, and rows marked [Feature] for features and enhancements. Priority and Severity share one row and one scale. Priority is for a Feature or Enhancement, and Severity for a Bug. Children are not nested. They sit at the top level and point back with Parent ID.

Template:

- Title
	- ID: YYYYmmDDHHMMSSNN
	- Type: [Bug|Feature|Enhancement|Task]
	- Status: [Queued|Waiting for answers|Waiting on signoff|Waiting for testing|Started|Testing|Stalled|Can't reproduce|Moot|Canceled|Deferred|Done]
	- Needs local test suite run?:
	- Needs external testing:
	- Priority [Feature|Enhancement] | Severity [Bug]: [Critical|High|Avg|Low]
	- Opened:
	- Opened by:
	- Assigned to:
	- Parent ID:
	- Prereq IDs:
	- Related IDs:
	- Target OS:
	- Test environment:
	- Version and build:
	- Requirements  [Feature]:
		- Hierarchical bulleted list.
	- Steps to reproduce [Bug]:
		- …
	- Incorrect behavior [Bug]:
	- Expected behavior [Bug]:
	- Reproduced [Bug]: [No, or when, where and how]
	- Possible cause [Bug]:
	- Actual cause [Bug]:
		- …
	- Estimated effort: [High|Avg|Low]
	- Actual effort: [High|Avg|Low]
	- Progress log:
		- …
	- Decisions:
		- …
	- Actual fix [Bug]:
	- Branch:
	- Commit:
	- Test case: [Reason not applicable, or CI test case #]
	- Acceptance signoff:
	- Superseded by ID:
	- Closed:
