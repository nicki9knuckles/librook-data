# librook-data

Reference data files for librook, a language-learning reader. Each file lets
the app look up a word offline: its base form (lemma), part of speech, short
definitions in English and in the language itself, and which grammatical
form a word is (for example "first-person imperfect indicative singular").

The files are built from the English Wiktionary and each language's own
Wiktionary via kaikki.org, and are licensed CC BY-SA 4.0. Details are below.

The files are attached to this repo's
[releases](https://github.com/nicki9knuckles/librook-data/releases), one
release per language and data date. The app downloads them from there.

## Files

| Release | File | Size | Language | Source data date |
|---|---|---|---|---|
| `es-2026-10-28` | `reference-es.db` | 46.8 MB | Spanish | 2026-10-25 (English), 2026-10-28 (Spanish) |
| `pt-2026-10-28` | `reference-pt.db` | 20.6 MB | Portuguese | 2026-10-25 (English), 2026-10-28 (Portuguese) |
| `fr-2026-10-27` | `reference-fr.db` | 24.4 MB | French | 2026-10-25 (English), 2026-10-27 (French) |
| `es-2026-10-21` | `reference-es.db` | 45.8 MB | Spanish | 2026-10-21 |
| `pt-2026-10-25` | `reference-pt.db` | 19.9 MB | Portuguese | 2026-10-25 |
| `fr-2026-10-25` | `reference-fr.db` | 24.4 MB | French | 2026-10-25 |

Each file is a SQLite database with these tables:

- `meta`: language, schema version, sources and licence
- `entries`: one row per dictionary entry (lemma, part of speech, English
  definitions, definitions in the language itself)
- `forms`: every spelling that leads to an entry, with its grammatical tags
- `tag_sets`: the distinct grammatical tag strings

The schema version is recorded in `meta`. The app refuses a file whose
schema version it doesn't recognise.

## Sources and attribution

The data comes from [Wiktionary](https://www.wiktionary.org/), written by
Wiktionary contributors. It was extracted with
[wiktextract](https://github.com/tatuylonen/wiktextract) and downloaded from
[kaikki.org](https://kaikki.org/):

- **English Wiktionary**: lemmas, parts of speech, English definitions,
  inflected forms and grammatical tags.
  [kaikki.org Spanish data](https://kaikki.org/dictionary/Spanish/index.html),
  [kaikki.org Portuguese data](https://kaikki.org/dictionary/Portuguese/index.html),
  [kaikki.org French data](https://kaikki.org/dictionary/French/index.html)
- **Spanish Wiktionary**: Spanish-language definitions.
  [kaikki.org raw data](https://kaikki.org/dictionary/rawdata.html)
- **Portuguese Wiktionary**: Portuguese-language definitions.
  [kaikki.org raw data](https://kaikki.org/dictionary/rawdata.html)
- **French Wiktionary**: French-language definitions.
  [kaikki.org raw data](https://kaikki.org/dictionary/rawdata.html)

wiktextract is described in: Tatu Ylonen, "Wiktextract: Wiktionary as
Machine-Readable Structured Data", *Proceedings of the 13th Conference on
Language Resources and Evaluation (LREC)*, pp. 1317–1325, Marseille, 2022.

## Changes made to the source data

These files are adapted from the sources above, not copies of them. The
build:

- keeps only entries in the file's language whose word is made of letters
  only, so multi-word expressions, and words containing hyphens or digits,
  are left out. Words with an apostrophe are kept (French file onwards):
  an elided word (*l'*, *jusqu'*) is looked up without its apostrophe, and
  leads to the word it stands for; each piece of a word joined by an
  apostrophe (*aujourd'hui*) leads to the whole word, unless the first
  piece is itself an elided word (*c'est*). Curly apostrophes are
  straightened;
- keeps at most 3 definitions per entry from each Wiktionary edition;
- drops inflection-table metadata, and any grammatical tag set that a more
  specific tag set for the same form fully contains;
- drops tags that describe the Wiktionary page rather than the word
  (`form-of`, `alt-of`), and `abbreviation` next to a more specific kind of
  shortening such as `apocopic` (releases from 2026-10-27 onwards);
- matches the language's own edition's entries to English-edition entries by
  word and part of speech. When the two editions have different numbers of
  entries for a word, an entry only gets definitions from entries of the
  same gender, or of no stated gender (releases from 2026-10-27 onwards);
- keeps contractions (such as Portuguese *do*, "contraction of de + o") as
  entries with their own definitions, rather than as pointers to another word
  (Portuguese file onwards);
- puts "(Brazil)" or "(Portugal)" in front of a definition the source marks
  as used in that region (Portuguese file onwards);
- links a form whose page points at another form's page (French
  *arrivés*, "masculine plural of *arrivé*", itself "past participle of
  *arriver*") to the lemma, with both pages' grammatical tags (French file
  onwards);
- puts "(Quebec)", "(Canada)", "(Louisiana)", "(France)", "(Belgium)" or
  "(Switzerland)" in front of a French definition the source marks as used
  there (French file onwards);
- adds, for Portuguese verbs, the infinitive spelling used before an attached
  pronoun (*fazê* in *fazê-lo*, *amá* in *amá-la*), tagged as the infinitive;
- leaves out pronunciation, etymology, examples, translations and all other
  fields.

## Licence

Wiktionary text is licensed under the
[Creative Commons Attribution-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-sa/4.0/)
licence (CC BY-SA 4.0), and so are the files here, as adapted material. You
may share and adapt them, including commercially, provided you give
attribution and release any adapted material under the same licence. The
full licence text is in [LICENSE](LICENSE).
