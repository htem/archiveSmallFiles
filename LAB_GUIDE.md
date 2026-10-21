# Archiving Large-Scale Datasets on O2: A Lab Guide

**Purpose:** Anyone who has finished generating a dataset with large file counts and needs to get it off active storage.

**Tool:** `archiveSmallFiles` (https://github.com/htem/archiveSmallFiles)

**Questions:** Ask Kevin Delgado, or HMS Research Computing at rchelp@hms.harvard.edu

The worked example throughout is a real dataset:
`/n/data3_vast/hms/neurobio/htem/temcagt/datasets/211122dAedes10_r1083`

---

## 1. What this does and why

Storage systems don't care much about total bytes — they care about the **number of files**. A single EM acquisition produces hundreds of millions of tiny files, which blows through our file-count quota, makes backups and `ls` crawl, and makes it effectively impossible to move data to cheaper long-term storage.

This tool walks the dataset and, **inside each folder**, bundles all the small files (under 1 GB each) into a single `.tar`. The folder structure stays exactly the same — you still see the same tree, just with one `.tar` per folder instead of thousands of loose files. For r1083 that's roughly **220 million files → 4 million**, a ~55× reduction.

Every archive is verified with MD5 checksums — once when written, and again in a separate checking pass — so nothing is silently lost.

**End goal:** archive the dataset → move it to HMS **Standby** storage (cheap, long-term) → free up active storage for the next acquisition.

---

## 2. When to run it

Run this **only when the dataset is completely aligned and ready for post-processing.** Confirm all of:

- [ ] Acquisition is done — the microscope is no longer writing to this directory.
- [ ] No pipeline (alignment, stitching, segmentation) is still reading or writing to it.
- [ ] Nobody else in the lab is actively using it.
- [ ] You have a destination on Standby, and RC has confirmed there's room.

The tool writes the archive to a new `tarred/` directory and never modifies the original, so for a while you need room for **both**. r1083 is 300+ TB — sort the destination out with RC *before* you start.

---

## 3. Key concepts

**Snapshots.** We never archive a live directory — we archive a **snapshot**, a frozen read-only copy the storage system makes nightly (16:00 UTC on data3_vast). This guarantees nothing changes underneath you mid-run. Most scripts will refuse to run on a path without `.snapshot` in it.

```
Live:      /n/data3_vast/hms/neurobio/htem/temcagt/datasets/211122dAedes10_r1083
Snapshot:  /n/data3_vast/hms/neurobio/htem/temcagt/datasets/.snapshot/data3_2026-09-22_16_00_02_UTC/211122dAedes10_r1083
```

Put `.snapshot` at the **parent** level as shown above, not inside the dataset. The scripts name their output after the last component of the path, and you want that to be the dataset name, not a timestamp.

**Starfish.** RC's storage browser: **https://starfish.med.harvard.edu**. Use it to look up your dataset's folder count — counting yourself takes hours. `convertStarfishPath.sh <path>` prints the matching Starfish URL.

**Passes.** The scripts work in rounds: `pass1` holds the complete folder list, and anything that fails is written to `pass2/folders.txt` so you can fix it and retry only those folders. Each pass is also a directory holding that round's logs.

---

## 4. Setup

```bash
git clone https://github.com/htem/archiveSmallFiles.git
export PATH=$PWD/archiveSmallFiles/bin:$PATH
```

Add the `export PATH` line to your `~/.bashrc`. **Note:** this will move to the htem/gtalign GitHub repo in the future.

---

## 5. Practice first (30 minutes, recommended)

This generates fake data with deliberately broken permissions, so you see what a failure looks like somewhere harmless.

```bash
srun -p short -t 2:0:0 --mem 2G --pty /bin/bash
mkdir -p ~/tarTesting && cd ~/tarTesting

createTestData.sh                       # ./TestingData: 65 folders, 365 files
scanFolders.sh TestingData/ 1 65 2>&1 | tee scan.log
archiveFolders.sh tar local pass1
findFoldersNotDoneArchiving.sh pass1    # reports 2 failures — on purpose
cat pass1/tarError*                     # see why

# fix permissions, retry only the failures
find TestingData -exec sh -c 'for f; do [ -d "$f" ] && chmod 775 "$f" || chmod 664 "$f"; done' sh {} +
archiveFolders.sh tar local pass2
findFoldersNotDoneArchiving.sh pass2

# verify
checkArchives.sh tar local pass1
findFoldersNotDoneChecking.sh pass1
randomUnArchiveToCheck.sh tar pass1 10
```

---

## 6. The real run

### Step 1 — Check permissions

Everything must be group-readable or `tar` will skip files. Check first:

```bash
find /n/data3_vast/hms/neurobio/htem/temcagt/datasets/211122dAedes10_r1083/sections \
  -maxdepth 3 ! -perm -g=r -print | head
```

Nothing printed means you're fine (this is the case for r1083). Otherwise fix them:

```bash
find <dataset> -exec sh -c 'for f; do [ -d "$f" ] && chmod 775 "$f" || chmod 664 "$f"; done' sh {} +
```

If you get "Operation not permitted," the files belong to someone who has left the lab and you can't change them — email rchelp@hms.harvard.edu with the path. **Permissions are the most common cause of failures later, so settle this first.** If you did change anything, wait for the next nightly snapshot before continuing.

### Step 2 — Pick a snapshot and get the folder count

```bash
ls /n/data3_vast/hms/neurobio/htem/temcagt/datasets/.snapshot/ | tail -1
convertStarfishPath.sh /n/data3_vast/hms/neurobio/htem/temcagt/datasets/211122dAedes10_r1083
```

Open the Starfish URL and note the **number of directories** (~4 million for r1083). The scripts use this to confirm they found the whole tree.

### Step 3 — Scan

Work from a location with room for the output — **not** your home directory.

```bash
srun -p short -t 12:0:0 --mem 8G --pty /bin/bash
mkdir -p /n/data3_vast/hms/neurobio/htem/temcagt/archiveWork/211122dAedes10_r1083
cd /n/data3_vast/hms/neurobio/htem/temcagt/archiveWork/211122dAedes10_r1083

scanFolders.sh \
  /n/data3_vast/hms/neurobio/htem/temcagt/datasets/.snapshot/data3_2026-09-22_16_00_02_UTC/211122dAedes10_r1083 \
  20 4000000 2>&1 | tee scan.log
```

Arguments are `<snapshot path> <parallel processes> <expected folder count>`. Twenty processes suits a dataset this size; use 1–4 for a small one.

Confirm the output says the folder count **matches**. If it doesn't, stop — it usually means part of the tree wasn't readable. Write down the folder and file counts it reports; you'll compare against them in Step 6. This produces `pass1/folders.txt`.

### Step 4 — Archive

```bash
archiveFolders.sh tar sbatch pass1
```

`sbatch` runs the work in parallel across the cluster — always use it for real data (`local` is for the practice run). At 4 million folders this runs for **days**. Monitor with `squeue -u $USER` and `summarizeRun.sh pass1`. Output lands in `tarred/`, mirroring the original tree.

### Step 5 — Retry failures

```bash
findFoldersNotDoneArchiving.sh pass1
cat pass1/tarError*
```

Failures go to `pass2/folders.txt`. A handful out of millions is normal, usually permissions. Fix on the **live** data, wait for the next snapshot, then:

```bash
updateSnapshotVersion.sh          # interactive: repoints paths at the newest snapshot
archiveFolders.sh tar sbatch pass2
findFoldersNotDoneArchiving.sh pass2
```

Repeat until nothing is left. **Don't move on until the not-done count is zero.**

### Step 6 — Verify

Always check against `pass1` — it has the complete folder list.

```bash
checkArchives.sh tar sbatch pass1
findFoldersNotDoneChecking.sh pass1
```

This re-reads every archive and re-verifies checksums; same retry drill until clean. In the summary it prints, the original file count and the untarred file count should both match Step 3's scan, and the post-tar count is your new, much smaller file count.

### Step 7 — Spot-check

```bash
randomUnArchiveToCheck.sh tar pass1 100
```

Unpacks 100 random folders and `diff`s them against the originals. Should report no differences. Do this before deleting anything.

### Step 8 — Move to Standby

Standby (`/n/standby`) is RC's cheap long-term tier; you need an allocation there first.

1. Email **rchelp@hms.harvard.edu** to request one (or ask Kevin whether the lab already has one), with the dataset path, size, and post-archive file count. At 300 TB this is a conversation, not a form — start it early.
2. Transfer on the dedicated transfer partition:

```bash
sbatch -p transfer -t 5-00:00:00 --mem 4G -o rsync.log \
  --wrap="rsync -av .../archiveWork/211122dAedes10_r1083/tarred/ /n/standby/hms/neurobio/lee/211122dAedes10_r1083/"
```

*Ask Kevin first:* if the Standby allocation already exists, you may be able to run Step 3 onward with your working directory **on Standby** so the tars are written there directly, skipping this transfer and a full second copy of the data.

3. Confirm in Starfish that the counts on Standby match what `checkArchives.sh` reported.
4. **Only then** delete the original. Keep the `pass1/` directory — it's your record of what was archived and verified.

---

## 7. Getting data back out

`.tar` is a standard format — you don't need this repo to read it. To restore one folder, untar in place:

```bash
cd /n/standby/.../sections/230404124142_211122dAedes10_r1083_7441/0/0001
tar -xf 0001.tar
```

To restore a whole subtree, copy it off Standby to a working location, then:

```bash
find <subtree> -name "*.tar" -print0 | xargs -0 -P 4 -I {} sh -c \
  'tar --overwrite -xf "$1" -C "$(dirname "$1")" && rm "$1" "${1%.tar}.md5sum"' _ {}
```

Drop the `&& rm ...` if you want to keep the tars.

**Restore only what you need.** The folder structure is preserved, so you can browse the archived tree on Standby and unpack a single section (~45 GB) instead of the whole dataset. Unpacking everything back onto active storage recreates the exact problem we archived to solve.

---

## 8. Troubleshooting

| Symptom | What to do |
|---|---|
| "Source directory is not snapshot folder" | Use the `.snapshot/data3_..._UTC/...` path, not live data |
| Folder count doesn't match Starfish | Part of the tree wasn't readable — fix permissions, wait for the next snapshot, rescan |
| `tarError*` shows "Permission denied" | `chmod` on the live data, wait a night, `updateSnapshotVersion.sh`, retry |
| `chmod` says "Operation not permitted" | Files owned by a departed user — email rchelp@hms.harvard.edu |
| Snapshot no longer exists | They expire after a few weeks — `updateSnapshotVersion.sh` |
| `sbatch` rejected (account or QOS error) | Ask Kevin, or email rchelp@hms.harvard.edu |
| Jobs seem stuck | Normal; this takes days. `squeue -u $USER`, `summarizeRun.sh pass1` |
| Your own `find` never finishes | Expected at this file count — use Starfish for counts |

---

## 9. Quick reference

```bash
LIVE=/n/data3_vast/hms/neurobio/htem/temcagt/datasets/211122dAedes10_r1083
SNAP=/n/data3_vast/hms/neurobio/htem/temcagt/datasets/.snapshot/data3_2026-09-22_16_00_02_UTC/211122dAedes10_r1083

convertStarfishPath.sh $LIVE               # O2 path -> Starfish URL (get folder count there)
scanFolders.sh $SNAP 20 4000000            # build pass1/folders.txt
archiveFolders.sh tar sbatch pass1         # create the .tar archives
findFoldersNotDoneArchiving.sh pass1       # failures -> pass2/folders.txt
checkArchives.sh tar sbatch pass1          # verify checksums
findFoldersNotDoneChecking.sh pass1        # verification failures -> next pass
randomUnArchiveToCheck.sh tar pass1 100    # spot-check by unpacking
summarizeRun.sh pass1                      # job progress
updateSnapshotVersion.sh                   # repoint paths at a newer snapshot
```

**Rules to follow**

1. Only archive finished data.
2. Always archive from a `.snapshot` path.
3. Sort out permissions and the Standby destination *before* you start.
4. Never delete the original until `checkArchives.sh` is clean, `randomUnArchiveToCheck.sh` shows no diffs, and the copy on Standby is confirmed in Starfish.
