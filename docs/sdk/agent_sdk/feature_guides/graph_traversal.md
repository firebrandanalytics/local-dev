# Entity Graph Traversal Guide

## Overview

The FireFoundry SDK entity framework uses a graph-based architecture where entities are connected through typed edges. This guide explains how to navigate the entity graph by traversing these connections.

## Core Graph Navigation Concepts

### 1. Edge Collections

Every `EntityNode` has two edge collections:
- `edges_from`: Edges where this entity is the source (outgoing edges)
- `edges_to`: Edges where this entity is the destination (incoming edges)

```typescript
public edges_from: Partial<ArrayifyEdgeMap<ENH['eth'], ENH['edges_from']>>;
public edges_to: Partial<ArrayifyEdgeMap<ENH['eth'], ENH['edges_to']>>;
```

### 2. Edge Types

The framework provides several built-in edge types:
- `Contains`: Hierarchical containment relationships
- `Owns`: Ownership relationships
- `Calls`: Function/method call relationships
- `Dispatch`: Dynamic dispatch relationships
- `InResponseTo`: Response/reply relationships
- `ScheduledCall`: Scheduled execution relationships
- `QueuedWork`: Work queue relationships
- `TriggersRun`: Trigger relationships

### 3. EntityPointer

Edges use `EntityPointer` for lazy loading of connected entities:
```typescript
// EntityPointer provides lazy-loaded access to entities
const entityPointer = new EntityPointer('User', userId);
const entity = await entityPointer.val(); // Loads the actual entity
```

## Finding Connected Nodes - From Source Entity

### Example 1: Finding All Contained Entities

```typescript
async function findContainedEntities(sourceEntity: EntityNode): Promise<EntityNode[]> {
    // Ensure the entity is loaded with its edges
    await sourceEntity.load();
    
    // Get all "Contains" edges from this entity
    const containsEdges = sourceEntity.edges_from.Contains || [];
    
    // Navigate to the contained entities
    const containedEntities: EntityNode[] = [];
    
    for (const edge of containsEdges) {
        // Each edge has get_to() method to access the destination entity
        const containedEntity = await edge.get_to();
        containedEntities.push(containedEntity);
    }
    
    return containedEntities;
}

// Usage
const container = factory.get_entity_known_type('Container', containerId);
const contained = await findContainedEntities(container);
console.log(`Found ${contained.length} contained entities`);
```

### Example 2: Finding Specific Edge Type with Filtering

```typescript
async function findOwnedEntitiesOfType<T extends keyof ETH['constructors']>(
    owner: EntityNode,
    targetType: T
): Promise<InstanceType<ETH['constructors'][T]>[]> {
    await owner.load();
    
    // Get all "Owns" edges
    const ownsEdges = owner.edges_from.Owns || [];
    
    const ownedEntities: InstanceType<ETH['constructors'][T]>[] = [];
    
    for (const edge of ownsEdges) {
        const ownedEntity = await edge.get_to();
        const dto = await ownedEntity.get_dto();
        
        // Filter by specific type
        if (dto.specific_type_name === targetType) {
            ownedEntities.push(ownedEntity as InstanceType<ETH['constructors'][T]>);
        }
    }
    
    return ownedEntities;
}

// Usage: Find all conversations owned by a user
const user = factory.get_entity_known_type('User', userId);
const conversations = await findOwnedEntitiesOfType(user, 'Conversation');
```

### Example 3: Following Call Chains

```typescript
async function findAllCalledEntities(caller: EntityNode): Promise<{
    entity: EntityNode;
    edge: EntityEdge;
    edgeData: any;
}[]> {
    await caller.load();
    
    const callsEdges = caller.edges_from.Calls || [];
    const results = [];
    
    for (const edge of callsEdges) {
        const calledEntity = await edge.get_to();
        const edgeDto = await edge.get_dto();
        
        results.push({
            entity: calledEntity,
            edge: edge,
            edgeData: edgeDto.data
        });
    }
    
    return results;
}
```

## Finding Connected Nodes - From Destination Entity

### Example 4: Finding All Entities That Contain This Entity

```typescript
async function findContainerEntities(containedEntity: EntityNode): Promise<EntityNode[]> {
    // Ensure entity is loaded with incoming edges
    await containedEntity.load();
    
    // Get all incoming "Contains" edges
    const containedByEdges = containedEntity.edges_to.Contains || [];
    
    const containers: EntityNode[] = [];
    
    for (const edge of containedByEdges) {
        // Use get_from() to access the source entity
        const containerEntity = await edge.get_from();
        containers.push(containerEntity);
    }
    
    return containers;
}

// Usage
const item = factory.get_entity_known_type('Item', itemId);
const containers = await findContainerEntities(item);
console.log(`Item is contained in ${containers.length} containers`);
```

