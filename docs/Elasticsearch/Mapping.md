# Mapping

With mapping you can determine how a document is defined and the fields it contains are stored and indexed.
Basically your document contains a collection of fields, which have their own data types. With mapping you can determine the data type of the fields.

There are two mappings: *dynamic* and *explicit* mapping.

---

## Dynamic mapping
Elasticsearch use dynamic mapping automatically, when you don't use explicit mapping. 

## Explicit mapping
This mapping allows you to choose how to define the mapping definitions. You can decide which fields contains which data types. 
Explicit mapping is useful, because you know your data best.
Field mappings can be described when you create a new index and also can be added to existing indices. 