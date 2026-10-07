# glibc Heap Playground

An interactive, single-file visualizer for the glibc (ptmalloc) heap fast paths on x86-64. Drive `malloc` and `free` by hand and watch the tcache, fastbins, and unsorted bin mutate in real time, then step through the classic heap-exploitation primitives.

It's a teaching tool: the invisible linked lists are made visible so the *why* behind each primitive is obvious.

## Run it

No build, no dependencies. Open `index.html` in any modern browser, or serve the folder:

```bash
python3 -m http.server 8000
# → http://localhost:8000
```

It also drops straight onto GitHub Pages (serve from the repo root).

## What it models

A faithful model of the free-chunk fast paths in glibc ~2.31–2.39:

- **`request2size`** rounding: `chunk = max(0x20, (req + 8 + 15) & ~15)`, with usable bytes reusing the next chunk's `prev_size`.
- **Allocation order**: exact-size tcache → exact-size fastbin → first-fit unsorted (split, remainder back to unsorted) → carve from top.
- **Free routing**: tcache (size ≤ `0x410`, bin not full) → fastbin (size ≤ `0x80`) → unsorted.
- **tcache** singly linked / LIFO with per-entry `key`; **fastbins** singly linked / LIFO; **unsorted** doubly linked.

### Hardening you can defeat

- **tcache double-free detection** (2.29+): the per-entry `key` triggers a bin scan; a hit aborts (`free(): double free detected in tcache 2`). Clear the key with `edit` to slip past it.
- **fastbin double-free**: the single fasttop check (`double free or corruption (fasttop)`); free a second chunk in between and the dup goes through.
- **safe-linking** (2.32+): the stored `fd` is mangled, `(pos >> 12) ^ ptr`. The playground shows both the value you want returned and the mangled bytes you'd actually have to write at `pos`.

### Built-in scenarios (step-through with a visible recipe)

1. tcache poisoning → arbitrary `malloc` at a target
2. fastbin dup (bypassing the fasttop check)
3. use-after-free aliasing
4. tcache double-free (caught by the key check)
5. tcache double-free bypass (clear the key, then poison)

## Deliberately out of scope

Chunk consolidation (forward/backward), small/large-bin sorting and binning, last-remainder nuance, per-thread arenas, and top-chunk integrity checks. They don't change the tcache/fastbin lessons and add noise. Treat returned sizes and the key as representative, not byte-exact to any one release.

## License

MIT
