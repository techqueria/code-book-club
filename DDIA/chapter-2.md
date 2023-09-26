## Chapter 2 - Data Models and Query Languages
- Most applications are built by layering one data model on top of another
- For each layer, the key question is: how is it represented in terms of the next-lower layer
- The basic idea is that each layer hides the complexity of the layers below it by providing a clean data model aka abstractions

### Relational Model Versus Document Model
- Best-known data model today is probably that of SQL
- Relational model
  - Data is organized into relations (called tables in SQL)
  - Where each relation is an unordered collection of tuples (rows in SQL)
- Transaction processing i.e entering sales or banking transactions, airline reservations, stock-keeping in warehouses
- Batch processing i.e (customer invoicing, payroll, reporting)
- The goal of the relational model was to hide that implementation detail (the internal representation of the data in the database) behind a cleaner interface

### The Birth of NoSQL
- Several driving forces behind the adoption of NoSQL databases, including:
  - A need for greater scalability than relational databases can easily achieve, including very large datasets or very high write throughput
  - A widespread preference for free and open source software over commercial database products
  - Specialized query operations that are not well supported by the relational model
  - Frustration with the restrictiveness of relational schemas, and a desire for a more dynamic and expressive data model
- Polyglot persistence - the idea that relational databases will continue to be used alongside a broad variety of nonrelational datastores

### The Object-Relational Mismatch
- For OOP programming languages, if data is stored in relational tables, there is an awkward transition layer required between the objects in the application code and db model of table, rows, and columns
- Impedance mismatch - The disconnect between the models
- Object-relational mapping (ORM) frameworks can try to reduce the amount of boilerplate code required for this transition layer but can't completely hide the differences between two models
- Some developers feel that the JSON model reduces the impedance mismatch between the application code and the storage layer
- The JSON representation has better locality than the multi-table schema
- Fetching info like a LinkedIn profile might require messy multi-way joins and queries in a multi-table schema

### Many-to-One and Many-to-Many Relationships
- Whether you store an ID or a text string is a question of duplication:
  - When you use an ID, the information that is meaningful to humans (such as the word Philanthropy) is stored in only one place, and everything that refers to it uses an ID (which only has meaning within the database)
  - When you store the text directly, you are duplicating the human-meaningful information in every record that uses it
- The advantage of using an ID is that because it has no meaning to humans, it never needs to change: the ID can remain the same, even if the information it identifies changes
- Removing such duplication is the key idea behind normalization in databases
  - As a rule of thumb, if you’re duplicating values that could be stored in just one place, the schema is not normalized.
- Normalizing data requires many-to-one relationships
- In relational databases, it’s normal to refer to rows in other tables by ID, because joins are easy
- In document databases, joins are not needed for one-to-many tree structures, and support for joins is often weak
- If the database itself does not support joins, the work of making the join is shifted from the database to the application code

### Are Document Databases Repeating History?
- The most popular database for business data processing in the 1970s was IBM’s Information Management System (IMS)
- It used a simple data model called the hierarchical model, similar to the JSON model used by document databases
- Various solutions were proposed to solve the limitations of the hierarchical model
- The two most prominent were the relational model (which became SQL) and the network model

### The Network Model
- network model was standardized by a committee called the Conference on Data Systems Languages (CODASYL) and implemented by several different database vendors; it is also known as the CODASYL model 
- The CODASYL model was a generalization of the hierarchical model
- In the tree structure of the hierarchical model, every record has exactly one parent; in the net‐ work model, a record could have multiple parents
- access path - a path from a root record along these chains of links (how to access info)
- an access path could be like the traversal of a linked list: start at the head of the list, and look at one record at a time until you find the one you want 
- a record could have multiple parents, meaning there could be multiple paths 
- however, if you didn't have the path you wanted, writing in a way required handwritten database query code 

### The Relational Model
- relational models lay out all the data in the open: a relation (table) is simply a collection of tuples (rows)
- You can insert a new row into any table without worrying about foreign key relationships to and from other tables 
- Foreign key constraints allow you to restrict modifications, but such constraints are not required by the relational model
- Even with constraints, joins on foreign keys are performed at query time, whereas in CODASYL, the join was effectively done at insert time.
- the query optimizer automatically decides which parts of the query to execute in which order, and which indexes to use ie the “access path” 
- the big difference is that they are made automatically by the query optimizer, not by the application developer, so we rarely need to think about them
- key insight of the relational model was this: you only need to build a query optimizer once, and then all applications that use the database can benefit from it 
- If you don’t have a query optimizer, it’s easier to handcode the access paths for a query than to write a general-purpose optimizer—but the general-purpose solution wins in the long run

### Comparison to Document Databases
- Document databases are similar to the hierarchical model: storing nested records within their parent record rather than in a separate table
- For representing many-to-one and many-to-many relationships, both models are not fundamentally different: both use a unique identifier, known as a foreign key in the relational model and a document reference in the document model
- That identifier is resolved at read time by using a join or follow-up queries

### Relational Versus Document Databases Today
- Document data model
  - Schema flexibility
  - Better performance due to locality
  - Closer to the data structures used by the application