### Example 5: Finding All Owners of an Entity

```typescript
async function findAllOwners(ownedEntity: EntityNode): Promise<{
    owner: EntityNode;
    ownershipDetails: any;
}[]> {
    await ownedEntity.load();
    
    // Get incoming "Owns" edges
    const ownedByEdges = ownedEntity.edges_to.Owns || [];
    
    const owners = [];
    
    for (const edge of ownedByEdges) {
        const owner = await edge.get_from();
        const edgeDto = await edge.get_dto();
        
        owners.push({
            owner: owner,
            ownershipDetails: edgeDto.data
        });
    }
    
    return owners;
}
```

### Example 6: Finding Entities That Call This Entity

```typescript
async function findCallers(targetEntity: EntityNode): Promise<EntityNode[]> {
    await targetEntity.load();
    
    // Get incoming "Calls" edges
    const calledByEdges = targetEntity.edges_to.Calls || [];
    
    const callers: EntityNode[] = [];
    
    for (const edge of calledByEdges) {
        const caller = await edge.get_from();
        callers.push(caller);
    }
    
    return callers;
}
```

## Advanced Graph Traversal Patterns

### Example 7: Multi-Hop Traversal

```typescript
async function findDescendantsRecursive(
    entity: EntityNode,
    edgeType: string,
    maxDepth: number = 5,
    visited: Set<string> = new Set()
): Promise<EntityNode[]> {
    if (maxDepth <= 0 || visited.has(entity.id)) {
        return [];
    }
    
    visited.add(entity.id);
    await entity.load();
    
    const descendants: EntityNode[] = [];
    const edges = entity.edges_from[edgeType] || [];
    
    for (const edge of edges) {
        const child = await edge.get_to();
        descendants.push(child);
        
        // Recursively find descendants
        const childDescendants = await findDescendantsRecursive(
            child,
            edgeType,
            maxDepth - 1,
            visited
        );
        descendants.push(...childDescendants);
    }
    
    return descendants;
}

// Usage: Find all descendants in a containment hierarchy
const root = factory.get_entity_known_type('Container', rootId);
const allDescendants = await findDescendantsRecursive(root, 'Contains');
```

### Example 8: Breadth-First Path Finding

```typescript
async function findPath(
    startEntity: EntityNode,
    endEntity: EntityNode,
    edgeType: string,
    maxHops: number = 10
): Promise<EntityNode[] | null> {
    const queue: { entity: EntityNode; path: EntityNode[] }[] = [
        { entity: startEntity, path: [startEntity] }
    ];
    const visited = new Set<string>([startEntity.id]);
    
    while (queue.length > 0 && queue[0].path.length <= maxHops) {
        const { entity: currentEntity, path } = queue.shift()!;
        
        if (currentEntity.id === endEntity.id) {
            return path;
        }
        
        await currentEntity.load();
        const edges = currentEntity.edges_from[edgeType] || [];
        
        for (const edge of edges) {
            const nextEntity = await edge.get_to();
            
            if (!visited.has(nextEntity.id)) {
                visited.add(nextEntity.id);
                queue.push({
                    entity: nextEntity,
                    path: [...path, nextEntity]
                });
            }
        }
    }
    
    return null; // No path found
}
```

### Example 9: Working with Dispatch Edges

**What are Dispatch Edges?**

Dispatch edges implement dynamic method invocation in the entity graph. They allow entities to define callable functions that can be invoked remotely by other entities. This enables:

- **Dynamic API Endpoints**: Entities can expose callable methods through dispatch edges
- **Microservice-like Architecture**: Entities act as services with discoverable functions
- **Workflow Orchestration**: Complex processes can dispatch work to specialized entities
- **Plugin Systems**: New functionality can be added by creating entities with dispatch targets

Each dispatch edge contains a `function_name` in its data that identifies which specific function to call on the target entity.

**Built-in Memoization**: A key feature of dispatch edges is that they provide automatic caching/memoization, but it's **per-source-node** rather than per-arguments. This means:

- Once a source entity dispatches to a target entity, the connection and result are cached
- Subsequent dispatches from the same source entity to the same target will use the cached relationship
- This is different from traditional function memoization (which caches based on input arguments)
- The caching happens at the graph relationship level, making repeated dispatches from the same source very efficient
- Different source entities dispatching to the same target maintain separate cached relationships

**Modeling Optional Computations**: This caching behavior makes dispatch edges excellent for modeling optional or conditional computations:

