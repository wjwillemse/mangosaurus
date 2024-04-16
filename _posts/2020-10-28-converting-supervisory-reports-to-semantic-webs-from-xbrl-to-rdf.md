---
title: "Converting supervisory reports to Semantic Webs: from XBRL to RDF"
date: "2020-10-28"
categories: ["linked-data"]
tags: ["rdf", "xbrl"]
---

A growing number of supervisory reports across Europe are based on the XML Extensible Business Reporting Language standard (XBRL). Financial entities such as banks, insurance undertakings and pension institutions are required to submit their reports to their supervisors in this format.

XBRL is a language for modeling, exchanging and automatically processing business and financial information. Reports in this format (called instance documents) are based on metadata (set out in taxonomies) that add semantic meaning to the data points that are reported. You can choose different implementations but overall an XBRL taxonomy provides a semantically rich data model and that has always been one of the main advantages of XBRL.

However, in its raw format (an XML file) each report is basically a machine readable document with a tree structure that does not enable easy integration with related data from other sources or integration with text documents and their contents.

In this blog, I will show that converting the XBRL reports to another format allows easier integration and understanding. That other format is based on Semantic Webs. It has been shown that XBRL converted to Semantic Webs can be done without any loss of information (see for example this [article](http://events.linkeddata.org/ldow2009/papers/ldow2009_paper6.pdf)). So if we convert the XBRL format to a Semantic Web then we keep the structure and the meaning provided by the taxonomy. The result is basically a graph and this format enables integration with other linked data that is much easier.

A Semantic Web consists of formats and technologies that are rather old (from a computer science perspective): it originated around the same time as XBRL, some twenty years ago. And because it tried to solve similar problems (lack of semantic meaning in the World Wide Web) as the XBRL standard (lack of semantic meaning in business and financial data), to some extent it is based on similar concepts. It was however developed completely separate from XBRL.

The general concept of a Semantic Webs, where data is linked together to provide semantic meaning, is also known as a knowledge graph.

How does a Semantic Web work? One of the formats of the Semantic Web is the Resource Description Framework (RDF), originally designed as a metadata data model. RDF was adopted as a World Wide Web Consortium recommendation in 1999. The RDF 1.0 specification was published in 2004, and RDF 1.1 followed in 2014.

The RDF format is based on expressions in the form of subject-predicate-object, called triples. The subject and object denote (web) resources and the predicate denotes the relationship between the subject and the object. For example the expression 'Spinoza has written the book Ethica Ordine Geometrico Demonstrata' in RDF is a triple with a subject denoting "Spinoza", a predicate denoting "has written", and an object denoting "the book Ethica Ordine Geometrico Demonstrata". This is a different approach than for example object-oriented models with an entity (Spinoza), attribute (book) and value (Ethica).

The RDF format could potentially solve some problems with the XBRL format. To explain this, I converted an XBRL-instance (a test instance file from EIOPA for Solvency 2) to RDF format.

Below you see the representation of one arbitrary data point in the report (called a fact) in RDF format and visualized as a network (I used the Python package networkx). The predicates contain the complete web resource so I limited the name to the last word to make it readable.

![](images/index-1.png)

The red node is the starting point of the data point. The red labels on the lines describe the predicate between subject and object. You see that the fact (subject) 'has decimals' (predicate) 2 (object), and furthermore has unit EUR, has value 838522076.03, has type metmi503 (an internal code describing _Payments for reported but not settled claims_) and some other properties.

The data point also has a so-called context that defines the entity to which the fact applies, the period of time the fact is relevant (in this case 2019-12-31) and also a scenario, which consists of additional metadata of the data point. In this case we see that the data point is related to _statutory accounts_, _non-life and health non-STL_, _direct business_ and _accepted during the period_ (and a node without a label).

All facts in every XBRL instance are structured in this way, which means that for example you can search all facts with the label _statutory accounts_. Furthermore, because XBRL uses namespaces you can unambiguously identify predicates and objects in the report. For example, you see that the entity node has an identifier (starting with 0LFF1...) and a scheme (17442). The scheme refers to the web resource for the ISO standard 17442 which specifies the Legal Entity Identifier (LEI), so the entity is unambiguously identified with the given (LEI-)code. If you add other XBRL instances with references to that entity then the data is automatically linked because other instances will contain exactly the same entity node.

The RDF representation of the XBRL fact above is:

```rdf-turtle
_:provenance1 xl:instance "filename".
_:unit_u xbrli:measure iso4217:EUR.
_:fact926
  xl:provenance :provenance1;
  xl:type xbrli:fact; 
  rdf:type s2md_met:mi503;
  rdf:value "838522076.03"^^xsd:decimal;
  xbrli:decimals "2"^^xsd:integer;
  xbrli:unit :unit_u; 
  xbrli:context :context_BLx79_DIx5_IZx1_TBx28_VGx84.
_:context_BLx79_DIx5_IZx1_TBx28_VGx84
  xl:type xbrli:context;
  xbrli:entity [
    xbrli:identifier "0LFF1WMNTWG5PTIYYI38";
    xbrli:scheme http://standards.iso.org/iso/17442;
    ];
  xbrli:scenario [
    xbrldi:explicitMember "s2c_LB:x79"^^rdf:XMLLiteral;
    xbrldi:explicitMember "s2c_DI:x5"^^rdf:XMLLiteral;
    xbrldi:explicitMember "s2c_RT:x1"^^rdf:XMLLiteral;
    xbrldi:explicitMember "s2c_LB:x28"^^rdf:XMLLiteral;
    xbrldi:explicitMember "s2c_AM:x84"^^rdf:XMLLiteral;
    ];
  xbrli:instant "2019-12-31"^^xsd:date.
```

Instead of storing the data in separate templates with often unclear code names you can also convert the XBRL data to one large Semantic Web where all facts are linked together. The RDF format thus provides a graph model which allows easier integration and visualization (and, for me at least, easier understanding). It allows adding and linking data from other sources, such as Solvency 2 documents and external data, in the same graph.

Typically, supervisory reports consists of thousands of data points and supervisors receive reports from many entities each period. How would you store that information? I think that the natural way to store an XBRL instance is not a relational database but a graph database (like graphDB or Neo4j). These databases can store the facts with all the metadata in a structured way and enable to query the graph efficiently. Next blog, I will explore graph databases and query languages for XBRL reports converted to the RDF format.