- The relational model
  - Provides better support for joins, and many-to-one and many-to-many relationships

### Which Data Model Leads to Simpler Application Code?
- if the data in your application has a document-like structure (i.e., a tree of one-to-many relationships, where typically the entire tree is loaded at once) -> a document model
 - shredding— the relational technique of splitting a document-like structure into multiple tables 
- document model limitations
  - example, you cannot refer directly to a nested item within a document
  - not an issue if data not deeply nested 
- document model performs bad when  
  - your application does use many-to-many relationships
  - it’s possible to reduce the need for joins by denormalizing, but then the application code needs to do additional work to keep the denormalized data consistent. 
  - joins can be emulated in application code by making multiple requests to the database, but that also moves complexity into the application and is usually slower than a join performed by specialized code inside the db

### Schema Flexibility in the Document Model
- document databases, and the JSON support in relational databases, do not enforce any schema on the data in documents
- XML support in relational databases usually comes with optional schema validation
- No schema means that arbitrary keys and values can be added to a document, and when reading, clients have no guaran‐tees as to what fields the documents may contain
- document db's can be described as schema-on-read (the structure of the data is implicit, and only interpreted when the data is read)
- schema-on-write (the traditional approach of relational databases, where the schema is explicit and the database ensures all written data conforms to it) 
- Schema-on-read is similar to dynamic (runtime) type checking in programming languages
- Schema-on-write is similar to static (compile-time) type checking
- The difference between the approaches is noticeable when applications want to change the format of their data 
- schema is not as advantageous in examples like 
  - the structure of the data is determined by external systems over which you have no control and which may change at any time
  - there are many different types of objects, and it is not practical to put each type of object in its own table
- in cases where all records are expected to have the same structure, schemas are a useful mechanism for document‐ ing and enforcing that structure

### Data locality for queries
- A document is usually stored as a single continuous string, encoded as JSON, XML, or a binary variant  
- If your application often needs to access the entire document (for example, to render it on a web page), there is a performance advantage to this storage locality

### Convergence of document and relational databases
- relational and document databases are becoming more similar over time, and that is a good thing: the data models complement each other
- If a database is able to handle document-like data and also perform relational queries on it, appli‐ cations can use the combination of features that best fits their needs
- A hybrid of the relational and document models is a good route for databases to take in the future
## Query Languages for Data
- SQL is a declarative query language
- In a declarative query language, like SQL or relational algebra, you just specify the pattern of the data you want—what conditions the results must meet, and how you want the data to be transformed (e.g., sorted, grouped, and aggregated)—but not how to achieve that goal
- It is up to the database system’s query optimizer to decide which indexes and which join methods to use, and in which order to execute various parts of the query
- declarative languages often lend themselves to parallel execution
- declarative query language hides implementation details of the database engine, which makes it possible for the database system to introduce performance improvements without requiring any changes to queries
- An imperative language tells the computer to perform certain operations in a certain order 
- You can imagine stepping through the code line by line, evaluating conditions, updating variables, and deciding whether to go around the loop one more time 
- Imperative code is very hard to parallelize across multiple cores and multiple machines, because it specifies instructions that must be performed in a particular order
### Declarative Queries on the Web
- In a web browser, using declarative CSS styling is much better than manipulating styles imperatively in JavaScript 
- Similarly, in databases, declarative query languages like SQL turned out to be much better than imperative query APIs 
### MapReduce Querying
- MapReduce is a programming model for processing large amounts of data in bulk across many machines, popularized by Google
- MapReduce is neither a declarative query language nor a fully imperative query API, but somewhere in between: the logic of the query is expressed with snippets of code, which are called repeatedly by the processing framework
- It is based on the map (also known as collect) and reduce (also known as fold or inject) functions that exist in many functional programming languages
- The map and reduce functions must be pure functions, which means they only use the data that is passed to them as input, they cannot perform additional database queries, and they must not have any side effects
- These restrictions allow the database to run the functions anywhere, in any order, and rerun them on failure
- However, they can parse strings, call library functions, perform calculations, and more
-  a usability problem with MapReduce is that you have to write two carefully coordinated JavaScript functions
- a declarative query language offers more opportunities for a query optimizer to improve the performance of a query

## Graph-Like Data Models 

- If your application has mostly one-to-many relationships (tree-structured data) or no relationships between records, the document model is appropriate
- The relational model can handle simple cases of many-to-many relationships, but as the con‐ nections within your data become more complex, it becomes more natural to start modeling your data as a graph
- A graph consists of two kinds of objects: 
  - vertices (also known as nodes or entities) 
  - edges (also known as relationships or arcs)

- Many kinds of data can be modeled as a graph. Typical examples include:
  - Social graphs
    - Vertices are people, and edges indicate which people know each other
  - The web graph
    - Vertices are web pages, and edges indicate HTML links to other pages  
  - Road or rail networks
    - Vertices are junctions, and edges represent the roads or railway lines between them

### Property Graphs
- In the property graph model, each vertex consists of:
  • A unique identifier
  • A set of outgoing edges
  • A set of incoming edges
  • A collection of properties (key-value pairs)
