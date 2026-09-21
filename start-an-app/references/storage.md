# File uploads (local folder → Vercel Blob)

Last verified: 2026-09-21

**Purpose:** Let users upload files (photos, avatars, documents) with **one storage module and two backends chosen at runtime**. In local development files are written into `public/uploads/` and served straight off disk — nothing to sign up for. In production the same code writes to Vercel Blob. The calling code never knows which one it got.

> Why the switch is necessary, in plain words for the user: on Vercel the app's filesystem is read-only and thrown away between requests, so "save it in a folder" simply cannot work there. And `public/` is snapshotted at build time, so even a file written at runtime would never be served. Local folder for development, blob storage for the real thing.

**The switch is presence-based, not a mode flag.** If a blob credential is in the environment, blob wins; otherwise local. That means the user configures nothing locally, and Vercel configures itself the moment a Blob store is connected to the project.

## Decide first: who may read a file

Everything under Configure below stores files where **anyone who has the URL can open them**: `public/uploads/` locally, a public blob store in production. That is right for content meant for strangers: a product photo on a public page, an avatar, a cover image on a published post.

It is wrong for anything private or about third parties. A plumber's photos of a customer's home, a scanned invoice, a document one user uploaded for themselves: a long random URL is not access control, because it gets pasted into a message, stays in browser history, and keeps working after the person who shared it has left the team.

**The default for anything private or about other people: store it privately and serve it through a route handler that checks the session and the same role or ownership rule the record itself uses. Never hand out a public URL for it.** Use public access only for content meant for strangers. Write the choice on the build sheet in one line ("job photos: private, visible to the assigned plumber and the office").

"Private files" below has the changes. Decide before building, because moving files from a public store to a private one later means re-uploading every file.

## Install

```bash
pnpm add @vercel/blob
```

Nothing goes in `.env` for local development — the absence of the token *is* the local-mode signal.

Ignore the upload folder so user files never land in git:

```bash
echo "/public/uploads" >> .gitignore
```

## Configure

Four small files under `src/lib/storage/`.

`src/lib/storage/types.ts` — shared shape, kept separate so the two backends don't import each other:

```ts
export type StoredFile = {
  /** Public URL to render in an <img> or link. */
  url: string;
  /** Stable key used to delete the file later. */
  pathname: string;
};
```

`src/lib/storage/local.ts`:

```ts
import { mkdir, writeFile, unlink } from "node:fs/promises";
import { randomUUID } from "node:crypto";
import path from "node:path";
import type { StoredFile } from "./types";

const root = () => path.join(process.cwd(), "public");

export async function saveLocal(file: File, folder: string): Promise<StoredFile> {
  const name = `${randomUUID()}${path.extname(file.name)}`;
  const pathname = `${folder}/${name}`;
  await mkdir(path.join(root(), folder), { recursive: true });
  await writeFile(path.join(root(), pathname), Buffer.from(await file.arrayBuffer()));
  return { url: `/${pathname}`, pathname };
}

export async function deleteLocal(pathname: string): Promise<void> {
  await unlink(path.join(root(), pathname)).catch(() => {});
}
```

`src/lib/storage/blob.ts`:

```ts
import { put, del } from "@vercel/blob";
import type { StoredFile } from "./types";

export async function saveBlob(file: File, folder: string): Promise<StoredFile> {
  const blob = await put(`${folder}/${file.name}`, file, {
    access: "public",
    addRandomSuffix: true,
  });
  return { url: blob.url, pathname: blob.pathname };
}

export async function deleteBlob(pathname: string): Promise<void> {
  await del(pathname);
}
```

`src/lib/storage/index.ts` — the only file the rest of the app imports:

```ts
import { saveLocal, deleteLocal } from "./local";
import { saveBlob, deleteBlob } from "./blob";
import type { StoredFile } from "./types";

export type { StoredFile };

/**
 * Blob storage wins whenever a credential is present. On Vercel these are set
 * automatically once a Blob store is connected to the project:
 *   - BLOB_STORE_ID   (+ VERCEL_OIDC_TOKEN) — the default, short-lived credentials
 *   - BLOB_READ_WRITE_TOKEN — long-lived fallback, also used off-platform
 */
export const usingBlobStorage = Boolean(
  process.env.BLOB_READ_WRITE_TOKEN || process.env.BLOB_STORE_ID,
);

export function saveFile(file: File, folder = "uploads"): Promise<StoredFile> {
  return usingBlobStorage ? saveBlob(file, folder) : saveLocal(file, folder);
}

export function deleteFile(pathname: string): Promise<void> {
  return usingBlobStorage ? deleteBlob(pathname) : deleteLocal(pathname);
}
```

