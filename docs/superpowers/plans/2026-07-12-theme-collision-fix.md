# Theme Collision Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the duplicate `no-clown-fiesta-dark` discovery path so Pi loads the theme exclusively from its installed Git package.

**Architecture:** Keep `package.json` as the sole theme registration point and remove the standalone symlink installer. Preserve pi-vim color guidance as documentation, migrate existing users with a symlink-only cleanup command, and protect the package-only contract with a dependency-free Node test.

**Tech Stack:** Pi package manifest JSON, Markdown, Node.js built-in test runner, Bash, Jujutsu.

## Global Constraints

- Pi's package manager is the only supported installation and theme-discovery path.
- Keep the `pi.themes` package manifest entry.
- Cleanup may remove `~/.pi/agent/themes/no-clown-fiesta-dark.json` only when the path is a symbolic link.
- Do not automatically edit users' settings during package installation.
- Do not add a Pi extension solely to configure pi-vim.
- Do not rename the theme or change its colors.

---

## File Map

- `package.json`: Declares the package theme and exposes the dependency-free test command.
- `README.md`: Documents package installation, theme selection, safe legacy cleanup, and optional pi-vim colors.
- `install-pi`: Deleted because it creates the conflicting user-scoped theme symlink.
- `test/package-installation.test.mjs`: Enforces the package-only discovery and documentation contract.

### Task 1: Enforce Package-Only Theme Installation

**Files:**

- Create: `test/package-installation.test.mjs`
- Modify: `package.json`
- Modify: `README.md`
- Delete: `install-pi`

**Interfaces:**

- Consumes: Pi's documented `pi install git:github.com/aktersnurra/no-clown-fiesta.pi` package source and `package.json#pi.themes` manifest shape.
- Produces: A package with one repository-defined theme registration, an `npm test` command, and safe migration instructions.

- [ ] **Step 1: Write the failing package-installation contract test**

Create `test/package-installation.test.mjs`:

```javascript
import assert from "node:assert/strict";
import { existsSync, readFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";
import test from "node:test";

const root = dirname(dirname(fileURLToPath(import.meta.url)));
const read = (path) => readFileSync(join(root, path), "utf8");

test("the Pi package is the only theme discovery path", () => {
  const manifest = JSON.parse(read("package.json"));
  const readme = read("README.md");

  assert.deepEqual(manifest.pi.themes, ["themes"]);
  assert.equal(existsSync(join(root, "install-pi")), false);
  assert.match(
    readme,
    /pi install git:github\.com\/aktersnurra\/no-clown-fiesta\.pi/,
  );
  assert.doesNotMatch(readme, /```bash\n\.\/install-pi\n```/);
});

test("legacy cleanup removes only a symbolic link", () => {
  const readme = read("README.md");

  assert.match(readme, /if \[ -L "\$THEME" \]; then/);
  assert.match(readme, /rm "\$THEME"/);
});

test("the package theme remains valid and named consistently", () => {
  const theme = JSON.parse(read("themes/no-clown-fiesta-dark.json"));

  assert.equal(theme.name, "no-clown-fiesta-dark");
  assert.equal(typeof theme.colors, "object");
});
```

- [ ] **Step 2: Run the contract test and verify it fails for the existing installer**

Run:

```bash
node --test test/package-installation.test.mjs
```

Expected: FAIL in `the Pi package is the only theme discovery path` because `install-pi` still exists and the README still installs with `./install-pi`.

- [ ] **Step 3: Remove the competing installer**

Delete `install-pi`:

```bash
rm install-pi
```

- [ ] **Step 4: Expose the test command without adding dependencies**

Replace `package.json` with:

```json
{
  "name": "no-clown-fiesta.pi",
  "version": "0.1.0",
  "description": "Pi theme inspired by no-clown-fiesta.nvim",
  "scripts": {
    "test": "node --test"
  },
  "pi": {
    "themes": [
      "themes"
    ]
  }
}
```

- [ ] **Step 5: Replace installer documentation with package installation and safe migration**

Replace `README.md` with:

````markdown
# no-clown-fiesta.pi

