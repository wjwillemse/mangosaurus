---
title: "The Solvency termbase for NLP"
date: "2022-02-09"
categories: ["NLP"]
tags: ["solvency", "termbase"]
---

This blog describes a way to construct a terminology database for the insurance supervision knowledge domain. The goal of this termbase is provide a reliable basis to extract insurance supervision terminology within different NLP analyses.

The terminology of solvency and insurance supervision forms an expert domain of terminology based on economics, mathematics, accounting and finance terminologies. Like probably many other knowledge domains the terminology used is very technical and specific. Some terms are only used within this domain with only a limited number of occurrences (which often hinders the use of statistical methods for finding terms). And many other words have general meanings outside the domain that do not coincide with the specific meanings within the domain. Translation of terms from this specific domain often requires extensive knowledge about the meaning and use of these terms.

### What is a termbase?

A termbase is a database containing terminology and related information. It consists of concepts with their verbal designations (terms, i.e. single words or composed of multi-word strings) of a specific knowledge domain, often in different languages. It contains the full form of concepts, but also abbreviations, synonyms and variants and additional information of concepts, such as definitions and external references. To indicate the accuracy or completeness often a reliability code is added to individual terms of a concept. A proper termbase is an important terminology tool to achieve standardization of information and consistent use of (translations) of concepts in documents. And because of that, they are often used by professional translators.

