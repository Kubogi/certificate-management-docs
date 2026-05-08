# CLI scripts

Two ad-hoc Node scripts that connect directly to MongoDB and exit. Both load the repo-root `.env` and use the same `URI` as the running server.

## register.js — create a user

```sh
node backend/register.js <username> <password> <role>
```

Source: [backend/register.js](../../backend/register.js).

### What it does

1. Connects to `process.env.URI`.
2. Checks whether `username` already exists. If so, **logs an error but continues anyway** ([register.js:19-21](../../backend/register.js#L19-L21)) — and then attempts to insert a duplicate. Mongoose will not raise a uniqueness error because `User.username` has no unique index, so you'll end up with two users sharing a username. Useful as a "force replace" only by accident.
3. Hashes `password` with bcryptjs (12 rounds).
4. Inserts a new `User` document.
5. Logs `"User '<username>' created successfully."` and exits 0.

### When to use it

- **First-time setup.** There is no self-serve registration; an admin must exist before anyone can log in.
- **Recovering a locked-out admin.** If everyone forgets the admin password and there's no other admin to reset it via `/api/auth/update_user`, run this script with a fresh admin record.

### Caveats

- `<role>` is not validated — pass `admin`, `manager`, or `viewer`. Any other string creates a user that authenticates but fails every role-gated route.
- Running it twice with the same username creates duplicates. Check with `db.users.find({ username: ... })` first, or use `POST /api/auth/update_user` instead.

## remove_dupes.js — dedupe students

```sh
node backend/remove_dupes.js
```

No CLI arguments. Source: [backend/remove_dupes.js](../../backend/remove_dupes.js).

### What it does

1. Connects to `process.env.URI`.
2. Aggregates the `students` collection grouping by `(ma_sinh_vien, truong_id)`.
3. For every group with more than one document, keeps the first `_id` and deletes the rest.
4. Logs how many were removed per group and a grand total.
5. Disconnects and exits.

The aggregate filters out groups where either field is `null` ([remove_dupes.js:19](../../backend/remove_dupes.js#L19)), so students with missing `ma_sinh_vien` are never deduped.

### Why this dedupe key

`ma_sinh_vien + truong_id` is the natural identity for a student record (a student ID is unique within a school but not across schools). The schema's only unique index is on `so_hieu_chung_chi` (certificate serial), so duplicates by student-ID can creep in via uploads where the same student appears twice with different certificate serials.

### When to use it

After bulk imports, if you suspect the source spreadsheet contained duplicates that slipped through. There is no UI button for this — it's purely a maintenance script.

### Safety notes

- The script picks "the first" `_id` per group (the order returned by `$group`, which is *not* stable). If two duplicates differ in any field other than `ma_sinh_vien` and `truong_id`, you may lose data unpredictably. Take a backup first.
- There is no dry-run mode. The deletes happen immediately.

## What's missing

There are no other scripts. Specifically there is no:

- Database seeder (no fixtures live in the repo)
- Migration runner (no schema versioning)
- Backup/restore helper
- Import-from-CSV tool (the bulk import path is HTTP-only via `/api/admin/upload`)

If you need any of those, write them as new scripts in `backend/` and document them here.
