---
name: build-a-clawnify-website
description: Build and ship a Clawnify website — Astro on Cloudflare Workers, the built-in form system (never hand-roll a form), images and assets, and the draft-then-publish deploy flow. Use when creating or editing a Clawnify website, or when a site needs a contact/waitlist/signup form.
---

# Build a Clawnify website

A Clawnify website is an **Astro** project deployed to Cloudflare Workers. One
site per organization. You write pages; the platform owns hosting, forms and
the publish flow.

```bash
clawnify init --type website --name "My Site" website
cd website && pnpm install && pnpm dev     # http://localhost:4321
```

```
website/
  src/pages/          # one .astro file per route (index.astro → /)
  src/layouts/        # shared shell: nav, footer, <head>
  src/blocks/         # platform blocks you've adopted (see Forms below)
  public/             # images, favicons — served as-is from /
```

## Forms: use the built-in system. Do not build your own.

**This is the single most common mistake.** A site needs a contact or waitlist
form, and the obvious move is to write a `<form>` and POST it somewhere — your
own endpoint, a third-party form service, or the platform's submit URL with the
website id pasted in. All three are wrong, and the last one looks right enough
to survive review.

Clawnify has a form system. It captures submissions, stores them, can email you,
and can hand each one to an agent or a flow. You get all of it by using the
**Forms block** rather than writing a form yourself.

### How it works

- The submit target is **computed at render time** from `WEBSITE_ID` — a
  variable the platform bakes into your site's Worker — plus the form's id:

  ```astro
  const env = (Astro.locals as { runtime?: { env?: Record<string, string> } })?.runtime?.env;
  const websiteId = env?.WEBSITE_ID ?? "";
  const action = websiteId ? `https://provision.clawnify.com/forms/submit/${websiteId}/${formId}` : "";
  ```

  **Never hardcode the website id.** It is baked per Worker so a site can only
  ever submit to itself — hardcoding throws that away, and the literal goes
  stale the moment the site is recreated. An empty `action` in local `astro dev`
  is expected: there are no Worker vars locally, so the form simply no-ops.

- The submit script POSTs JSON with `Content-Type: text/plain;charset=UTF-8`.
  That is not a typo — `text/plain` is CORS-safelisted, so the request needs no
  preflight, which matters because the platform can't know your site's origin in
  advance. The route parses the text body as JSON.

- **Keep the Turnstile widget.** The ingest rejects a submission with no token,
  so a form without the widget fails every time with "Verification required".
  The sitekey shipped in the block is public and pairs with the platform's
  secret; the widget will not validate on `localhost`, only on a real hostname.

### Getting the block

Sites are seeded with the platform's block library at creation. If yours has
`src/blocks/Forms/Contact.astro`, import it and go. If it doesn't — an older
site, or one scaffolded bare by the CLI — copy it in from the website template
and adapt it.

**The design is yours; the plumbing is not.** Restyle the markup, labels and
classes freely. Preserve, exactly:

- the computed `action` (never a prop, never a literal),
- the Turnstile widget,
- the `data-clawnify-form` attribute and its submit script,
- `export const schema`,
- destructuring only known props — never splat leftover props onto `<form>`,
  or an injected `action` becomes a way to point your form at someone else.

### Fields

The stock block collects **name, email and message**. Two ways to go beyond
that, and the right one depends on whether your form is registered:

- **Registered forms** (a tree node named `Forms/Contact`) are schema-locked:
  the ingest 422s any field it doesn't expect. Give extra inputs **no `name`
  attribute** and have the submit script fold them into `message`.
- **Unregistered forms** — hand-built pages like this one — are not locked, so
  extra fields post as themselves and are stored verbatim in the payload. Better
  data, but if the site later moves onto the tree renderer, registration will
  start rejecting them. Leave a comment saying so.

### PDF attachments

The Contact block can collect one PDF per submission (job applications, RFQs).
Off by default — enable it with the block's `attachment` prop (and optionally
`attachmentLabel`), then **publish**: the file field is registered at publish
time, and the ingest rejects files on forms that haven't registered one. Rules
the platform enforces server-side (don't fight them client-side): PDF only,
verified by content not by extension, max 10 MB. The submission's payload
carries the filename and a download link that only the site's own organization
can open — the file is never on a public URL. When a file is attached the
submit script posts `multipart/form-data` instead of JSON; both are
preflight-free, and the block already handles the switch.

### Where submissions go

Every submission is stored and visible in the dashboard. From there it can
notify an email address, or fan out to an agent or a flow — which is how a
signup becomes a CRM record or a WhatsApp message without you writing a backend.
Set the notification address in the dashboard, not in the site.

## Images and assets

Put them in `public/` and reference them with an absolute path (`/img/hero.jpg`).
They're served as static files. Pull in real assets at real resolution rather
than linking someone else's CDN from your markup.

## Deploying: draft first, then publish

```bash
clawnify deploy            # → the DRAFT site, visible only to your org
```

Deploying **never** changes the live site. Going live is a separate, explicit
step: review the draft, then **Publish to live** in the dashboard. Tell the user
this — a deploy that "worked" but didn't appear on the live site is almost
always someone expecting deploy to publish.

## Pinning the org

Every command resolves which organization it targets, folder-first. In a linked
folder you get it for free; otherwise pass `--org`. If you work across several
organizations, pass `--org` explicitly on anything that writes — the global
active org is shared machine-wide and another session can change it under you.
