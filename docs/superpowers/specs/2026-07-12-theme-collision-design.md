# Theme Collision Fix Design

## Problem

Pi discovers `no-clown-fiesta-dark` twice when this repository is installed as a Git package and `./install-pi` has also been run:

1. The installer creates `~/.pi/agent/themes/no-clown-fiesta-dark.json` as a user-scoped symlink.
2. The installed package exposes `themes/no-clown-fiesta-dark.json` through `package.json`.

Pi gives the user-scoped theme precedence and reports the package theme as a skipped collision. The two files are byte-identical, so the collision comes from duplicate discovery paths rather than conflicting theme content.

## Decision

Use Pi's package manager as the only supported installation and theme-discovery path.

- Keep the `pi.themes` package manifest entry.
- Delete `install-pi` so the project no longer creates a global theme symlink.
- Document installation with `pi install git:github.com/aktersnurra/no-clown-fiesta.pi`.
- Tell users to select `no-clown-fiesta-dark` through `/settings`.
- Keep the recommended pi-vim mode colors as an optional manual `settings.json` snippet. The theme package must not mutate unrelated extension settings.

## Legacy Cleanup

Document a one-time cleanup command for existing users. It must remove `~/.pi/agent/themes/no-clown-fiesta-dark.json` only when that path is a symbolic link. A regular file at the same path may be independently maintained and must not be deleted automatically.

The current development environment's legacy symlink will be removed as part of implementation so only the installed package copy remains discoverable.

## Resulting Data Flow

1. `~/.pi/agent/settings.json` contains the Git package source.
2. Pi loads the package clone.
3. `package.json` exposes the repository's `themes/` directory.
4. Pi discovers one theme named `no-clown-fiesta-dark`.
5. The active theme is selected through normal Pi settings.

No installer-side file linking or settings mutation remains.

## Error Handling

There is no new runtime code. Migration instructions use a symbolic-link guard so cleanup is safe and idempotent. Package installation and theme selection rely on Pi's documented commands and settings UI.

## Verification

Implementation is complete when all of the following hold:

- `install-pi` no longer exists.
- README installation instructions use Pi's package manager exclusively.
- README cleanup instructions preserve a regular file and remove only a legacy symbolic link.
- `package.json` and the theme JSON parse successfully.
- The manifest still exposes `themes/`.
- The current global legacy symlink is absent.
- Exactly one discoverable `no-clown-fiesta-dark` theme remains in the current Pi setup.
- A focused Pi startup or resource-discovery check no longer reports the collision.
- Repository diagnostics report no blocking errors.

## Out of Scope

- Automatically editing users' settings during package installation.
- Adding a Pi extension solely to configure pi-vim.
- Renaming the theme.
- Changing theme colors.