The European Union translates legal documents in all member state languages and uses for this one common publicly available termbase: the [IATE](https://iate.europa.eu/home) (Interactive Terminology for Europe) terminology database. The IATE termbase is used in the EU institutions and agencies since 2004 for the collection, dissemination and management of EU-specific terminology. This helps to avoid divergences in the application of European Law within Europe (there exists a vast amount of literature on the effects on language ambiguity in European legislation). The translations of European legislation are therefore of the highest quality with strong consistency between different directives, regulations, delegated and implementing acts and so on. This termbase is very useful for information extraction from documents and for linking terminology concepts between different documents. They can be extended with abbreviations, synonyms and common variants of terms.

Termbases is very useful for information extraction from documents and for linking terminology concepts between different documents. They can be extended with abbreviations, synonyms and common variants of terms.

### The Solvency termbase for NLP

To create a first Solvency termbase for NLP purposes, I extracted terms from Solvency 2 Delegated Acts in a number of languages, looked up these terms in the IATE database and copied the corresponding concepts. It often happens that for one language the same term refers to different concepts (for example, the term 'balance' means something different in chemistry and in accounting). But if for one legal document the terms from different languages refer to the same concept, then we probably have the right concept (that was used in the translation of the legal document). So, the more references from the same legal document, the more reliable the term-concept relation is. And if we have the proper term-concept relationship, we automatically have all reliable translations of that concept.

Term extraction was done with part-of-speech patterns (such as adj-noun and adj-noun-noun patterns). To do this, for every language the Delegated Acts was converted to the NLP Annotation Format (NAF). The functionality for conversion to NAF and for extracting terms based on pos patterns is part of the [nafigator package](https://github.com/DeNederlandscheBank/nafigator). As an NLP engine for nafigator, I used the Stanford Stanza package that contains tokenizers and part-of-speech models for every European language. The termbase itself was made with the [terminator repository](https://github.com/DeNederlandscheBank/terminator) (currently under construction).

For terms in Dutch, I also added to the termbase additional part-of-speech tags, lemma's and morphological properties from the Lassy Klein-corpus from the [Instituut voor de Nederlandse taal](https://taalmaterialen.ivdnt.org/download/lassy-klein-corpus6/) (Dutch Language Institute). This data set consists of approximately 1 million words with manually verified syntactic annotations. I expanded this data set with solvency related words. Linguistical properties of terms of other languages can be added it a reliable data set is available.

Below, you see one concept from the resulting termbase (the concept of which 'solvency capital requirement' is the English term) in [TermBase eXchange format](https://www.tbxinfo.net/) (TBX). This is an international standard (ISO 30042:2019) for the representation of structured concept-oriented terminological data, based on xml.

```xml
<conceptEntry id="249">
 <descrip type="subjectField">insurance</descrip>
 <xref>IATE_2246604</xref>
 <ref>https://iate.europa.eu/entry/result/2246604/en</ref>
 <langSec xml:lang="nl">
  <termSec>
   <term>solvabiliteitskapitaalvereiste</term>
   <termNote type="partOfSpeech">noun</termNote>
   <note>source: ../naf-data/data/legislation/Solvency II Delegated Acts - NL.txt (#hits=331)</note>
   <termNote type="termType">fullForm</termNote>
   <descrip type="reliabilityCode">9</descrip>
   <termNote type="lemma">solvabiliteits_kapitaalvereiste</termNote>
   <termNote type="grammaticalNumber">singular</termNote>
   <termNoteGrp>
    <termNote type="component">solvabiliteits-</termNote>
    <termNote type="component">kapitaal-</termNote>
    <termNote type="component">vereiste</termNote>
   </termNoteGrp>
  </termSec>
 </langSec>
 <langSec xml:lang="en">
  <termSec>
   <term>SCR</term>
   <termNote type="termType">abbreviation</termNote>
   <descrip type="reliabilityCode">9</descrip>
  </termSec>
  <termSec>
   <term>solvency capital requirement</term>
   <termNote type="termType">fullForm</termNote>
   <descrip type="reliabilityCode">9</descrip>
   <termNote type="partOfSpeech">noun, noun, noun</termNote>
   <note>source: ../naf-data/data/legislation/Solvency II Delegated Acts - EN.txt (#hits=266)</note>
  </termSec>
 </langSec>
 <langSec xml:lang="fr">
  <termSec>
   <term>capital de solvabilité requis</term>
   <termNote type="termType">fullForm</termNote>
   <descrip type="reliabilityCode">9</descrip>
   <termNote type="partOfSpeech">noun, adp, noun, adj</termNote>
   <note>source: ../naf-data/data/legislation/Solvency II Delegated Acts - FR.txt (#hits=198)</note>
  </termSec>
  <termSec>
   <term>CSR</term>
   <termNote type="termType">abbreviation</termNote>
   <descrip type="reliabilityCode">9</descrip>
  </termSec>
 </langSec>
</conceptEntry>
```

You see that the concept contains a link to the IATE database entry with the definition of the concept (the link in this blog actually works so you can try it out). Then a number of language sections contain terms of this concept for different languages. The English section contains the term SCR as an English abbreviation of this concept (the French section contains the abbreviation CSR for the same concept). For every term the part-of-speech tags were added (which are not part of the IATE database) and, for Dutch only, with the lemma and grammatical number of the term and its word components. These additional linguistical attributes allow easier use within NLP analyses. Furthermore, as a note the number of all occurrences in the original legal document are included.

The concept entry contains related terms in all European languages. In Greek the SCR is κεφαλαιακή απαίτηση φερεγγυότητας, in Irish it is 'ceanglas maidir le caipiteal sócmhainneachta' (although the Solvency 2 Delegated Acts is not available in the Irish language), in Portuguese it is 'requisito de capital de solvência', in Estonian 'solventsuskapitalinõue', and so on. These are reliable translations as they are used in legal documents of that language.

The termbase contains all terms from the Solvency 2 Delegated Acts that can be found in the IATE database. In addition, terms that were not found in that database are added with the termNote "NewTerm", to indicate that this term has yet to be reviewed by a knowledge domain expert. This would also be the way to add synonyms and variants of terms.

The Solvency termbase basically allows to scan for a Solvency 2 concept in a document in any of the 23 European languages (given that it is in the IATE database). This is of course an initial approach to construct a termbase to test whether it is feasible and practical. The terminology that insurance undertakings use in their solvency reports is very likely to differ from the one used in legal documents. I will be testing this with a number of documents to identify Solvency 2 terminology to get an idea of how many synonyms and variants are missing.

Besides this Solvency termbase, it is in the same way possible to construct a Climate termbase based on the European Climate Law (a European regulation from 2021). This law contains a large number of climate-related terminology and is available in all European languages. A Climate termbase gives the possibility to extract climate-related information from all kinds of documents. Furthermore, we have the Sustainable Finance Disclosure Regulation (a European regulation also from 2021) for environmental, social, and governance (ESG) terminology, which could provide a starting point for an ESG termbase. And of course I eagerly await the European Regulation on Artificial Intelligence.
