# GraphStorage API Reference

This document provides a detailed reference for the public API of the GraphStorage class.

## Lifecycle & Instantiation

Methods for creating, opening, and closing a database instance.

`public static GraphStorage open(Path dbPath)`

Opens or creates a new database at the specified file system path. This is the main entry point for the library.

- Parameters: `dbPath`, the `java.nio.file.Path` to the directory where the database will be stored.

- Returns: A new `GraphStorage` instance, ready for use.

- Throws: `GraphStorageException`, If an error occurs while opening or creating the database (e.g., I/O error or RocksDB-level issue).

```java
import java.nio.file.Paths;

try (GraphStorage db = GraphStorage.open(Paths.get("./my-graph-db"))) {
    // Use the database instance
    System.out.println("Database is open.");
} catch (GraphStorageException e) {
    e.printStackTrace();
}
```

`public void close()`

Closes the database connection and releases all underlying resources (including file handles and RocksDB instances).

This method is part of the `AutoCloseable` interface and is called automatically when you use a try-with-resources block, which is the recommended practice.

- Parameters: None.

- Returns: void.

## Ingestion API

Methods for bulk-loading data into the database, currently you can only use `.pgdf` files

`public void insertNodesByFile(String filepath)`

Performs a bulk ingestion of nodes from a specified data file.

- Parameters: `filepath`, A String representing the path to the node data file.

- Returns: void.

- Throws: `GraphStorageException`: If the file cannot be read or a database error occurs during ingestion.

```java
try (GraphStorage db = GraphStorage.open(Paths.get("./my-graph-db"))) {
    db.insertNodesByFile("path/to/my_nodes.pgdf");
}
```

`public void insertEdgesByFile(String filepath)`

Performs a bulk ingestion of edges from a specified data file.

- Parameters: `filepath`, A String representing the path to the edge data file.

- Returns: void.

- Throws: `GraphStorageException`, If the file cannot be read or a database error occurs during ingestion.

```java
try (GraphStorage db = GraphStorage.open(Paths.get("./my-graph-db"))) {
    db.insertEdgesByFile("path/to/my_edges.pgdf");
}
```

## Single Entity Queries

Methods for retrieving a single, specific entity by its unique ID.

`public NodeBlob getNode(String nodeId)`

Retrieves a single node and its data given its unique ID.

- Parameters: `nodeId`, The unique ID of the node to retrieve.

- Returns: A `NodeBlob` object containing the node's label and properties, or null if no node with that ID is found.

- Throws: `GraphStorageException`, If a database read error occurs.

```java
NodeBlob node = db.getNode("user_123");
if (node != null) {
    System.out.println("Label: " + node.label);
    System.out.println("Name: " + node.props.get("name"));
}
```

`public EdgeBlob getEdge(String edgeId)`

Retrieves a single edge and its data given its unique ID.

- Parameters: `edgeId`, The unique ID of the edge to retrieve.

- Returns: An `EdgeBlob` object containing the edge's label, source/target IDs, and properties, or null if no edge with that ID is found.

- Throws: `GraphStorageException`, If a database read error occurs.

```java
EdgeBlob edge = db.getEdge("rel_456");
if (edge != null) {
    System.out.println("Label: " + edge.label);
    System.out.println(edge.src + " -> " + edge.dst);
}
```

## Iterator Queries

Methods for querying multiple entities. These methods return lazy iterators for high performance and low memory usage.

Important: Always Close Iterators

All methods in this section return an `AutoCloseableIterable`. You must use them in a try-with-resources block to prevent resource leaks.

`public AutoCloseableIterable<NodeEntry> getNodeIterator()`

Returns a lazy iterator over all nodes in the database.

- Parameters: None.

- Returns: `AutoCloseableIterable<GraphQueries.NodeEntry>`.

```java
    try (var nodes = db.getNodeIterator()) {
    for (var nodeEntry : nodes) {
        System.out.println("ID: " + nodeEntry.id() + ", Label: " + nodeEntry.blob().label);
    }

}
```

`public AutoCloseableIterable<EdgeEntry> getEdgeIterator()`

Returns a lazy iterator over all edges in the database.

- Parameters: None.

- Returns: An `AutoCloseableIterable<GraphQueries.EdgeEntry>`.

```java
try (var edges = db.getEdgeIterator()) {
    for (var edgeEntry : edges) {
        System.out.println("ID: " + edgeEntry.id() + ", Label: " + edgeEntry.blob().label);
    }
}

```

`public AutoCloseableIterable<EdgeEntry> getEdgeIteratorByLabel(String label)`

Returns a lazy iterator over all edges that have a specific label.

