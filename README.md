<!--
# SPDX-License-Identifier: Apache-2.0
# SPDX-FileCopyrightText: 2026 The Linux Foundation
-->

# 📋 SBOM Generator Action

<!-- prettier-ignore-start -->
<!-- markdownlint-disable-next-line MD013 -->
[![Linux Foundation](https://img.shields.io/badge/Linux-Foundation-blue)](https://linuxfoundation.org/) [![Source Code](https://img.shields.io/badge/GitHub-100000?logo=github&logoColor=white&color=blue)](https://github.com/lfreleng-actions/sbom-action) [![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![pre-commit.ci status badge]][pre-commit.ci results page] [![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/lfreleng-actions/sbom-action/badge)](https://scorecard.dev/viewer/?uri=github.com/lfreleng-actions/sbom-action)
<!-- prettier-ignore-end -->

Generates CycloneDX Software Bill of Materials (SBOM) reports for
projects in any language/ecosystem.

## sbom-action

This action provides a common interface over pluggable SBOM generation
backends. The default backend wraps
[syft](https://github.com/anchore/syft), which performs static analysis
of lockfiles and filesystem content, covering Go modules, Node.js/npm,
Java, Rust, containers, binaries and further ecosystems with a
single tool. Syft shares a vendor with
[Grype](https://github.com/anchore/grype), which the reusable workflows
in this organisation use to audit the generated SBOM in a separate,
downstream job.

The interface mirrors
[python-sbom-action](https://github.com/lfreleng-actions/python-sbom-action),
giving callers one contract across the actions estate. Future backends
(such as `cyclonedx-npm`, `cyclonedx-gomod` or an environment-based
Python backend) can slot in behind the same interface through the
`backend` input, without changes to calling workflows.

## Usage Example

<!-- markdownlint-disable MD046 -->

```yaml
steps:
  - name: "Generate SBOM"
    id: sbom
    uses: lfreleng-actions/sbom-action@main
    with:
      path_prefix: '.'
```

<!-- markdownlint-enable MD046 -->

## Requirements

The action needs `jq`, `realpath` (GNU coreutils, including `-m`
support) and `mktemp` on the runner. GitHub-hosted Ubuntu runners
include these tools; minimal self-hosted or non-Linux runners must
provide them. The action checks for them up front and fails with a
clear error naming any missing tool. The syft binary downloads
via the pinned `anchore/sbom-action/download-syft` helper, so runners
need egress to GitHub release assets.

## Inputs

<!-- markdownlint-disable MD013 -->

| Name              | Required | Default          | Description                                                           |
| ----------------- | -------- | ---------------- | --------------------------------------------------------------------- |
| backend           | False    | `syft`           | SBOM generation backend; supports `syft`                              |
| path_prefix       | False    | `.`              | Project directory; must resolve within the workspace                  |
| sbom_format       | False    | `both`           | SBOM output format: `json`, `xml`, or `both`                          |
| sbom_spec_version | False    | `1.5`            | CycloneDX specification version to use                                |
| filename_prefix   | False    | `sbom-cyclonedx` | Base filename for SBOM output (without extension)                     |
| output_directory  | False    | `.`              | SBOM report directory, within workspace or runner temp                |
| include_dev       | False    | `false`          | Include development dependencies in SBOM                              |
| fail_on_error     | False    | `true`           | Fail the action if SBOM generation encounters errors                  |
| syft_version      | False    | `''`             | Syft version to download (defaults to the installer's pinned version) |

<!-- markdownlint-enable MD013 -->

## Outputs

<!-- markdownlint-disable MD013 -->

| Name            | Description                                |
| --------------- | ------------------------------------------ |
| sbom_json_path  | Path to generated JSON SBOM file           |
| sbom_xml_path   | Path to generated XML SBOM file            |
| component_count | Number of components in the generated SBOM |
| backend         | SBOM generation backend used               |

<!-- markdownlint-enable MD013 -->

The action emits `sbom_json_path` for the `json` and `both` formats,
and `sbom_xml_path` for the `xml` and `both` formats; the output for a
format the caller did not request stays empty.

## Path Constraints

Relative values for `path_prefix` and `output_directory` resolve
against `GITHUB_WORKSPACE`, not the current working directory, so
behaviour stays deterministic when a calling workflow sets a custom
working directory. The action checks both directory inputs against
the runner filesystem before use: `path_prefix` must resolve within
`GITHUB_WORKSPACE`, and `output_directory` must resolve within
`GITHUB_WORKSPACE` or `RUNNER_TEMP`. Paths that escape these
locations fail the action, preventing scans or writes against
arbitrary runner filesystem locations.

## Development Dependency Scoping

The `include_dev` input controls whether development dependencies
appear in the SBOM. The syft backend maps this onto its JavaScript
cataloger (`SYFT_JAVASCRIPT_INCLUDE_DEV_DEPENDENCIES`), so for Node.js
projects the SBOM covers production dependencies by default. Go modules
have no development scope, so the input has no effect there. Further
per-ecosystem scoping options join the mapping as syft exposes them.

## Monorepo and Nested Module Support

Point `path_prefix` at the directory containing the project's
lockfile/manifest. This supports repositories where the module is not
at the repository root, for example Gerrit-hosted monorepos such as
`onap/multicloud-k8s`, where Go modules live under `src/`:

<!-- markdownlint-disable MD046 -->

```yaml
steps:
  - name: "Generate SBOM for nested module"
    uses: lfreleng-actions/sbom-action@main
    with:
      path_prefix: 'src/k8splugin'
```

<!-- markdownlint-enable MD046 -->

## Workflow Integration

The reusable workflows in this organisation keep SBOM generation and
SBOM auditing in separate jobs, connected by an artifact. The
generation job runs this action and uploads the results; a downstream
job downloads them and audits with Grype. This separation keeps
failure modes independent and preserves the SBOM for inspection even
when the audit fails:

<!-- markdownlint-disable MD046 -->

```yaml
- name: "Generate SBOM"
  id: sbom
  uses: lfreleng-actions/sbom-action@main

- name: "Upload SBOM artifact"
  uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
  with:
    name: sbom-files
    path: sbom-cyclonedx.*
    if-no-files-found: error
```

<!-- markdownlint-enable MD046 -->

## Implementation Details

<!-- markdownlint-disable MD013 -->

1. **Input Validation**: Validates backend, format, boolean flags, specification version and filename prefix (restricted character set) before use; verifies the project directory exists
2. **Syft Download**: Fetches the syft binary via the pinned `anchore/sbom-action/download-syft` helper action
3. **SBOM Generation**: Runs a single syft scan emitting the requested CycloneDX formats (`format@version=path` syntax); the JSON document always gets generated internally to compute the component count
4. **Outputs and Summary**: Emits output paths for the requested formats, the component count, and a step summary

<!-- markdownlint-enable MD013 -->

## Notes

- The generated filenames follow the `sbom-cyclonedx.*` convention the
  organisation's reusable workflows consume (the `sbom-files` artifact
  contract)
- Static analysis reads lockfiles/manifests without installing project
  dependencies, so generation is fast and needs no language toolchain
- For Python projects, prefer
  [python-sbom-action](https://github.com/lfreleng-actions/python-sbom-action):
  its environment-based generation gives higher-fidelity results for
  resolved Python dependency graphs

[pre-commit.ci results page]: https://results.pre-commit.ci/latest/github/lfreleng-actions/sbom-action/main
[pre-commit.ci status badge]: https://results.pre-commit.ci/badge/github/lfreleng-actions/sbom-action/main.svg
