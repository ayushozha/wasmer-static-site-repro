# Static site in a folder fails to deploy from GitHub on Wasmer

A plain HTML site (`index.html`, `about.html`, `style.css`) lives in `website/`. Deploying this repo from GitHub
fails before the build starts, because no framework is detected. Uploading the `website/` folder as a ZIP deploys
and serves the site.

| How it's deployed | Result |
| --- | --- |
| GitHub, default settings | ❌ Fails: “Could not detect repo configuration … Anybuild could not detect a provider for this project” |
| ZIP of the `website/` folder ([website.zip](https://github.com/ayushozha/wasmer-static-site-repro/releases/download/v1/website.zip)) | ✅ Deploys, `/` and `/about.html` return HTTP 200 |
| GitHub, root directory set to `website` | ✅ Deploys, HTTP 200 |
| GitHub's own “Download ZIP” of the repo, uploaded | ❌ Fails with the same detection error |

Reproduced on production on Nov 12, 2026, with Anybuild 0.33.3.

## Steps to reproduce

1. In the Wasmer dashboard, choose **Deploy from GitHub** and pick this repository. Leave the root directory empty.
2. Deploy.

**Expected:** Wasmer finds `website/index.html` and serves it as a static site.

**Actual:** detection fails and the deploy stops:

```
invalid input: Could not detect repo configuration: invalid input: anybuild plan failed with status exit status: 1
while running `anybuild plan /tmp/.tmpwcCpi0/out --wasmer -o /tmp/.tmpwcCpi0/anybuild.plan.json`;
stderr: Error: plan failed for /tmp/.tmpwcCpi0/out: Anybuild could not detect a provider for this project
```

The build log for the failed deploy shows only:

```
Anybuild 0.33.3
Error: auto failed for /tmp/.tmpwGIV74/out: Anybuild could not detect a provider for this project
```

To compare, download [website.zip](https://github.com/ayushozha/wasmer-static-site-repro/releases/download/v1/website.zip)
(the `website/` folder, zipped the way macOS **Compress** does it) and deploy it with **Upload a ZIP**. It
deploys and serves both pages.

## Why GitHub and ZIP behave differently

The backend treats the two sources differently before and after Anybuild runs:

- **ZIP uploads step into a single top-level folder.** A ZIP that holds only `website/` is built from inside
  `website/`, where `index.html` sits at the top. Anybuild detects a static site.
- **GitHub deploys start at the repository root.** The root holds `README.md` and `website/`, Anybuild finds no
  provider there, and the deploy stops.
- **The backend's static-site fallback never runs.** For ZIP uploads, the backend has a fallback: if Anybuild
  returns no plan and the files contain an `index.html` anywhere (or a `wasmer.toml`), deploy as a static site.
  Anybuild 0.33.3 no longer returns “no plan”; it exits with an error, so the fallback is skipped. That's why
  GitHub's “Download ZIP” of this repo fails too.
- **GitHub deploys have no fallback at all.** When Anybuild finds nothing, the GitHub path returns
  “Could not detect repo configuration.”

So a static site that isn't at the top of the repository can't be deployed from GitHub without manually setting the
root directory, and the error doesn't say that.

## Workarounds today

- Set the root directory to `website`, or
- Move the site to the repository root, or
- Zip the `website/` folder itself and upload that.

## Suggested fixes

1. Run the static-site fallback when Anybuild reports “could not detect a provider”, not only when it returns no
   plan, and use the folder that contains `index.html` as the site root.
2. Give GitHub deploys the same fallback as ZIP uploads.
3. When nothing is detected, tell the user which folders contain an `index.html` and offer them as the root
   directory.