Upload endpoint `src/app/api/upload/route.ts`:

```ts
import { saveFile } from "@/lib/storage";

const MAX_BYTES = 4 * 1024 * 1024; // stay under Vercel's 4.5 MB request limit
const ALLOWED = ["image/jpeg", "image/png", "image/webp", "image/gif"];

export async function POST(req: Request) {
  const form = await req.formData();
  const file = form.get("file");

  if (!(file instanceof File)) {
    return Response.json({ error: "No file uploaded." }, { status: 400 });
  }
  if (file.size > MAX_BYTES) {
    return Response.json({ error: "That file is larger than 4 MB." }, { status: 413 });
  }
  if (!ALLOWED.includes(file.type)) {
    return Response.json({ error: "Only images are allowed." }, { status: 415 });
  }

  return Response.json(await saveFile(file));
}
```

**If sign-in was chosen, guard this route** — an open upload endpoint is a free file host for the whole internet. Check the session first and return 401 when there isn't one:

```ts
const session = await auth.api.getSession({ headers: req.headers });
if (!session) return Response.json({ error: "Sign in first." }, { status: 401 });
```

**If you render uploads with `next/image`, allow the blob host in `next.config.ts`** — otherwise images work locally (where the URL is a same-origin `/uploads/...` path) and break the moment they come from blob storage, which is a miserable thing to debug after deploying:

```ts
images: {
  remotePatterns: [{ protocol: "https", hostname: "*.public.blob.vercel-storage.com" }],
},
```

A plain `<img>` needs none of this — fine for a first version.

**One file per record: store the returned `url` and `pathname` on the row they belong to** — an `imageUrl` / `imagePath` column on the record's own table. `url` is what you render; `pathname` is what you pass to `deleteFile` when the row is deleted.

**Many files per record need their own table, not a column on the parent.** Photos of a job, attachments on a ticket: a column holds one file, and a JSON list in a column cannot be queried, cannot be deleted one at a time, and cannot say who uploaded what.

```ts
export const jobPhotos = pgTable("job_photos", {
  id: uuid("id").primaryKey().defaultRandom(),
  jobId: uuid("job_id")
    .notNull()
    .references(() => jobs.id, { onDelete: "cascade" }),
  pathname: text("pathname").notNull(),
  uploadedBy: text("uploaded_by").references(() => user.id, { onDelete: "set null" }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});
```

Postgres branch shown; SQLite uses `text` ids and `integer` timestamps per `references/database.md`. Name the table after the app's own noun. Add a `url` column only when the files are public; a private file has no URL worth storing. `uploadedBy` is `"set null"` on shared team data so the photos stay when a member leaves, and `"cascade"` where each user's files are private to them, the same choice `references/database.md` describes for every other table. Drop the column if the app has no accounts.

**The upload route then takes the parent id and checks the uploader may attach to that parent.** Read a `jobId` field from the same `FormData`, load the job, and apply the rule the job's own pages apply (the session user owns it, or their role may edit it) before saving anything. Answer 404 when the parent does not exist or is not theirs. Insert the photo row in the same request. A route that accepts any signed-in user's file and leaves the client to attach it later lets one user attach files to another's record. This pattern has not yet been built with this skill; the Verify item below proves it.

### Private files

Not yet built with this skill: confirm the blob store's current API for private access in the check-what's-current step, and prove the result with the Verify items below.

What changes from the public setup:

- **Local backend:** write outside `public/`, for example to `data/uploads/` (`data/` is already gitignored on most branches; add it if not). Anything under `public/` is served to anyone, with no code in between.
- **Blob backend:** create the store with private access and pass the matching `access` value to `put`. The exact option, and how the server reads a private blob back, is what the research step confirms.
- **`StoredFile` carries only `pathname`.** Nothing stores or renders a storage URL.
- **One route handler serves every private file**, for example `src/app/api/files/[id]/route.ts`. It looks the file row up by id, loads the parent record, applies the same session and role or ownership check the parent's page uses, and only then streams the bytes with the right `Content-Type`. It answers 401 signed out and 404 when the row does not exist or the caller may not see it. Pages render `<img src="/api/files/<id>">`.
- **Do not set `Content-Length` from a size the storage SDK reports.** Let the stream set it, or leave it out: a wrong length makes the browser save a short or empty file.
- `next/image` and the `remotePatterns` entry above are for public files. A private file served by the route handler uses a plain `<img>`.

### Photos taken on a phone

When the app's main upload is a photo taken on a phone (a site visit, a receipt, a plant), plan for it from day one. Two things break the simple route above:

