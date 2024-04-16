---
title: "Europe's insurance register linked to the GLEIF RDF dataset"
date: "2022-01-08"
categories: ["linked-data"]
tags: ["eiopa", "gleif", "rdf"]
---

Number 7 of my New Year's Resolutions list reads "only use and provide linked data". So, to start the year well, I decided to do a little experiment to combine insurance undertakings register data with publicly available legal entity data. In concrete terms, I wanted to provide the European insurance register published by EIOPA (containing all licensed insurance undertakings in Europe) as linked data with the Legal Entity data from the Global Legal Entity Identifier Foundation (GLEIF). In this blog I will describe what I have done to do so.

### The GLEIF data

The GLEIF data consists of information on all legal entities in the world (entities with a [Legal Entity Identifier](https://www.gleif.org/en/about-lei/introducing-the-legal-entity-identifier-lei)). A LEI is required by any legal entity who is involved with financial transactions or operates within the financial system. If an organization needs a LEI then it requests one at a local registration agent. For the Netherlands these are the Authority for the Financial Markets (AFM), the Chamber of Commerce (KvK) and the tax authority and others. GLEIF receives data from these agents in each country and makes the collected LEI data available in a number of forms (api, csv, etc).

The really cool thing is that in 2019, together with data.world, GLEIF developed an RDFS/OWL Ontology for Legal Entities, and began in 2020 to publish regularly the LEI data as a linked [RDF](https://www.w3.org/RDF/) dataset on data.world (see [https://data.world/gleif](https://data.world/gleif), you need a (free) account to obtain the data). At the time of this writing, the size of the level 1 data (specifying who is who) is around 10.2 Gb with almost 92 million triples (subject-predicate-object), containing information about entity name, legal form, headquarter and legal address, geographical location, etc. Also related data such as who owns whom is published in this forms.

### The EIOPA insurance register

The European Supervisory Authority EIOPA publishes the Register of Insurance undertakings based on information provided by the National Competent Authorities (NCAs). The NCA in each member state is responsible for authorization and registration of the insurance undertakings activities. EIOPA collects the data in the national registers and publishes an European insurance register, which includes more than 3.200 domestic insurance undertakings. The register contains entity data like international and commercial name, name of NCA, addresses, cross border status, registration dates etc. Every insurance undertaking requires a LEI and the LEI is included in the register; this enables us to link the data easily to the GLEIF data.

The EIOPA insurance register is available as CSV and Excel file, without formal naming and clear definitions of column names. Linking the register data with other sources is a tedious job, because it must be done by hand. Take for example the LEI data in the register, which is referred to with the column name 'LEI'; this is perfectly understandable for humans, but for computers this is just a string of characters. Now that the GLEIF has published its ontologies there is a proper way to refer to a LEI, and that is with the Uniform Resource Identifier (URI) _https://www.gleif.org/ontology/L1/LEI_, or in a short form _gleif-L1:LEI_.

The idea is to publish the data in the European insurance register in the same manner; as linked data in RDF format using, where applicable, the GLEIF ontology for legal entities and creating an EIOPA ontology for the data that is unique for the insurance register. This allows users of the data to incorporate the insurance register data into the GLEIF RDF dataset and thereby extending the data available on the legal entities with the data from the insurance register.

### Creating triples from the EIOPA register

To convert the EIOPA insurance register to linked data in RDF format, I did the following:

- extract from the GLEIF RDF level 1 dataset on data.world all insurance undertakings and related data, based on the LEI in the EIOPA register;

- create a provisional ontology with URIs based on _https://www.eiopa.europe.eu/ontology/base_ (this should ideally be done by EIOPA, as they own the domain name in the URI);

- transform, with this ontology, the data in the EIOPA register to triples (omitting all data from the EIOPA register that is already included in the GLEIF RDF dataset, like names and addresses);

- publish the triples for the insurance register in RDF Turtle format on [data.world](https://data.world/wjwillemse/european-insurance-register).

Because I used the GLEIF ontology where applicable, the triples I created are automatically linked to the relevant data in the GLEIF dataset. Combining the insurance register dataset with the GLEIF RDF dataset results in a set where you have all the GLEIF level 1 data and all data in the EIOPA insurance register combined for all European insurance undertakings.

### Querying the data

Let's look what we have in this combined dataset. Querying the data in RDF is done with the SPARQL language. Here is an example to return the data on Achmea Schadeverzekeringen.

```sql
SELECT DISTINCT ?p ?o
WHERE
{ ?s gleif-L1:hasLegalName "Achmea schadeverzekeringen N.V." . 
  ?s ?p ?o .}
```

The query looks for triples where the predicate is _gleif-base:hasLegalName_ and the object is _Achmea Schadeverzekeringen N.V._ and returns all data of the subject that satisfies this constraint. This returns (where I omitted the prefix of the objects):

```rdf-turtle
gleif-L1-data:L-72450067SU8C745IAV11
    rdf#type                             LegalEntity
    gleif-base:hasLegalJurisdiction      NL  
    gleif-base:hasEntityStatus           EntityStatusActive  
    gleif-l1:hasLegalName                Achmea Schadeverzekeringen     
                                         N.V.
    gleif-l1:hasLegalForm                ELF-B5PM
    gleif-L1:hasHeadquartersAddress      L-72450067SU8C745IAV11-LAL  
    gleif-L1:hasLegalAddress             L-72450067SU8C745IAV11-LAL  
    gleif-base:hasRegistrationIdentifier BID-RA000463-08053410
    rdf#type                             InsuranceUndertaking
    eiopa-base:hasRegisterIdentifier     IURI-De-Nederlandsche-Bank-
                                         W1686
```

We see that the _rdf#type_ of this entity is LegalEntity (from the GLEIF data) and the jurisdiction is NL (this has a prefix that refers to the [ISO 3166-1](https://en.wikipedia.org/wiki/ISO_3166-1) country codes). The legal form refers to another subject called ELF-B5PM. The headquarters and legal address both refer to the same subject that contains the address data of this entity. Then there is a business identifier to the registration data. The last two lines are added by me: a triple to specify that this subject is not only a LegalEntity but also an InsuranceUndertaking (defined in the ontology), and a triple for the Insurance Undertaking Register Identifier (IURI) of this subject (also defined in the ontology).

Let's look more closely at the references in this list. First the legal form of Achmea (i.e. the predicate and objects of legal form code ELF-B5PM). Included in the GLEIF data is the following (again omitting the prefix of the object):

```
rdf#type                         EntityLegalForm  
rdf#type                         EntityLegalFormIdentifier  
gleif-base:identifies            ELF-B5PM  
gleif-base:tag                   B5PM  
gleif-base:hasCoverageArea       NL  
gleif-base:hasNameTransliterated naamloze vennootschap  
gleif-base:hasNameLocal          naamloze vennootschap  
gleif-base:hasAbbreviationLocal  NV, N.V., n.v., nv
```

With the GLEIF data we have this data on all legal entity forms of insurance undertakings in Europe. The local abbreviations are particularly handy as they help us to link an entity's name extracted from documents or other data sources with its corresponding LEI.

If we look more closely at the EIOPA Register Identifier _IURI-De-Nederlandsche-Bank-W1686_ then we find the register data of this Achmea entity:

```
owl:a                         InsuranceUndertakingRegisterIdentifier
gleif-base:identifies         L-72450067SU8C745IAV11
eiopa-base:hasNCA                          De Nederlandsche Bank  
eiopa-base:hasInsuranceUndertakingID       W1686  
eiopa-base:hasEUCountryWhereEntityOperates NL  
eiopa-base:hasCrossBorderStatus            DomesticUndertaking  
eiopa-base:hasRegistrationStartDate        23/12/1991 01:00:00  
eiopa-base:hasRegistrationEndDate          None  
eiopa-base:hasOperationStartDate           23/12/1991 01:00:00  
eiopa-base:hasOperationEndDate             None
```

The predicate _gleif-base:identifies_ refers back to the subject where _gleif-L1:hasLegalName_ equals the Achmea entity. The other predicates are based on the provisional ontology I made that contains the definitions of the attributes of the EIOPA insurance register. Here we see for example that _W1686_ is the identifier of this entity in DNB's insurance register.

Let me give a tiny example of the advantage of using linked data. The GLEIF data contains the geographical location of all legal entities. With the combined dataset it is easy to obtain the location for the insurance undertakings in, for example, the Netherlands. This query returns entity names with latitude and longitude of the legal address of the entity.

```sql
SELECT DISTINCT ?name ?lat ?long
WHERE {?sub rdf:type eiopa-base:InsuranceUndertaking ;
            gleif-base:hasLegalJurisdiction CountryCodes:NL ;
            gleif-L1:hasLegalName ?name ;
            gleif-L1:hasLegalAddress/gleif-base:hasCity ?city .
       ?geo gleif-base:hasCity ?city ;
            geo:lat ?lat ; 
            geo:long ?long .}
```

This result can be plotted on a map, see the link below. If you click on one of the dots then the name of the insurance undertaking will appear.

[eiopa\_register\_nl-1](https://www.mangosaurus.nl/wp-content/uploads/2023/10/eiopa_register_nl-1.html)[Download](https://www.mangosaurus.nl/wp-content/uploads/2023/10/eiopa_register_nl-1.html)

All queries above and the code to make the map are included in the notebook [EIOPA Register RDF datase - SPARQL queries](https://gist.github.com/wjwillemse/36d1ec4b67710715f866e1492086239e).

The provisional ontology I created is not yet semantically correct and should be improved, for example by incorporating data on NCAs and providing formal definitions. And other data sources could be added, for example the level 2 dataset to identify insurance groups, and the ISIN to LEI relations that are published daily by GLEIF.

By introducing the RDFS/OWL ontologies, the Global LEI Foundation has set an example on how to publish (financial) entity data in an useful manner. The GLEIF RDF dataset reduces the time needed to link the data with other data sources significantly. I hope other organizations that publish financial entity data as part of their mandate will follow that example.
