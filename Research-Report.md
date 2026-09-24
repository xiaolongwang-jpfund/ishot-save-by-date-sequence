# Research Report: iShot Save by Date Sequence

Date: 2026-09-24
Status: Research complete; implementation not started.

## Executive Conclusion

The project is technically feasible on this Mac without administrator privileges or a background service. The recommended solution is one Automator Folder Action attached to the iShot destination folder. It should invoke a small local script only for the files passed to it, rename each supported image to `MM-DD-NN.ext`, and reveal the successfully renamed file in Finder.

This is the best option because macOS runs the workflow when an item is added to the folder. It is simpler and less error-prone than a continuously running watcher, and it does not scan or rename old images.

## Evidence Collected

### Local Mac

- The Mac is an Apple M1 Pro running macOS 27.0, so it meets the platform requirement.
- The target folder `~/Downloads/2026/Images` exists and currently contains 166 files.
- The folder contains multiple image formats and files from more than one source. Any automation must act only on the newly added file supplied by the Folder Action; it must never batch-rename the existing folder contents.
- The IT Guy visit log records that a real Folder Action passed synthetic PNG/JPEG renaming and Finder reveal checks on 2026-09-23. This is strong evidence that the design works on this Mac.
- The standard Folder Action preference file and common workflow locations did not expose a current matching configuration during this read-only check. The prior successful setup therefore cannot be assumed to remain present or correct; perform one non-sensitive live test before treating it as operational.

### Public Documentation

- iShot's App Store listing states that it supports local saving, custom file names, and PNG, JPG, and TIFF output formats.
- Apple documents Folder Actions as workflows that run when items are added to an attached folder.
- Apple documents Automator Folder Actions as receiving the files added to the folder as workflow input.

Sources:

- https://apps.apple.com/jp/app/ishot-screenshot-recording-ocr/id1485844094?mt=12
- https://developer.apple.com/library/archive/documentation/LanguagesUtilities/Conceptual/MacAutomationScriptingGuide/WatchFolders.html
- https://support.apple.com/en-hk/guide/automator/aut7cac58839/2.10/mac/15.0

## Recommended Design

### Components

1. iShot saves a screenshot locally into `~/Downloads/<year>/Images`.
2. An Automator Folder Action is attached to that folder.
3. The Folder Action receives only the newly added item paths and passes them to a local renaming script.
4. The script validates the file, derives `MM-DD` from its local modification time, allocates the next two-digit sequence number for that date, renames without overwriting, and asks Finder to reveal the renamed file.

### Required Safety Rules

- Process regular image files only: PNG, JPG, JPEG, HEIC, and TIFF, case-insensitively.
- Ignore folders, hidden files, temporary files, and unsupported formats.
- Never scan the folder as the work list. Use only the files received from the Folder Action event.
- Wait briefly for a newly written file to become stable before renaming it; a screenshot application may create the file before it has finished writing.
- Determine the sequence by checking all supported image extensions for the same `MM-DD` prefix. The next number must be greater than every existing number for that date.
- Use an exclusive lock during sequence allocation so two near-simultaneous screenshots cannot receive the same number.
- Never overwrite a file. If a collision remains after sequence allocation, stop and report it instead of replacing a file.
- Record the original and new names in a local undo manifest before each rename. The manifest must not be committed to GitHub.
- Reveal the renamed item only after the rename succeeds. If several files arrive together, reveal each only after its own successful result.

## Privacy and Permission Considerations

- The workflow works locally. It does not need to send screenshots to a server.
- Dragging a screenshot into Codex is a separate, user-controlled upload action. Before doing so, inspect the screenshot for personal information, passwords, API keys, access tokens, account pages, orders, emails, or other sensitive material. Crop, redact, or do not upload sensitive images.
- macOS may request Files and Folders permission for Downloads, Finder automation permission, or Automation permission for Automator. These prompts require user approval. No administrator password should be required.
- Do not commit screenshots, undo manifests, logs, `.env` files, credentials, or personal paths to GitHub. Review changes and sensitive information before every commit and push.

## Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| Existing images are renamed accidentally | Process only the event-provided new files; never run a folder-wide batch rename. |
| A duplicate name replaces an image | Use a lock and collision check; never overwrite. |
| The file is renamed before iShot finishes saving it | Check that the file is stable before renaming. |
| The Finder reveal interrupts work | Reveal only after success; test the user experience with one screenshot first. |
| A macOS privacy prompt blocks access | Let the user approve only the required Downloads/Finder permission, then re-test. |
| Sensitive screenshot content is uploaded to Codex or GitHub | Keep processing local and require manual privacy review before any upload or commit. |

## Implementation Plan

1. Confirm iShot's configured save folder and output formats in its user interface.
2. Build the renaming script with a dry-run mode and a synthetic test folder.
3. Test the dry-run using PNG, JPG, JPEG, HEIC, and TIFF sample files, including an existing sequence number and a simulated name collision.
4. Create and attach the Automator Folder Action to the target folder only after the user approves the plan.
5. Perform a one-file live test with a non-sensitive screenshot created by the user.
6. Verify the final name, source-to-destination record, Finder reveal, and that the original image content remains unchanged.
7. Run a two-file near-simultaneous test to verify sequence locking.
8. Write a short user guide and keep the script, guide, and tests in the repository. Keep logs, manifests, screenshots, and local settings out of Git.

## Acceptance Criteria

- A newly saved supported image in the configured folder is renamed once to `MM-DD-NN.ext`.
- `NN` is the next available two-digit sequence for that date across supported image extensions.
- No existing image is overwritten, renamed, or deleted.
- The renamed image appears in Finder after a successful rename.
- Unsupported files and existing images are untouched.
- The workflow operates locally, without administrator privileges or network access.
- A user can confirm the changed name and reverse the rename from the local undo manifest.
