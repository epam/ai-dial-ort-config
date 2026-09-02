---
name: ort-package-curations
description: 'Create or update ORT package curations to resolve license rule violations. Use when the user pastes ORT/evaluator output containing violations such as NO_LICENSE_IN_DEPENDENCY, UNMAPPED_DECLARED_LICENSE, MISSING_CONTRIBUTING_FILE, COPYLEFT_IN_DEPENDENCY, COPYLEFT_LIMITED_IN_DEPENDENCY, or any missing/invalid/unmappable/incompletely-detected license metadata for a package (PyPI, NPM, Maven, Go, Gem, NuGet, Composer, Crate, Pod). Independently verifies the REAL license from package metadata and source, then writes a curation file with declared_license_mapping or concluded_license using relaxed Ivy-style version ranges.'
argument-hint: 'Paste the ORT rule violation(s) or the package coordinates'
---

# ORT Package Curations

Resolve ORT license rule violations by verifying a package's real license and writing a curation
into this repo. Curations amend package metadata during `ort analyze`/`evaluate`.
Reference: <https://oss-review-toolkit.org/ort/docs/configuration/package-curations>

## When to Use

Trigger on ORT/evaluator output about a package, especially:

- `UNMAPPED_DECLARED_LICENSE` - means license is found but declared string is not a valid SPDX identifier.
- `NO_LICENSE_IN_DEPENDENCY` - means no license metadata found.
- `GENERIC_IN_DEPENDENCY` - a generic trigger (e.g. `LicenseRef-scancode-proprietary-license`) was **detected** in the files; it is not a real license. Needs `concluded_license` (a `declared_license_mapping` won't clear a detected finding).
- `COPYLEFT_IN_DEPENDENCY` / `COPYLEFT_LIMITED_IN_DEPENDENCY` - a copyleft license was detected.
- `UNHANDLED_LICENSE` - the license is correct but not covered by policy rules yet.
-
- Any other invalid, missing, non-SPDX, or incompletely-detected declared license.

## Procedure

### 1. Parse the violations

Extract per violation: **type**, **namespace**, **name**, **version**, and the **declared license string** (if any).

> [!tip]
> Package IDs are `<Type>:<namespace>:<name>:<version>`, e.g. `PyPI::httpcore:1.0.9` or
`Maven:androidx.core:core:1.2.0` (PyPI/NPM namespace is empty).

ORT often emits several violations for the same package. Group them by package id and act on the **authoritative** finding; the rest are usually side effects that clear once it's fixed.
Precedence: a finding that implies that _some_ (even non-compliant, or unmapped) license was found beats a generic trigger. Examples:

- `PROPRIETARY_FREE_IN_DEPENDENCY` > `GENERIC_IN_DEPENDENCY`
- `UNMAPPED_DECLARED_LICENSE` > `NO_LICENSE_IN_DEPENDENCY`

### 2. Verify the REAL license

Verify all sources independently. Registry metadata _may be_ wrong or incomplete and does
NOT win: when the source code `LICENSE` conflicts with the declared string, the source text is
authoritative (e.g. a package declaring `"BSD License"` whose repo `LICENSE` is Apache-2.0 --> it's Apache-2.0). Cross-check available sources, don't stop at one.

1. Source repo license files, e.g. `LICENSE`/`COPYING`/`LICENSE.txt`/`LICENSE.md` and similar - fetch the raw file, read the actual text; this is the ground truth. Watch for a main license **plus** there _may be_ additional licenses (e.g. vendored components) that _may be_ applicable.
2. Source repo files `pyproject.toml` / `package.json` / `pom.xml` license fields.
3. Registry metadata
    - PyPI: `https://pypi.org/pypi/<name>/json` (`info.license`, `info.license_expression`, `info.classifiers`)
    - NPM: `https://registry.npmjs.org/<name>`
    - Maven: `https://central.sonatype.com/artifact/<group>/<name>`
    - etc

Map the finding to a valid identifier and confirm which **kind** it is:

- an SPDX license (<https://spdx.org/licenses/>)
- an SPDX **exception** used after `WITH` (<https://spdx.org/licenses/exceptions-index.html>)
- a non-SPDX `LicenseRef-scancode-...` to be classified (<https://scancode-licensedb.aboutcode.org/>)

> [!tip]
> Don't mix them up (`LLVM-exception` is an exception, only valid as `Apache-2.0 WITH LLVM-exception`).

Common non-SPDX strings:

| Declared string                    | SPDX                                                    |
| ---------------------------------- | ------------------------------------------------------- |
| `MIT License`, `MIT license`       | `MIT`                                                   |
| `Apache 2.0`, `Apache License 2.0` | `Apache-2.0`                                            |
| `BSD`, `BSD License`               | usually `BSD-3-Clause` (verify against the text)        |
| `GPLv3`, `GNU GPL v3`              | `GPL-3.0-only` or `GPL-3.0-or-later` (check "or later") |
| `ISC License`                      | `ISC`                                                   |

> [!tip]
> Use `fetch_webpage` or similar tool for registry JSON and raw LICENSE URLs. Never conclude a license from the
package name or a vague string alone. It's absolutely legitimate to claim you can't find a license, or that the evidence is ambiguous/conflicting, and to stop there instead of guessing.

**Reuse identifiers; never invent them.** For non-SPDX / vendor licenses, find an EXISTING id
before writing: search `license-classifications.yml` in this repo and
<https://scancode-licensedb.aboutcode.org/> for a `LicenseRef-scancode-...`. Never coin a new
`LicenseRef-...`. Match on the license's **category/text**, not its name: `LicenseRef-scancode-nvidia`
is _permissive_ and is WRONG for proprietary CUDA libraries (use
`LicenseRef-scancode-nvidia-cuda-supplement-2020`). A `WITH`-exception expression combines a valid
SPDX license id with a valid SPDX **exception** id (e.g. `Apache-2.0 WITH LLVM-exception`).

**NON-NEGOTIABLE.**

- NEVER invent, assume, or downgrade to a permissive license (MIT, BSD, Apache-2.0, ...) to clear a violation or please the requester. Mirror the package's **ACTUAL** evidenced license - even if copyleft (GPL/LGPL/MPL/AGPL), proprietary, dual-licensed, or `NOASSERTION`.
- Be bureaucratically precise. If evidence is ambiguous, conflicting, or no authoritative source exists, **STOP** and report your findings instead of writing a curation.
- If the package is genuinely unlicensed, simply raise a flag - per ORT's guidance an unlicensed dependency must not be used, and a user must decide how to handle it (e.g. by adding resolutions directly to their projects `.ort.yml`). Your work on this package is done.

### 3. Choose the curation field

| Situation                                                                       | Field                      | Value                                                                          |
| ------------------------------------------------------------------------------- | -------------------------- | ------------------------------------------------------------------------------ |
| 1. `UNMAPPED_DECLARED_LICENSE`                                                  | `declared_license_mapping` | `"<exact declared string>": "<SPDX>"`                                          |
| 2. `NO_LICENSE_IN_DEPENDENCY`                                                   | `concluded_license`        | `"<SPDX expression>"`                                                          |
| 3. Detected license is **incomplete** vs. source, e.g. `COPYLEFT_IN_DEPENDENCY` | `concluded_license`        | full `"<SPDX expression>"` (e.g. `"EPL-2.0 OR GPL-2.0-only OR LGPL-2.1-only"`) |
| 4. Source shows a main license **plus** additional licenses                     | `concluded_license`        | combined `"<A> AND <B>"` (e.g. `"Apache-2.0 AND BSD-3-Clause"`)                |
| 5. `GENERIC_IN_DEPENDENCY` (detected trigger, e.g. `LicenseRef-scancode-proprietary-license`) | `concluded_license` | `"<SPDX expression>"` (real license, e.g. `LicenseRef-scancode-nvidia-cuda-supplement-2020`) |
| 6. `UNHANDLED_LICENSE`                                                          | -                          | -                                                                              |

Cases explanation:

1. License metadata found, but it is not SPDX-compliant. Fix by mapping the detected string to matching SPDX identifier.
2. License metadata wasn't found at all. Fix by finding and asserting the actual license.
3. License metadata found, but it's offensive. The source may offer a choice, but ORT detected only one branch. Worth checking and asserting the **complete** expression with `concluded_license`, joining alternatives with `OR` (`AND` only if genuinely combined). But keep **every** alternative - never drop the copyleft branch just because a permissive one exists.
4. The `LICENSE` file has a main license and additional license(s), e.g. for vendored components. Resolve as situation 3: assert the combined `concluded_license` but use `AND` (e.g. `"Apache-2.0 AND BSD-3-Clause"`).
5. ScanCode **detected** a generic license trigger (e.g. `LicenseRef-scancode-proprietary-license`) inside the package files - this is a detected finding, not just declared metadata. A `declared_license_mapping` only rewrites the declared string and will NOT override a detected license, so the trigger persists. Assert the verified real license with `concluded_license` (which overrides detected findings). Keep any existing `declared_license_mapping` too, so the declared metadata still maps cleanly.
6. The identifier is correct but not covered by policy. Nothing to do with curations, subject to `license-classifications.yml` adjustments (see step 5b).

### 4. Pick a relaxed version range (Ivy-style)

Curations use **Ivy version matchers**, not glob/semver wildcards.
Relax the range so the curation survives patch/minor bumps but stops at the next major (license may change on a major bump).
Put it in the `id`'s version segment, using the repo's **open-lower** house style `(,<next-major>.0.0[`.

| Subject version    | `id` version segment | Meaning                  |
| ------------------ | -------------------- | ------------------------ |
| `1.0.9`, `1.2.3`   | `(,2.0.0[`           | all versions `< 2.0.0`   |
| any / all versions | empty                | applies to every version |

Ivy notation - the bracket **direction** sets inclusivity: an opening `[` / closing `]` is inclusive, an opening `]` / closing `[` is exclusive, and `(` / `)` are unbounded. So `(,2.0.0[` means `v < 2.0.0` (unbounded below, exclusive above).

Reference matches:

| Range       | Matches           |
| ----------- | ----------------- |
| `[1.0,2.0]` | `1.0 <= v <= 2.0` |
| `[1.0,2.0[` | `1.0 <= v < 2.0`  |
| `]1.0,2.0]` | `1.0 < v <= 2.0`  |
| `]1.0,2.0[` | `1.0 < v < 2.0`   |
| `[1.0,)`    | `v >= 1.0`        |
| `]1.0,)`    | `v > 1.0`         |
| `(,2.0]`    | `v <= 2.0`        |
| `(,2.0[`    | `v < 2.0`         |

### 5. Write the file

Path convention: `curations/<Type>/<namespace>/<name>.yml`

- PyPI/NPM (empty namespace): use `_` as the namespace dir, e.g. `curations/PyPI/_/httpcore.yml`.
- Maven: namespace is the group, e.g. `curations/Maven/androidx.core/core.yml` (existing files may use `_.yml` for a whole namespace).

### 5b. Classify the license

If the violation is `UNHANDLED_LICENSE`, or the license you mapped/concluded (SPDX or `LicenseRef-scancode-...`) is not yet in `license-classifications.yml`, add a `categorizations` entry there, inserted **alphabetically by `id`**, mirroring a sibling entry's category.
Use the id verbatim - for `WITH`-exception licenses the full compound string is the `id` (e.g. `Apache-2.0 WITH LLVM-exception`, category `permissive`, next to `Apache-2.0 WITH Swift-exception`).
Pick the category from the license's real nature (proprietary CUDA --> `proprietary-free`, not `permissive`).
No `LicenseRef-...` may be invented; it must exist in ScanCode LicenseDB.

### 6. Comment: concise and verifiable

Keep `comment` to 1-3 lines plus a URL to the strong evidence (e.g. LICENSE file).
No speculation. The evidence URL MUST point to a version that actually exists.
Prefer the exact version from the violation (e.g. tag `0.39.0`), never a fabricated one - even when the `id` range spans up to the next major.
Pin to a **tag/release**, not a moving branch (`main`, `master`, `dev`).
For a named proprietary/vendor license, link the **canonical upstream license** (e.g. the CUDA EULA page for `LicenseRef-scancode-nvidia-cuda-supplement-2020`), not just the registry project page.

### 7. Verify

- curation YAML is valid
- `id` type/namespace/name/version match the violation
- the SPDX expression is valid
- the file sits under the correct `curations/<Type>/<namespace>/` path
- Every `LicenseRef-...` used exists in ScanCode LicenseDB **and** is classified in `license-classifications.yml` (no invented ids leaked)

## Quick Reference: worked examples

### `UNMAPPED_DECLARED_LICENSE` --> `declared_license_mapping`

> WARNING: UNMAPPED_DECLARED_LICENSE - PyPI::httpcore:1.0.9 - The declared license 'BSD License' could not be mapped to a valid license or parsed as an SPDX expression. The license was found in package 'PyPI::httpcore:1.0.9'.

1. **Read the violation** A declared string exists (`"BSD License"`) but isn't SPDX, so there is evidence to preserve: `declared_license_mapping` (situation 1).
2. **Find the REAL license** "BSD" could be 2- or 3-clause; read the repo `LICENSE` at tag `1.0.9`. It's 3-clause: `BSD-3-Clause`.
3. **Id kind?** Plain SPDX license. No exception, no `LicenseRef`, no classification.
4. **Range?** `1.0.9` --> assume all versions till next major, `(,2.0.0[`.
5. **Write** `curations/PyPI/_/httpcore.yml`

    ```yaml
    - id: "PyPI::httpcore:(,2.0.0["
      curations:
        comment: |
          The library is licensed under BSD-3-Clause, see:
          https://github.com/encode/httpcore/blob/1.0.9/LICENSE.md
        declared_license_mapping:
          "BSD License": "BSD-3-Clause"
    ```

### `COPYLEFT_IN_DEPENDENCY` --> `concluded_license`

> ERROR: COPYLEFT_IN_DEPENDENCY - Maven:com.github.jnr:jnr-posix:3.1.21 - GPL-2.0-only - The dependency 'Maven:com.github.jnr:jnr-posix:3.1.21' is licensed under the ScanCode 'copyleft' categorized license GPL-2.0-only.

1. **Read the violation** ORT detected `GPL-2.0-only` but it worth checking is it a valid and complete conclusion.
1. **Find the REAL license** The tagged `LICENSE.txt` says "released under a tri EPL/GPL/LGPL license". So ORT caught only the GPL branch, the detection is incomplete (situation 3). Side note: however, if the source confirmed a single `GPL-2.0-only`, the violation would be accurate: I'd report it and do nothing instead of adding curation.
1. **Decide the expression** A choice, so join with `OR` and keep **every** branch: `EPL-2.0 OR GPL-2.0-only OR LGPL-2.1-only`.
1. **Range?** `3.1.21` --> assume all versions till next major, `(,4.0.0[`.
1. **Write** `curations/Maven/com.github.jnr/jnr-posix.yml`

    ```yaml
    - id: "Maven:com.github.jnr:jnr-posix:(,4.0.0["
      curations:
        comment: |
          The library is tri-licensed under EPL-2.0 OR GPL-2.0-only OR LGPL-2.1-only, see:
          https://github.com/jnr/jnr-posix/blob/jnr-posix-3.1.21/LICENSE.txt
        concluded_license: "EPL-2.0 OR GPL-2.0-only OR LGPL-2.1-only"
    ```

### `UNMAPPED_DECLARED_LICENSE`: reuse an existing `LicenseRef` (+ classify)

> WARNING: UNMAPPED_DECLARED_LICENSE - PyPI::nvidia-cublas-cu12:12.8.4.1 - The declared license 'NVIDIA Proprietary Software' could not be mapped ...

1. **Id kind?** A vendor license, not SPDX. Expect a `LicenseRef-scancode-...`.
2. **Search before inventing** Check `license-classifications.yml` and ScanCode LicenseDB. Found `LicenseRef-scancode-nvidia`, but won't use it: it is _permissive_ and misrepresents a proprietary library. The package is a CUDA Toolkit component, so should fail under the CUDA EULA supplement --> Found `LicenseRef-scancode-nvidia-cuda-supplement-2020` (proprietary-free type).
3. **Classify** It's already in `license-classifications.yml`, no need to add this time.
4. **Write** `curations/PyPI/_/nvidia-cublas-cu12.yml` with mapping `"NVIDIA Proprietary Software": "LicenseRef-scancode-nvidia-cuda-supplement-2020"`.

### `GENERIC_IN_DEPENDENCY` (detected trigger) --> `concluded_license`

> ERROR: GENERIC_IN_DEPENDENCY - PyPI::nvidia-cublas-cu12:12.8.4.1 - The dependency 'PyPI::nvidia-cublas-cu12:12.8.4.1' might contain a license which is unknown to the tooling. It was detected as LicenseRef-scancode-proprietary-license which is just a trigger, but not a real license.

1. **Read the violation** ScanCode **detected** the generic trigger `LicenseRef-scancode-proprietary-license` inside the package files. This is a detected finding, so a `declared_license_mapping` alone (which only rewrites the declared metadata string) will NOT clear it - the trigger persists (situation 5).
2. **Find the REAL license** Same CUDA Toolkit component as above; the proprietary trigger is the CUDA EULA supplement --> `LicenseRef-scancode-nvidia-cuda-supplement-2020` (already classified as proprietary-free).
3. **Choose the field** Assert `concluded_license` to override the detected finding. Keep the existing `declared_license_mapping` so the declared string still maps cleanly.
4. **Range?** `12.8.4.1` --> all versions till next major, `(,13.0.0[`.
5. **Write** `curations/PyPI/_/nvidia-cublas-cu12.yml`

    ```yaml
    - id: "PyPI::nvidia-cublas-cu12:(,13.0.0["
      curations:
        comment: |
          The package is part of the NVIDIA CUDA Toolkit; it declares 'NVIDIA Proprietary
          Software', covered by the CUDA Toolkit Supplement to the SLA for NVIDIA SDKs, see:
          https://docs.nvidia.com/cuda/eula/index.html#cuda-toolkit-supplement-to-software-license-agreement-for-nvidia-software-development-kits
        concluded_license: "LicenseRef-scancode-nvidia-cuda-supplement-2020"
        declared_license_mapping:
          "NVIDIA Proprietary Software": "LicenseRef-scancode-nvidia-cuda-supplement-2020"
    ```

### Source overrides metadata

> ERROR: GENERIC_IN_DEPENDENCY - PyPI::nvidia-nvshmem-cu12:3.3.20 - LicenseRef-scancode-proprietary-license - The dependency 'PyPI::nvidia-nvshmem-cu12:3.3.20' might contain a license which is unknown to the tooling.

1. **Find the REAL license**: PyPI's classifier says `BSD-3-Clause`. Good, but metadata doesn't win, this package have the source code open, should verify that. The repo `License.txt` shows the project is `Apache-2.0`, with an "Additional Licenses" section for vendored `BSD-3-Clause` components.
2. **Decide the expression** Both apply and ship in the artifact --> `AND` (situation 4): `Apache-2.0 AND BSD-3-Clause`.
3. **Write** `curations/PyPI/_/nvidia-nvshmem-cu12.yml` with `concluded_license`.
