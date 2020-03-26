# Chapter 2 - Data Models and Query Languages

Relational Model - Relations (tables) and tuples (rows)

LinkedIn profile example -
- *locality* - If stored in JSON, entire document can be obtained in 1 query. For relational, multiple queries with multi-way join will be needed. JSON is more suited for *tree* structures for easier querying.
- *normalization/joins* - Difficult to support joins for many-to-one relations eg If multiple people live in a city and it changes it name, in document databases, the name will have to be changed everywhere, whereas in relational, the new name should be updated in only 1 place.
- *schema flexibility* - Different users can have different objects in their profile in document.


In RDBMS, user submits the query and the query optimizer finds the best path/indexes to use. In document, user/application has to figure out the access path. This leads to more complex application code and worse performance.

- Document database - `schema-on-read`: 'the structure of the data is implicit, and only interpreted when the
data is read'
- Relational database - `schema-on-write`: 'the schema is explicit and the database ensures all written data con‐
forms to it'

Normally, only the whole document can be accessed in a document database and not just a small portion of it. And the whole document is rewritten even if there are only small updates to the document. So the document sizes should be kept small to improve performance.


- declarative language: Tell the database what you want. The database query optimizer will figure out the best way to do it. More suited for parallel execution.

SQL
```
SELECT * FROM animals WHERE family = 'Sharks'
```
CSS
```
li.selected > p {
  background-color: blue;
}
```
- imperative language: Tell the compute exactly how to do get what you want. Hard to parallelize.
```
function getSharks() {
    var sharks = [];
    for (var i = 0; i < animals.length; i++) {
      if (animals[i].family === "Sharks") {
        sharks.push(animals[i]);
      }
    }
    return sharks;
}
```


## Graph data models
### Property Graphs
- Vertex:
  - ID
  - Outgoing edges
  - Incoming edges
  - Properties (key-value pairs)
- Edges:
  - ID
  - Head Vertex (end of edge)
  - Tail Vertex (start of edge)
  - Label
  - Properties (key-value pairs)
