# Rule-Based Chunker

A rule-based system that splits a text into **chunks**. Following Abney's definition, a chunk is one content word (noun, verb, adjective) together with the small grammatical words around it, for example *sur votre liste* or *vous êtes parvenus*.

The system uses a hand-written lexicon of function words and a small set of rules. There is no ML or NLP library, everything is written from scratch in Python.

It was built on a French text, tested on a second French text, then transferred to English by translating the lexicon and keeping the same rules.

Course project for *Formalismes pour le TAL* and *Représentation des connaissances*, M1 Language Industries (NLP), Université Grenoble Alpes (2025–2026).
Full report, in French: [report_fr.md](report_fr.md)

## Example

```
[ConjSub] Alors que
[SV] vous êtes parvenus
[PN] à prendre
[N] tous les produits inscrits
[PN] sur votre liste
[PN] de courses
[PN] en seulement 23 minutes
[Pct] ,
```

The same result in the XML output:

```xml
<texte_chunke>
  <chunk cat="ConjSub">Alors que</chunk>
  <chunk cat="SV">vous êtes parvenus</chunk>
  <chunk cat="PN">à prendre</chunk>
  ...
</texte_chunke>
```

## How it works

1. **Tokenization**: a regular expression splits the text into words and punctuation. French elisions (*l'*, *qu'*, *c'*...) and hyphenated words (*peut-on*) are kept as single tokens.
2. **Tagging**: each token gets a category from the lexicon. Two-word expressions (*parce que*, *alors que*) are checked first. For words not in the lexicon, simple suffix rules help: *-ment* (French) or *-ly* (English) means adverb.
3. **Chunking**: when a token's category matches a rule, the current chunk is closed and a new one is opened. Otherwise, the token is added to the current chunk.
4. **Clean-up**: fragments without a label are merged into the next chunk.
5. **Export**: results are saved as XML and HTML.

## Lexicon and rules

The lexicon only contains **function words**: prepositions, determiners, subject pronouns, relative pronouns, conjunctions, modals and auxiliaries, punctuation and quotation marks. Nouns, verbs, adjectives and adverbs are open classes, so they are not listed.

```
Prep = {sur, à, en, de, du, selon, pour, ...}
Det  = {tous, votre, un, une, la, les, le, ...}
```

Rules have the form `MarkerCategory → [ChunkType` and cover at most 2 tokens:

- `Det → [N`: a determiner opens a noun chunk.
- `Prep + _ → [PN`: a preposition opens a prepositional chunk, and the next token is added to it whatever its category.

If a determiner appears while a noun chunk is already open, it stays in that chunk, so *tous les produits* is one chunk and not two.

### Chunk types

| Label | Meaning | Opened by |
|-------|---------|-----------|
| `N` | noun chunk | a determiner |
| `PN` | prepositional chunk | a preposition |
| `SV` | verbal chunk | a subject pronoun |
| `V` | isolated verb (*sont*, *seront*, *was*) | a modal or auxiliary |
| `Adv` / `Adj` | adverbial / adjectival chunk | a suffix rule |
| `ConjSub` / `ConjCoor` | subordinating / coordinating conjunction | the conjunction |
| `Pct` / `PctFin` | punctuation / end-of-sentence punctuation | the sign |
| `GO` / `GF` | opening / closing quotation mark | the sign |

## Data

| Text | Language | Source |
|------|----------|--------|
| 1 (used to build the rules) | French | Le Gorafi, [*Selon une étude, la file d'attente de l'autre caisse avançait plus vite*](https://www.legorafi.fr/2025/08/22/selon-une-etude-la-file-dattente-de-lautre-caisse-avancait-plus-vite/), 22 Aug 2025 |
| 2 | French | Le Gorafi, [*83% des Français avouent dormir au bureau pour récupérer de leur week-end en famille*](https://www.legorafi.fr/2026/03/04/83-des-francais-avouent-dormir-au-bureau-pour-recuperer-de-leur-week-end-en-famille/), 4 Mar 2026 |
| 3 | English | The Onion, [*Overambitious Man Wants To Get 2 Things Done Today*](https://theonion.com/overambitious-man-wants-to-get-2-things-done-today/) |

Each text was chunked by hand to create a reference for evaluation. The articles belong to their publishers and are used here for educational purposes only.

## Results

The output was compared with the manual reference. Scores are F1 for chunk boundaries and for chunk labels.

| # | Text | Lexicon | F1 boundaries | F1 labels |
|---|------|---------|:---:|:---:|
| 1 | French, text used to build the rules | `lex.xlsx` | 0.76 | 0.74 |
| 2.1 | French, new text | `lex.xlsx` | 0.43 | 0.43 |
| 2.2 | Same text | `lex2.xlsx` (enriched) | 0.73 | 0.73 |
| 3.1 | English, same rules | `lex_en.xlsx` (translated) | 0.43 | 0.40 |
| 3.2 | Same text | `lex_en2.xlsx` (enriched) | 0.54 | 0.52 |

**What this shows**

- **The rules generalize; the lexicon is the bottleneck.** On text 2, common words like *je*, *mes*, *son*, *des* and *quand* were missing from the lexicon, so chunks became far too long. Adding them raised F1 from 0.43 to 0.73, close to text 1, without changing any rule.
- **The same rules work on English** with a translated lexicon, but with lower scores.
- **Precision is higher than recall** in every experiment: the system produces fewer, longer chunks than the reference.

The evaluation only counts exact chunk matches, on three short texts.

## Main errors

- **Content words are not in the lexicon.** Verbs, nouns and adjectives are added to whatever chunk is open, so *l'univers a décidé* becomes a single noun chunk. Adjective chunks and proper-noun chunks are never produced.
- **Some function words are ambiguous.** *vous* and *nous* are always treated as subjects, even when they are objects (*pour vous contrarier*). *de* can be a preposition or a determiner, and English *that* can be a determiner or a conjunction.
- **English-specific problems.** Phrasal verbs (*pick up*, *slow down*) are split. Contractions (*you're*, *there's*) are badly tokenized because the tokenizer was designed for French elisions. Proper nouns (*James Chao*) are not detected.

## Possible improvements

- Suffix rules for verb forms (French *-er*, *-ir*, *-é*, *-ant*; English *-ing*, *-ed*), with a list of exceptions, since *métier* is not a verb.
- A capital-letter rule to detect proper nouns.
- A more systematic lexicon, with all subject pronouns and all possessive determiners.
- For English: split contractions in the tokenizer (*you're* → *you* + *'re*), list phrasal verbs as two-word expressions, and add a context rule for *that*.

## Project structure

```
mbd.ipynb          notebook: chunker, experiments and evaluation
report_fr.md       project report (French)
bd.xlsx            chunking rules
lex.xlsx           French lexicon
lex2.xlsx          enriched French lexicon
lex_en.xlsx        English lexicon (translated)
lex_en2.xlsx       enriched English lexicon
article1-3.txt     input texts
chunks1-3.xlsx     manual reference annotation
sortie*.xml/.html  outputs
```

## How to run

Requires Python 3.9 or later.

```bash
pip install pandas openpyxl jupyter
jupyter notebook mbd.ipynb
```

Run the cells in order. The output files are written to the same folder.


## Author

Darya Zdrelyuk, Master 1 in Language Industries (NLP), Université Grenoble Alpes, 2025–2026