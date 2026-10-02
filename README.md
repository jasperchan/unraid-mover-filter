# Mover Filter

An Unraid plugin that excludes files from the mover by filename pattern.

Defaults to `*.part`, so downloads still in progress are never moved off the
cache pool mid-write.

## Why

Unraid's mover moves files from the cache pool to the array on a schedule. It does
not check whether a file is still being written.

So a download in progress — `something.mkv.part` — gets moved to the array
mid-download, and the download client carries on writing to it *there*. Three
things follow:

1. **Every subsequent write becomes a parity read-modify-write.** The target disk
   reads and writes the same blocks while both parity disks do the same, instead of
   the download running at cache-pool speed.
2. **One download's files scatter across disks.** Each part lands wherever the
   allocator put it, so a single folder ends up spread over several drives.
3. **Downloads can fail with `ENOSPC`.** Unraid's *minimum free space* is checked
   before a file is placed, not enforced during the write — so a disk chosen with
   10 GB free can be filled to zero by one large file, and any download already
   living on that disk then fails.

## What it does

Wraps `/usr/libexec/unraid/move` — the binary Unraid's mover pipes its file list
into — and drops any path matching a pattern in `filters.conf`:

```bash
awk '...build regexes from filters.conf...' | move.stock "$@"
```

Unraid's own binary is kept alongside as `move.stock`.

## Why wrap the binary rather than the mover script

Every way of invoking the mover — the scheduled cron job, the **Move Now** button
in the web UI, and any custom script calling `/usr/local/sbin/mover` — runs the
same mover script, which pipes through this one binary. Wrapping here covers all
of them, and it is unaffected by changes to mover's own logic between Unraid
releases.

## Install

Plugins → Install Plugin:

```
https://raw.githubusercontent.com/jasperchan/unraid-mover-filter/main/mover.filter.plg
```

`/usr/libexec` is squashfs+overlay, so the wrapper is reinstalled on every boot by
the plugin. Removing the plugin restores Unraid's binary.

## Configuration

`/boot/config/plugins/mover.filter/filters.conf` — one glob per line, matched
against the full path of each file the mover considers. Patterns are implicitly
anchored, so `*.part` means "any path ending in `.part`".

```
*.part

#*.!qB                  # qBittorrent incomplete files
#*.crdownload           # Chrome partial downloads
#*/incomplete/*         # anything under an "incomplete" directory
```

`*` matches any run of characters (including `/`), `?` matches one character.
Lines beginning with `#` and blank lines are ignored. Regex metacharacters in your
patterns are escaped, so `file (1).mkv` is matched literally.

Changes take effect on the next mover run — no restart needed. The config is left
in place if you uninstall, so a reinstall keeps your patterns.

## Verifying

When the mover skips something, it says so in syslog:

```
mover.filter: excluded 14 file(s) matching filters.conf
```

To find files matching your patterns that are already on the array:

```bash
for d in /mnt/disk*; do
  n=$(find "$d" -name '*.part' 2>/dev/null | wc -l)
  [ "$n" -gt 0 ] && echo "$d: $n"
done
```

## Scope

Deliberately small: a filename filter and nothing else. No scheduling changes, no
age thresholds, no per-share rules, no settings page. For those, use
[Mover Tuning](https://forums.unraid.net/topic/70783-plugin-mover-tuning/).

Two limits worth knowing:

- It matches on filenames. A download client that writes incomplete files under
  their final name cannot be detected this way.
- It prevents new occurrences only. Files already stranded on the array stay there.

## License

MIT