Pi theme inspired by the dark palette from [no-clown-fiesta.nvim](https://github.com/aktersnurra/no-clown-fiesta.nvim).

## Install

```bash
pi install git:github.com/aktersnurra/no-clown-fiesta.pi
```

Select `no-clown-fiesta-dark` in Pi through `/settings`.

### Migrating from the legacy installer

Older versions of this project linked the theme into Pi's user theme directory. Remove that legacy link before restarting Pi:

```bash
THEME="$HOME/.pi/agent/themes/no-clown-fiesta-dark.json"
if [ -L "$THEME" ]; then
  rm "$THEME"
fi
```

The symbolic-link check preserves a regular file at the same path.

### Optional pi-vim mode colors

To keep pi-vim mode badges readable, merge these values into `~/.pi/agent/settings.json`:

```json
{
  "piVim": {
    "modeColors": {
      "insert": "text",
      "normal": "yellow"
    }
  }
}
```

## Theme

- `themes/no-clown-fiesta-dark.json`
````

- [ ] **Step 6: Run the repository regression test**

Run:

```bash
npm test
```

Expected: three passing tests, zero failures.

- [ ] **Step 7: Reproduce and record the current collision before environment cleanup**

Run:

```bash
rm -f /tmp/pi-theme-startup.log
script -q -c 'timeout 3 pi --verbose' /tmp/pi-theme-startup.log >/dev/null 2>&1 || true
python3 - <<'PY'
import re
from pathlib import Path

text = Path("/tmp/pi-theme-startup.log").read_text(errors="replace")
text = re.sub(r"\x1b(?:\[[0-?]*[ -/]*[@-~]|\][^\x07]*(?:\x07|\x1b\\))", "", text)
lines = [
    line.strip()
    for line in text.splitlines()
    if re.search(r"theme conflicts|collision|no-clown-fiesta", line, re.I)
]
print("\n".join(lines[-20:]))
PY
```

Expected before cleanup: output includes `[Theme conflicts]` and `"no-clown-fiesta-dark" collision:`.

- [ ] **Step 8: Remove the current legacy path only if it is a symbolic link**

Run:

```bash
THEME="$HOME/.pi/agent/themes/no-clown-fiesta-dark.json"
if [ -L "$THEME" ]; then
  rm "$THEME"
fi
```

Expected: the legacy symlink is absent. A regular file would remain untouched.

- [ ] **Step 9: Verify exactly one current discovery source remains**

Run:

```bash
python3 - <<'PY'
from pathlib import Path

paths = [
    Path.home() / ".pi/agent/themes/no-clown-fiesta-dark.json",
    Path.home() / ".pi/agent/git/github.com/aktersnurra/no-clown-fiesta.pi/themes/no-clown-fiesta-dark.json",
]
existing = [path for path in paths if path.exists()]
assert existing == [paths[1]], existing
print(f"PASS: one discoverable theme remains: {existing[0]}")
PY
```

Expected: `PASS: one discoverable theme remains:` followed by the installed Git package theme path.

- [ ] **Step 10: Verify Pi startup no longer reports a theme collision**

Run:

```bash
rm -f /tmp/pi-theme-startup.log
script -q -c 'timeout 3 pi --verbose' /tmp/pi-theme-startup.log >/dev/null 2>&1 || true
python3 - <<'PY'
import re
from pathlib import Path

text = Path("/tmp/pi-theme-startup.log").read_text(errors="replace")
text = re.sub(r"\x1b(?:\[[0-?]*[ -/]*[@-~]|\][^\x07]*(?:\x07|\x1b\\))", "", text)
conflicts = [
    line.strip()
    for line in text.splitlines()
    if re.search(r"theme conflicts|no-clown-fiesta-dark.*collision", line, re.I)
]
assert not conflicts, conflicts
print("PASS: Pi startup reports no no-clown-fiesta theme collision")
PY
```

Expected: `PASS: Pi startup reports no no-clown-fiesta theme collision`.

- [ ] **Step 11: Run final repository checks**

Run:

```bash
npm test
```

Expected: three passing tests, zero failures.

Run Pi Lens diagnostics for these edited files:

```text
lens_diagnostics(mode="all", paths=["package.json", "README.md", "test/package-installation.test.mjs"])
```

Expected: no blocking errors.

- [ ] **Step 12: Commit the implementation**

```bash
jj describe -m "fix(theme): remove duplicate installation path"
jj new
```
