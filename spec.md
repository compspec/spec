# Composition Spec

 - [Goals](#goals)
 - [Definitions](#definitions)
   - [Entity](#entity)
   - [Namespace](#namespace)
   - [Facts](#facts)
   - [Corpus](#corpus)
 - [Procedures](#procedures)
   - [Extract](#extract)
     - [Domain](#domain)
     - [Node](#node)
     - [Relation](#relation)
   - [Diff](#diff)
     - [Rules](#rules)

## Goals

The Composition Specification aims to provide simple procedures, models, and implementations
for comparing compositions.

## Definitions

### Entity

An entity is a digital or physical object that be represented by nodes an relationships. 
Examples in computer science include libraries and binaries. An entity typically
has one or more of each of nodes and relationships, all of which exist under a namespace.

### Namespace

A namespace is generally a named scope under which unique identifiers are comparable.
As an example, if we have two versions of the same function "hello-world" (versions A and B) 
in different libraries, all references for within each can be thought of as within the namespaces
A and B. E.g.,:

```python
A.hello-world
B.hello-world
```

And children of these abstract entities will inherit the namespace. This also means
that both namespaces could have overlapping identifiers, however they are not meaningful
in context of on another, e.g., `A.id0` != `A.id1`.

### Facts

Facts are sets of logical statements about an entity. Under compspec, facts can be one
of two types:

 - **node**: a node represents an object or attribute
 - **relation**: a relation is a relationship between facts, with an optional identifier for a type of relationship that defaults to "has" to indicate a parent "has" a child of some kind.
 
In this context, by default the relation type is "has" and the direction of relationship
flows from parent to child.

### Corpus

A corpus is a collection of facts about an entity.


## Procedures

The composition specification currently defines two kinds of procedures,
extraction and diff.

### Extract

Given an entity, we can perform some extraction to derive a list of facts.

```
[entity] -- extraction --> [facts]
```

This initial step of loading in the entity and defining facts can be done for a single
isolated entity, or with two entities to carry on the facts for later comparison.
Generally this extraction means:

 1. reading in the entity
 2. defining types of nodes and relations
 3. deriving the set
 4. saving to an output format

And a compspec implementation should provide either domain-specific or -agnostic
tools for making this easy to do.

#### Domain

Every comparison has a domain. For any given domain, the implementer should
decide what they want to model. Let's inspect a simple Python function that has
a name, and a parameter with a default.

```python
def hello_world(name="Vanessa"):
    return f"Hello, {name}"
```

In this simple example, I might decide that I want to model the following ideas:

 - There is a function called "hello_world"
 - The function "hello_world" has a parameter "name"
 - The parmater "name" has type string
 - The parmater "name" has a default of "Vanessa"

While the above is concretely a listing of facts (nodes) there are also implicit relationships!
For example:

```bash
                                                 --> has default "Vanessa"
function hello_world -- has parameter -->  name
                                                 --> is type string
```

And the comparison spec is going to make it easy to flatten these ideas
into nodes and relationships (to define a graph) that can understand any set of
entities, attributes describing entities, or relationshps.

#### Node

A node is a flat set of four attributes that define an entity, including:

 - a **namespace** [defined above](#namespace).
 - a unique identifier within the namespace to understand relationships with other nodes
 - a generic name or type (e.g., function or func)
 - a value that represents the particular type (e.g., "hello_world")

And structured as follows:

```
node("<namespace>", "<identifier>", "<name>", "<value>").
```

As an example from the facts above, let's say that our function is in a hypothetical
namespace "A." We might define the following nodes:

```asp
node("A", "id0", "func", "goodbye_world").
node("A", "id1", "func", "hello_world").

% Library A has these flattened attributes attributes
node("A", "id2", "parameter", "name").
node("A", "id3", "default", "Vanessa").
node("A", "id4", "type", "string").
```

The above representation is in ASP (answer set programming or a logic program) but
you can imagine an implementation chooing another data structure, e.g.:

```python
[
    {
        "namespace": "A",
        "id": "id0",
        "name": "func",
        "value": "hello_world"
    }
]

```

Given a set of nodes, it should be possible to derive a list of unique node types
within a namespace via:

```asp
nodes(Namespace, Type) := node(Namespace, _, Type, _).
```


#### Relation

A relation is simple - it describes the relationship between two nodes.
To continue the example above:

```asp
% function hello_world has parameter "name"
relation("A", "id1", "has", "id2").

% parameter name has default Vanessa
relation("A", "id2", "has", "id3").

% parameter name has type string
relation("A", "id2", "has", "id4").
```

The relations above are defined by a type, with a default type of "has" to indicate a parent to child
relationships, a parent entity "has" a child entity and points to it.
Indeed, the relations are intended to be generated by a tool, and are primarily for use
within an implementation. The identifiers within a namespace for both nodes and relations
are scoped to that namespace.

Given a set of relations, it should be possible to derive a list of unique relation types
within a namespace via:

```asp
relations(Namespace, Relation) := relation(Namespace, _, Relation, _).
```

### Diff

Two extractions can be further compared, such as with a diff. A diff
is exactly that - we essentially have two graphs and subtract them to understand
what is added, removed, or changed. Unchanged items are generally not important
to show. For example, let's say we want to compare two entities. We'd first 
extract facts, and then (given a specification for doing a diff) we would run the diff:

```
[entity A] -- extraction --> [facts A]
                                           -- facts diff --> [change set] 
[entity B] -- extraction --> [facts B]
```

The result of the diff is a set of additions, subtractions, and changes, or more generally, 
a change set. In more abstract terms:

```
A 
   --> A-B --> change set
B
```

This means the direction of the diff should be "changes resulting in moving from A to B." 

#### Rules

The composition specification for doing a diff "diffspec" should provide the following
functionality. Note that some of these are *optional* but suggested. Given an entity
in namespace A, and another in namespace B, each with a corpus extracted, a diff should:

##### Assertion of Difference

Make an assertion that the two comparison entries are different *optional*

```asp
is_different(A, B) :- is_a(A), is_b(B), A != B.
```

##### Changed Node Values

A node is asserted to have a change of value if:

 - Given namespaces A and B for comparison
 - the node exists in A (based on name)
 - the node exists in B (based on name)
 - there exists a nearest shared parent (one level up)
 - the values are different.

```asp
changed_node_value(A, B, EntityA, EntityB, Name, ValueA, ValueB) :-
    is_a(A), is_b(B),
    node(A, EntityA, Name, ValueA),
    node(B, EntityB, Name, ValueB),
    shared_parent(EntityA, EntityB, Name),
    ValueA != ValueB.
```

##### Removed Nodes

A node is asserted to be removed in B if:

 - Given namespaces A and B for comparison
 - the node exists in A (based on name, and we don't care about value)
 - the node does not exist in B (same)
 - there exists a nearest parent (one level up)

```asp
What is removed if we move from A to B? 
% This covers anything with a parent (not a root)
removed_node(A, B, Name) :-
    is_a(A), is_b(B),
    node(A, NodeA, Name, _),
    not node(B, NodeB, Name, _),
    shared_parent_missing(NodeA, NodeB, Name).
```

##### Added Nodes
 
A node is asserted to be added in B if:

 - Given namespaces A and B for comparison
 - the node exists in B (based on name, and we don't care about value)
 - the node does not exist in A (same)
 - the node is not represented in the set of changed values

```asp
% What is added if we move from A to B?
% We have the entity in B but not A
added_node(A, B, Name, Value) :- 
    is_a(A), is_b(B),
    node(A, _, Name, Value),
    not node(B, _, Name, Value),
    not changed_node_value(A, B, _, _, Name, _, _).    
```

Of course the above require supporting rules, which is up to the implementation (and we
will provide in the implementations here. These rules are the basis of a comparison, 
and the model to derive a diff.

### Implementation

In practice, an implementation should simply allow a user to define these relations and
nodes, and then output a change set. The change set should include groups of rules that are "true,"
e.g.,:

- added_node
- removed_node
- changed_node_value
