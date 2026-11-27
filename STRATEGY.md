# Strategy — joshburnsm.github.io — 2026-11-27

As of `2a67e45` on `main`. Proposals only; the plan of record is none.

## Where it stands

- A five-file static GitHub Pages site (legacy build from `main` at `/`, HTTPS enforced; `gh api repos/joshburnsm/joshburnsm.github.io/pages`). Last three `pages-build-deployment` runs succeeded, the latest at 2026-11-27T04:22Z (`gh run list -L 5`).
- Its real job is infrastructure for `joshburnsm/pw-charge-by`: it serves the Tesla Fleet API public key at `/.well-known/appspecific/com.tesla.3p.public-key.pem` and the OAuth redirect page `callback/index.html`. pw-charge-by's README names both (`README.md:59`, `:72-76` in that repo).
- Live check passes: all three URLs return 200, and the served key is byte-identical to the committed one (`curl … | diff - <pem>` → match). The key is a 256-bit EC public key with no private material (`openssl pkey -pubin -noout -text`; `grep -c PRIVATE` → 0), so the snapshot's "looks like a secret" flag is a false positive.
- `.nojekyll` is load-bearing: without it Jekyll drops dot-directories and the key path 404s. Nothing in the repo says so; there is no README, and `main` has no branch protection (`gh api …/branches/main/protection` → 404).
- `callback/index.html:9-18` reads `code`/`error` from the query string and writes them with `textContent` (no HTML injection) and loads no third-party resources, so the code isn't leaked via Referer.
- No issues, no open PRs, no milestones, no CI beyond the Pages build; not in the issue-triage scheme's per-repo list, so there is no board to order against.

## Proposed next steps

1. **Record why the files are there, so a tidy-up can't break Tesla auth.** A short README stating that `.nojekyll`, the `.well-known` key and `callback/` serve pw-charge-by, that the key must match pw-charge-by's private key, and how to rotate it (replace the PEM, rerun `pw-charge-by setup register`). Why now: the repo was just trimmed (#2), and the next trim is the one that deletes something that looks unused. No issue yet. Acceptance: README exists and names each load-bearing file and the consuming repo.
2. **Detect a missing or mismatched public key before Tesla does.** Tesla re-fetches the key on registration; a 404 or wrong key surfaces only as a failed `setup register` or failed commands. The cheapest guard is in pw-charge-by (it holds the private key): a check, e.g. in its `dry-run`, that fetches the URL and compares it to the locally derived public key. No issue yet (belongs in pw-charge-by). Acceptance: deleting `.nojekyll` on a test branch, or swapping the PEM, makes that check fail with a clear message.

## Future directions

- **Real homepage content** (speculative, S): only if the owner wants a public profile. Prerequisite: the owner decides what goes on it; #2 deliberately cut it to one line.
- **Custom domain** (M): would change the Tesla allowed origin, redirect URI and `--domain` in pw-charge-by and force re-registration. Prerequisite: a reason to have one; none in the evidence.

## Decisions for the owner

- **Branch protection on `main`.** Options: (a) leave as is, (b) require PRs (no required checks, since there is no CI). Recommendation: (b), cheap and consistent with "changes reach main only through a PR"; pushes to `main` deploy live within ~25 s.
- **Where the key check lives** (step 2). Options: here as a scheduled workflow that curls the URL, or in pw-charge-by. Recommendation: pw-charge-by, since only it can compare against the private key; a curl-only check here would catch a 404 but not a mismatch.
- **Stale merged branches** `normalize-line-endings` and `trim-homepage-text` remain on origin and `delete_branch_on_merge` is `false`. Recommendation: delete them and turn the setting on.

## Not recommended

- **Adding the template-repository CI workflows.** There is no Python, no tests and no build; a lint/test job would check nothing. The Pages build is the only meaningful check and already runs.
- **Moving the key out of the repo or gitignoring it** because a scanner flagged it. It is a public key that Tesla requires at exactly this path.
- **A static-site generator or framework.** Five files, two of them functional; a generator adds a build step that could itself drop `.well-known`.
