# Build and run a personal blog with Claude Code

> How this site was built end to end with Claude Code: Hugo + PaperMod, GitHub Pages deploy with a deploy key, custom domain with HTTPS, and an Obsidian writing workflow. Includes the pitfalls that cost real time.

- Source: https://uestcyz.top/playbooks/build-a-blog-with-claude-code/
- Applies to: Hugo extended 0.158+ (CI pinned to 0.169.0), PaperMod theme, GitHub Pages, macOS
- Last verified: 2026-11-09
- Skill version of this page: https://uestcyz.top/playbooks/build-a-blog-with-claude-code/skill.md


## Goal

A personal blog that is cheap to run and easy to write for:

- Hugo + PaperMod, restyled only through extension points (the theme itself is never edited).
- Source repo and output repo are separate. A GitHub Action builds the source and force-pushes a single commit to `<user>/<user>.github.io`.
- Custom domain with HTTPS.
- Posts are written in Obsidian and published with one button.

This is the process behind <https://uestcyz.top>. Names such as `<user>` and `<domain>` are placeholders.

## Who does what

Do the technical steps yourself. Stop and ask the user for these, because they involve credentials or accounts the agent should not create on its own:

1. Creating or approving a GitHub login (`gh auth login`, a device code the user enters in the browser).
2. Writing the deploy key into repository secrets. Give the user the exact command and let them run it (in Claude Code: `! gh secret set ...`).
3. Changing DNS records at the domain registrar.

Never commit tokens or private keys. Never print a private key into the chat.

## Prerequisites

- `hugo` (extended, 0.158 or newer), `git`, `gh` (logged in), `ssh-keygen`.
- A GitHub account. A domain is optional.
- Obsidian is optional and only needed for the writing workflow in step 5.

## Steps

### 1. Scaffold the site

```bash
hugo new site blog && cd blog && git init
git submodule add https://github.com/adityatelange/hugo-PaperMod themes/PaperMod
```

In `hugo.yaml` set at least:

```yaml
baseURL: https://<domain>/
theme: PaperMod
hasCJKLanguage: true            # only if you write Chinese, Japanese or Korean
timeZone: Asia/Shanghai         # the time zone you write in, not the machine's
permalinks: { page: { posts: /posts/:slugorcontentbasename/ } }
```

Keep article URLs short and stable (`slug` in front matter). Changing a URL later breaks every link to it.

### 2. Customize only through extension points

Never edit files under `themes/PaperMod/`. A theme upgrade would overwrite them. Use these instead:

| Goal | File |
|---|---|
| Styles | `assets/css/extended/*.css` (one file, design tokens at the top) |
| Extra `<head>` or scripts | `layouts/_partials/extend_head.html`, `extend_footer.html` |
| Extra content after a post | `layouts/_partials/extend_post_content.html` |
| Markdown behavior | `layouts/_markup/render-*.html` (images, callouts, math) |
| A page type | a copy of the theme template in your own `layouts/`, with a comment recording the theme commit it came from |

Write the project rules into `AGENTS.md` (structure, commands, "never edit the theme", date format). The agent reads it at the start of every session.

Math: enable goldmark `passthrough` and render with `transform.ToMath` in a render hook; load the KaTeX CSS only on pages that contain math. Images: a render hook that resizes and converts to WebP keeps the repo and pages light.

### 3. Bring content in

- Put posts in `content/posts/`. Required front matter: `title`, `description`, `date` with an offset (`2026-01-02T10:00:00+08:00`), `categories`, `tags`.
- When migrating from another generator, keep the old URLs working with `aliases` in front matter. Redirect old taxonomy URLs in CI, not locally (see pitfall 6).
- Check locally: `hugo --gc --minify` must finish with no warnings.

### 4. Deploy: two repos and a deploy key

Why a separate output repo: the source repo can hold drafts and automatic backups, and the Pages repo stays tiny (one commit).

1. Create `<user>/<source>` (the source) and `<user>/<user>.github.io` (the output). In the output repo set Pages to deploy from the `main` branch.
2. Create a deploy key pair on the machine, in a temporary directory:

   ```bash
   ssh-keygen -t ed25519 -N "" -f /tmp/deploy_key -C "pages-deploy"
   ```

