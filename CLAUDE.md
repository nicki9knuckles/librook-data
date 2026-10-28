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

Each language is its own librook feature (`features/<NNN>-<language>/`),
with an intent, spec and plan like any other. Portuguese
(`features/010j-portuguese/`) is the worked example: its `plan.md` records
every check below and what it found. The steps run from the librook repo
unless they say otherwise.

**The spec must answer these** before any data is built, since each has
needed a decision before:
- Which regional variety is the target (Portuguese: Brazilian), and
  whether regional meanings get labels.
- How words the Reader splits apart are handled. The Reader splits at
  every non-letter, so *fazê-lo* is tapped as *fazê* and *lo*, and
  *l'homme* as *l* and *homme*. Changing that is an app-wide change, so
  the default is to fix lookups in the data instead.
- How contractions and elisions are explained (*do* = *de* + *o*).
- What counts as good enough coverage (see the baselines in step 6).

1. **Check that the app supports the language.** It must be in librook's
   `SUPPORTED_LANGUAGES` (`src/features/library/languageDetection.ts`).
   Use the same two-letter code everywhere.

2. **Get the data.** Keep downloads outside both repos, for example in a
   scratch folder.
   - **English edition** (required): lemmas, parts of speech, English
     definitions and grammatical tags. Use the per-language file
     `https://kaikki.org/dictionary/<Language>/kaikki.org-dictionary-<Language>.jsonl`
     while it exists (Portuguese: 582 MB). If it doesn't, use the full
     `raw-wiktextract-data.jsonl.gz` (about 3 GB) from
     https://kaikki.org/dictionary/rawdata.html; the build filters by
     `lang_code`.
   - **Native edition** (optional): definitions written in the language
     itself, from
     `https://kaikki.org/dictionary/downloads/<code>/<code>-extract.jsonl.gz`.
     It is the language's *own* Wiktionary, not a source of English. With no
     edition, build without `--native` and the app shows English straight
     away.

3. **Inspect the source before building.** These questions found every
   problem Portuguese had. Use `jq` on the English-edition file (`$EN`),
   and `gzcat` piped into `jq` for the native edition:
   - *Do contractions and elisions survive?* Look at a few common ones:
     `jq -c 'select(.word=="do") | {pos, senses: [.senses[] | {glosses, tags, form_of, alt_of}]}' $EN`.
     A word whose senses are all `form_of`/`alt_of` becomes a pointer, not
     an entry, unless its part of speech is `contraction`. Spellings with a
     hyphen are dropped entirely, because the Reader can never tap them
     whole. Spellings with an apostrophe are kept (010m): *l'* is looked up
     as *l*, and *aujourd'hui* by its pieces. Check the native edition's
     apostrophes too: French writes them curly (*aujourd’hui*).
   - *Do form-of pages point at other form-of pages?* French *arrivés* is
     "plural of *arrivé*", itself "participle of *arriver*". The build
     follows one such step.
   - *Do inflection tables list joined forms?* Look at a common verb's
     `forms` for hyphens or apostrophes. If the table doesn't list the
     shortened stem the Reader will look up (*fazê*), a rule is needed.
   - *Which region tags do senses use, in both editions?*
     `jq -r '.senses[]?.tags[]?' $EN | grep '^[A-Z]' | sort | uniq -c | sort -rn | head`.
     The editions spell them differently (*Brazil* and *Brazilian*), and
     may put them in `raw_tags` instead (French: *Québec*, *Suisse*):
     check `.senses[]?.raw_tags[]?` as well.

4. **Add any per-language rules** to `LANGUAGE_RULES` in
   `scripts/referencePack.ts`. It is the only place the build treats
   languages differently: the rest of the build names no language. A
   language with no entry gets the default processing. A new rule:
   - must be needed by a finding in step 3 or 6, not by guesswork;
   - must be general if it can be. Contractions are kept for every
     language, by part of speech. Only a rule that truly varies by language
     belongs in `LANGUAGE_RULES`;
   - comes with real, trimmed fixture lines
     (`__tests__/fixtures/wiktextract-<code>.jsonl` and
     `-native.jsonl`; `jq -c` with the fields the build reads) and tests in
     `__tests__/reference-pack-build.test.ts`. That includes a test that
     another language's build is unaffected.

5. **Build:**
   ```
   npm run build:reference -- --language <code> \
     --english <english file> --native <native file> \
     --source "English Wiktionary via wiktextract (kaikki.org), <date>" \
     --native-source "<Language> Wiktionary via wiktextract (kaikki.org), <date>" \
     --out reference-<code>.db
   ```
   `<date>` is the date of the kaikki.org extracts. The script prints the
   size and the number of entries, forms and tag sets.

6. **Measure, and stop at a gate.**
   - **Size:** under about 50 MB (the devices are e-ink readers with limited
     storage). Spanish is 45.8 MB, half of it pronoun-combined verb forms
     (*háblame*). Portuguese is 19.9 MB, because it writes those forms with
     a hyphen and the build drops hyphenated spellings. French is 24.4 MB.
     Over 50 MB: discuss
     trimming with the user before changing the build.
   - **Coverage**, against a real book in the language (ideally one the
     user means to read, in modern spelling):
     ```
     npm run measure:reference -- --pack reference-<code>.db --text book.txt \
       --words <common contractions and joined-word parts to inspect>
     ```
     The script's header explains how to extract an EPUB's text. It prints
     running-text and different-word coverage, the share with native
     definitions, how hyphenated and apostrophe-joined words fare, and the
     most frequent misses.

     | Baseline | Running text found | Different words found | Native definitions |
     |---|---|---|---|
     | Spanish, *Éramos unos niños* (modern) | 94.5% | 89.9% | 95.6% of found |
     | Portuguese, *Dom Casmurro* (1899 spelling) | 93.7% | 80.8% | 95.0% of found |
     | French, *Maria Chapdelaine* (1913, Quebec) | 98.8% | 97.7% | 99.3% of found |

     A new language should land near these figures. **Read the most frequent
     misses**: names and old spellings are expected, but a whole class of
     common words (Portuguese contractions, a pronoun stem) means the
     source marks them in a way the build doesn't handle. Go back to step 3
     or 4. If the gap is still clearly below the baselines, stop and bring
     the figures to the user.
   - **Spot-check** with `--words`: a common inflected word, an ambiguous
     word, a base form, and each kind of joined word or contraction the
     spec covers.

   Record the figures in the feature's `plan.md`.

7. **Publish**, after asking the user, since the release is public and
   permanent:
   ```
   gh release create <code>-<date> reference-<code>.db --repo nicki9knuckles/librook-data \
     --title "<Language> reference data (<date>)" \
     --notes "Built from kaikki.org extracts dated <date>. CC BY-SA 4.0."
   ```
   The release tags this repo's current `main`, so no commit is needed
   first. Confirm the download works without signing in:
   `curl -sIL <download URL>` should end in `200`, with a `content-length`
   matching the file's size in bytes.

8. **Update this repo's `README.md`.** Add a row to the Files table, the
   native-edition source under "Sources and attribution", and any new
   processing (including new `LANGUAGE_RULES`) under "Changes made to the
   source data". The licence requires all three.

9. **Update the app** (in librook): add the language to `REFERENCE_PACKS`
   in `src/features/reference/referencePacks.ts`, with the download URL,
   the exact size in bytes, and both source texts. A test checks that the
   URL's release and file name match the language. Nothing else in the app
   should need to change. If something does, it belongs in the feature's
   plan, and it must not assume the new language.

10. **Verify on the device:** install the language from Settings, look up
    the spot-check words offline, and check that another language's lookups
    still work.

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
