The Hypertopic protocol is to the Social Semantic Web what SPARQL is to the Formal Semantic Web. 
It is based on the Hypertopic model which is an alternative to RDF/RDFS.

In the Hypertopic model:

* attribute types are defined by users,
* knowledge models depends on a subjective modality.

In the Hypertopic protocol:

* updating models is considered as important as reading them,
* horizontal scaling is preferred to formal expressivity. 

To achieve that, we defined it as a "RESTful" protocol.

# Requirements & Design choices

This version of the protocol (v2) meets additional requirements:

* Features
  * Confrontation of data from different services
* Performance
  * Complex tasks with a fixed number of HTTP requests
  * Easy cache managing (logs could be enough)
  * Batch creations (imports)
* Quality & project management
  * Reuse services instead of code
  * v1 and v2 usable at the same time on the same data (for transition)
* Ease of coding
  * Atomic resources (no more partial updates needed)
  * No more ad-hoc URL parsing
  * Same parsing code for every type format
  * Implementable on SQL and noSQL datastore
  * Implementable with REST frameworks

To meet these requirements we had to opt for: 

* Non-located items

To get the description of an item, clients often need to mash-up attributes and topics stored or generated on different servers. 
We once implemented a "remote item" mechanism, however this handles as a special case something that will get more and more common 
(esp. with [Argos](https://github.com/Hypertopic/Argos/wiki) & [Cassandre](https://github.com/Hypertopic/Cassandre/wiki), 
[Argos](https://github.com/Hypertopic/Argos/wiki) & [Steatite](https://github.com/Hypertopic/Steatite/wiki)). 
Instead we will consider that every item is identified by a UUID which must be checked for links on any known server.
The same approach will be used for other objects, to optimize distribution and service reuse.

* Documentary items

There was no efficient way to do complex algorithms if the link between sources and fragments were handled by item-item. Moreover, most of algorithms were not tractable without being limited to a single corpus. Therefore we decided that what would be tagged would have "(corpus,source,fragment)" as its primary key (fragment can be equals to "none").

We considered standard or well-known URI schemes for [user-named contents](http://www.ietf.org/rfc/rfc4151.txt) such as corpora, [hash contents](http://en.wikipedia.org/wiki/Uniform_Resource_Name#Non-standard_usage) such as resources, and [plain text fragment identifiers](http://tools.ietf.org/html/rfc5147). But we chose the simplest way to refer them (UUIDs and coordinates).

![Data model](https://github.com/Hypertopic/Protocol/raw/master/class_diagram.png "UML class diagram")

Note: Item-Item will remain unspecified until we have more real cases for it.

# Objects

Objects can be created, read, updated and deleted through the standard [HTTP](http://tools.ietf.org/html/rfc2616) methods (POST, GET, PUT, and DELETE respectively).

Their payloads complies with the [JSON](http://tools.ietf.org/html/rfc4627) standard format.

On creation, objects are given an "_id" by the service. If it handles conflict detection, they are given a "_rev" on each write action.

**TODO**: explain _rev and _id handling.

**TODO**: explain _changes handling.

Please note that, in order to allow specialized processing and to reduce network latency, the normal way to read an object is through a view. The only time to GET an object is before modifying it.

**TODO**: explain the data model (esp. corpus, items, highlights)

## Corpus

A corpus MUST have a ``corpus_name`` (which can be empty), and MAY have ``users``.

```javascript
{
  "corpus_name": "N1", 
  "users": ["U1", "U2"]
}
```

## Item

An item MUST have an ``item_corpus``. It MAY have an ``item_name``, a ``resource``, ``topics`` (with a ``viewpoint``) and ``highlights`` (with ``coordinates``, ``viewpoint`` and ``topic``). 

```javascript
{
  "item_name": "N2", 
  "item_corpus": "C", 
  "resource": "http://acme/bar", 
  "A": "V", 
  "topics": {
    "T1": {"viewpoint":"V1"}
  }, 
  "highlights": {
    "H": {
      "coordinates": [X,Y],
      "viewpoint": "V2",
      "topic": "T2"
    }
  }
}
```

Reserved attribute names:

* ``thumbnail`` (for items and highlights),
* ``text`` and ``actor`` (for highlights).

## Viewpoint

A viewpoint MUST have a ``viewpoint_name`` (which can be empty). It MAY have ``users`` and ``topics`` (with ``name`` and ``broader`` topics). The topics graph should be acyclic.

```javascript
{
  "viewpoint_name": "N3", 
  "users": ["U1"],
  "topics": {
    "T2": {"name": "N4"},
    "T3": {"name": "N5", "broader":["T2"]}
  }
}
```

Reserved attribute names:

* ``icon`` (for topics).

# Views

Views can be read through the HTTP GET method. Their payloads complies with the JSON standard format.

They are made of ``rows``. Each row has a ``key`` (a JSON array) and a ``value`` (a JSON object).

**TODO**: explain distributed objects

**TODO**: explain distributed links

## Corpus

GET /corpus/C

```javascript
{"rows":[
  {"key":["C"], "value":{"name":"N1"}},
  {"key":["C"], "value":{"user":"U1"}},
  {"key":["C"], "value":{"user":"U2"}},
  {"key":["C","I"], "value":{"name":"N2","resource":"http://acme/bar"}},
  {"key":["C","I"], "value":{"A":"V"}},
  {"key":["C","I"], "value":{"topic":{"viewpoint":"V1","id":"T1"}}},
  {"key":["C","I", "H"], "value":{"coordinates":[X,Y], "topic":{"viewpoint":"V2", "id":"T2"}}}
]}
```

## Item

GET /item/C/I

```javascript
{"rows":[
  {"key":["C","I"], "value":{"name":"N2","resource":"http://acme/bar"}}, 
  {"key":["C","I"], "value":{"A":"V"}},
  {"key":["C","I"], "value":{"topic":{"viewpoint":"V1","id":"T1"}}},
  {"key":["C","I", "H"], "value":{"coordinates":[X,Y], "topic":{"viewpoint":"V2", "id":"T2"}}}
]}
```

## Resource

GET /item/?resource=http://acme/bar

```javascript
{"rows":[
  {"key":["http://acme/bar"],"value":{"item":{"corpus":"C","id":"I"}}}
]}
```

## User

GET /user/U1

```javascript
{"rows":[
  {"key":["U1"], "value":{"corpus":{"id":"C", "name":"N1"}}},
  {"key":["U1"], "value":{"viewpoint":{"id":"V2", "name":"N3"}}}
]}
```

## Viewpoint

GET /viewpoint/V2

```javascript
{"rows":[
  {"key":["V2"], "value":{"name":"N3"}},
  {"key":["V2"], "value":{"user":"U1"}},
  {"key":["V2"], "value":{"upper":{"id":"T2","name":"N4"}}},
  {"key":["V2","T2"], "value":{"name":"N4"}},
  {"key":["V2","T2"], "value":{"narrower":{"id":"T3", "name":"N5"}}},
  {"key":["V2","T3"], "value":{"name":"N5"}},
  {"key":["V2","T3"], "value":{"broader":{"id":"T2", "name":"N4"}}}
]}
```

## Topic
 
GET /topic/V2/T2

```javascript
{"rows":[
  {"key":["V2","T2"], "value":{"name":"N4"}},
  {"key":["V2","T2"], "value":{"narrower":{"id":"T3", "name":"N5"}}}
]}
```

## Attributes by corpus

GET /attribute/C/

```javascript
{"rows":[
  {"key":["C","A"],"value":"1"}
]}
```

## Attribute

GET /attribute/C/A/

```javascript
{"rows":[
  {"key":["C","A", "V"],"value":"1"}
]}
```

## Attribute value

GET /attribute/C/A/V

```javascript
{"rows":[
  {"key":["C","A", "V"],"value":{"item":{"id":"I","name":"N2"}}}
]}
```

# Implementation

## Services

<table><tr>
<td>GET, POST, PUT, DELETE</td>
<th>Argos</th>
<th>Cassandre</th>
</tr><tr>
<th>/CorpusID</th>
<td><a href="http://argos2.hypertopic.org/5a03d6d794ec4b9215f7cba8600c24db">X</a></td>
<td><a href="http://cassandre.hypertopic.org/MISS">X</a></td>
</tr><tr>
<th>/ItemID</th>
<td><a href="http://argos2.hypertopic.org/5a03d6d794ec4b9215f7cba8600c15af">X</a></td>
<td></td>
</tr><tr>
<th>/ViewpointID</th>
<td><a href="http://argos2.hypertopic.org/5a03d6d794ec4b9215f7cba8600c11f7">X</a></td>
<td></td>
</tr></table>

<table><tr>
<td>GET</td>
<th>Argos</th>
<th>Cassandre</th>
<th>Steatite</th>
</tr><tr>
<th>/corpus/ID</th>
<td><a href="http://argos2.hypertopic.org/corpus/5a03d6d794ec4b9215f7cba8600c24db">X</a></td>
<td><a href="http://cassandre.hypertopic.org/corpus/MISS">X</a></td>
<td><a href="http://steatite.hypertopic.org/corpus/Megara%20Hyblaea">X</a></td>
</tr><tr>
<th>/item/ID/ID</th>
<td><a href="http://argos2.hypertopic.org/item/5a03d6d794ec4b9215f7cba8600c24db/5a03d6d794ec4b9215f7cba8600c15af">X</a></td>
<td><a href="http://cassandre.hypertopic.org/item/MISS/e29902013074340b1b52155376000cf0">X</a></td>
<td><a href="http://steatite.hypertopic.org/item/Megara%20Hyblaea/11b01fcdf9c091452791306f60ae38ce59b2c4f1">X</a></td>
</tr><tr>
<th>/item/?resource=URL</th>
<td><a href="http://argos2.hypertopic.org/item/?resource=http://cassandre/text/d0">X</a></td>
<td><a href="http://cassandre.hypertopic.org/item/?resource=http://cassandre.hypertopic.org/text/MISS/e29902013074340b1b521553760036a7">X</a></td>
<td><a href="http://steatite.hypertopic.org/item/?resource=http://steatite.hypertopic.org/picture/11b01fcdf9c091452791306f60ae38ce59b2c4f1">X</a></td>
</tr><tr>
<th>/user/ID</th>
<td><a href="http://argos2.hypertopic.org/user/cecile@hypertopic.org">X</a></td>
<td><a href="http://cassandre.hypertopic.org/user/nadia@hypertopic.org">X</a></td>
<td><a href="http://steatite.hypertopic.org/user/aurelien@hypertopic.org">X</a></td>
</tr><tr>
<th>/viewpoint/ID</th>
<td><a href="http://argos2.hypertopic.org/viewpoint/5a03d6d794ec4b9215f7cba8600c0739">X</a></td>
<td><a href="http://cassandre.hypertopic.org/viewpoint/2a">X</a></td>
<td></td>
</tr><tr>
<th>/topic/ID/ID</th>
<td><a href="http://argos2.hypertopic.org/topic/5a03d6d794ec4b9215f7cba8600c11f7/10">X</a></td>
<td><a href="http://cassandre.hypertopic.org/topic/2a/2c">X</a></td>
<td></td>
</tr><tr>
<th>/attribute/ID/</th>
<td><a href="http://argos2.hypertopic.org/attribute/5a03d6d794ec4b9215f7cba8600c24db/">X</a></td>
<td><a href="http://cassandre.hypertopic.org/attribute/IF14/">X</a></td>
<td></td>
</tr><tr>
<th>/attribute/ID/ZZ/</th>
<td><a href="http://argos2.hypertopic.org/attribute/5a03d6d794ec4b9215f7cba8600c24db/foo/">X</a></td>
<td><a href="http://cassandre.hypertopic.org/attribute/IF14/secteur/">X</a></td>
<td></td>
</tr><tr>
<th>/attribute/ID/ZZ/ZZ/</th>
<td><a href="http://argos2.hypertopic.org/attribute/5a03d6d794ec4b9215f7cba8600c24db/foo/bar">X</a></td>
<td><a href="http://cassandre.hypertopic.org/attribute/IF14/secteur/Automobile">X</a></td>
<td></td>
</tr></table>

Hypertopic services can be tested with a generic [REST client](https://addons.mozilla.org/en-US/firefox/addon/restclient/).

## Client APIs

* [for Java](https://github.com/hypertopic/Porphyry/tree/master/src/org/hypertopic) 
* [for JavaScript](https://github.com/hypertopic/LaSuli/blob/master/modules/HypertopicMap.js) 
* [for PHP](https://github.com/Hypertopic/lib-php)
* [for Objective-C](https://github.com/Hypertopic/lib-objc)

# References

* Jean-Pierre Cahier, Manuel Zacklad, [Expérimentation d’une approche coopérative et multipoint de vue de la construction et de l’exploitation de catalogues commerciaux « actifs »](http://publications.tech-cico.fr/b743575e7c79c308b22e7703b98a81b6). *Document numérique n°3-4, Volume 5*. Hermes, 2001. p.45-64. [FRENCH]

* Jean Caussanel, Jean-Pierre Cahier, Manuel Zacklad, Jean Charlet. [Les Topic Maps sont-ils un bon candidat pour l'ingénierie du Web Sémantique ?](http://publications.tech-cico.fr/b743575e7c79c308b22e7703b98a4bf8), *Conférence Ingénierie des Connaissances IC'2002, Rouen, Mai 2002*. Prix AFIA de la meilleure présentation. [FRENCH]

* Manuel Zacklad, Jean Caussanel, Jean-Pierre Cahier. [Proposition d’un Méta-modèle basé sur les Topic Maps pour la structuration et la recherche d’information](http://publications.tech-cico.fr/b743575e7c79c308b22e7703b98ac908). Atelier Web Sémantique du GRACQ, 2002. [FRENCH]

* Chao Zhou, Aurélien Bénel, Christophe Lejeune. [Towards a standard protocol for community-driven organizations of knowledge](http://publications.tech-cico.fr/ce329c153e7b8873a03ec02847023ca9). *Proceedings of the thirteenth international conference on Concurrent Engineering, Antibes, September 18-22, 2006. Frontiers in Artificial Intelligence and Appl. vol. 143*. Amsterdam: IOS Press, 2006. p.438-449. 

* Aurélien Bénel, Chao Zhou, Jean-Pierre Cahier. [Beyond Web 2.0... And Beyond the Semantic Web](http://publications.tech-cico.fr/71376a63935238483d1e86d569000d5b). Randall, David; Salembier, Pascal (Eds.). *From CSCW to Web 2.0: European Developments in Collaborative Design*. Computer Supported Cooperative Work. Springer Verlag, 2010. p.155-171.
