# Contributing

Thank you for helping improve this playbook. This guide covers the workflow
for making changes and keeping the version history consistent.

---

## Versioning Scheme

This project uses [Semantic Versioning](https://semver.org): `MAJOR.MINOR.PATCH`

| Increment | When to use | Example |
|---|---|---|
| **MAJOR** | Breaking change — renamed variable, restructured inventory, removed section | `2.0.0` |
| **MINOR** | New hardening section or backwards-compatible feature | `1.4.0` |
| **PATCH** | Bug fix, doc correction, minor tweak to an existing task | `1.3.1` |

---

## Making a Change — Step by Step

### 1. Fork and branch

```bash
git clone https://github.com/n8xja/RPI-Hardening-Playbook.git
cd RPI-Hardening-Playbook
git checkout -b feature/my-new-section   # or fix/descriptive-name
```

### 2. Make your changes

Edit `rpi_harden.yml` and any templates in `templates/`.

For new hardening sections, follow the established pattern:

- Add a `do_<section>: true` toggle in the **SECTION TOGGLES** vars block
- Add any supporting variables in their own named config block below the toggles
- Add the task block gated with `when: do_<section>` and a matching tag
- Add a status check in the post-hardening report collection tasks
- Add a line to the **HARDENING SECTIONS APPLIED** summary in the report content
- Add the new tag to the **Available tags** list in `README.md`
- Add a row to the features table in `README.md`

### 3. Syntax check

Always run the Ansible syntax check before opening a PR:

```bash
ansible-playbook --syntax-check -i inventory.ini.example rpi_harden.yml
```

(Create a temporary `inventory.ini` from the example for this to work.)

### 4. Determine the new version number

Follow the scheme above. If the current version is `1.3.0`:

- New hardening section → `1.4.0`
- Bug fix → `1.3.1`
- Renamed a config variable → `2.0.0`

### 5. Update the version in three places

**`rpi_harden.yml`** — update the header comment:
```yaml
# Version : 1.4.0
```
And add a line to the inline changelog in the same header:
```yaml
#   1.4.0 — Brief description of what changed
```

**`README.md`** — update the version badge:
```markdown
[![Version](https://img.shields.io/badge/version-1.4.0-blue.svg)](CHANGELOG.md)
```

**`CHANGELOG.md`** — add a new entry at the top (above the previous release):
```markdown
## [1.4.0] — YYYY-MM-DD

### Added
- Description of what was added.

### Changed
- Description of what changed.

### Fixed
- Description of what was fixed.
```
And add a comparison link at the bottom of the file:
```markdown
[1.4.0]: https://github.com/n8xja/RPI-Hardening-Playbook/compare/v1.3.0...v1.4.0
```

### 6. Commit and open a PR

```bash
git add rpi_harden.yml README.md CHANGELOG.md
git commit -m "feat: add <section name> hardening section (v1.4.0)"
git push origin feature/my-new-section
```

Then open a pull request against `main` on GitHub.

---

## Commit Message Convention

Use a short prefix to make the history scannable:

| Prefix | When to use |
|---|---|
| `feat:` | New hardening section or feature |
| `fix:` | Bug fix |
| `docs:` | README, CHANGELOG, or comment changes only |
| `refactor:` | Code restructure with no behaviour change |
| `chore:` | Dependency update, CI config, tooling |

Example: `feat: add log2ram SD card protection section (v1.2.0)`

---

## Releasing

Maintainers — after merging a PR that bumps the version:

```bash
git tag -a v1.4.0 -m "Release v1.4.0"
git push origin v1.4.0
```

Then create a GitHub Release from that tag, copying the relevant
`CHANGELOG.md` section as the release notes.
