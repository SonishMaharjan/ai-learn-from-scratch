# Step 7: Keyword search

You store notes in a SQLite database and search them by keyword with FTS5. No AI here: it is the fast, exact search that AI search is compared with.

## The idea

**SQLite** is a whole database in one file. **FTS5** (full-text search) is built into SQLite. It builds an **index**: every word and which notes contain it, like the index at the back of a book. Results are ranked by **BM25**. The weakness: it matches **words**, not **meaning**.

```text
notes table ──► FTS5 index (word → notes)
                  chord   → note 1
                  chords  → note 4
                  braked  → note 3
search "chord"    ──► note 1  (ranked by BM25)
search "stopping" ──► nothing (even though note 3 is about braking!)
```

| Word | Simple meaning |
| --- | --- |
| SQLite | A database stored in a single file. |
| FTS5 | SQLite's full-text search feature. |
| Index | A lookup from words to the rows that contain them. |
| BM25 | A ranking score. In SQLite, lower (more negative) = better match. |
| Stemming | Cutting words to their root: "braking", "braked" → "brake". |

## The code

`npm install better-sqlite3`. Make `step07.ts`.

**Piece 1: create the database and table.**

```ts
import Database from "better-sqlite3";

const db = new Database("notes.db");
db.exec(`
  DROP TABLE IF EXISTS notes;
  CREATE VIRTUAL TABLE notes USING fts5(day UNINDEXED, text);
`);
```

**Piece 2: add notes.**

```ts
const notes = [
  ["2026-11-01", "Guitar: G to C chord change, 20 min, still slow"],
  ["2026-11-02", "Football: 50 left-foot passes against the wall"],
  ["2026-11-03", "MTB: rode the rocky descent twice, braked too late"],
  ["2026-11-04", "Guitar: barre chords with a metronome"],
  ["2026-11-05", "Football: first touch drills, both feet"],
];

const insert = db.prepare("INSERT INTO notes (day, text) VALUES (?, ?)");
for (const [day, text] of notes) insert.run(day, text);
```

**Piece 3: search.**

```ts
const search = db.prepare(`
  SELECT day, text, round(bm25(notes), 2) AS score
  FROM notes
  WHERE notes MATCH ?
  ORDER BY rank
  LIMIT 5
`);

for (const q of ["chord", "chords", "chord*", "foot*", "guitar AND slow", "braking"]) {
  console.log(`\n"${q}"`);
  console.table(search.all(q));
}
```

- `chord*` = words starting with "chord"; `AND` / `OR` / `NOT` combine words; `"two words"` = exact phrase.

## Run it and read the output

```text
"chord"            → 2026-11-01  Guitar: G to C chord change, 20 min, still slow
"chords"           → 2026-11-04  Guitar: barre chords with a metronome
"chord*"           → both guitar notes
"foot*"            → both football notes (words starting with "foot")
"guitar AND slow"  → 2026-11-01 only
"braking"          → (no rows)
```

## Try this

- Stemming: `fts5(day UNINDEXED, text, tokenize='porter')`. Now "braking" finds "braked".
- Then search "stopping". Still nothing: different word, similar meaning. Step 8 fixes this.
- Insert 1,000 copies and time a search with `console.time`.

## Quiz

1. What does the FTS5 index store?
2. Why does "braking" find nothing at first?
3. What can keyword search never do on its own?

**Answers:** 1. Each word and the rows that contain it. 2. The note says "braked" and stemming is off. 3. Match by meaning.

## Your practice-video app

Logs, tags and notes in SQLite with FTS5 keyword search: best for exact words, names and dates.

## Learn more

- [SQLite FTS5](https://sqlite.org/fts5.html)
- [Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval) (Anthropic)
