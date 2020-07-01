simple inverted index search engine

example usage

```
const {
  Index,
  AND,
  OR,
  TERM,
  CONSTANT,
  DISMAX,
  analyzers
} = require("paxx");

let ix = new Index({
  name: analyzers.autocompleteAnalyzer,
  type: analyzers.IDanalyzer
});

ix.doIndex(
  // documents to be indexed
  [
    { name: "john Crème Brulée", type: "user" },
    { name: "hello world k777bb k9 bzz", type: "user" },
    { name: "jack", type: "admin" },
    { name: "doe" }
  ],
  // which fields to index (must be strings)
  ["name", "user"]
);

// iterate over all the matches
ix.forEach(
  new OR(
    ix.TERM("name", "creme"),
    new AND(
      // matches on k9 because it splits k and 9
      ix.TERM("name", "9"),
      ix.TERM("name", "k"),
      ix.TERM("name", "hell")
    ),
    new AND(ix.TERM("name", "ja"), new CONSTANT(1, ix.TERM("type", "user"))),
    new DISMAX(ix.TERM("name", "doe"), ix.TERM("type", "user"))
  ),
  // callback called with the document and its IDF score
  function(doc, score) {
    console.log({ doc, score });
  }
);

outputs:

{ doc: { name: 'john Crème Brulée', type: 'user' },
  score: 2.6931471805599454 }
{ doc: { name: 'hello world k777bb k9 bzz', type: 'user' },
  score: 7.673976433571672 }
{ doc: { name: 'doe' }, score: 2.6931471805599454 }

```

## queries

* TERM

```

new TERM(numberOfDocumentsInIndex, postingsList)
e.g.

let t = new Term(5, [1,2,34])
```

the term is the most primitive query, it uses binary search to advance its
position. (same as lucens's TermQuery)

its score is (unnormalized) IDF as follows:

```
this.idf = 1 + Math.log(nDocumentsInIndex / (postingsList.length + 1));
```

tf is not used or stored.

* AND

```
new AND(queryA, queryB, queryC)
```

returns (queryA AND queryB AND queryC) boolean query (similar to lucene's Bool MUST), its score is the sum of the scores of the matching subqueries (unnormalized)

* OR

```
new OR(queryA, queryB, queryC)
```

returns (queryA OR queryB OR queryC) boolean query (similar to lucene's Bool SHOULD), its score is the sum of the scores of the matching subqueries (unnormalized)

* DISMAX

```
new DISMAX(tiebreaker, queryA, queryB, queryC)

e.g.
let q = new DISMAX(0.1, queryA, queryB, queryC)
```

returns (queryA DISMAX queryB DISMAX queryC) boolean OR query (similar to lucene's DisMax), its score is the max of the matching subqueries plus the tiebreaker multiplier by the rest of the scores.

* CONSTANT

```
new CONSTANT(boost, query)

e.g.
let q = new CONSTANT(0.1, query)
```

returns a constant score query, that will score with whatever boost you give it, in the example used `new CONSTANT(0.1...)` it will score with 0.1

## TOP N

```
let limit = 2 // -1 for all matches, sorted by idf score
let matches = ix.topN(
  new DISMAX(0.5, ix.TERM("name", "hello"), ix.TERM("name", "world")),
  limit
)

outputs: 
[ { name: 'hello world k777bb k9 bzz', type: 'user' } ]
```


## more examples

```
ix.forEach(
  new OR(0.5, ix.TERM("name", "hello"), ix.TERM("name", "world")),
  function(doc, score) {
    console.log({ doc, score });
  }
);

ix.forEach(
  new DISMAX(
    // tiebreaker
    0.5,
    // variable argument list of queries
    ix.TERM("name", "hello"),
    new CONSTANT(1000, ix.TERM("name", "world"))
  ),
  function(doc, score) {
    console.log({ doc, score });
  }
);

```

## Index

create inverted index (which is a handy way to store the postings lists and create term queries)
```
let ix = new Index({
  name: analyzers.autocompleteAnalyzer,
  type: analyzers.IDanalyzer
});

ix.doIndex(
  // documents to be indexed
  [
    { name: "john Crème Brulée", type: "user" },
    { name: "hello world k777bb k9 bzz", type: "user" },
    { name: "jack", type: "admin" },
    { name: "doe" }
  ],
  // which fields to index (must be strings)
  ["name", "user"]
);

```

## analyzers

analyzer is a group of tokenizers and normalizers
* Autocomplete

```
tokenize at index: whitespace, edge
tokenize at search: whitespace
normalize: lowercase, unaccent, spaceBetweenDigits
```


* NOOP
```
tokenize at index: noop
tokenize at search: noop
normalize: noop
```


### tokenizers

a tokenizer takes a string and produces tokens from that string, at the moment those are available:

* whitespace: 'a b c' -> ['a','b','c']
* noop: 'a b c' -> ['a b c']
* edge: 'hello' -> ['h','he','hel','hell','hello']

any object that has `apply([string]) -> [string]` function can be used as tokenizer

### normalizers

normalizes apply transformation to the string, used both at search and index time

* lowercase: 'ABC' -> 'abc'
* unaccent: 'Crème' - 'Creme'
* space between digits: k9 -> 'k 9'

any object that has `apply(string) -> string` function can be used as normalizer

