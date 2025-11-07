# Get started with GraphStorage

Welcome to GraphStorage, an on-disk property graph storage library.

This guide will walk you through adding the library to your project, opening a database, ingesting data from files, and performing basic queries.

## Prerequisites

- Java 17 or higher.

- Maven.

## Project Setup (Maven)

First you need to clone the [github repo](https://github.com/dbgutalca/graph-storage).

```bash
git clone https://github.com/dbgutalca/graph-storage
cd graph-storage
```


To use GraphStorage in your Maven project, you must first install the library to your local repository. Run the following command in the `graph-storage` directory:
```
mvn install
```

Once installed, you can add `GraphStorage` as a dependency in your other project's pom.xml:

```xml
<dependency>
    <groupId>com.gdblab.graphstorage</groupId> 
    <artifactId>graphstorage</artifactId>
    <version>1.0-SNAPSHOT</version> 
</dependency>
```

## Opening and Closing the Database

The main entry point is the GraphStorage class. It implements AutoCloseable, so you should always use it within a try-with-resources block. This ensures that the database connection (and underlying RocksDB resources) are closed properly.

```java
import com.gdblab.graphstorage.GraphStorage;
import com.gdblab.graphstorage.storage.Utils.GraphStorageException;
import java.nio.file.Path;
import java.nio.file.Paths;

public class App {
    public static void main(String[] args) {
        // Define the path where the database will be stored
        Path dbPath = Paths.get("./my-graph-db");

        try (GraphStorage db = GraphStorage.open(dbPath)) {
            // ... your database interaction code goes here ...
            System.out.println("Database opened successfully at: " + dbPath.toAbsolutePath());
            
        } catch (GraphStorageException e) {
            System.err.println("Error opening or using the database:");
            e.printStackTrace();
        }
    }
}
```

## Ingesting Data

GraphStorage is optimized for bulk ingesting data from `.pgdf` files.

Use the `insertNodesByFile` and `insertEdgesByFile` methods.

```java
try (GraphStorage db = GraphStorage.open(dbPath)) {
    
    System.out.println("Ingesting nodes...");
    db.insertNodesByFile("nodes.pgdf");
    
    System.out.println("Ingesting edges...");
    db.insertEdgesByFile("edges.pgdf");
    
    System.out.println("Ingestion complete.");

} 
```

## Querying Data

The API provides two main ways to query: fetching single entities by ID and getting lazy iterators for collections of entities. The query methods return NodeBlob and EdgeBlob objects, which contain public fields for easy data access.

### Single Entity Queries

You can get a specific node or edge if you know its unique ID.

```java
import com.gdblab.graphstorage.storage.NodeBlob;
import com.gdblab.graphstorage.storage.EdgeBlob;

//...

// ... inside the try-with-resources block ...
try {
    // Get a single node by its ID
    NodeBlob node = db.getNode("user_101");
    
    if (node != null) {
        System.out.println("Found node. Label: " + node.label);
        // Access properties directly from the 'props' map
        System.out.println("Property 'name': " + node.props.get("name"));
    } else {
        System.out.println("Node with ID 'user_101' not found.");
    }

    // Get a single edge by its ID
    EdgeBlob edge = db.getEdge("edge_50");
    if (edge != null) {
        System.out.println("Found edge. Label: " + edge.label);
        System.out.println("Source: " + edge.src);
        System.out.println("Target: " + edge.dst);
    }

} catch (GraphStorageException e) {
    e.printStackTrace();
}

```

### Iterator Queries

For queries that return multiple results (like neighbors, nodes by property, or all nodes), the API returns an AutoCloseableIterable.

This is important! You must always use these iterators within a try-with-resources block to prevent resource leaks.

The iterator returns NodeEntry or EdgeEntry objects. Use the .id() method to get the ID and the .blob() method to get the NodeBlob or EdgeBlob.

```java
// Example: Get all neighbors of node "user_101"

System.out.println("Neighbors of 'user_101':");
try (var neighbors = db.getNeighbours("user_101")) {
    
    for (var edgeEntry : neighbors) {
        EdgeBlob blob = edgeEntry.blob(); // Get the EdgeBlob
        
        System.out.println("  - Edge ID: " + edgeEntry.id());
        System.out.println("    Label: " + blob.label);
        System.out.println("    Source: " + blob.src);
        System.out.println("    Target: " + blob.dst);
        System.out.println("    Weight: " + blob.props.get("weight")); // Access props
    }
    
} catch (GraphStorageException e) {
    e.printStackTrace();
}

// Example: Find nodes by a specific property
System.out.println("Nodes with property 'country'='Chile':");
try (var nodes = db.getNodesByPropertyEquals("country", "Chile")) {
    
    for (var nodeEntry : nodes) {
        NodeBlob blob = nodeEntry.blob(); // Get the NodeBlob
        
        System.out.println("  - Node ID: " + nodeEntry.id());
        System.out.println("    Label: " + blob.label);
        System.out.println("    Name: " + blob.props.get("name"));
    }
    
} catch (GraphStorageException e) {
    e.printStackTrace();
}
```

## Querying Metadata

You can quickly retrieve counts and other schema-related information.

```java

try {
    long nodeCount = db.getNodesQuantity();
    long edgeCount = db.getEdgesQuantity();
    
    System.out.println("Graph Metadata");
    System.out.println("Total Nodes: " + nodeCount);
    System.out.println("Total Edges: " + edgeCount);
    
    var edgesByLabel = db.getEdgesQuantityByLabel();
    System.out.println("Edge Counts by Label:");
    edgesByLabel.forEach((label, count) -> {
        System.out.println("  - " + label + ": " + count);
    });

} catch (GraphStorageException e) {
    e.printStackTrace();
}

```
