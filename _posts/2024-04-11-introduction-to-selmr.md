---
title: "Introduction to SELMR"
date: "2024-04-17"
categories: ["natural language processing", "NLP", "explainability"]
math: true
---

The selmr crate [on crates.io](https://crates.io/crates/selmr) provides a library 
for generating and using simple text data structures that work like language models. 
The data structures do not use real-valued vector embeddings; instead they use the
mathematical concept of multisets and are derived directly from plain text data.

The data structures are named Simple Explainable Language Multiset Representations
(SELMRs) and consist of multisets created from all multi-word expressions and all
multi-word-context combinations contained in a collection of documents given some
contraints. The multisets can be used for downstream NLP tasks like text classifications
and searching, in a similar manner as real-valued vector embeddings.

SELMRs produce explainable results without any randomness and enable explicit links
with lexical, linguistical and terminological annotations. No model is trained and no
dimensionality reduction is applied.

# Simple model based on DBpedia pages

The model used below is based on the plain text of 10.000 DBpedia pages (217 Mb), with
phrase length of one to three words, and  left and right context length of also one to 
three words. It and can be downloaded from here:

* [dbpedia_1_10_p=3_lc=3_rc=3_lang=en.zip](https://mangosaurus.nl/models/dbpedia_1_10_p=3_lc=3_rc=3_lang=en.zip)

Read the file with:

```rust
    use selmr::selmr::SELMR;
    let mut s = SELMR::new();
    s.read("dbpedia_1_10_p=3_lc=3_rc=3_lang=en.zip", "zip");
```

# Example: (multi-)word similarities

Take for example the multi-word "has been suggested". The data structure based on the
plain text data of 10.000 DBpedia pages returns the following top ten most similar
multi-words

```rust
let actual = s.most_similar("has been suggested".to_string(), None, 15, 15, 10, "count").unwrap();
println!("{:?}", actual);
```

This gives:
```rust
[
    ("has been suggested", 15.0),
    ("has been argued", 10.0),
    ("has been speculated", 10.0),
    ("is argued", 9.0),
    ("has been proposed", 7.0),
    ("is suggested", 7.0),
    ("has been claimed", 7.0),
    ("is clear", 7.0),
    ("is probable", 7.0),
    ("can be shown", 7.0)
]
```

This shows the multi-words with the number of contexts that they have in common with "has been
suggested" (see below). So "is probable" has 11 contexts in common with "has been suggested".
The more contexts that a multi-word has in common the more "similar" it is. Note that this
similarity measure is only based on patterns in text; it is not a syntactical or semantic
similarity measure.

An example with the name "Aldous Huxley" which returns the following.

```rust
let actual = s.most_similar("Aldous Huxley".to_string(), None, 15, 15, 10, "count").unwrap();
println!("{:?}", actual);
``` 

This gives:
```rust
[
    ("Aldous Huxley", 15.0),
    ("Ben Bova", 6.0),
    ("Clark Ashton Smith", 5.0),
    ("Apuleius", 5.0),
    ("E E Smith", 5.0),
    ("Blaise Pascal", 5.0),
    ("L Frank Baum", 5.0),
    ("A E Housman", 5.0),
    ("Anatole France", 5.0),
    ("Marcel Proust", 4.0)
]
```

The results are based on the common contexts with "Aldous Huxley". For example
the coinciding contexts of "Aldous Huxley" and "Ben Bova" are:

```rust
    ("by ... at", 3),
    ("by ... at LibriVox", 1),
    ("by ... at Open", 1),
    ("by ... at Project", 1),
    ("Works by ... at LibriVox", 1),
    ("Works by ... at Open", 1),
    ("Works by ... at Project", 1),
    ("about ... at", 1),
    ("article ... bibliography", 1),
```

So in this case similarities are found because DBpedia contains a bibliography of
their work and (some of) their works are available on LibriVox, Open (Library) and
Project (Gutenberg).

# Taking into account contexts

Some words have multiple meanings. The different usages of a word can, to some extent,
be derived from the contexts in which they are used. Take for example the word "deal"
which is used as a noun and as a verb. By adding the context to the function call we
can restrict the results to similar words that occur in the input context. We can
restrict the output by providing the context "a ... with":

```rust
let actual = s.most_similar("deal", ("a", "with"), 10).unwrap();
println!("{:?}", actual);
``` 

This gives:
```rust
[
    ("deal", 25.0)
    ("contract", 7.0),
    ("treaty", 6.0),
    ("dispute", 5.0), 
    ("meeting", 4.0), 
    ("close relationship", 4.0), 
    ("peace treaty", 4.0),
    ("problem", 4.0),
    ("partnership", 4.0),
    ("par", 3.0)
]
```

Compare these results to the results when the context is "to ... with":

```rust
use std::collections::HashMap;
let actual = s.most_similar("deal", ("to", "with"), 10).unwrap();
println!("{:?}", actual);
``` 

This gives:
```rust
[
    ("deal", 25.0), 
    ("coincide", 5.0), 
    ("come up", 5.0), 
    ("interact", 5.0), 
    ("cope", 5.0), 
    ("experiment", 4.0), 
    ("cooperate", 4.0), 
    ("comply", 4.0), 
    ("interfere", 4.0), 
    ("communicate", 4.0)
]
```

# How does it work

## Multiset representations

A multiset is a modification of a set that allows for multiple instances for
each of its elements. It contains elements with their number of instances,
called multiplicities. Instead of real-valued vectors, multisets contain a
variable number of elements with their multiplicities. The corresponding data
structure of a multiset is not an fixed length array of numbers but a dictionary
with elements as keys and multiplicities as values.

This concept can be used to represent multi-words in a corpus by creating a
multiset for each multi-word that contains all contexts in which the multi-word
occurs with their number of occurrences. A context of a multi-word is a combination
(a tuple) of the preceding list of words (the left side) and the following list of
words (the right side) of that multiword, with a certain maximum length and a certain
minimum number of unique occurrences.

The most similar multi-words are found by looking at the number of common contexts
of an input multi-word and another mult-word in the corpus. The multi-words that have
the highest number of contexts in common are the most "similar" of the input multi-word.

## Similarity measures

The measure can be

* count: a simple count of the common contexts of phrase $p1$ and $p2$:

$$
\operatorname{count}(p1, p2) = | \operatorname{C}(p1) \cap \operatorname{C}(p2)|
$$

where $\operatorname{C}(p)$ is the set of contexts of multi-word $p$,


* Jaccard index: the intersection of contexts divided by the union of contexts of two phrases

$$
\operatorname{jaccard}(p1, p2) = \frac{\operatorname{C}(p1) \cap \operatorname{C}(p2)}{\operatorname{C}(p1) \cup \operatorname{C}(p2)}
$$

* weighted Jaccard index: 

$$
\operatorname{weighted\_jaccard}(p1, p2) = \frac{\displaystyle\sum min(m_{\operatorname{C}(p1)}(c), m_{\operatorname{C}(p2)}(c))}{\displaystyle\sum max(m_{\operatorname{C}(p1)}(c), m_{\operatorname{C}(p2)}(c))}
$$

where $m_{C}(c)$ is the multiplicity (the number of occurrences) of $c$ in multiset $C$.

The most similar words are calculated accordingly. For example To find the most similar words, 
the Jaccard distance of the contexts of the input multi-words and the contexts of each multi-word 
in the corpus is calculated. The most similar multi-words of input multi-word $p$ are calculated with

$$
\displaystyle\max_{k \in \mathrm{\mathbf{P}}} 1 - \operatorname{jaccard}(k, p)
$$

where
- $\mathrm{\mathbf{P}}$ is the set of phrases that fits one of the contexts of multi-word $p$

If an input context is also given then we restrict the phrases by replacing $\mathrm{\mathbf{P}}$ with
$\operatorname{Phrases}(c)$, i.e. the phrases that fit into context $c$:

$$
\displaystyle\max_{k \in \operatorname{Phrases}(c)} 1 - \operatorname{jaccard}(k, p)
$$

where
- $p$ is the input phrase and $c$ is the input context,
- $\operatorname{Phrases}(c)$ is the set of phrases that fit into context $c$.

Note that the most_similar function has an argument $topcontexts$ to specify the number
of most common contexts that has to be used when calculating the most similar multi-words.
Taking into account only a limited number of contexts yields better results. Similarly,
if a context is specified, then the argument $topphrases$ specifies the number of most
common multi-words that are used. Also note that we do not use the actual number of
occurrences of multi-words and contexts.

# Context-multi-word matches

You can search in the data structure with regex. For example common nouns ending with "-ion"
and start with one of the determiners "the", "a" or "an":

```ignore
let binding = s.matches(("the|a|an", "^[a-z]*ion$", ".*"))
    .expect("REASON");
let words_ion = binding
    .keys()
    .map(|(_, p, _)| p.as_str())
    .collect::<Vec<_>>();
```

# Constructing a SELMR

Create an empty SELMR data structure with

```ignore
let selmr = selmr::selmr::SELMR::new(
    1, // min_phrase_len=
    3, // max_phrase_len
    1, // min_context_len
    3, // max_context_len
    1, // min_phrase_keys
    2, // min_context_keys
    "en", // language
);
```

- min_phrase_len: the minimum number of single words in the multi-words
- max_phrase_len: the maximum number of single words in the multi-words
- min_context_len: the minimum number of single words in the left and right part of the contexts
- min_context_len: the maximum number of single words in the left and right part of the contexts
- min_phrase_keys: the minimum number of unique contexts each phrase must contain
- min_context_keys: the minimum number of unique phrases each context must contain
- language: the language of the data structure content

Then add text with:

```ignore
s.add("We went to the park to walk. And then we went to the city to shop.")
```

Then you can run:

```ignore
let r = s.most_similar("city");
assert!(r == [("city", 1), ("park", 1)])
```

Contexts with more words on left or right will only be added if the number of occurrences is different, so
if "the ... to" has two occurrences and "to the ... to" has also two occurrences then this latter context
will not be added because it does not add any information.

Write the data structure to a zip-file:

```ignore
s.write(file_name, "zip");
```

The zip-file contains one file named "data.json" that contains all data of the structure in json format.

Read the data structure from a zip file:

```ignore
s.read(file_name, "zip");
```
