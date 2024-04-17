---
title: "Multilingual termbases with metadata from reporting templates"
date: "2022-07-26"
categories: ["NLP"]
tags: ["termbases", "xbrl"]
---

Domain-specific termbases are of great importance to many domain-specific NLP-tasks. They enable identification and annotation of terms in documents in situations where often not enough text is available to use statistical approaches. And more importantly, they form a step towards extracting structured facts from unstructured text data.

This blog shows how to construct and use multilingual termbases to annotate text from supervisory documents in different European languages with references to relevant parts of (quantitative) supervisory templates. By linking qualitative data (text) to quantitative data (numbers) we connect initially unstructured text data to data that are often to a high degree structured and well-defined in data point models and taxonomies. I will do this by constructing termbases that contain terminology data combined with linguistic data and metadata from the supervisory templates from different financial sectors.

For terminology data I will start with the [IATE-database](https://iate.europa.eu/home). Most terminology that is used in European quantitative reporting templates is based on and derived from European legislation. Having multilingualism as one of its founding principles is, the EU publishes terminology in the IATE-database in all official European languages to provide consistent translation of terms in European legislation. The IATE-database is published in the form of a file in TBX-format (**TermBase eXchange). But termbases can also be in form of SKOS (Simple Knowledge Organization System, built upon the RDF-format). Both formats are data models that contain descriptions and properties of concepts and are to some extent interchangeable (see for example [here](https://hal.inria.fr/hal-02398820/document)).

For metadata on reporting templates I will use relevant XBRL Taxonomies in RDF-format (see [here](https://mangosaurus.eu/2021/06/18/eiopas-solvency-2-taxonomy-in-rdf/)). Normally, XBRL Taxonomy are developed specifically for a single sector and therefore covers to some extent the financial terminology used within that sector. XBRL Taxonomies contain metadata of all data point in the reporting templates. From a XBRL Taxonomy a Data Point Model can be derived (that is: the taxonomy contains all definitions) and is often published together with the taxonomy which is only computer readable.

For linguistic data I will use the Python NLP package of Stanford Stanza, which provide pretrained NLP-models for all official European languages (in order of becoming an official EU language: Dutch, French, German, Italian (1958), Danish, English (1973), Greek (1981), Portuguese, Spanish (1986), Finnish, Swedish (1995), Czech, Estonian, Hungarian, Latvian, Lithuanian, Maltese, Polish, Slovak, Slovenian (2004), Bulgarian, Irish, Romanian (2007) and Croatian (2013)).

So we add semantic and linguistic structure to a terminology database. The resulting data structure is sometimes called an ontology, a taxonomy or a vocabulary, but these terms have no clear distinctive definitions. And moreover, the XBRL-people use the term taxonomy to refer to a structure that contains concepts with properties for definitions, labels, calculations and (table) presentations. To some extent it contains structured metadata of data points (i.e. the semantics of the data). Because of that you can say that it corresponds to an ontology within a Linked Data context. On the other hand a taxonomy within a Linked Data context (and everywhere else if I might add) is basically a description of concepts with sub-class relationships (concepts with hierarchical information). In the remainder of this blog I will use the term termbase for the resulting structure with semantic, linguistic and terminological data combined.

## Constructing a termbase from IATE and XBRL

In a previous [blog](https://mangosaurus.eu/2022/02/09/the-solvency-termbase-for-nlp/) I have described how to set up a terminology database (**Termbase) specifically for insurance-related terms. Now I will add links from the concepts and terms in the IATE-database to the data point model of insurance reporting templates (thereby adding semantics to the termbase), and secondly I will add linguistic information at term-level like lemmas and part-of-speech tags to allow for easy usage in NLP-tasks. The TBX-format in which the IATE-database is published allows for storing references as well as linguistic data on term-level, so we can construct the termbase as a standalone file in TBX-format (another solution would be to add the terminology and linguistic information to the XBRL Taxonomy and use that as a basis).

The IATE-database currently contains almost 930.000 concepts and many of them have verbal expressions in multiple languages resulting in over 8.000.000 terms. A single (English) expression of a concept in the IATE-database looks like this.

```xml
<conceptEntry id="3539858">
  <descrip type="subjectField">insurance</descrip>
  <langSec xml:lang="en">
    <termSec>
      <term>basic own funds</term>
      <termNote type="termType">fullForm</termNote>
      <descrip type="reliabilityCode">9</descrip>
    </termSec>
  </langSec>
```

### Adding labels from the XBRL Taxonomy

For the termbase, we add every element in the XBRL Taxonomy that has a label (tables, concepts, elements, dimensions and members) to the termbase and we add an external cross reference to the template and the location in that template where the element is used (the row or column within the template). The TBX-format allows for fields called externalCrossReference which refer to a resource that is external to the terminology database. Then you get concept entries like this:

```xml
<conceptEntry id="http://eiopa.europa.eu/xbrl/s2md/fws/solvency/solvency2/2021-07-15/tab/s.22.01.01.01#s2md_c4071">
  <descrip type="xbrlTypes">element</descrip>
  <xref type="externalCrossReference">S.22.01.01.01,R0020</xref>
  <langSec xml:lang="en">
    <termSec>
      <term>Basic own funds</term>
      <termNote type="termType">fullForm</termNote>
    </termSec>
    <termSec>
      <term>R0020</term>
      <termNote type="termType">shortForm</termNote>
    </termSec>
  </langSec>
</conceptEntry>
```

This means that the termbase concept with the id "http://eiopa.europa.eu/xbrl/.../s.22.01.01.01#s2md\_c4071" has English labels "Basic own funds" (full form) and "R0020" (short form). These are the labels of row 0020 of template S.22.01.01.01, i.e. in the template definition you will see Basic own funds on row 0020.

The template and rc-codes where elements refer to were extracted using SPARQL-queries on the XBRl Taxonomy in RDF-format.

### Adding references between IATE- and XBRL-concepts

Now that we have added terms for labelled elements from the XBRL Taxonomy, the next step is to add cross references between the IATE-concepts and the XBRL-concepts. In the TBX-format the crossReference is a pointer to another related location, such as another entry or another term, in this case to a XBRL-concept. Below the references to the XBRL-concepts are added.

```xml
<conceptEntry id="3539858">
  <descrip type="subjectField">insurance</descrip>
  <langSec xml:lang="en">
    <termSec>
      <term>basic own funds</term>
      <termNote type="termType">fullForm</termNote>
      <descrip type="reliabilityCode">9</descrip>
    </termSec>
  </langSec>
  <ref type="crossReference">http://eiopa.europa.eu/xbrl/s2c/dict/dom/el#x7</descrip>
  <ref type="crossReference">http://eiopa.europa.eu/xbrl/s2md/fws/solvency/solvency2/2021-07-15/tab/s.22.01.22.01#s2md_c4121</descrip>
  <ref type="crossReference">http://eiopa.europa.eu/xbrl/s2md/fws/solvency/solvency2/2021-07-15/tab/s.22.01.04.01#s2md_c4094</descrip>
  <ref type="crossReference">http://eiopa.europa.eu/xbrl/s2md/fws/solvency/solvency2/2021-07-15/tab/s.22.01.01.01#s2md_c4071</descrip>
  ...
```

The IATE-concept with id 3539858 points to the domain item el#x7 (a domain in XBRL is a set of related items, in this case own funds items), and furthermore to (table) elements s.22.01.22.01#s2md\_c4121, s.22.01.04.01#s2md\_c4094 and s.22.01.01.01#s2md\_c4071. These all refer to a single row or a single column within a template and the last one is given as an example above. It is the row in the table S.22.01.01.01.01 with label 'Basic own funds'. IATE-concepts and XBRL-concepts are considered equal when their lowercase lemmas are the same (except for abbreviations).

For XBRL Taxonomy of Solvency 2 we find in this way 740 unique terms in the IATE-database that are identical to a XBRL-concept (in English), and 1.500 terms that occur in the labels of XBRL-concepts but are not identical, for example 'net best estimate' is a part of the label 'Net Best Estimate of Premium Provisions'. For now I focused on identifying identical terms. How to process long labels with more than one term is a remaining challenge ([this](http://ceur-ws.org/Vol-673/paper3.pdf) describes probably a useful approach).

### Adding part-of-speech tags and lemmas

To make the termbase applicable for NLP-tasks we need to add additional linguistic information, such as part-of-speech patterns and lemmas. This improves the annotation process later on. Premium as an adjective has another meaning than premium as a common noun. By matching the PoS-pattern we get better matches (i.e. less false positives). And also the lemmas of a term will find terms irrespective of whether they are in grammatical singular or plural form. This is the concept to which PoS-patterns and lemma are added:

```xml
<conceptEntry id="3539858">
  <descrip type="subjectField">insurance</descrip>
  <langSec xml:lang="en">
    <termSec>
      <term>basic own funds</term>
      <termNote type="termType">fullForm</termNote>
      <descrip type="reliabilityCode">9</descrip>
      <termNote type="termLemma">basic own fund</termNote>
      <termNote type="partOfSpeech">adj, adj, noun</termNote>
    </termSec>
  </langSec>
  <langSec xml:lang="it">
    <termSec>
      <term>fondi propri di base</term>
      <termNote type="termType">fullForm</termNote>
      <descrip type="reliabilityCode">9</descrip>
      <termNote type="termLemma">fondo proprio di base</termNote>
      <termNote type="partOfSpeech">noun, det, adp, noun</termNote>
    </termSec>
  </langSec>
  ...
```

## Annotating text in different languages

Because the termbase contains multilingual terms with references to templates and locations we are now able to annotate terms in documents in different European languages. Below you find some examples of what you can do with the termbase. I took an identical text from the Solvency 2 Delegated Acts in English, Finnish and Italian (the first part of article 52), converted the text to NAF and added annotations with the termbase (Nafigator has a function that processes a complete termbase and adds annotations to the NAF-file). This results in the following (using a visualizer from the spaCy package):

In English

Article 52 **Mortality risk (Term [S.26.03.01.01,R0100]**) stress The **mortality risk (Term [S.26.03.01.01,R0100]**) stress referred to in Article 77b(1)(f) of Directive 2009/138/EC shall be the more adverse of the 1. following two scenarios in terms of its impact on **basic own funds (Term [S.22.01.01.01,R0020]**): (a) an instantaneous permanent increase of 15 % in the mortality rates used for the calculation of the **best estimate (Term [S.02.01.01.01,R0540]**); (b) an instantaneous increase of 0.15 percentage points in the mortality rates (expressed as percentages) which are used in the calculation of **technical provisions (Term [S.22.01.01.01,R0010]**) to reflect the mortality experience in the following 12 months.

In Finnish

52 artikla **Kuolevuusriskiin (Term [S.26.03.01.01,R0100]**) liittyvä stressi Direktiivin 2009/138/EY 77 b artiklan 1 kohdan f alakohdassa tarkoitetun, **kuolevuusriskiin (Term [S.26.03.01.01,R0100]**) liittyvän stressin on 1. oltava seuraavista kahdesta skenaariosta se, jonka epäsuotuisa vaikutus **omaan perusvarallisuuteen (Term [S.22.01.01.01,R0020]**) on suurempi: a) välitön, pysyvä 15 %:n nousu **parhaan estimaatin (Term [S.02.01.01.01,R0540]**) laskennassa käytetyssä kuolevuudessa; b) välitön 0,15 prosenttiyksikön nousu prosentteina ilmaistussa kuolevuudessa, jota käytetään **vakuutusteknisen vastuuvelan (Term [S.22.01.01.01,R0010]**) laskennassa ilmentämään havaittua kuolevuutta seuraavien 12 kuukauden aikana. Sovellettaessa 1 kohtaa kuolevuuden nousua sovelletaan ainoastaan vakuutuksiin, joissa kuolevuuden nousu johtaa

In German

Artikel 52 **Sterblichkeitsrisikostress (Term [S.26.03.01.01,R0100]**) Der in Artikel 77b Absatz 1 Buchstabe f der Richtlinie 2009/138/EG genannte **Sterblichkeitsrisikostress (Term [S.26.03.01.01,R0100]**) ist das im 1. Hinblick auf die Auswirkungen auf die **Basiseigenmittel (Term [S.22.01.01.01,R0020]**) ungünstigere der beiden folgenden Szenarien: (a) plötzlicher dauerhafter Anstieg der bei der Berechnung des besten Schätzwerts zugrunde gelegten Sterblichkeitsraten um 15 %; (b) plötzlicher Anstieg der bei der Berechnung der **versicherungstechnischen Rückstellungen (Term [S.22.01.01.01,R0010]**) zugrunde gelegten Sterblichkeitsraten (ausgedrückt als Prozentsätze) um 0,15 Prozentpunkte, um die Sterblichkeit in den folgenden zwölf Monaten widerzuspiegeln.

In French

Article 52 Choc de **risque de mortalité (Term [S.26.03.01.01,R0100]**) Le choc de **risque de mortalité (Term [S.26.03.01.01,R0100]**) visé à l'article 77 ter, paragraphe 1, point f), de la directive 2009/138/CE 1. correspond au plus défavorable des deux scénarios suivants en termes d'impact sur les **fonds propres de base (Term [S.22.01.01.01,R0020]**): (a) une hausse permanente soudaine de 15 % des taux de mortalité utilisés pour le calcul de la **meilleure estimation (Term [S.02.01.01.01,R0540]**); (b) une hausse soudaine de 0,15 point de pourcentage des taux de mortalité (exprimés en pourcentage) qui sont utilisés dans le calcul des **provisions techniques (Term [S.22.01.01.01,R0010]**) pour refléter l'évolution de la mortalité au cours des 12 mois à venir.

In Italian

Articolo 52 Stress legato al **rischio di mortalità (Term [S.26.03.01.01,R0100]**) Lo stress legato al **rischio di mortalità (Term [S.26.03.01.01,R0100]**) di cui all'articolo 77 ter, paragrafo 1, lettera f), della direttiva 2009/138/CE è 1. il più sfavorevole dei due seguenti scenari in termini di impatto sui **fondi propri di base (Term [S.22.01.01.01,R0020]**): (a) un incremento permanente istantaneo del 15 % dei tassi di mortalità utilizzati per il calcolo della migliore stima; (b) un incremento istantaneo di 0,15 punti percentuali dei tassi di mortalità (espressi in percentuale) utilizzati nel calcolo delle **riserve tecniche (Term [S.22.01.01.01,R0010]**) per tener conto dei dati tratti dall'esperienza relativi alla mortalità nei 12 mesi successivi. Ai fini del paragrafo 1, l'incremento dei tassi di mortalità si applica soltanto alle polizze di assicurazione per le 2. quali tale incremento comporta un aumento delle **riserve tecniche (Term [S.22.01.01.01,R0010]**) tenendo conto di tutto quanto segue:

In the Italian text one reference is missing: the IATE-database does not yet contain an Italian translation for the English term best estimate. This happens because the IATE-database is far from complete. Not all terms are available in all languages, and the IATE-database probably does not contain all terminology from the reporting templates. And although the IATE-database is constantly updated, it might be necessary for certain use cases to add additional translations and concepts to the database.

This works for every XBRL Taxonomy, although every taxonomy has its own peculiarities (it's a "standard" as one of my IT-colleagues likes to say). Following the procedure described above, I made the following termbases for insurance undertakings, credit institutions and Dutch pension funds based on the taxonomy versions mentioned:

- Termbase of EIOPA Solvency 2, taxonomy version 2.6.0

- Termbase of EBA CRD IV, taxonomy version 3.2.1.0

- Termbase of DNB FTK, taxonomy version 2.3.0

These can be found on [data.world](https://data.world/wjwillemse/termbases).
