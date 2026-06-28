# CONTRIBUTING

## Why Read This?

Following these guidelines helps ensure your time and ours is well respected.
Clear, thoughtful contributions improve response time, reduce confusion, and
increase the likelihood of your changes being merged.

## What You Can Contribute

We welcome contributions of all kinds:

- Code (new features or bug fixes)
- Bug reports or feature requests
- Documentation improvements
- Molecule scenario improvements or CI fixes

## What We Are Not Looking For

Please avoid:

- Feature suggestions that lack context or a use case
- Support questions (use Discussions or Stack Overflow instead)
- Unreviewed PRs touching core design without prior discussion

If unsure, open a GitHub Issue to talk first.

## Code of Conduct

Please review our [Code of Conduct](CODE_OF_CONDUCT.md). We are committed to
fostering a safe, inclusive environment for everyone.

## Getting Started

```bash
# Fork the repo and clone it locally
git checkout -b feature/your-feature

# Install Python dependencies
pip install -r requirements.txt

# Install Ansible collections
ansible-galaxy collection install -r requirements.yml

# Run lint checks
yamllint .
ansible-lint --profile production

# Run the offline test scenario
molecule test -s render
```

If your change is non-trivial, open an issue first so we can discuss the approach.

## Obvious Fix Policy

You do not need to open an issue for:

- Fixing typos or formatting
- Comment updates
- Minor README changes
- `.gitignore` tweaks

## Commit Conventions

We follow [Conventional Commits](https://www.conventionalcommits.org/):

- `feat:` — new feature
- `fix:` — bug fix
- `docs:` — documentation only
- `refactor:` — code restructuring without feature change
- `test:` — Molecule / CI changes
- `chore:` — maintenance (deps, lint config, etc.)

Breaking changes: append `!` after the type (e.g. `feat!:`) and add a
`BREAKING CHANGE:` footer.

## Code Standards

- FQCN for every module (`ansible.builtin.*`, `community.crypto.*`).
- `loop:` not `with_*`; `loop_control.label` on every loop.
- `include_tasks` (dynamic) for conditional file inclusion; never `import_tasks` + `when`.
- `notify:` handlers; never `when: result.changed`.
- Explicit quoted-octal `mode:` on every file/directory task.
- Bracket access (`result['stdout']`); never dot-notation.
- `no_log: true` on every task touching secrets, tokens, or keys.
- `changed_when:` / `failed_when:` on every `command`/`shell` task.
- Any new variable must have a matching entry in `meta/argument_specs.yml` and a comment in `defaults/main.yml`.
- `ansible-lint --profile production` and `yamllint .` must both be clean (0 violations).

## Security Issues

Please do **not** open a public issue for security vulnerabilities.
Report them privately by email at info@devopsgroup.sk.

## Community

Maintained by [DevOpsGroup s.r.o.](https://devopsgroup.sk/) — info@devopsgroup.sk