3. Add the public key (`/tmp/deploy_key.pub`) to the output repo as a deploy key with write access. Add the private key to the source repo as the secret `ACTIONS_DEPLOY_KEY`. This is the credential step: ask the user to run the secret command. Delete the key files afterward.
4. Add `.github/workflows/deploy.yml`. It triggers only on the `publish` branch, so everyday backups on `main` never go live:

   ```yaml
   on: { push: { branches: [publish] }, workflow_dispatch: {} }
   jobs:
     build:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
           with: { submodules: recursive }
         - uses: peaceiris/actions-hugo@v3
           with: { hugo-version: '0.169.0', extended: true }
         - run: hugo --gc --minify --baseURL "https://$(tr -d '[:space:]' < static/CNAME)/"
         - uses: peaceiris/actions-gh-pages@v4
           with:
             deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }}
             publish_dir: ./public
             external_repository: <user>/<user>.github.io
             publish_branch: main
             force_orphan: true
   ```

5. Ship with `git push origin main && git push origin main:publish`.

Use a deploy key rather than a personal access token. Tokens expire, and an expired one makes the deploy fail with a confusing error.

### 5. Custom domain and HTTPS

1. Put the domain in `static/CNAME` (one line, no protocol). The workflow reads the base URL from this file.
2. At the registrar add four `A` records for the apex domain (`@`) pointing to `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`. This is a human step. A `CNAME` record on the apex does not work with some registrars, so use `A` records.
3. Wait until GitHub has issued the certificate, then enforce HTTPS:

   ```bash
   gh api -X PUT repos/<user>/<user>.github.io/pages -f cname=<domain> -F https_enforced=true
   ```

   If this returns an error saying the certificate does not exist yet, wait and retry. It can take minutes to an hour.

### 6. Writing workflow with Obsidian

Use the open-source toolkit <https://github.com/1-pluto1/obsidian-hugo-blog>. Its installer turns `content/posts` into an Obsidian vault and adds a plugin with buttons for new post, preview and publish, image paste with WebP compression, and checks before publishing.

- Back up automatically to `main` (Obsidian Git); publish with the plugin, which pushes to `publish`.
- Exclude vault tooling from the site: `ignoreFiles: ['/posts/Templates/', '/posts/Scripts/', '/\.obsidian/']`. Mount the images folder as assets with `module.mounts` so Markdown links like `attachments/a.png` resolve.

### 7. Work with the agent

- Iterate locally with `hugo server` and look at real screenshots (light, dark, mobile). Do not push each small tweak.
- Batch the changes, then push once.
- After every push, read the deploy result. Do not assume it worked:

  ```bash
  gh run watch "$(gh run list -b publish -L1 --json databaseId -q '.[0].databaseId')" --exit-status
  ```

  then fetch the live page and check for the specific change you made.

## Verify

All of these must hold:

- `hugo --gc --minify` prints no warnings.
- The latest run on the `publish` branch ends with `success`, and its head commit equals your local `HEAD`.
- `curl -sI https://<domain>/` returns `200` and the certificate is valid. `http://<domain>/` redirects to HTTPS.
- A post with math, a post with an image, the search page and dark mode all render.
- Old URLs (if you migrated) answer with a redirect, not `404`.

## Pitfalls

1. **A web font works locally and fails on the live site.** Some mirrors (for example npmmirror) reject requests whose `Referer` is your domain. Test font URLs with `curl -H "Origin: https://<domain>" -e "https://<domain>/"`. jsDelivr worked. Compare local and live screenshots in the same browser.
2. **Math shows unstyled.** The page flag for "has math" is only known after the content renders, so a `<head>` partial must touch `.Content` first.
3. **Words like `vector<int>` or `<PAD>` vanish from a post.** They are parsed as HTML. Wrap them in backticks.
4. **Remote data fails locally.** `resources.GetRemote` can be blocked by a local proxy. Make the template silently skip on error, so the build works everywhere and shows the data on CI.
5. **Deploy fails before any job starts, with "workflow file issue".** Check <https://www.githubstatus.com> first. GitHub Actions had an outage during this build and nothing in the repository was wrong. Editing the workflow does not help.
6. **Case-insensitive file systems.** If old URLs differ only by letter case (`/tags/LLM` and `/tags/llm`), generate those redirects in CI on Linux, never on macOS.
7. **Obsidian Git commits half-finished edits.** Make larger refactors in a scratch copy or when the vault is idle, then apply them at once.
8. **`echo ok` after a watch command hides failures.** Use the exit code of `gh run watch --exit-status`.

## Prompts you can give an agent

- "Read `AGENTS.md`, then restyle the home page. Change only CSS and `layouts/_partials/`. Preview locally with screenshots in light, dark and mobile, and do not push until I say so."
- "Deploy failed. Read the run log and tell me whether it is our workflow or a GitHub incident before changing anything."
- "Migrate these old posts. Keep the old URLs working with `aliases`, convert external images to local files, and report any link that cannot be fixed."

