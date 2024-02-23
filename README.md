# KG microbe: TSV files into one RDF (turtle) file

Source: https://kg-hub.berkeleybop.io/kg-microbe/current/index.html
Bioregistry source: https://bioregistry.io/registry/


```python
import csv
import json
import re
```

Relative paths to the input and output files


```python
edgesPath = "input/merged-kg_edges.tsv"
nodesPath = "input/merged-kg_nodes.tsv"
bioregistryPath = "input/registry.json"
outputPath = "output/kg-microbe.ttl"
```

All used prefixes are defined here


```python
prefixes = {
    "biolink": "https://w3id.org/biolink/vocab/",
    "rdfs": "http://www.w3.org/2000/01/rdf-schema#",
    "rdf": "http://www.w3.org/1999/02/22-rdf-syntax-ns#",
    "owl": "http://www.w3.org/2002/07/owl#",
    "dc": "http://purl.org/dc/terms/",
    "obo": "http://purl.obolibrary.org/obo/",
    "oio": "http://www.geneontology.org/formats/oboInOwl#",
    "wd": "https://www.wikidata.org/wiki/",
    "bioregistry": "https://bioregistry.io/",
    "medi": "https://mediadive.dsmz.de/ingredients/",
    "meds": "https://mediadive.dsmz.de/solutions/",
    "medm": "https://mediadive.dsmz.de/medium/"
}
```

Load prefixes from `registry.json` which should be downloaded from https://bioregistry.io/registry/


```python
bioregistry_prefixes = {}
f = open(bioregistryPath)
data = json.load(f)
for entity in data.values():
    bioregistry_prefixes[entity["prefix"]] = {"name": entity["name"], "uri_format": entity["uri_format"]}
f.close()
del data
```

Here we start to write prefixes into the output file


```python
outputStream = open(outputPath, "w")

for p, ns in prefixes.items():
    outputStream.write(f"@prefix {p}: <{ns}> .\n")

outputStream.write("\n")
```

Helpers for saving triples and uri extraction. All uris are resolved from the `iri` columns and then from https://bioregistry.io/registry/ if iri is unknown. All unresolved uris are replaced with the following urn format: `<urn:unknown:id>`.


```python
def add_triple(s: str, p: str, o: str):
    outputStream.write(f"{s} {p} {o} .\n")
    # print(f"{s} {p} {o} .")

def add_label(s: str, label: str):
    add_triple(s, "rdfs:label", json.dumps(label))

def add_type(s: str, t: str):
    add_triple(s, "rdf:type", t)

def add_synonym(s: str, syn: str):
    add_triple(s, "biolink:synonym", json.dumps(syn))
    
def add_reference(s: str, ref: str):
    add_triple(s, "dc:identifier", json.dumps(ref))

def extract_uri(id: str):
    uri_parts = id.split(":", 1)
    if len(uri_parts) == 1:
        return f"urn:unknown:{id}"
    elif len(uri_parts) > 1:
        prefix = uri_parts[0].lower()
        def_prefix = bioregistry_prefixes.get(prefix)
        if def_prefix is None:
            return f"urn:unknown:{id}"
        else:
            uf = def_prefix["uri_format"]  # type: str
            if uf is None or "$1" not in uf:
                return f"https://bioregistry.io/{prefix}:{uri_parts[1]}"
            else:
                return uf.replace("$1", uri_parts[1])
    return None
```

First, we add user-defined triples.


```python
add_triple("biolink:synonym", "rdfs:label", "\"Synonym\"")
add_triple("dc:identifier", "rdfs:label", "\"Reference\"")
```

## Nodes extraction into the output file

It includes:
- labels `rdfs:label`
- types `rdf:type` as `biolink:<type>`
- synonyms `biolink:synonym`
- reference `dc:identifier` with the entity uri.


```python
i = 0
f_in = open(nodesPath, newline="")
reader = csv.reader(f_in, delimiter="\t")
rowsIt = iter(reader)
header = {k: v for v, k in enumerate(next(rowsIt))}
id_to_uri = {}
for row in rowsIt:
    i += 1
    if i % 50000 == 0: print(f"processed lines: {i}")
    uri = row[header["iri"]].split("|")[0]
    if not uri:
        uri = extract_uri(row[header["id"]])
    if not uri: continue
    puri = None
    for p, ns in prefixes.items():
        if uri.startswith(ns):
            puri = f"{p}:{uri.lstrip(ns)}"
            break
    s = puri if puri else f"<{uri}>"
    id_to_uri[row[header["id"]]] = s
    n = str(row[header["name"]]).strip()
    if len(n) > 0: add_label(s, n)
    if not uri.startswith("urn:unknown:"):
        add_reference(s, uri)
    for syn in row[header["synonym"]].split("|"):
        syn = syn.strip()
        if len(syn) > 0 and syn != n: add_synonym(s, syn)
    for t in str(row[header['category']]).split("|"):
        t = t.strip()
        if len(t) > 0: add_type(s, t)
print(f"(Done) processed lines: {i}")
f_in.close()
```

## Edges extraction into the output file

Triples with unknown uris with unknown prefix are skipped.


```python
i = 0
f_in = open(edgesPath, newline="")
reader = csv.reader(f_in, delimiter="\t")
rowsIt = iter(reader)
header = {k: v for v, k in enumerate(next(rowsIt))}
unknown_predicates = set()
used_predicates = set()
for row in rowsIt:
    i += 1
    if i % 50000 == 0: print(f"processed lines: {i}")
    s = id_to_uri.get(row[header["subject"]])
    o = id_to_uri.get(row[header["object"]])
    p = row[header["predicate"]]
    p_parts = p.split(":", 1)
    if s and o and len(p_parts) == 2 and p_parts[0] in prefixes:
        used_predicates.add(p)
        add_triple(s, p, o)
    else:
        unknown_predicates.add(p)
print(f"(Done) processed lines: {i}")
f_in.close()
if len(unknown_predicates) > 0:
    print(f"Unknown predicates: {unknown_predicates}")
```

Add labels of all used predicates into the output file.


```python
for p in used_predicates:
    add_label(p, re.sub("([A-Z])", r"_\1", p.split(":", 1)[1]).replace("_", " ").lower().strip())
```

Close the output file writing.


```python
outputStream.close()
```
