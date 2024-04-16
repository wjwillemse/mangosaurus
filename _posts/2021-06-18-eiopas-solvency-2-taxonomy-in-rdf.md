---
title: "EIOPA's Solvency 2 taxonomy in RDF"
date: "2021-06-18"
categories: ["linked-data"]
tags: ["eiopa", "rdf", "xbrl"]
---

To use the metadata from XBRL taxonomies, like labels, hierarchies, template structures and formulas, often licensed software is needed to process the taxonomy and convert the XML content to readable formats. In an earlier [blog](https://mangosaurus.eu/2020/10/28/converting-supervisory-reports-to-semantic-webs-from-xbrl-to-rdf) I have shown that it is useful to convert XBRL instance data to a linked data set in RDF and then query that data to retrieve the desired information. In this blog I will show how to do this with taxonomies: by using a number of small SPARQL queries the complete Data Point Model (DPM) of (European) XBRL taxonomies can be retrieved.

The main purpose of retrieving metadata in this manner is to be able to use taxonomy metadata in data science environments, for example to be able to apply machine learning models that use taxonomy metadata like hierarchies and to use concept and element labels from a taxonomy in NLP, for example Named Entity Recognition tasks to link quantitative reports to (unstructured, or: not yet structured) text data.

The lightweight solution that I show here is completely based on open source code, in the form of my [xbrl2rdf](https://pypi.org/project/xbrl2rdf/) package. This package converts XBRL instance files and all related taxonomy files (schemas and linkbases) to RDF and RDF-star and uses for this lxml and rdflib, and nothing else.

The examples below use the Solvency 2 taxonomy, but other taxonomies works as well. Gist with the notebook with the code below can be found [here](https://gist.github.com/wjwillemse/66437705f673f935cc84ff2c971a8a2d).

### Importing the data

With the xbrl2rdf-package I have converted the EIOPA instance example file for quarterly reports for solo undertaking (QRS) to RDF. All taxonomy concepts that are used in that instance are included in the RDF data set. This result can be read with rdflib into memory.

```python
# RDF graph loading
path = "../data/rdf/qrs_240_instance.ttl"

g = RDFGraph()
g.parse(path, format='turtle')

print("rdflib Graph loaded successfully with {} triples".format(len(g)))
```

This returns

```
rdflib Graph loaded successfully with 1203744 triples
```

So we have a RDF graph with 1.2 million triples that contains all data (facts related to concepts in the taxonomy, including all labels, validation rules, template structures, etc). The original RDF data file is around 64 Mb (combining instance and taxonomy triples). Reading and processing this file into an in-memory RDF graph takes some time, but then the graph can easily be queried.

### Extracting template URIs

Let's start with a simple query. Table or template URIs are subjects in the triple "subject xl:type table:table". To get a list with all templates of an instance (in this case the first five) we run

```python
q = """
  SELECT ?a
  WHERE {
    ?a xl:type table:table .
  }"""
tables = [str(row[0]) for row in g.query(q)]
tables.sort()
tables[0:5]
```

This returns a list of the URIs of the templates contained in the instance file.

```python
['http://eiopa.europa.eu/xbrl/s2md/fws/solvency/solvency2/2019-07-15/tab/S.01.01.02.01#s2md_tS.01.01.02.01',
 'http://eiopa.europa.eu/xbrl/s2md/fws/solvency/solvency2/2019-07-15/tab/S.01.02.01.01#s2md_tS.01.02.01.01',
 'http://eiopa.europa.eu/xbrl/s2md/fws/solvency/solvency2/2019-07-15/tab/S.02.01.02.01#s2md_tS.02.01.02.01',
 'http://eiopa.europa.eu/xbrl/s2md/fws/solvency/solvency2/2019-07-15/tab/S.05.01.02.01#s2md_tS.05.01.02.01',
 'http://eiopa.europa.eu/xbrl/s2md/fws/solvency/solvency2/2019-07-15/tab/S.05.01.02.02#s2md_tS.05.01.02.02']
```

### Extracting the explicit domains

Next, we extract the explicit domains and related data in the taxonomy. A domain is specific XBRL terminology and means a set of elements sharing a specified semantic nature. An explicit domain has its elements enumerated in the taxonomy and can be found with the subject in the triple 'subject rdf:type model:explicitDomainType'.

```python
q = """
  SELECT DISTINCT ?t ?x1 ?x2 ?x3 ?x4
  WHERE {
    ?t rdf:type model:explicitDomainType .
    ?t xbrli:periodType ?x1 .
    ?t model:creationDate ?x2 .
    ?t xbrli:nillable ?x3 .
    ?t xbrli:abstract ?x4 .
  }"""
```

The first five domains (of 41 in total) are

| index | Domain name | Domain label | period type | creation date | nillable | abstract |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | LB | Lines of businesses | instant | 2014-07-07 | true | true |
| 1 | MC | Main categories | instant | 2014-07-07 | true | true |
| 2 | TI | Time intervals | instant | 2014-07-07 | true | true |
| 3 | AO | Article 112 and 167 | instant | 2014-07-07 | true | true |
| 4 | CG | Collaterals/Guarantees | instant | 2014-07-07 | true | true |

So, the label of the domain LB is Lines of businesses; it has been there since the early versions of the taxonomy. If a domain is modified then this is also included as a triple in the data set.

### Extracting domain members

Elements of an explicit domain are called domain members. A domain member (or simply a member) is enumerated element of an explicit domain. All members from a domain share a certain common nature. To get the members of a domain, we define a function that finds all domain-members relations of a given domain and retrieve the label of the member. In SPARQL this is:

```python
def members(domain):
    q = """
      SELECT DISTINCT ?t ?label
      WHERE {
        ?l arcrole7:domain-member [ xl:from <"""+str(domain)+"""> ;
                                    xl:to ?t ] .
        ?t rdf:type nonnum:domainItemType .
        ?x arcrole3:concept-label [ xl:from ?t ;
                                    xl:to [rdf:value ?label ] ] .
        }"""
    return g.query(q)
```

All members of all domains can be retrieved by running this function for all domains defined earlier. Based on the output we create a Pandas DataFrame with the results.

```python
df_members = pd.DataFrame()
for d in df_domains.iloc[:, 0]:
    data = [[urldefrag(d)[1]]+[urldefrag(row[0])[1]]+list(row[1:]) for row in members(d)]
    columns = ['Domain',
               'Member',
               'Member label']
    df_members = df_members.append(pd.DataFrame(data=data,
                                                columns=columns))
```

In total there are 4.879 members of all domains (in this taxonomy).

The first five member of the domain LB (Lines of Businesses):

| index  | Domain | Member | Member label |
| --- | --- | --- | --- |
| 0 | LB | x0 | Total/NA |
| 1 | LB | x1 | Accident and sickness |
| 2 | LB | x2 | Motor |
| 3 | LB | x3 | Fire and other damage to property |
| 4 | LB | x4 | Aviation, marine and transport |

This allows us, for example, to retrieve all facts in a report that are related to the term 'motor' because a reported fact contains references to the domain members to which the fact relates.

### Extracting the template structure

Template structures are stored in the taxonomy as a tree of linked elements along the axes x, y and, if applicable, z. The elements have a label and a row-column code (this holds at least for EIOPA and DNB taxonomies), and have a certain depth, i.e. they can be subcategories of other elements. For example in the Solvency 2 balance sheet template, the element 'Equities' has subcategories 'Equities - listed' and 'Equities unlisted'. I have not included the code here but with a few lines you can extract the complete template structures as they are stored in the taxonomy. For example for the balance sheet (S.02.01.02.01) we get:

The first 25 lines of the balance sheet template

|   | axis | depth | rc-code | label |
| --- | --- | --- | --- | --- |
| 1 | x | 1 | C0010 | Solvency II value |
| 3 | y | 1 |   | Assets |
| 4 | y | 1 |   | Liabilities |
| 5 | y | 2 | R0010 | Goodwill |
| 6 | y | 2 | R0020 | Deferred acquisition costs |
| 7 | y | 2 | R0030 | Intangible assets |
| 8 | y | 2 | R0040 | Deferred tax assets |
| 9 | y | 2 | R0050 | Pension benefit surplus |
| 10 | y | 2 | R0060 | Property, plant & equipment held for own use |
| 11 | y | 2 | R0070 | Investments (other than assets held for index-... |
| 12 | y | 3 | R0080 | Property (other than for own use) |
| 13 | y | 3 | R0090 | Holdings in related undertakings, including pa... |
| 14 | y | 3 | R0100 | Equities |
| 15 | y | 4 | R0110 | Equities - listed |
| 16 | y | 4 | R0120 | Equities - unlisted |
| 17 | y | 3 | R0130 | Bonds |
| 18 | y | 4 | R0140 | Government Bonds |
| 19 | y | 4 | R0150 | Corporate Bonds |
| 20 | y | 4 | R0160 | Structured notes |
| 21 | y | 4 | R0170 | Collateralised securities |
| 22 | y | 3 | R0180 | Collective Investments Undertakings |
| 23 | y | 3 | R0190 | Derivatives |
| 24 | y | 3 | R0200 | Deposits other than cash equivalents |

With relatively small SPARQL queries, it is possible to retrieve metadata from XBRL taxonomies. This works well because we have converted the original taxonomy (in xml) to the linked data format RDF; and this format is especially well suited for representing and querying XBRL data.

The examples above show that it is possible to retrieve the complete Solvency 2 Data Point Model (and more) from the the taxonomy in RDF and make it available in a Python environment. This allows incorporation of metadata in machine learning models and in NLP applications. I hope that this approach will allow more data scientists to use existing metadata from XBRL taxonomies.
