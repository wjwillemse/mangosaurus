---
title: "Converting XBRL to RDF-star"
date: "2021-03-21"
categories: ["linked-data"]
tags: ["rdf", "rdf-star", "xbrl"]
---

Lately I have been working on the conversion of XBRL instances and related taxonomy schemas and linkbases to RDF and RDF-star. In these semantic data formats, you can link data in XBRL data with other data sources and you can query the data in a fairly easy manner. RDF-star is an extension of RDF that in some situations allows a more compact description of linked data, and by that it narrows the gap between RDF and property graphs. How this works, I will show in this blog using the XBRL taxonomy definitions as an example.

In a previous [blog](https://mangosaurus.eu/2020/10/28/converting-supervisory-reports-to-semantic-webs-from-xbrl-to-rdf/) I showed that XBRL instance facts can be converted to RDF and visualized as a network. The same can be done with the related taxonomy elements. An XBRL taxonomy consists of concepts and relations between concepts that define calculations, presentations, labels and definitions. The concepts are laid down (mostly) in XML schemas and the relations in linkbases using XML schemas and XLinks. By converting the XBRL taxonomy to RDF, the XBRL fact data is linked to its corresponding metadata in the taxonomy.

### XBRL to RDF

There has been done some work on the conversion of XBRL to RDF, most notably by Dave Raggett. His project [xbrlimport](https://sourceforge.net/projects/xbrlimport/), written in C++ and available on SourceForge, converts XBRL data to RDF triples. His approach is clean and straightforward and reuses the original namespaces of the XBRL data (with some obvious elements translated to predicates with RDF namespaces).

I used Raggett's xbrlimport as a starting point, translated it to Python, added XBRL items that were introduced after publication of the code and improved a number of things. The code is now for example able to convert all EIOPA's Solvency 2 taxonomy elements with all metadata available to RDF format. This code is available under the same license as xbrlimport (GNU General Public License) as a Python Package on [pypi.org](https://pypi.org/project/xbrl2rdf/). You can take an XBRL instance with corresponding taxonomy (in the form of a zip-file) and convert the contents to RDF and RDF-star. This code will look up any references (URIs) in the XBRL instance to the taxonomy in the zip-file and convert the relevant files to RDF.

Let's look at some examples of the Solvency 2 taxonomy converted to RDF. The RDF triple of an arbitrary XBRL concept from the Solvency 2 taxonomy looks like this (in turtle format):

```rdf-turtle
s2md_met:mi362 
    rdf:type xbrli:monetaryItemType ;
    xbrli:periodType """instant"""^^rdf:XMLLiteral ;
    model:creationDate """2014-07-07"""^^xsd:dateTime ;
    xbrli:substitutionGroup xbrli:item ;
    xbrli:nillable "true"^^xsd:boolean ;
```

This example describes the triples of concept s2md\_met:mi362 (a Solvency 2 metric). With these triples we have exactly the same data as in the related XML file but now in the form of triples. Namespaces are derived from the XML file (except rdf:type) and datatypes are transformed to RDF datatypes with proper RDF syntax.

This can be done with all concepts used to which the facts of an XBRL instance refer. If you have facts in RDF format, then in RDF these concept are automatically linked with the concepts in the taxonomy because the URIs of the concepts are the same. This creates a network of facts with all related metadata of the facts.

An XBRL taxonomy also contains links that relate concepts to each other for several purposes (to provide labels, definitions, presentations and calculations) . An example of a link is the following.

```rdf-turtle
_:link2 arcrole:concept-label [
    xl:type xl:link ;
    xl:role role2:link ;
    xl:from s2md_met:mi362 ;
    xl:to s2md_met:label_s2md_mi362 ;
    ] .
```

The link relates concept mi362 with label mi362 by creating a new subject \_:link2 with predicate arcrole:concept-label and an object which contains all data about the link (including the xl:from and xl:to and the attributes of the link). This way of introducing a new subject to specify a link between two concepts is called reification and a bit artificial because you would like to link the concept directly with the label, such as

```rdf-turtle
s2md_met:mi281 arcrole:concept-label s2md_met:label_s2md_mi281
```

However, then you are unable in RDF to link the attributes (like the order and the role) to the predicates. It is one of the disadvantages of the current RDF format. There appears to be no easy way to do this in RDF, other than by using this artificial reification approach (some other solutions exist like the singleton property approach, but all of them have disadvantages.)

### The new RDF-star format

Recently, the [RDF-star working group](https://w3c.github.io/rdf-star/) published their first [Draft Community Report](https://w3c.github.io/rdf-star/cg-spec/2021-02-18.html). In this report they introduced new RDF-star and SPARQL-star specifications. These new specifications, although not yet a W3C standard, enable more compact specification of linked datasets and simpler graphs and less nodes.

Let's look what this means for the XBRL linkbases with the following example. Suppose we have the following link definition.

```rdf-turtle
_:link1 arcrole:breakdown-tree [
    xl:from _:s2md_a1 ;
    xl:to _:s2md_a1.root ;
    xl:type xl:link ;
    xl:role tab:S.01.01.02.01 ;
    xl:order "0"^^xsd:decimal ;
    ] .
```

The subject in this case is \_:link1 with predicate arcrole:breakdown-tree, so this link describes a part of a table template. It points to a subject with all the information of the link, i.e. from, to, type, role and order from the xl namespace. Note that there is no triple with \_:s2md\_a1 (xl:from) as a subject and \_:s2md\_a1.root (xl:to) as an object. So if you want to know the relations of the concept \_:s2md\_a1 you need to look at the link triples and look for entries where xl:from equals the concept.

With the new RDF-star specifications you can just add the triple and then add properties to the triple as a whole, so the example would read

```rdf-turtle
_:s2md_a1 arcrole:breakdown-tree _:s2md_a1.root .
<<_:s2md_a1 arcrole:breakdown-tree _:s2md_a1.root>> 
    xl:role tab:S.01.01.02.01 ;
    xl:order "0"^^xsd:decimal ;
    .
```

Which is basically what we need to define. If you now want to know the relations of the subject \_:s2md\_a1 then you just look for triples with this subject. In the visual presentation of the RDF dataset you will see a direct link between the two concepts. This new RDF format also implies simplifications of the SPARQL queries.

This blog has become a bit technical but I hope you see that the RDF-star specification allows a much needed simplification of RDF triples. I showed that the conversion of XBRL taxonomies to RDF-star leads to a smaller amount of triples and also to less complex triples. The resulting taxonomy triples lead to less complex graphs and can be used to derive the XBRL labels, template structures, validation rules and definitions, just by using SPARQL queries.
