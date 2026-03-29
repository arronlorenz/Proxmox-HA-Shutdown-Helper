# AGENTS instructions

- Use 4 spaces for indentation.
- Keep shell scripts POSIX compliant.
- For Markdown files, wrap lines at 80 characters.
- Before committing changes, run the following checks:
  - `bash -n pve-nut-shutdown pve-nut-shutdown-restore`
  - `shellcheck pve-nut-shutdown pve-nut-shutdown-restore`
  - `markdownlint README.md AGENTS.md`

If `shellcheck` or `markdownlint` is not installed, install them with
`apt-get install -y shellcheck` and `npm install -g markdownlint-cli`.
