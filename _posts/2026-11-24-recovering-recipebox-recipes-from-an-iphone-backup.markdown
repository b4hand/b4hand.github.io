---
layout: post
title:  "Recovering RecipeBox recipes from a local iPhone backup"
date:   2026-11-24 19:55:00 -0600
---

I had recipes I'd typed into [RecipeBox](https://www.getrecipebox.com)
that I could no longer get to in the app. I did still have old local
backups of the phone on my Mac, and those backups keep a copy of each
app's data. This post walks through pulling an app's files out of a
local backup, including an encrypted one, and turning RecipeBox's
database back into readable recipes.

The extraction script works for any app; RecipeBox is just the
default. Both scripts are at the end of the post. They only need
Python's standard library, plus one open-source package for encrypted
backups.

**Try the easy way first.** Every recipe RecipeBox saves also gets a
page on getrecipebox.com, so if you can still sign in to your
RecipeBox account on the web, look there before digging through
backups.

## What you need

- A Mac with a local backup of the phone, made with Finder or iTunes,
  from when the recipes were still in the app. iCloud backups aren't
  stored on your Mac, so this won't help with those.
- Python 3. macOS offers to install it the first time you run `python3`.
- Full Disk Access for your terminal app, under System Settings →
  Privacy & Security → Full Disk Access. Without it, macOS won't let
  the terminal read the backup folder.
