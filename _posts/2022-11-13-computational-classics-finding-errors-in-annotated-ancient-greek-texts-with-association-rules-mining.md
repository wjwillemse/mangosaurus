---
title: "Computational Classics: Finding errors in annotated ancient Greek texts with association rules mining"
date: "2022-11-13"
categories: ["association rules mining", "NLP"]
---

This blog describes some experiments with ruleminer for finding morphological patterns in annotated data of ancient Greek texts. [Ruleminer](https://github.com/DeNederlandscheBank/ruleminer) is a Python package for association rules mining, a rules-based machine learning method for discovering patterns in large data sets. The regex and dataframe approach in ruleminer (set out in this [article](https://github.com/wjwillemse/ruleminer/blob/main/docs/paper.pdf)) is used to enable a controlled search in the data set. Previously, I have used ruleminer mainly for quantitative data, but it might be worth investigating whether it is applicable to annotated (NLP) text data.

### Finding annotation errors

The idea is to extract morphological patterns from annotated data and with these patterns detect annotation errors made by the NLP parser that was used for the annotations. Morphological patterns are recurrent relations between word forms and features, such as part of speech, tense, mood, case, number and gender. These recurrent relations can be expressed as association rules and mining algorithms can be used to find these relations. By looking at those situation where patterns were not satisfied it could be possible to identify errors made by the NLP parser. This is useful because many NLP pipelines use these annotation for subsequent analyses.

For many languages NLP parsers are available to annotate documents and determine lemmas and the morphological features of word forms within these documents. The performance of these models is often measured in the percentage of correct annotations against predefined treebanks, text corpora with annotations verified by linguists. Normally these models use deep learning algorithms and no model is yet able to achieve fully correct annotations; for many models these percentages lie around 95%. The [Perseus model in Stanford Stanza](https://stanfordnlp.github.io/stanza/available_models.html) for ancient Greek texts provides annotations with the following scores: universal part-of-speech tags (92,4%), treebank-specific part-of-speech tags (85,0%), treebank-specific morphological features (91,0%), and lemmas (88.3%).

### Preprocessing steps

As a basis for ancient Greek data I took a number of dialogues of Plato (Apology, Crito, Euthydemos, Euthypron, Gorgias, Laws, Phaedon, Phaedrus, Republic, Symposium and Timaois). The text documents were annotated with the Perseus model (the model was originally not trained with this data) and the result was converted to the NLP Interchange Format ([NIF 2.0](https://persistence.uni-leipzig.org/nlp2rdf/)) in RDF with [OLiA](https://github.com/acoli-repo/olia) annotations (all done with the [nafigator package](https://pypi.org/project/nafigator/) using the Stanford Stanza pipeline).

The data set consists of 312.955 annotated word forms. In order to apply ruleminer all words were extracted from the RDF-graph and stored in a Pandas DataFrame. Each word is a row in the DataFrame with the original text of the word (**nif:anchorOf)**, the lemma of the word derived from the Perseus model (**nif:lemma**) and, for each feature (57 in total), whether or not the word is annotated with this feature (columns start with **olia:**, for example **olia:Adverb**).

The only changes to the original text were the deletion of diacritic signs acute and grave (like ά and ὰ) because sometimes the place of these signs changes when suffixes are added or deleted, which makes it harder to find patterns. All other diacritic signs were unchanged.

This [notebook](https://gist.github.com/wjwillemse/4f93be44f44aa1e6d14a5a9249f3d25f) contains all examples mentioned below (and many more examples for a range of nominal forms).

### Deriving morphological patterns in ancient Greek

In what follows I will show some examples of rules that can be found in the annotated text. To check whether word forms have the annotated features and whether they have multiple meanings (with different features) I used [Perseus Greek Word Study Tool](http://www.perseus.tufts.edu/hopper).

#### Particle patterns

Particles in ancient Greek are short word forms that are never inflected, i.e. the form or ending does not change. So ideal for finding morphological patterns. First we look at the morphological features. Let's run the following expression to identify the OLiA annotations of the word **γαρ**:

```
if (({"nif:anchorOf"}=="γαρ")) then ({"olia:.*"}==1))
```

The **olia:.\*** in this expression means all columns that start with **olia:**, i.e. OLiA annotations except the lemma and the original text. Ruleminer will match all these columns and evaluate the metrics of the resulting candidate rule. If the candidate rule satisfies predefined constraints (here a minimum confidence of 90% was used) it will be added to the resulting rules:

Results of the expression: _if (({"nif:anchorOf"}=="γαρ")) then ({"olia:.\*"}==1))_

| rule\_definition | abs support | abs exceptions | confidence |
| --- | --- | --- | --- |
| if ({"nif:anchorOf"}=="γαρ") then ({"olia:Adverb"}==1) | 2165 | 42 | 0.98097 |

This produces one rule which states that the word γαρ is identified correctly as an adverb (the Perseus model maps particles to adverbs or conjunctions). This rule has a confidence of just over 98% (in 2.165 cases the word γαρ is annotated as an adverb). There are 42 exceptions, meaning that the word was not annotated as an adverb. These exceptions might point to situations where a word has different features (and meanings) depending on the context. In this case it is strange because γαρ has only one other meaning: the noun γάρος, which does not occur in Plato's work. I therefore expect that these are all annotation errors by the Perseus model. We can check this by looking at the associations between word forms and their lemmas with the following rule.

```
if (({"nif:anchorOf"}=="γαρ")) then ({"nif:lemma"}==".*"))
```

Here are the results of the expression: _if (({"nif:anchorOf"}=="γαρ")) then ({"nif:lemma"}==".\*"))_:

| rule\_definition | abs support | abs exceptions | confidence |
| --- | --- | --- | --- |
| if ({"nif:anchorOf"}=="γαρ") then ({"nif:lemma"}=="γαρ") | 2187 | 20 | 0.990938 |
| if ({"nif:anchorOf"}=="γαρ") then ({"nif:lemma"}=="γιγνομαι") | 9 | 2198 | 0.004078 |
| if ({"nif:anchorOf"}=="γαρ") then ({"nif:lemma"}=="ἐγώ") | 6 | 2201 | 0.002719 |
| if ({"nif:anchorOf"}=="γαρ") then ({"nif:lemma"}=="γιρ") | 4 | 2203 | 0.001812 |
| if ({"nif:anchorOf"}=="γαρ") then ({"nif:lemma"}=="γιπτω") | 1 | 2206 | 0.000453 |

Here the first rule in the table is the only one that is correct. The others have very low confidence and are obvious errors by the Perseus model: **γιρ** and **γιπτω** are nonexistent words, and **ἐγώ** and **γιγνομαι** are not the lemmas of **γαρ**. So these are incorrect annotations, and errors by the NLP parser.

Next we consider the particles **οὐ**, **οὐκ**, **οὐχ** (negating particles) and **μη** (a particle indicating privation). In this case for each word two annotations are found, the **olia:Adverb** and the **olia:Negation**.

Results of the expression with negating particles:

| rule\_definition | abs support | abs exceptions | confidence |
| --- | --- | --- | --- |
| if ({"nif:anchorOf"}=="μη") then ({"olia:Adverb"}==1) | 1753 | 59 | 0.967439 |
| if ({"nif:anchorOf"}=="μη") then ({"olia:Negation"}==1) | 1694 | 118 | 0.934879 |
| if ({"nif:anchorOf"}=="οὐ") then ({"olia:Adverb"}==1) | 1333 | 0 | 1.0000 |
| if ({"nif:anchorOf"}=="οὐ") then ({"olia:Negation"}==1) | 1332 | 1 | 0.9993 |
| if ({"nif:anchorOf"}=="οὐκ") then ({"olia:Adverb"}==1) | 1248 | 0 | 1.0000 |
| if ({"nif:anchorOf"}=="οὐκ") then ({"olia:Negation"}==1) | 1248 | 0 | 1.0000 |
| if ({"nif:anchorOf"}=="οὐχ") then ({"olia:Adverb"}==1) | 322 | 0 | 1.0000 |
| if ({"nif:anchorOf"}=="οὐχ") then ({"olia:Negation"}==1) | 322 | 0 | 1.0000 |

Most of the times the word **μη** is annotated as an adverb and as a negation. There are however a number of exceptions. Looking into this a bit further shows that the word is sometimes annotated as a subordinating conjunction and sometimes the lemma is mistakenly set to **μεμω** or **εἰμι** resulting in incorrect verb related annotations. Here are the lemmas in case the word is not an adverb:

| rule\_definition | abs support | abs exceptions | confidence |
| --- | --- | --- | --- |
| if ({"nif:anchorOf"}=="μη") then (({"olia:Adverb"}!=1)&({"nif:lemma"}=="μη")) | 42 | 1770 | 0.0232 |
| if ({"nif:anchorOf"}=="μη") then (({"olia:Adverb"}!=1)&({"nif:lemma"}=="μεμω")) | 16 | 1796 | 0.0088 |
| if ({"nif:anchorOf"}=="μη") then (({"olia:Adverb"}!=1)&({"nif:lemma"}=="εἰμι")) | 1 | 1811 | 0.0006 |

#### Pronoun patterns

The word form of Ancient Greek pronouns depend on the case and grammatical number. In most of the cases the personal pronoun does not have other meanings depending on the context, so this should lead to strong patterns. To run ruleminer with a list of expressions we can use the following code.

```python
# personal pronouns
# first person, second person
pronouns = [
    'ἐγω', 'ἐμοῦ', 'ἐμοι', 'ἐμε', 
    'μου', 'μοι', 'με',
    'συ', 'σοῦ', 'σοι', 'σε', 
    'σου',
    'ἡμεῖς', 'ἡμῶν', 'ἡμῖν', 'ἡμᾶς',
    'ὑμεῖς', 'ὑμῶν', 'ὑμῖν', 'ὑμᾶς'
]
expressions = [
    'if (({"nif:anchorOf"}=="'+pn+'")) then ({"olia:Pronoun"}==1)'
    for pn in pronouns
]
```

Then we get, sorted with highest support, the following result

Results of pronoun patterns

| rule\_definition | abs support | abs exceptions | confidence |
| --- | --- | --- | --- |
| if ({"nif:anchorOf"}=="ἡμῖν") then ({"olia:Pronoun"}==1) | 671 | 0 | 1.0000 |
| if ({"nif:anchorOf"}=="μοι") then ({"olia:Pronoun"}==1) | 499 | 4 | 0.9920 |
| if ({"nif:anchorOf"}=="ἐγω") then ({"olia:Pronoun"}==1) | 383 | 36 | 0.9141 |
| if ({"nif:anchorOf"}=="σοι") then ({"olia:Pronoun"}==1) | 344 | 11 | 0.9690 |
| if ({"nif:anchorOf"}=="ἡμῶν") then ({"olia:Pronoun"}==1) | 230 | 0 | 1.0000 |
| if ({"nif:anchorOf"}=="συ") then ({"olia:Pronoun"}==1) | 229 | 27 | 0.8945 |
| if ({"nif:anchorOf"}=="ἡμᾶς") then ({"olia:Pronoun"}==1) | 208 | 0 | 1.0000 |
| ... | ... | ... | ... |

Again this points to many errors in the Perseus model. Word forms like **ἐγω**, **μοι** and **συ** cannot be taken as anything other than a pronoun. However, the word **σοι** could also be a possessive adjective depending on the context.

#### Verbs patterns

We now have seen some easy examples with straightforward rules. For verbs we need more complex rules, but this is still feasible with ruleminer.

In ancient Greek if a verb is thematic, in present tense, indicative mood, plural and third person then the ending of that verb (if it is not contracted) is **stem+ουσι(ν)**. To formulate a rule for this we want to keep the stem of the verb that was found in the antecedent (the if-part) and use it later on in the consequent of the rule (the then-part). This can be done by defining a regex group (by using with parentheses) in the following way:

```
if (({"nif:anchorOf"}=="(\w+[^εαο])ουσιν?")) then (({"nif:lemma"}=="\1ω"))
```

The if-part of the rule is true if the column **nif:anchorOf** matches the regex **(\\w+\[^εαο\])ουσιν?**. The first part of this regex (between parenthesis) consists of one or more characters not ending with **ε**, **α**, and **ο**. This is the stem of the word and it is stored as a regex group (to be used in the consequent of the rule). The second part is **ουσιν?**, which is regex for either **ουσι** or **ουσιν**. The then-part of the rule is true is **nif:lemma** contains the stem of the word (in regex this is **\\1**) plus **ω**.

The first five lines of the expression: _if (({"nif:anchorOf"}=="(\\w+\[^εαο\])ουσιν?")) then (({"nif:lemma"}=="\\1ω"))_ (85 rules in total).

| rule\_definition | abs support | abs exceptions | confidence |
| --- | --- | --- | --- |
| if({"nif:anchorOf"}=="ἐχουσιν")then({"nif:lemma"}=="ἐχω") | 31 | 0 | 1.0000 |
| if({"nif:anchorOf"}=="ἐχουσι")then({"nif:lemma"}=="ἐχω") | 20 | 1 | 0.9524 |
| if({"nif:anchorOf"}=="μελλουσιν")then({"nif:lemma"}=="μελλω") | 16 | 0 | 1.0000 |
| if({"nif:anchorOf"}=="μελλουσι")then({"nif:lemma"}=="μελλω") | 10 | 0 | 1.0000 |
| if({"nif:anchorOf"}=="τυγχανουσιν")then({"nif:lemma"}=="τυγχανω") | 7 | 0 | 1.0000 |
| ... | ... | ... | ... |

#### Aggregate text analysis

Let's end these examples with an aggregate analysis of the data set of all word forms with lemmas and morphological features. To find out if there is a prevalence in the text with respect to certain morphological features let's run the following simple rules:

```
if (({"olia:ProperNoun"}==1)) then ({"olia:Neuter"}==1)
if (({"olia:ProperNoun"}==1)) then ({"olia:Feminine"}==1)
if (({"olia:ProperNoun"}==1)) then ({"olia:Masculine"}==1)
```

These rules identify the grammatical gender of all the proper nouns in the text (word forms that start with a capital letter and name people, places, things, and ideas). Here are the results:

| rule\_definition | abs support | confidence |
| --- | --- | --- |
| if({"olia:ProperNoun"}==1)then({"olia:Masculine"}==1) | 3559 | 0.7928 |
| if({"olia:ProperNoun"}==1)then({"olia:Feminine"}==1) | 468 | 0.1043 |
| if({"olia:ProperNoun"}==1)then({"olia:Neuter"}==1) | 34 | 0.0076 |

Almost 80% of the proper nouns in the text have masculine gender, and just over 10% have feminine gender. Remember that this is derived from Plato's dialogues, so no surprise there. Most protagonists in the dialogues, if not all, are male and related word forms are therefore masculine. I specifically looked at the feminine proper nouns with more that five occurrences: they are geographical locations like Δῆλος (most frequent, 48 times), Συράκουσαι (7 times), Αίγυπτος (6 times). It also appeared that a number of male protagonists were incorrectly given annotations with feminine gender (Σιμμιας, 10 times and Μελητος, 6 times). Furthermore some word forms were mistakenly taken as pronoun, and that some pronouns did not have an annotation for gender (that is why it does not sum up).

### Conclusion

As you can see this all works quite well. If a word form has one meaning then it is fairly easy to create reliable patterns and find erroneous annotations from a NLP parser. The main problem that cannot be solved in this approach (by looking at word forms only) is that in ancient Greek a word form can have more than one meaning, and therefore different morphological features, depending on the specific context of the word form. For example the meaning of a word form also depends on the (features of) preceding and following words in the sentence. To take that into account a different approach for mining is necessary.

I wonder whether it is feasible to automatically correct the output of the NLP parser in case of high confidence or humanly verified morphological patterns. That would increase the accuracy of the annotations. Furthermore, if the association rules are used for prediction then it might perhaps even be possible to construct a complete rules-based annotation model, and thereby replacing the deep learning model with a transparent rules-based approach.

So it must first be possible to create more complex rules that take into account the context of the word forms. This could be achieved by querying the RDF-graph directly to mine for reliable triple associations and with that find erroneous triples and missing triples in the graph. To be continued.
