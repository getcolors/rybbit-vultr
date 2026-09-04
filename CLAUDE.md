# CLAUDE.md

## Repository

Desired state for `rybbit-vultr`: Rybbit privacy-friendly web & product
analytics stack on one Vultr instance in Amsterdam, published at
`https://rybbit.getcolors.ai` through Cloudflare and Caddy. Behavior lives
in `../rybbit`.

Tracked source is `colors.yml`, toolchain and documentation, the installed
Package Skill, and a root launcher copied from its payload.
`.colors/` is generated private state and `.envrc.private` contains credentials;
never read, edit or commit either. The machine SSH keypair is
`~/.ssh/rybbit-vultr`(`.pub`), named by profile per
`workspace/standards/ssh-keypair.md`; the matching `~/.ssh/config` entry covers
the alias and the instance address, so converges authenticate without an agent.
Losing that private key means losing SSH access to the live instance — never
regenerate it while the instance exists.

## Commands

```sh
./green build
./green create --dry-run
./green create
./green delete
```

Build and dry-run require no credentials. Never export `COLORS_PAR_PROFILE`.
Keep `compute-prevent-destroy: true`; deletion requires separate authorization
and a one-run `COLORS_PAR_COMPUTE_PREVENT_DESTROY=false` override.

The root `green` is a copy, not a symlink. After a Package Skill update copy
`.agents/skills/package-rybbit-green/green` over it. Never hand-edit its SHA.

Vultr treats the instance `hostname` and `ssh_key_ids` as ForceNew: the
playbook sets the hostname from `vultr-name`, and rotating the SSH key means
rebuilding the instance, never editing it in place.

## Documentation

`index.html` is this repository's landing page and carries two analytics tags:
GA4 measurement ID `G-4VKP1WY4QJ`, whose explicit `page_title` must exactly
equal the decoded HTML `<title>` and stay distinct and stable so one Analytics
property can separate repositories, and the self-hosted Rybbit snippet
`<script src="https://rybbit.getcolors.ai/api/script.js" data-site-id="9fb9c41a6d49" defer></script>`,
which shares one site ID across every page because `getcolors.github.io/<repo>/`
paths already encode the repository. Never add one tag without the other.

## Git

Work on the current branch. Do not commit or push unless explicitly authorized.

## The first real create after the SSH Config Standard will refuse, by design

The rybbit package writes a managed `Host rybbit-vultr` block into the
operator's `~/.ssh/config` and, per `workspace/standards/ssh-config.md` §5,
never adopts a stanza it did not write. This machine's `~/.ssh/config`
opens with a hand-written `Host rybbit-vultr <ip>` stanza from the
2026-08-24 migration, outside any managed markers, carrying the
`IdentityFile ~/.ssh/rybbit-vultr` that is the only SSH path into the live
host. A real `create` (once the revoked credentials are re-minted) will
stop before rendering anything with `refusing to manage ~/.ssh/config: it
already declares Host rybbit-vultr at line 1 outside this package's managed
block`. That is correct.

Do not clear it by deleting the stanza. This deployment is in opt-out mode
(`vultr-ssh-keys` is set), and an opt-out managed block carries no
`IdentityFile`, so the replacement would be strictly less capable than the
hand-written entry and `ssh rybbit-vultr` would stop authenticating. Decide
first: keep opt-out and keep an operator-owned identity stanza for the
alias below the managed block (the managed block wins on HostName and User
only), or move the deployment to keygen mode, which is a rebuild because
`vultr-ssh-keys` is ForceNew. Back the line-1 stanza up verbatim before
either. Never delete or regenerate `~/.ssh/rybbit-vultr`.