- If the backup is encrypted (the "Encrypt local backup" checkbox in
  Finder), the backup password and the
  [`iphone_backup_decrypt`](https://github.com/jsharkey13/iphone_backup_decrypt)
  Python package. The password is the one chosen when encryption was
  turned on which is probably not the same as your Apple ID
  password. There's no way to recover it if it's lost.

## How a local backup is laid out

Backups live in `~/Library/Application Support/MobileSync/Backup/`,
one folder per device. Folders with a date and time on the end are
archived copies of older backups.

Each backup folder holds:

- `Info.plist`: the device name, model, iOS version, backup date and
  the list of installed apps.
- `Manifest.plist`: whether the backup is encrypted, plus the apps and
  their data containers.
- `Manifest.db`: a SQLite index of every file in the backup. Each row
  has a *domain*, the file's path within that domain, and a
  `fileID`. The file's contents are stored at
  `$FILE_ID_PREFIX/$fileID` where `FILE_ID_PREFIX` is the first two
  characters of `fileID` inside the backup folder.

In an encrypted backup, `Manifest.db` and every file are encrypted,
but the two `.plist` files aren't. That's how the script can tell
which backups contain an app before you enter a password.

An app's files are spread across a few domains:

| Domain | What it holds |
|---|---|
| `AppDomain-$BUNDLE_ID` | The app's own folder |
| `AppDomainPlugin-$EXTENSION_ID` | Folders for the app's extensions, such as share-sheet extensions |
| `AppDomainGroup-$GROUP_ID` | Shared "app group" folders that the app and its extensions both use |

RecipeBox's bundle ID is `com.7thfrequency.Recipe`, and it keeps its
recipes in an app group: the database is `Recipe.sqlite` in the
`AppDomainGroup-group.com.7thfrequency.recipe` domain. The backup
doesn't record which app owns which app group, so the script includes
RecipeBox's two groups by default and suggests likely groups for other
apps.

## Step 1: Find the backups that have the app

Save the two scripts from the end of this post into a new folder,
open Terminal in that folder, and run:

```sh
python3 extract_app_files.py --list
```

This reads only the unencrypted `.plist` files, so it doesn't need a
password. For each backup that has the app, it prints something like:

```
XXXXXXXX-XXXXXXXXXXXXXXXX
  device:    $DEVICE_NAME ($MODEL, iOS $VERSION)
  backed up: $DATE_TIME
  encrypted: yes
  app groups that may belong to it: group.com.7thfrequency.recipe, group.com.codefortravel.recipe
```

The first line is the backup's folder name. You can pass it to the
next step to extract from just that backup.

For a different app, pass its bundle ID with `--app` and any app
groups with `--group`. To find a bundle ID, search the backups' app
lists for part of the app's name or its developer's name:

```sh
plutil -p ~/Library/Application\ Support/MobileSync/Backup/*/Info.plist | grep -i recipe
```

## Step 2: Extract the app's files

If any of the backups are encrypted, install the decryption package
into a virtual environment first:

```sh
python3 -m venv .venv
source .venv/bin/activate
pip install iphone_backup_decrypt
```

Then run the script:

```sh
python extract_app_files.py
```

For unencrypted backups, `python3 extract_app_files.py` is enough.

With no arguments, it extracts from every backup that has the app. Add
one or more backup folder names to limit it to those. Run it in a real
Terminal window: the password prompt hides what you type, and that
only works in an interactive terminal. The password is never saved. If
it doesn't open a backup, the script asks again; leave it blank to
skip that backup.

The script only reads the backups and only writes out files that
belong to the app. Each backup gets its own folder under `recovered/`:

```
recovered/$BACKUP_DATE-$BACKUP_SUFFIX/
  files/$DOMAIN/$PATH     the app's files
  listing.tsv             every entry for the app in the backup's index
  Manifest.db             the backup's (decrypted) index
```

`Manifest.db` holds the name of every file on the phone, though not
their contents. It's handy if the data you want turns out to live
somewhere unexpected, but delete it when you're done.

At the end, the script lists the largest files it extracted. For
RecipeBox, the one that matters is:

```
recovered/$BACKUP/files/AppDomainGroup-group.com.7thfrequency.recipe/Recipe.sqlite
```

## Step 3: Turn the database into readable recipes

`Recipe.sqlite` is a Core Data store. Each recipe is a row in the
`ZRECIPE` table. The name is in `ZNAME`, the ingredients and steps are
JSON arrays of strings in `ZINGREDIENTSJSON` and `ZINSTRUCTIONSJSON`,
and dates are counted in seconds from January 1, 2001. To see which
recipes are in it:

```sh
sqlite3 -readonly "recovered/$BACKUP/files/AppDomainGroup-group.com.7thfrequency.recipe/Recipe.sqlite" "SELECT ZNAME FROM ZRECIPE"
```

To export them all:

```sh
python3 export_recipes.py "recovered/$BACKUP/files/AppDomainGroup-group.com.7thfrequency.recipe/Recipe.sqlite" recipes
```

This writes `recipes.md`, with every recipe's times, ingredients,
steps and notes, and `recipes.json`, with the same data for importing
into another app. The Markdown is plain enough for Google Docs to
import: create a new document, choose **File → Open → Upload**, and
pick `recipes.md`.

## Limits

- A backup only has what was on the phone when it was made, so recipes
  added later won't be in it. Use the newest backup that has the app.
- Backups from iOS 9 and earlier use an older format (`Manifest.mbdb`)
  that the script skips.
- The database layout belongs to RecipeBox and could change in later
  versions of the app.

## The scripts

### extract_app_files.py

Finds backups that contain an app and copies that app's files out of
them, decrypting if needed.

{% raw %}
```python
#!/usr/bin/env python3
"""Copy one app's files out of a local iPhone/iPad backup made by Finder or iTunes.

  python3 extract_app_files.py --list
      List the backups on this Mac that contain the app. No password needed.

  python3 extract_app_files.py [BACKUP_ID ...]
      Extract the app's files from those backups, or from every backup that has
      the app if none are named. Encrypted backups ask for the backup password at
      a hidden prompt (it is never saved) and need:  pip install iphone_backup_decrypt

Each backup gets its own folder, $out/$BACKUP_DATE-$BACKUP_ID_SUFFIX/:
  files/$DOMAIN/$PATH     the extracted files
  listing.tsv             every entry for the app in the backup's file index
  Manifest.db             the backup's file index (file names only; delete it when done)
"""
import argparse
import getpass
import plistlib
import shutil
import sqlite3
import sys
from pathlib import Path, PurePosixPath

DEFAULT_BACKUP_ROOT = Path.home() / "Library/Application Support/MobileSync/Backup"
DEFAULT_APP = "com.7thfrequency.Recipe"  # RecipeBox
# App-group containers (shared folders an app and its extensions use) aren't linked
# to their app in the backup metadata, so they have to be named. RecipeBox keeps its
# recipe database in the first of these.
KNOWN_GROUPS = {
    "com.7thfrequency.Recipe": ["group.com.7thfrequency.recipe", "group.com.codefortravel.recipe"],
}
KINDS = {1: "file", 2: "dir", 4: "link"}
NO_ACCESS = ("Can't read {root}.\nGive your terminal app Full Disk Access in "
             "System Settings > Privacy & Security, then open a new terminal window.")


def read_plist(path):
    try:
        with open(path, "rb") as fh:
            return plistlib.load(fh)
    except (OSError, plistlib.InvalidFileException):
        return {}


def find_backups(root, app, wanted_ids):
    """Return (path, info, manifest) for each backup that contains the app."""
    found = []
    for d in sorted(p for p in root.iterdir() if p.is_dir()):
        if wanted_ids and d.name not in wanted_ids:
            continue
        info, manifest = read_plist(d / "Info.plist"), read_plist(d / "Manifest.plist")
        if app in manifest.get("Applications", {}) or app in info.get("Installed Applications", []):
            found.append((d, info, manifest))
    return found


def related_groups(manifest, app):
    """App groups whose names share a word with the app's bundle id: hints for --group."""
    words = {w for w in app.lower().split(".")[1:] if len(w) > 3}
    return sorted(k for k, v in manifest.get("Applications", {}).items()
                  if v.get("ContainerContentClass") == "Shared/AppGroup"
                  and any(w in k.lower() for w in words))


def label(path, info):
    when = info.get("Last Backup Date")
    return f"{when:%Y-%m-%d}-{path.name[-8:]}" if when else path.name[-8:]


class PlainBackup:
    """Unencrypted backup: the index and the files are stored as-is."""

    def __init__(self, path):
        self.path = path

    def save_manifest(self, dest):
        shutil.copyfile(self.path / "Manifest.db", dest)

    def extract(self, file_id, domain, rel, dest):
        shutil.copyfile(self.path / file_id[:2] / file_id, dest)


class EncryptedBackupFiles:
    """Encrypted backup, read through the iphone_backup_decrypt library."""

    def __init__(self, backup):
        self.backup = backup

    def save_manifest(self, dest):
        self.backup.save_manifest_file(str(dest))

    def extract(self, file_id, domain, rel, dest):
        self.backup.extract_file(relative_path=rel, domain_like=domain, output_filename=str(dest))


def unlock(path, password):
    try:
        from iphone_backup_decrypt import EncryptedBackup
    except ImportError:
        sys.exit("This backup is encrypted, so the decryption library is needed:\n"
                 "  python3 -m venv .venv && .venv/bin/pip install iphone_backup_decrypt\n"
                 "then run this script with .venv/bin/python")
    backup = EncryptedBackup(backup_directory=str(path), passphrase=password)
    try:
        backup.test_decryption()
    except Exception as exc:  # a wrong password raises from inside the library
        print(f"  could not unlock: {exc}")
        return None
    return EncryptedBackupFiles(backup)


def app_entries(index, app, groups):
    conds = ["domain = ?", "domain LIKE ?", "relativePath LIKE ?"]
    params = [f"AppDomain-{app}", f"AppDomainPlugin-{app}.%", f"%{app}%"]
    for g in groups:
        conds.append("domain = ?")
        params.append(f"AppDomainGroup-{g}")
    sql = ("SELECT fileID, domain, relativePath, flags FROM Files WHERE "
           + " OR ".join(conds) + " ORDER BY domain, relativePath")
    db = sqlite3.connect(f"file:{index}?mode=ro", uri=True)
    try:
        return db.execute(sql, params).fetchall()
    finally:
        db.close()


def safe_dest(base, domain, rel):
    p = PurePosixPath(rel)
    if not rel or p.is_absolute() or ".." in p.parts or "/" in domain:
        return None
    return base.joinpath(domain, *p.parts)


def process(path, info, manifest, args, groups, password):
    """Extract one backup. Returns the password that worked, for the next backup."""
    out = args.out / label(path, info)
    print(f"\n=== {path.name}")
    if not (path / "Manifest.db").exists():
        print("  skipped: this backup uses the pre-iOS 10 format, which this script doesn't read")
        return password

    if manifest.get("IsEncrypted"):
        backup = unlock(path, password) if password else None
        while backup is None:
            entered = getpass.getpass("  Backup password (hidden; blank to skip this backup): ")
            if not entered:
                print("  skipped")
                return password
            backup = unlock(path, entered)
            if backup:
                password = entered
    else:
        backup = PlainBackup(path)

    out.mkdir(parents=True, exist_ok=True)
    index = out / "Manifest.db"
    backup.save_manifest(index)
    rows = app_entries(index, args.app, groups)
    files = [r for r in rows if r[3] == 1]

    with open(out / "listing.tsv", "w") as fh:
        fh.write("domain\trelativePath\tkind\n")
        for _, domain, rel, flags in rows:
            fh.write(f"{domain}\t{rel}\t{KINDS.get(flags, flags)}\n")

    ok, failed = 0, []
    for file_id, domain, rel, _ in files:
        dest = safe_dest(out / "files", domain, rel)
        if dest is None:
            failed.append((domain, rel, "unsafe path, skipped"))
            continue
        dest.parent.mkdir(parents=True, exist_ok=True)
        try:
            backup.extract(file_id, domain, rel, dest)
            ok += 1
        except Exception as exc:
            failed.append((domain, rel, str(exc)))

    print(f"  {len(rows)} entries in the index, {len(files)} of them files")
    print(f"  extracted {ok} files to {out / 'files'}")
    for domain, rel, err in failed:
        print(f"    FAILED {domain}/{rel}: {err}")
    extracted = [p for p in (out / "files").rglob("*") if p.is_file()]
    if extracted:
        print("  largest files:")
        for p in sorted(extracted, key=lambda p: -p.stat().st_size)[:10]:
            print(f"    {p.stat().st_size:>12,}  {p.relative_to(out / 'files')}")
    return password


def main():
    ap = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    ap.add_argument("backup_ids", nargs="*", metavar="BACKUP_ID", help="backup folder names (default: all with the app)")
    ap.add_argument("--list", action="store_true", help="only list backups that contain the app")
    ap.add_argument("--app", default=DEFAULT_APP, help=f"app bundle id (default: {DEFAULT_APP}, RecipeBox)")
    ap.add_argument("--group", action="append", help="app-group container to include as well (repeatable)")
    ap.add_argument("--backup-root", type=Path, default=DEFAULT_BACKUP_ROOT, help="folder that holds the backups")
    ap.add_argument("--out", type=Path, default=Path("recovered"), help="output folder (default: ./recovered)")
    args = ap.parse_args()
    groups = args.group if args.group is not None else KNOWN_GROUPS.get(args.app, [])

    try:
        found = find_backups(args.backup_root, args.app, set(args.backup_ids))
    except (PermissionError, FileNotFoundError):
        sys.exit(NO_ACCESS.format(root=args.backup_root))
    if not found:
        sys.exit(f"No backup in {args.backup_root} contains {args.app}.")

    if args.list:
        for path, info, manifest in found:
            print(path.name)
            print(f"  device:    {info.get('Device Name', '?')} "
                  f"({info.get('Product Type', '?')}, iOS {info.get('Product Version', '?')})")
            print(f"  backed up: {info.get('Last Backup Date', '?')}")
            print(f"  encrypted: {'yes' if manifest.get('IsEncrypted') else 'no'}")
            hints = related_groups(manifest, args.app)
            if hints:
                print(f"  app groups that may belong to it: {', '.join(hints)}")
        print(f"\nApp groups included when extracting: {', '.join(groups) or 'none (use --group)'}")
        return

    password = None
    for path, info, manifest in found:
        password = process(path, info, manifest, args, groups, password)
    print(f"\nDone. Results are in {args.out.resolve()}")


if __name__ == "__main__":
    main()
```
{% endraw %}

### export_recipes.py

Reads RecipeBox's `Recipe.sqlite` and writes the recipes as Markdown and JSON.

{% raw %}
```python
#!/usr/bin/env python3
"""Export the recipes in a recovered RecipeBox Recipe.sqlite to Markdown and JSON.

Usage: python3 export_recipes.py <Recipe.sqlite> <output-basename>
Writes <output-basename>.md and <output-basename>.json. The database is opened read-only.
The Markdown is plain (no HTML), so Google Docs can import it with File > Open > Upload.
"""
import json
import sqlite3
import sys
from datetime import datetime, timezone
from pathlib import Path

APPLE_EPOCH = 978307200  # Core Data timestamps count seconds from 2001-01-01


def load(db_path):
    con = sqlite3.connect(f"file:{db_path}?immutable=1", uri=True)
    con.row_factory = sqlite3.Row
    recipes = []
    for r in con.execute("SELECT * FROM ZRECIPE ORDER BY ZCREATED, Z_PK"):
        def j(col):
            try:
                return json.loads(r[col]) if r[col] else []
            except json.JSONDecodeError:
                return [r[col]]

        def day(col):
            if r[col] is None:
                return None
            return datetime.fromtimestamp(r[col] + APPLE_EPOCH, timezone.utc).date().isoformat()

        recipes.append({
            "name": r["ZNAME"],
            "url": r["ZURL"] or None,
            "created": day("ZCREATED"),
            "updated": day("ZUPDATEDAT"),
            "prep_time": r["ZPREPTIME"] or None,
            "cook_time": r["ZCOOKTIME"] or None,
            "total_time": r["ZTOTALTIME"] or None,
            "yield": r["ZYIELD"] or None,
            "categories": j("ZCATEGORIESJSON"),
            "ingredients": j("ZINGREDIENTSJSON"),
            "instructions": j("ZINSTRUCTIONSJSON"),
            "notes": r["ZNOTES"] or None,
            "favorite": bool(r["ZFAVORITE"]),
        })
    con.close()
    return recipes


def to_markdown(recipes):
    out = [f"# Recipes\n\n{len(recipes)} recipes recovered from RecipeBox.\n"]
    for r in recipes:
        out.append(f"\n---\n\n## {r['name']}\n")
        times = [f"{label} {r[k]}" for label, k in
                 (("Prep", "prep_time"), ("Cook", "cook_time"), ("Total", "total_time"), ("Yield", "yield")) if r[k]]
        if times:
            out.append("*" + " · ".join(times) + "*\n")
        if r["ingredients"]:
            out.append("### Ingredients\n\n" + "\n".join(f"- {i}" for i in r["ingredients"]) + "\n")
        if r["instructions"]:
            out.append("### Instructions\n\n"
                       + "\n".join(f"{n}. {s.strip()}" for n, s in enumerate(r["instructions"], 1)) + "\n")
        if r["notes"]:
            out.append("### Notes\n\n" + r["notes"].strip() + "\n")
        info = [f"Added {r['created']}"] if r["created"] else []
        if r["url"]:
            info.append(f"[Saved in RecipeBox]({r['url']})")
        if info:
            out.append("_" + " · ".join(info) + "_\n")
    return "\n".join(out)


def main():
    if len(sys.argv) != 3:
        sys.exit(__doc__)
    db, base = Path(sys.argv[1]), Path(sys.argv[2])
    recipes = load(db)
    base.with_suffix(".json").write_text(json.dumps(recipes, indent=2, ensure_ascii=False) + "\n")
    base.with_suffix(".md").write_text(to_markdown(recipes))
    print(f"{len(recipes)} recipes -> {base.with_suffix('.md')}, {base.with_suffix('.json')}")


if __name__ == "__main__":
    main()
```
{% endraw %}
