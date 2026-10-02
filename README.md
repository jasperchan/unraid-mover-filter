# mover-partfilter

A one-line Unraid plugin: stop the mover relocating in-progress downloads to the array.

## The problem

Unraid's mover moves files from the cache pool to the array. It does not care
whether a file is still being written.

So a download in progress — `something.mkv.part` — gets moved to the array
mid-download, and the download client carries on writing to it *there*. Three
things follow:

1. **Every subsequent write becomes a parity read-modify-write.** The target disk
   reads and writes the same blocks while both parity disks do the same. Measured
   on a 28-disk array: ~72 MB/s, versus SSD speed had the file stayed on cache.
2. **One download's files scatter across disks.** Each `.part` lands wherever shfs
   put it, so a single folder ends up spread over several drives.
3. **Downloads die with `ENOSPC`.** Unraid's "minimum free space" is checked
   *before* a file is placed, not enforced during the write. A disk picked with
   10 GB free can be filled to zero by one large file — and any download already
   living on it then fails.

Observed on the array this was written for: one disk at **36 KB free**, 22 stalled
`.part` files on it, and `No space left on device` in syslog.

## What this does

Wraps `/usr/libexec/unraid/move` — the binary Unraid's mover pipes its file list
into — and drops any path ending in `.part`:

```bash
awk '/\.part$/ { skipped++; next } { print }' | move.stock "$@"
```

That's the whole thing. The real binary is kept at `move.stock`.

## Why wrap the binary rather than the mover script

Every way of invoking mover — the scheduled cron job, the **Move Now** button in
the web UI, and any custom script calling `/usr/local/sbin/mover` — runs the same
mover script, which pipes through this one binary. Wrapping here covers all of
them, and it does not care how Unraid changes mover's own logic between releases.

## Install

Plugins → Install Plugin:

```
https://raw.githubusercontent.com/jasperchan/unraid-mover-partfilter/main/mover-partfilter.plg
```

`/usr/libexec` is squashfs+overlay, so the wrapper is reinstalled on every boot by
the plugin. Removing the plugin restores Unraid's binary.

## Scope

Deliberately narrow. It filters exactly one suffix and nothing else — no
scheduling changes, no age thresholds, no share rules, no settings page. If you
want those, use
[Mover Tuning](https://forums.unraid.net/topic/70783-plugin-mover-tuning/).

It only matches the `.part` suffix, which is what qBittorrent, Transmission and
JDownloader use for incomplete files. A client configured to write incomplete
files under their final name is not covered — nothing reading a filename could be.

It also only prevents *new* instances of the problem. Files already stranded on
the array stay there.

## Verifying

When mover skips something it says so in syslog:

```
mover-partfilter: skipped 14 in-progress .part file(s)
```

Find `.part` files that already live on the array:

```bash
for i in /mnt/disk*; do
  n=$(find "$i" -name '*.part' 2>/dev/null | wc -l)
  [ "$n" -gt 0 ] && echo "$i: $n"
done
```

## License

MIT