- Each edge consists of:
  • A unique identifier
  • The vertex at which the edge starts (the tail vertex)
  • The vertex at which the edge ends (the head vertex)
  • A label to describe the kind of relationship between the two vertices
  • A collection of properties (key-value pairs)
Some important aspects of this model are:
  1. Any vertex can have an edge connecting it with any other vertex. There is no schema that restricts which kinds of things can or cannot be associated 
  2. Given any vertex, you can efficiently find both its incoming and its outgoing edges, and thus traverse the graph—i.e., follow a path through a chain of vertices —both forward and backward  
  3. By using different labels for different kinds of relationships, you can store several different kinds of information in a single graph, while still maintaining a clean data model
- Graphs are good for evolvability: as you add features to your application, a graph can easily be extended to accommodate changes in your application’s data structures
## The Cypher Query Language 
- Cypher is a declarative query language for property graphs, created for the Neo4j graph database
- no need to specify such execution details when writing the query: the query optimizer automatically chooses the strategy that is predicted to be the most efficient, so you can get on with writing the rest of your application
## Graph Queries in SQL 
- we can put graph data in a relational structure and use SQL to query it
- you need to know in advance which joins you need in your queries 
- in a graph query, you may need to traverse a variable number of edges before you find the vertex you’re looking for aka the number of joins is not fixed in advance
- the idea of variable-length traversal paths in a query can be expressed using something called recursive common table expressions

## Triple-Stores and SPARQL
 - In a triple-store, all information is stored in the form of very simple three-part statements: (subject, predicate, object) 
 - The subject of a triple is equivalent to a vertex in a graph 
 - The object is one of two things:
  1. A value in a primitive datatype, such as a string or a number
    - In that case, the predicate and object of the triple are equivalent to the key and value of a property on the subject vertex
    - example: (lucy, age, 33) is like a vertex lucy with properties {"age":33}
  2. Another vertex in the graph
    - In that case, the predicate is an edge in the graph, the subject is the tail vertex, and the object is the head vertex
    - example, in (lucy, marriedTo, alain) the subject and object lucy and alain are both vertices, and the predicate marriedTo is the label of the edge that connects them
  - triple-store data model is completely independent of the semantic web
## The RDF data model
- The Resource Description Framework (RDF) is a framework for representing information in the Web 
- It is based on the idea of making statements about resources (like documents or things) in the form of subject-predicate-object expressions called triples 
- The Turtle language we used in Example 2-7 is a human-readable format for RDF data
- The subject, predicate, and object of a triple are often URIs
## The SPARQL query language 
- SPARQL is a query language for triple-stores using the RDF data model (SPARQL Protocol and RDF Query Language)

## Graph Databases Compared to the Network Model 

 - Graph DB's vs CODASYL 
  • CODASYL: a database had a schema that specified which record type could be nested within which other record type     
  • Graph DB: there is no such restriction: any vertex can have an edge to any other vertex. This gives much greater flexibility for applications to adapt to changing requirements 
  • CODASYL: the only way to reach a particular record was to traverse one of the access paths to it.
  • Graph DB: you can refer directly to any vertex by its unique ID, or you can use an index to find vertices with a particular value
  • CODASYL: the children of a record were an ordered set, so the database had to maintain that ordering (which had consequences for the storage layout) and applications that inserted new records into the database had to worry about the positions of the new records in these sets
•  Graph DB: vertices and edges are not ordered (you can only sort the results when making a query)
• CODASYL: all queries were imperative, difficult to write and easily broken by changes in the schema
•  Graph DB: you can write your traversal in imperative code if you want to, but most graph databases also support high-level, declarative query languages such as Cypher or SPARQL 

### The Foundation: Datalog

- Datalog is used in a few data systems: for example, it is the query language of Datomic, and Cascalog is a Datalog implementation for querying large datasets in Hadoop 
- instead of writing a triple as (subject, predicate, object), dataog writes it as predicate(subject, object) 
- datalog can define rules to create new predicates 
- these new predicates aren’t triples stored in the database, but instead they are derived from data or from other rules.
- a rule applies if the system can find a match for all predicates on the righthand side of the :- operator
- when the rule applies, it’s as though the lefthand side of the :- was added to the database (with variables replaced by the values they matched)

## Summary
- data started out being represented as one big tree (the hierarchical model), but that wasn’t good for representing many-to-many relationships, so the relational model was invented to solve that problem
- developers found that some applications don’t fit well in the relational model either
-  New nonrelational “NoSQL” datastores have diverged in two main directions:
  1. Document databases target use cases where data comes in self-contained documents and relationships between one document and another are rare
  2. Graph databases go in the opposite direction, targeting use cases where anything is potentially related to everything
-  document and graph databases both typically don’t enforce a schema for the data they store, which can make it easier to adapt applications to changing requirements
- if your application most likely still assumes that data has a certain structure: it’s just a question of whether the schema is explicit (enforced on write) or implicit (handled on read)
- Each data model comes with its own query language or framework, and we discussed several examples:
  (SQL, MapReduce, MongoDB’s aggregation pipeline, Cypher, SPARQL, and Datalog)