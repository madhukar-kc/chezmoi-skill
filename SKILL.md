---
name: chezmoi
description: Manage dotfiles with chezmoi. Use when working with dotfiles, chezmoi templates, machine-specific configuration, syncing configs across machines, or when the user mentions chezmoi, dotfiles, or configuration management.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash(chezmoi diff *)
  - Bash(chezmoi cat *)
  - Bash(chezmoi data *)
  - Bash(chezmoi status *)
  - Bash(chezmoi managed *)
  - Bash(chezmoi unmanaged *)
  - Bash(chezmoi source-path *)
  - Bash(chezmoi execute-template *)
---

# Chezmoi Dotfiles Management

## Action Guidance

When the user asks about dotfiles or chezmoi operations, **implement the changes directly** using chezmoi commands rather than only suggesting them. If the user says "make this work on macOS only" or "add my bashrc to chezmoi", proceed with implementation.

Editing files in the source directory is fine: it is a git repo, so every edit can be reviewed and undone. Commands that change the home directory or publish anything need the user's approval first:

- `chezmoi apply`: show `chezmoi diff` first, and apply only after the user approves that diff.
- `chezmoi add`, `chezmoi re-add`, `chezmoi forget`, `chezmoi destroy`, `.chezmoiremove` entries, and git commit/push in the source directory: say what will change, then wait for approval.
- Never pass `--force` unless the user has approved overwriting the specific files involved. `--force` overwrites files in the home directory that differ from the source state without asking.

Before adding a file, check whether the repo is public (`chezmoi git -- remote -v`, then `gh repo view`) and search the file for secrets, tokens, employer names, internal hostnames and email addresses. Never add a whole directory such as `~/.claude` in one step; add the files you mean to share.

## Commands

```bash
chezmoi add <file>           # Add file to chezmoi
chezmoi apply                # Apply changes, only after the user approves the diff
chezmoi diff                 # Preview what would change
chezmoi cat <file>           # Preview rendered template output
chezmoi data                 # Show available template variables
chezmoi edit <file>          # Edit source file
chezmoi edit ~/.local/share/chezmoi/.chezmoiremove  # Edit removal list
```

## File Naming

| Prefix/Suffix | Effect |
|---------------|--------|
| `dot_` | Installed as `.` (`dot_gitconfig` → `~/.gitconfig`) |
| `private_` | Removes group and world permissions. It does not hide the contents: the file is still plain text in the repo |
| `executable_` | Executable permissions (755) |
| `.tmpl` | Processed as Go template |
| `symlink_` | Creates a symlink; the file's content is the link target (one trailing newline is removed). An empty target removes the link. With `.tmpl`, the target is a template |
| `exact_` | Directories only: removes everything in the target directory that chezmoi does not manage. Do not use it on directories that other tools also write to |
| `create_` | Creates the file only if it does not exist yet; later edits in the home directory are kept |
| `run_` | The file is a script that chezmoi runs |

Script names combine attributes in this order: `run_`, then `once_` or `onchange_`, then `before_` or `after_`, then the name, then an optional `.tmpl` (for example `run_onchange_after_install-deps.sh.tmpl`). `run_onchange_` scripts run when their rendered content changes. To re-run one when another file changes, put that file's hash in a comment: `# hash: {{ include "dot_config/foo.json" | sha256sum }}`. Scripts must be safe to run more than once.

## Ignoring Files (`.chezmoiignore`)

- Patterns match target paths (as in the home directory, e.g. `.config/foo`), not source names (`dot_config/foo`).
- The file is always processed as a template, so `{{ if }}` blocks can ignore files per machine.
- `dir/` ignores the directory itself; `dir/**` ignores its contents.
- A line starting with `!` excludes a pattern from ignoring, and these exclusions win over every ignore pattern. To share only some entries of a directory: `.claude/skills/*` followed by `!.claude/skills/aws-login`.
- Remember the repo may be public: an ignore line itself names the file it ignores.

## Machine-Specific Templates

Convert a static file to a template by renaming with `.tmpl` suffix, then use conditionals:

### By OS
```
{{- if eq .chezmoi.os "darwin" }}
# macOS config
{{- end }}

{{- if eq .chezmoi.os "linux" }}
# Linux config
{{- end }}
```

### By hostname
```
{{- if hasPrefix .chezmoi.hostname "work-" }}
# work machine config
{{- end }}
```

### By environment variable
```
{{- if env "WORK_ENV" }}
# when WORK_ENV is set
{{- end }}
```

### Combined conditions
```
{{- if or (eq .chezmoi.os "darwin") (env "WORK_ENV") }}
# macOS or when WORK_ENV is set
{{- end }}
```

## Escaping Nested Templates

When the target file itself uses `{{ }}` syntax (like mise, Jinja2, or Tera templates), escape the braces so chezmoi doesn't process them as Go template syntax:

```
SOME_VAR = "{{ "{{" }}env.OTHER_VAR{{ "}}" }}"
```

This renders as `SOME_VAR = "{{env.OTHER_VAR}}"` in the output. The `{{ "{{" }}` and `{{ "}}" }}` are Go template expressions that output literal brace characters.

## Removing Files

Use `.chezmoiremove` to clean up dotfiles that chezmoi should remove from your system:

```bash
# Create .chezmoiremove file listing targets to remove
cat > ~/.local/share/chezmoi/.chezmoiremove << 'EOF'
~/.old-config
~/.deprecated-rc
EOF

# Preview what would be removed
chezmoi diff

# Apply to remove the files, after the user approves the diff
chezmoi apply
```

### Template-based removal (`.chezmoiremove.tmpl`)

Conditionally remove files based on system variables:

```
{{- if eq .chezmoi.os "darwin" }}
~/.linux-only-config
{{- end }}

{{- if not (env "WORK_ENV") }}
~/.work-config
{{- end }}
```

The file is always processed as a template, even without the `.tmpl` extension.

## External Dependencies

Manage external git repos, archives or single files in `.chezmoiexternal.toml` (types `git-repo`, `archive`, `file`). A `git-repo` external hands the whole directory to git: chezmoi cannot manage other files in it.

```toml
[".zsh/plugins/some-plugin"]
    type = "git-repo"
    url = "https://github.com/user/some-plugin.git"
    refreshPeriod = "168h"
```

## Workflow: Converting Static File to Template

When converting a static config to machine-specific template:

1. Read the current file to understand its contents
2. Identify machine-specific values (paths, hostnames, environment-specific settings)
3. Run `chezmoi add --template <file>` to add as template
4. Edit the template source to add conditionals (see "Machine-Specific Templates")
5. Preview with `chezmoi cat <file>` to verify rendering on this machine
6. Show the diff, and apply with `chezmoi apply` once the user approves

## Gotchas

1. **Removing managed files**: Use `.chezmoiremove` to clean up files (see "Removing Files" section)
2. **Always preview**: Run `chezmoi cat <file>` before applying templates
3. **No `--force` without approval**: if `chezmoi apply` stops because a file in the home directory was changed outside chezmoi, show the user `chezmoi diff` for that file and ask whether to keep the local version (`chezmoi re-add <file>`) or overwrite it (`chezmoi apply --force <file>`). Claude cannot answer chezmoi's interactive prompts, so ask in the conversation instead.
4. **`--refresh-externals` only when wanted**: externals re-download at most every `refreshPeriod`; pass `--refresh-externals` (`-R`) only when the user wants externals updated now. chezmoi checks externals on every `diff` and `apply`, so avoid large archives
5. **Template errors**: Use `chezmoi execute-template '{{ .chezmoi.os }}'` to test template snippets

## Troubleshooting

**Template syntax errors**: Test small snippets with `chezmoi execute-template '{{ .chezmoi.os }}'`

**File not applying**: Check `chezmoi status` and `chezmoi diff` to see what changes are pending

**External not refreshing**: The refresh period might not have elapsed; use `--refresh-externals` flag to force refresh

**Permission errors**: Verify file has correct prefix (`private_` for 600, `executable_` for 755)
