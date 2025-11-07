# Workflow example

This example demonstrates the complete workflow: ingesting data, running various queries, and fetching metadata.

## Example Data Files

First, let's assume you have two files, nodes.pgdf and edges.pgdf, in your project's root directory. The ingestor expects these files to be in the .pgdf format.


### nodes.pgdf
```txt
@id|@label|name|country
user_101|User|Ana|Mexico
user_102|User|Bob|USA
user_103|User|Pedro|Chile
```

### edges.pgdf
```text
@id|@label|@dir|@out|@in|weight
edge_50|KNOWS|T|user_101|user_102|0.8
edge_51|WORKS_WITH|T|user_103|user_101|0.2
```

## Code example

This single Java class performs all operations on the data defined above.

```java
import com.gdblab.graphstorage.GraphStorage;
import com.gdblab.graphstorage.storage.EdgeBlob;
import com.gdblab.graphstorage.storage.NodeBlob;
import com.gdblab.graphstorage.storage.Utils.GraphStorageException;
import java.nio.file.Path;
import java.nio.file.Paths;

public class FullDemo {

    public static void main(String[] args) {
        Path dbPath = Paths.get("./my-graph-db");

        try (GraphStorage db = GraphStorage.open(dbPath)) {
            System.out.println("Database opened successfully at: " + dbPath.toAbsolutePath());

            // INGEST DATA
            // This only needs to be run once.
            // You can comment this out after the first successful run.
            System.out.println("\n--- Ingesting Data ---");
            try {
                db.insertNodesByFile("nodes.pgdf");
                db.insertEdgesByFile("edges.pgdf");
                System.out.println("Ingestion complete.");
            } catch (GraphStorageException e) {
                System.err.println("Ingestion failed: " + e.getMessage());
            }

            // SINGLE ENTITY QUERY (BY ID)
            System.out.println("\n--- Query 1: Get Node 'user_101' ---");
            try {
                NodeBlob node = db.getNode("user_101");
                if (node != null) {
                    System.out.println("Found node. Label: " + node.label);
                    System.out.println("Property 'name': " + node.props.get("name"));
                    System.out.println("Property 'country': " + node.props.get("country"));
                } else {
                    System.out.println("Node with ID 'user_101' not found.");
                }
            } catch (GraphStorageException e) {
                e.printStackTrace();
            }

            // ITERATOR QUERY (BY PROPERTY)
            // Note: This must be in a try-with-resources block
            System.out.println("\n--- Query 2: Find Nodes where 'country'='Chile' ---");
            try (var nodes = db.getNodesByPropertyEquals("country", "Chile")) {
                for (var nodeEntry : nodes) {
                    NodeBlob blob = nodeEntry.blob();
                    System.out.println("  - Found Node ID: " + nodeEntry.id());
                    System.out.println("    Label: " + blob.label);
                    System.out.println("    Name: " + blob.props.get("name"));
                }
            } catch (GraphStorageException e) {
                e.printStackTrace();
            }

            // ITERATOR QUERY (NEIGHBORS)
            // This also MUST be in a try-with-resources block
            System.out.println("\n--- Query 3: Get Neighbors of 'user_101' ---");
            try (var neighbors = db.getNeighbours("user_101")) {
                for (var edgeEntry : neighbors) {
                    EdgeBlob blob = edgeEntry.blob();
                    System.out.println("  - Found Edge ID: " + edgeEntry.id());
                    System.out.println("    Label: " + blob.label);
                    System.out.println(
                        String.format("    %s --[%s]--> %s", blob.src, blob.label, blob.dst)
                    );
                }
            } catch (GraphStorageException e) {
                e.printStackTrace();
            }

            //  METADATA QUERIES
            System.out.println("\n--- Query 4: Graph Metadata ---");
            try {
                long nodeCount = db.getNodesQuantity();
                long edgeCount = db.getEdgesQuantity();
                
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

        } catch (GraphStorageException e) {
            System.err.println("A critical error occurred with the database:");
            e.printStackTrace();
        }
    }
}

```

you should get in your terminal: 

```text
Database opened successfully at: /home/user/your_project/my-graph-db

--- Ingesting Data ---
Ingestion complete.

--- Query 1: Get Node 'user_101' ---
Found node. Label: User
Property 'name': Ana
Property 'country': Mexico

--- Query 2: Find Nodes where 'country'='Chile' ---
  - Found Node ID: user_103
    Label: User
    Name: Pedro

--- Query 3: Get Neighbors of 'user_101' ---
  - Found Edge ID: edge_50
    Label: KNOWS
    user_101 --[KNOWS]--> user_102
  - Found Edge ID: edge_51
    Label: WORKS_WITH
    user_103 --[WORKS_WITH]--> user_101

--- Query 4: Graph Metadata ---
Total Nodes: 3
Total Edges: 2
Edge Counts by Label:
  - KNOWS: 1
  - WORKS_WITH: 1

```