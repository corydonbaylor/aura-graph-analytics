## Loading in the Data

```cypher

CREATE INDEX id_index for (u:User) ON (u.id)

LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/corydonbaylor/aura-graph-analytics/refs/heads/main/fraud_p2p/raw_data/p2p_users.csv' AS row
MERGE (u:User {id: toInteger(row.nodeId)})
SET u.fraud_transfer_flag = toInteger(row.fraud_transfer_flag),
    u.ip_count = toInteger(row.ip_count),
    u.card_count = toInteger(row.card_count),
    u.device_count = toInteger(row.device_count);

// Load Transactions
LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/corydonbaylor/aura-graph-analytics/refs/heads/main/fraud_p2p/raw_data/p2p_transactions.csv' AS row
MATCH (source:User {id: toInteger(row.sourceNodeId)})
MATCH (target:User {id: toInteger(row.targetNodeId)})
MERGE (source)-[t:TRANSACTION]->(target)
SET t.amount = toFloat(row.transaction_amount),
    t.datetime = datetime(row.transaction_datetime);
```

Then we need to create a projection:

```python
query = """
CALL {
    MATCH (source:User)-[rel:TRANSACTION]->(target:User)
    WITH source, target, SUM(rel.amount) AS total_amount
    MERGE (source)-[aggRel:AGGREGATED_TRANSACTION]->(target)
    SET aggRel.amount = total_amount
    RETURN source, aggRel, target
}
RETURN gds.graph.project.remote(
    source, target, 
    {
        sourceNodeLabels: labels(source),
        targetNodeLabels: labels(target),
        relationshipType: 'aggRel',
        relationshipProperties: {amount: 'total_amount'}  
    }
);
"""

# Project the graph into GDS
full, result = gds.graph.project(
    graph_name="full-graph",
    query=query
)
```

