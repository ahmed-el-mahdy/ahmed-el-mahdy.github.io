# Portfolio Maintenance Guide

This repository publishes Ahmed ElMahdy's portfolio through GitHub Pages. Keep changes simple, local, responsive, and easy to verify.

## Source Of Truth

- `index.html`: production page and the only HTML file GitHub Pages serves at the root URL.
- `assets/`: optimized runtime images and local product or technology icons referenced by `index.html`.
- `design-qa.md`: design decisions, review history, and accepted desktop/mobile evidence.
- `design-reference/`: screenshots used by `design-qa.md` for visual regression review.
- `Ahmed-Elmahdy-CV-DevOps.pdf`: downloadable CV linked from the portfolio.

Draft HTML files are not production sources. Promote reviewed work into `index.html` before release.

## Local Preview

Run a local HTTP server from the repository root:

```powershell
python -m http.server 4173
```

Open `http://127.0.0.1:4173/`. Do not validate the site by opening `index.html` directly because browser security and relative asset behavior can differ under `file://`.

## Page Structure

Keep the existing section order unless the portfolio story changes materially:

1. Hero and delivery pipeline
2. Evidence and platform transformation
3. Casework
4. Experience
5. Projects
6. Operating model
7. Technical stack
8. Credentials
9. Contact

Navigation anchors, section IDs, and visible labels must remain aligned.

## Safe Change Workflow

1. Read the affected HTML, CSS, and JavaScript before editing.
2. Reuse existing CSS variables, component classes, spacing, and responsive breakpoints.
3. Change content and behavior in the smallest practical scope.
4. Keep runtime assets local and optimized.
5. Preview the full page on desktop and mobile.
6. Test links, animations, reduced-motion behavior, and horizontal overflow.
7. Update `design-qa.md` and add only the screenshots needed to prove the change.
8. Stage the production files explicitly, review the diff, then publish.

## Design Rules

- Preserve the dark platform-control visual language and the teal, cyan, green, and amber status colors.
- Use the existing compact radii, technical typography hierarchy, and thin borders.
- Keep headings left aligned and readable in one or two intentional lines where space allows.
- Use real brand icons. Prefer official Azure and AWS assets, then maintained Devicon or Simple Icons sources.
- Keep icons local after verifying their license and source. Do not substitute text glyphs, emoji, or invented logos.
- Do not add nested cards, decorative gradients, or animation that competes with the casework.

## Asset Policy

- Hero image: `assets/platform-hero-v2.avif`, with WebP and PNG fallbacks.
- Contact image: `assets/ahmed-at-work-v2.webp`.
- Technology icons: `assets/icons/`.
- Prefer AVIF or WebP for photographs and PNG only as a compatibility fallback.
- Resize images to the largest rendered size and remove unused copies before committing.
- Keep descriptive `alt` text for meaningful images and empty `alt` text for decorative icons.

## Motion And Performance

- Animate `transform` and `opacity` whenever possible.
- Avoid canvas, video backgrounds, GIFs, heavy animation frameworks, and scroll handlers that run every frame.
- Pause or simplify off-screen activity and honor `prefers-reduced-motion`.
- Keep automatic motion slow enough to read and ensure hover is never the only way to reveal information.
- Preserve the hero image preload and avoid adding render-blocking dependencies.
- The current Lucide script is pinned. Test the page before changing or removing that version.

## Responsive QA

Verify at minimum:

- Desktop: 1440 x 1024
- Mobile: 390 x 844
- No horizontal scrolling or clipped text
- Navigation remains usable
- Pipeline, metric, experience, project, stack, and credential layouts collapse cleanly
- Contact photo keeps a useful crop
- Touch targets remain easy to activate
- Animations remain smooth and reduced-motion mode is calm

Use the browser console to confirm there are no missing assets or JavaScript errors.

## Release Checklist

- `index.html` contains the intended release.
- Every local `assets/` reference resolves with HTTP 200.
- Email, GitHub, LinkedIn, Credly, project README, and CV links work.
- Desktop and mobile layouts have no overflow or overlap.
- Metadata, canonical URL, social image, and structured data use the live domain.
- `design-qa.md` reflects the final accepted state.
- `git diff --cached` contains no drafts, private files, or unused source images.
- GitHub Pages serves the new commit at `https://ahmed-el-mahdy.github.io/`.

## Rollback

Use a new revert commit instead of rewriting branch history:

```powershell
git revert <release-commit>
git push origin main
```

After rollback, verify the live page and its assets again. Do not use `git reset --hard` on the shared publishing branch.
