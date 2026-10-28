# librook-data

This repo hosts reference data files for the librook app, one SQLite file
per language, attached to GitHub releases. The repo itself holds only
documentation: `README.md` (attribution, changes, file table) and `LICENSE`
(CC BY-SA 4.0). Never commit `.db` files to git; they belong on releases.

The build code lives in the **librook** repo (private), not here:
`scripts/referencePack.ts` (the build, with its tests in
`__tests__/reference-pack-build.test.ts`) and
`scripts/build-reference-pack.mjs` (the command-line wrapper, run with
`npm run build:reference`). Design notes are in
`features/010-language-reference-data/plan.md` in that repo.

## Adding a language

1. **Check that the app supports the language.** It must be in librook's
   supported-language list (`SUPPORTED_LANGUAGES` in
   `src/features/library/languageDetection.ts`). Use the same two-letter
   code.

2. **Get the English-edition data.** This gives lemmas, parts of speech,
   English definitions and grammatical tags, and is required. The build
   filters by `lang_code`, so either works:
   - the full English Wiktionary extract
     (`raw-wiktextract-data.jsonl.gz` from
     https://kaikki.org/dictionary/rawdata.html), which contains every
     language; or
   - the per-language file under `https://kaikki.org/dictionary/<Language>/`,
     if it still exists. These are marked deprecated.

3. **Get the native-edition data, if there is one.** This gives definitions
   in the language itself, and is optional. It is
   `https://kaikki.org/dictionary/downloads/<code>/<code>-extract.jsonl.gz`,
   listed on the same raw-data page. Watch out: `<code>-extract` is that
   language's *own* Wiktionary, so its definitions are written in that
   language. It is not a source of English definitions. If no edition
   exists, build without `--native`, and the app shows English straight
   away.

4. **Build**, from the librook repo:
   ```
   npm run build:reference -- --language <code> \
     --english <english file> --native <native file> \
     --source "English Wiktionary via wiktextract (kaikki.org), <date>" \
     --native-source "<Language> Wiktionary via wiktextract (kaikki.org), <date>" \
     --out reference-<code>.db
   ```
   `<date>` is the date of the kaikki.org extract you downloaded. Each input
   may be plain JSONL or `.gz`. The script prints the size and the number of
   entries, forms and tag sets.

5. **Check the size.** The target is under about 50 MB per language (the
   devices are e-ink readers with limited storage). If it's over, discuss
   trimming with the user before changing the build. Spanish is 45.8 MB,
   and pronoun-combined verb forms (*háblame*) make up about half its forms.
   Portuguese is 19.9 MB: it writes those forms with a hyphen (*fala-me*),
   and the build drops every spelling with a hyphen, since the app's Reader
   splits words at hyphens and could never look one up.

6. **Sanity-check the output** with `sqlite3`. Look up a few common
   inflected words, an ambiguous word, and a base form, and check that the
   lemma, part of speech, tags and both sets of definitions look right.
   Also check how many entries have native definitions
   (`SELECT COUNT(*) FROM entries WHERE native_glosses <> '[]'`). For
   Spanish it's 47%, for Portuguese 50%.

   Then **measure coverage** against a real book in the language, from the
   librook repo (for an EPUB, extract its text first; the script's header
   says how):
   ```
   npm run measure:reference -- --pack reference-<code>.db --text book.txt \
     --words <a few common words to inspect>
   ```
   It prints the share of the book's different words found, the share of
   those with native definitions, how hyphenated words fare, and the
   readings of each listed word. Spanish finds 93.0% of the running text of
   a modern novel, and Portuguese 93.3% of an 1899 one. Most misses should
   be names or old spellings. If common words are missing, look at how the
   source marks them before publishing; Portuguese contractions (*do*,
   *na*) were dropped until the build learned to keep them.

   Also check whether the language needs a rule in `referencePack.ts`.
   `SHORTENED_INFINITIVES` lists, per language, how an infinitive is spelled
   before an attached pronoun written after a hyphen (Portuguese *fazê-lo*),
   since the Reader looks up *fazê* on its own.

7. **Publish.** Commit any README change first, because a release needs a
   commit to tag. Then:
   ```
   gh release create <code>-<date> reference-<code>.db --repo nicki9knuckles/librook-data \
     --title "<Language> reference data (<date>)" \
     --notes "Built from kaikki.org extracts dated <date>. CC BY-SA 4.0."
   ```
   Confirm the download works without signing in:
   `curl -sIL <download URL>` should end in `200` with the right
   `content-length`.

8. **Update `README.md`.** Add a row to the Files table, and add the
   native-edition source under "Sources and attribution". If the build
   changed how data is processed, update "Changes made to the source data".
   The licence requires both.

9. **Update the app** (in librook): add the language to the list of
   available reference data, with the download URL, the exact size in
   bytes, and the source text. That list is `REFERENCE_PACKS` in
   `src/features/reference/referencePacks.ts`.

## Rules

- **Never replace or delete a published release file.** Installed apps
  point at that exact URL. A rebuild with newer data gets a new release
  (`<code>-<new date>`) and an app update that points to it.
- **The data file's tables are a contract with the app.** Changing them
  means bumping `PACK_SCHEMA_VERSION` in `scripts/referencePack.ts`, and the
  app refuses data files with a schema version it doesn't recognise. So a
  schema change needs a matching app change and new releases for every
  language.
- **Keep the attribution and licence intact.** Wiktionary is CC BY-SA 4.0,
  and these files are adapted material under the same licence.
