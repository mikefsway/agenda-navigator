# docs/data/ — the conference goes here

This directory is empty on purpose. Everything in it is a **pipeline output**,
not a source file, and it is specific to one conference:

| file | written by | what it is |
|---|---|---|
| `sessions.json` | `pipeline/normalize.py` | the programme, in the shape `PORTING.md` §2 documents |
| `facets.json` | `pipeline/embed.py` | one row per embedded chunk → session index + evidence label |
| `embeddings.bin` | `pipeline/embed.py` | the matrix: `n_facets × dim`, float16 |
| `meta.json` | `pipeline/embed.py` | counts, model name, and `order_sig` |

Fill it by running the three steps:

```
python3 pipeline/fetch.py        # -> data/raw/
python3 pipeline/normalize.py    # -> docs/data/sessions.json
python3 pipeline/embed.py        # -> the other three
node test/data.test.mjs          # check them against each other
```

Out of the box those talk to Ex Ordo, because that is the conference this was
built for. Pointing them somewhere else is what `PORTING.md` is for — read it
before writing an adapter, particularly §0, which is about deciding whether the
conference is worth doing this to at all.

**The four files are only meaningful as a set.** `facets.json` addresses
sessions by row index, so `embeddings.bin` describes one exact ordering of
`sessions.json` and no other. `embed.py` stamps `order_sig` and the browser
re-derives it at load and refuses a mismatched pair, which is why re-running
`embed.py` after *any* `normalize.py` change — including one that touches no
text — is not optional.

Until they exist the page loads and says so rather than throwing.