- **Size.** Phone camera photos are often larger than the 4 MB limit above, and on Vercel a request over 4.5 MB never reaches the function at all. Use client-side direct upload (`@vercel/blob/client`) from the start: the browser sends the file straight to the store, and the app's route only issues the upload token after checking the session and the parent record. The same check applies; it moves to the token step. Locally there is no such limit, so the local backend keeps the simple route with a higher `MAX_BYTES`.
- **Format.** iPhones can hand over HEIC, which most browsers cannot display and which is not in `ALLOWED`. Either set `accept="image/jpeg,image/png,image/webp"` on the file input, which makes iOS convert to JPEG in most cases, or convert on the server. Whichever is chosen, a HEIC file must end in a displayed photo or a plain sentence, not a broken image.

Not yet built with this skill: confirm the client-upload API in the check-what's-current step, and test with a real phone photo rather than a small test image, because a small image passes every check here and proves nothing.

### When a record or an account is deleted

Deleting a row does not delete its file. The database cascade removes the row that held the `pathname`, and after that nothing knows the file exists.

- **Deleting a record:** read the pathnames of its files first, delete the rows, then call `deleteFile` for each pathname. Do this in the shared `src/lib/<domain>` function, so an agent's delete cleans up as well.
- **Deleting an account:** `references/settings.md` builds this, and the order matters. Its `beforeDelete` hook collects every pathname the user owns **before** the rows cascade away; its `afterDelete` hook calls `deleteFile` for each. Collected afterwards, there is nothing left to collect.
- **What makes that possible is decided here:** every stored `pathname` sits in a column on a row that reaches the user, either directly (`userId`, `uploadedBy`) or through its parent (`job_photos` → `jobs.userId`). List those tables on the build sheet when this step finishes, so the settings step knows where to look. A pathname held anywhere else, such as inside a JSON column or a rich-text body, will be missed.
- **Shared team data is the exception.** Photos attached to the team's records belong to the team. When a member's account is deleted, `uploadedBy` becomes null and the files stay. Only files that were that person's alone, such as an avatar, are collected and deleted.

Build the upload UI to match the interview (an avatar picker, a photo on each entry, an attachment list) — a bare "choose a file" test page is the fallback, not the goal. Post a `FormData` with a `file` field to `/api/upload`, then save the returned `url` with the record.

## Going to production

Tell the user at hand-off, in this order:

1. Deploy to Vercel.
2. In the project, open **Storage → Create Database → Blob**, set access to **Public** where the files are meant for strangers, or to private access where "Private files" above applies, and connect it to the project.
3. That's it — Vercel injects the credentials, `usingBlobStorage` flips to true on the next deploy, and uploads go to blob storage. No code change, no config file.

Files uploaded locally stay local; they are development data and were never in git.

For files over 4.5 MB the request never reaches the function — that needs client-side direct upload (`@vercel/blob/client`). Where the main upload is a phone photo it is planned from day one, as "Photos taken on a phone" above says; otherwise it is worth adding only when the app actually needs it.

## Verify

**No account is created to run these checks.** On an app with sign-in the upload route refuses anyone signed out, so everything that needs a real upload waits for the first account, which is the real person's and is made in Step 6. The upload UI also needs the pages it sits on, which are built in Step 5.

Checked now:

- Types compile, and `/public/uploads` (and `data/` for private files) is in `.gitignore`.
- Apps with accounts: signed out, a POST to the upload endpoint returns 401 and nothing is written to disk. Private files: signed out, `/api/files/<any id>` returns 401.
- No-accounts apps only: a POST of a small image with `curl -F "file=@test.png"` returns a `pathname`, the file appears in the upload folder, and oversized and wrong-type files produce the friendly error, not a crash.

Deferred to Step 6, after the user has signed up:

- Upload an image through the app's own UI: it appears in `public/uploads/` (or `data/uploads/` for private files) and renders on the page.
- The saved file survives a page refresh, so it is persisted on the record and not just held in React state.
- `git status` shows no uploaded files.
- Oversized and wrong-type files produce the friendly error, not a crash.
- Many files per record: a second file on the same record adds a row and does not replace the first, and an upload naming a parent id the account may not edit answers 404 and writes no file.
- Private files: the file's address opened in a private browser window, signed out, returns 401 and no bytes. Where a second account or role can exist, `references/verify.md`'s isolation probe requests the file as the account that must not see it and gets 404.
- Phone photos: a real photo from a phone, several megabytes, uploads and displays.
- Deleting the record removes its files from the upload folder. Deleting the account is checked in `references/settings.md`, which owns that flow.