- Parameters: `label`, The exact edge label to filter by (e.g., "KNOWS").

- Returns: An `AutoCloseableIterable<GraphQueries.EdgeEntry>`.

```java
try (var edges = db.getEdgeIteratorByLabel("KNOWS")) {
    for (var edgeEntry : edges) {
        EdgeBlob edge = edgeEntry.blob();
        System.out.println(edge.src + " KNOWS " + edge.dst);
    }
}
```

`public AutoCloseableIterable<EdgeEntry> getNeighbours(String nodeId)`

Returns a lazy iterator over all neighboring edges (both incoming and outgoing) for a specific node.

- Parameters: `nodeId`, The unique ID of the central node.

- Returns: An `AutoCloseableIterable<GraphQueries.EdgeEntry>`.

```java
try (var neighbors = db.getNeighbours("user_123")) {
    for (var edgeEntry : neighbors) {
        System.out.println("Found neighboring edge: " + edgeEntry.id());
    }
}
```

`public AutoCloseableIterable<NodeEntry> getNodesByPropertyEquals(String propName, String propValue)`

Returns a lazy iterator over all nodes that have a specific property key matching a specific value.

- Parameters: `propName`, The name of the property (e.g., "city"). `propValue`, The exact value to match (e.g., "London").

- Returns: An `AutoCloseableIterable<GraphQueries.NodeEntry>`.

```java
try (var nodes = db.getNodesByPropertyEquals("city", "London")) {
    for (var nodeEntry : nodes) {
        System.out.println("Found node: " + nodeEntry.id());
    }
}
```


`public AutoCloseableIterable<EdgeEntry> getEdgesByPropertyEquals(String propName, String propValue)`

Returns a lazy iterator over all edges that have a specific property key matching a specific value.

- Parameters: `propName`, The name of the property (e.g., "weight"). `propValue`: The exact value to match (e.g., "10.5").

- Returns: An `AutoCloseableIterable<GraphQueries.EdgeEntry>`.

```java
try (var edges = db.getEdgesByPropertyEquals("weight", "10.5")) {
    for (var edgeEntry : edges) {
        System.out.println("Found edge: " + edgeEntry.id());
    }
}
```

## Metadata Queries

Methods for retrieving metadata, counts, and schema information about the graph.

`public long getNodesQuantity()`

Gets the total number of nodes in the database.

- Parameters: None.

- Returns: A long representing the total node count.

- Throws: `GraphStorageException`, If a database read error occurs.

```java
    long nodeCount = db.getNodesQuantity();
    System.out.println("Total nodes: " + nodeCount);
```

`public long getEdgesQuantity()`

Gets the total number of edges in the database.

- Parameters: None.

- Returns: A long representing the total edge count.

- Throws: `GraphStorageException`, If a database read error occurs.

```java
    long edgeCount = db.getEdgesQuantity();
    System.out.println("Total edges: " + edgeCount);
```

`public Map<String, Long> getEdgesQuantityByLabel()`

Gets the total number of edges, grouped by their label.

- Parameters: None.

- Returns: A `Map<String, Long>` where the key is the label and the value is the count.

- Throws: `GraphStorageException`: If a database read error occurs.

```java
Map<String, Long> counts = db.getEdgesQuantityByLabel();
for (var entry : counts.entrySet()) {
    System.out.println("Label '" + entry.getKey() + "': " + entry.getValue() + " edges");
}
```

`public Map<String, Set<String>> getNodesStructure()`

Retrieves the detected "schema" of the nodes. It returns a map where each key is a node label, and the value is a set of all property keys found on nodes with that label.

- Parameters: None.

- Returns: `Map<String, Set<String>> (Map<Label, Set<PropertyKey>>)`.

- Throws: `GraphStorageException`: If a database read error occurs.

```java
Map<String, Set<String>> nodeSchema = db.getNodesStructure();
// E.g., prints: "Label 'User': [name, age, email]"
nodeSchema.forEach((label, props) -> {
    System.out.println("Label '" + label + "': " + props);
});
```

`public Map<String, Set<String>> getEdgesStructure()`

Retrieves the detected "schema" of the edges. It returns a map where each key is an edge label, and the value is a set of all property keys found on edges with that label.

- Parameters: None.

- Returns: `Map<String, Set<String>> (Map<Label, Set<PropertyKey>>)`.

- Throws: `GraphStorageException`: If a database read error occurs.

```java

Map<String, Set<String>> edgeSchema = db.getEdgesStructure();
// E.g., prints: "Label 'KNOWS': [since, weight]"
edgeSchema.forEach((label, props) -> {
    System.out.println("Label '" + label + "': " + props);
});
```