- **Lazy Evaluation**: Computations only occur when first requested by a source entity
- **Persistent Results**: Once computed for a source, the result persists in the graph relationship
- **Conditional Processing**: Different entities can have different computational needs - some may dispatch to expensive analysis functions, others may not
- **Incremental Complexity**: Systems can start simple and add optional computational capabilities through dispatch edges as needed

For example, a document entity might optionally dispatch to sentiment analysis, keyword extraction, or summarization services only when those capabilities are needed by specific workflows, with results cached for future use by the same source.

```typescript
// Example: Entity with dispatch capabilities using the @EntityDispatcherDecorator
@EntityDispatcherDecorator({
    "process_data": {
        entityClass: DataProcessor,
        entityType: "DataProcessor"
    },
    "validate_input": {
        entityClass: InputValidator, 
        entityType: "InputValidator"
    }
})
class WorkflowOrchestrator extends EntityNode<WorkflowOrchestratorTypeHelper> {
    // This entity can dispatch to "process_data" and "validate_input" functions
}

async function findDispatchTargets(
    dispatcherEntity: EntityNode,
    functionName?: string
): Promise<{
    entity: EntityNode;
    functionName: string;
}[]> {
    await dispatcherEntity.load();
    
    const dispatchEdges = dispatcherEntity.edges_from.Dispatch || [];
    const targets = [];
    
    for (const edge of dispatchEdges) {
        // Cast to EntityEdgeDispatch to access function_name
        const dispatchEdge = edge as EntityEdgeDispatch<any, any, any>;
        const targetEntity = await dispatchEdge.get_to();
        const edgeFunctionName = await dispatchEdge.get_function_name();
        
        // Filter by function name if specified
        if (!functionName || edgeFunctionName === functionName) {
            targets.push({
                entity: targetEntity,
                functionName: edgeFunctionName
            });
        }
    }
    
    return targets;
}

// Usage examples:
// Find all entities that can process data
const dataProcessors = await findDispatchTargets(orchestrator, 'process_data');

// Find all dispatch targets (any function)
const allTargets = await findDispatchTargets(orchestrator);

// Actual dispatch usage (from the EntityDispatcher pattern):
const result = await orchestrator.run_dispatch('process_data', { data: inputData });
```

## Best Practices

### 1. Always Load Before Traversing
```typescript
// ✅ Good: Load entity before accessing edges
await entity.load();
const edges = entity.edges_from.Contains || [];

// ❌ Bad: Accessing edges without loading
const edges = entity.edges_from.Contains || []; // May be empty/stale
```

### 2. Handle Missing Edges Gracefully
```typescript
// ✅ Good: Provide default empty array
const edges = entity.edges_from.Contains || [];

// ✅ Good: Check for existence
if (entity.edges_from.Contains) {
    for (const edge of entity.edges_from.Contains) {
        // Process edge
    }
}
```

### 3. Use Lazy Loading Efficiently
```typescript
// ✅ Good: Load entities as needed
for (const edge of edges) {
    const connectedEntity = await edge.get_to(); // Lazy loaded
    // Process entity
}

// ⚠️ Consider: Pre-load if accessing multiple properties
const entities = await Promise.all(
    edges.map(edge => edge.get_to())
);
```

### 4. Avoid Infinite Loops in Recursive Traversal
```typescript
// ✅ Good: Track visited entities
const visited = new Set<string>();

async function traverse(entity: EntityNode): Promise<void> {
    if (visited.has(entity.id)) return;
    visited.add(entity.id);
    
    // Process entity and continue traversal
}
```

### 5. Type Safety with Edge Navigation
```typescript
// ✅ Good: Use type assertions when you know the type
const userEntity = (await edge.get_to()) as User;

// ✅ Better: Check type before casting
const entity = await edge.get_to();
const dto = await entity.get_dto();
if (dto.specific_type_name === 'User') {
    const userEntity = entity as User;
    // Safe to use as User
}
```

## Common Patterns Summary

| Pattern | From Entity | To Entity | Use Case |
|---------|-------------|-----------|----------|
| **Outgoing Navigation** | `entity.edges_from[edgeType]` → `edge.get_to()` | Get entities this entity points to |
| **Incoming Navigation** | `entity.edges_to[edgeType]` → `edge.get_from()` | Get entities that point to this entity |
| **Hierarchical Traversal** | Use `Contains` edges with recursion | Navigate parent-child relationships |
| **Ownership Queries** | Use `Owns` edges bidirectionally | Find owners or owned entities |
| **Call Chain Analysis** | Use `Calls` edges in either direction | Trace execution flows |
| **Dynamic Dispatch** | Use `Dispatch` edges with function filtering | Find dispatch targets |

This graph traversal system provides a powerful way to navigate complex entity relationships while maintaining type safety and lazy loading for performance.