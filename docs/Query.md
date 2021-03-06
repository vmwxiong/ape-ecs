# Query

Queries are the primary way of retrieving data in ApeECS.
Retrieve Entities based on Component composition, specific sets, and reverse references.
Most ECS implementations implement a Union query, which ApeECS does through it's `query.fromAll()` method.

```js
// find all of the entities with both Mass and Point Components/Tags
const entities = world.createQuery().fromAll('Mass', 'Point').execute();
```

👆 A Queries must include at least one from* method call or init option.

👆 `fromAll`, `fromAny`, `fromReverse`, `from`, `not`, `only`, `persist` can all be done in the creation factory's init `Object`.

👆 `from*`, `not`, and `only` methods do not distinguish between Component types and Entity tags.

```js
// functionally equivalent to the previous example
const entities = world.createQuery({ all: ['Mass', 'Point'] }).execute();
```

## create

There are two Query factories -- [world.createQuery](./World.md#createquery) and [system.createQuery](./System.md#createquery).
Each returns a Query instance.
The main difference is that a Query created from a System is associated with that system, and thus can be persisted to track changes.

A common pattern is to create your persisted queries in a System init.

```js
class ApplyMove extends ApeECS.System {

  init() {
    this.moveQuery = this.createQuery().fromAll('Sprite', 'Position', 'MoveAction').persist();
  }

  update(tick) {
    for (const entity of this.moveQuery.execute()) {
      for (const action of entity.getComponents('MoveAction')) {
        entity.c.Position.x += action.x;
        entity.c.Position.y += action.y;
        entity.removeComponent(action);
      }
    }
  }
}
```

### Query init object

You can pass an init object to world or system `createQuery({})`

* init
  * from: `Array`, equiv to from()
  * all: `Array`, equiv to fromAll()
  * any: `Array`, equiv to fromAny()
  * not `Array`, equiv to not()
  * only: `Array`, equiv to only()
  * persist: `bool`, equiv to persist()
  * trackAdd: `bool`, track added entities between system runs on persisted query with the query.added Set.
  * trackRemoved: `bool`, track removed entities between system runs on persisted query with query.removed Set.
  * includeApeDestroy: `bool`, if world.config.useApeDestoy is true, queries default to removing Entities with the `ApeDestroy` tag, but you can include them by setting this to `true`.

## fromAll

Using `fromAll` adds to the Query execution results Entities with at least all of the Component types/Tags listed.
It is literally a set union.

**Arguments**:
* ...types: `[]String|Component class`: _required_, array of strings that are the tags and Component types you require from an entity

**Returns**:
* `Query` instance for chaining methods.

```js
const query = world.createQuery().fromAll('Sprite', 'Position', 'MoveAction');
```

```js
const query = world.createQuery({
  all: ['Sprite', 'Position', 'MoveAction']
});
```

## fromAny

Query `execute` results include Entities with at least one of the tags or Component types listed.

```js
//must have Character Component type or tag and must have one or more of Sprite, Image, or New.
const query = world.createQuery().fromAll('Character').fromAny('Sprite', 'Image', 'New');
```

```js
const query = world.createQuery({
  all: ['Character'],
  any: ['Sprite', 'Image', 'New']
});
```

**Arguments**:
* ...types: `[]String|Component class`: _required_, array of strings that are the tags and Component types you require at least one of from an entity

**Returns**:
* `Query` instance for chaining methods.

## fromReverse

Query `execute` results must include entities that have Components that reference a given entity with a given Component type.

**Arguments**:
* entity: `Entity`, _required_, Entity instance that must be refered to by a Component.
* type: `String`, _required_, Component type that contains the reference to the entity.

**Returns**:
* `Query` instance for chaining methods.

```js
// get all of the entities that have a component indicating that they're in the player's inventory
const entities = world.createQuery().fromReverse(player, 'InInventory').execute();
```
```js
const query = world.createQuery({
  reverse: {
    entity: player,
    type: 'InInventory'
  }
});
```

## from

Limit the Query `execute` results to only include a subset of these specified entities.

**Arguments**:
* ...entities: `[]Entity`, _required_, Array of entity lists that is a superset of the results

**Returns**:
* `Query` instance for chaining methods.

```js
const query = world.createQuery().from([player, enemy1]);
```

```js
const query = world.createQuery({
  from: [player, enemy1]
});
```

## not

Limit Query `execute` results to not include Entities that have any of these Component types or tags.
`not()` filters results, and thus the query must include at least one of `from`, `fromAll` or `fromAny` as well.

**Arguments**:
* ...types: `[]String|Component class`, _required_, Array of Component types and Tags to disqualify result entities

**Returns**:
* `Query` instance for chaining methods.

```js
const query = world.createQuery().fromAll('Character', 'Sprite').not('Invisible', 'MarkedForRemoval');
```

```js
const query = world.createQuery({
  all: ['Character', Sprite],
  not: ['Invisible', 'MarkedForRemoval']
});
```

## only

Limit Query `execute` results to only include Entities that have at least one of these Component types or tags.
`only()` filters results, and thus the query must include at least one of `from`, `fromAll` or `fromAny` as well.

**Arguments**:
* ...types: `[]String|Component class`, _required_, Array of Component types and Tags to disqualify result entities

**Returns**:
* `Query` instance for chaining methods.

```js
const query = world.createQuery().fromAll('Character', 'Sprite').only('Invisible', 'MarkedForRemoval');
```

```js
const query = world.createQuery({
  all: ['Character', Sprite],
  only: ['Invisible', 'MarkedForRemoval']
});
```

## persist

Indicate that the Query should be persisted, turning it into a live index of Entities.

**Arguments**:
* trackAdded: `Boolean`, _optional_, flag to track new Entity results from the Query since the last `system.update`
* trackRemoved: `Boolean`, _optional_, flag to track removed Entity results from the Query since the last `system.update`

The properties `query.added` and `query.removed` are `Sets` that you can check during your `system.update` if tracked.

👆 A query can be persisted without having to track added or removed.
Whenever an Entity changes Component or Tag composition, it's checked against all persisted Queries when the `world.tick()` or after a `system.update(tick)` happens.

👆 Peristed queries only update their results after `system.update` or during `world.tick()`.
If you want to update your persisted queries at other times, run [world.updateIndexes()](./World.md#updateindexes).


⚠ Queries cannot be persisted if they use `from` a static set of Entities, or if they're not created from a System.

💭 If you persist a LOT of Queries, it can have a performance from creating Entities, or adding/removing Components or Tags.

## refresh

[execute](#execute) will not update results for changed entities, and persisted queries won't update within a single system update. To get new results in these situations, use refresh.

```js
const world.registerTags('A', 'B');
const entity1 = world.createEntity({ tags: ['A'] });
const query = world.createQuery().fromAll('A', 'B');
const results1 = query.execute(); // doesn't include entity1
entity1.addTag('B');
const results2 = query.execute(); // doesn't include entity1
const result3 = query.refresh().execute(); //does include entity1
```

👆 After each system run and each tick, [world.updateIndexes()](./World.md#updateindexes) which will refresh persisted queries from changed entities.
You can run this directly as well.

## execute

Execute a Query, returning all of the resulting Entities.

**Arguments**:
* filter: `Object`, _optional_, Filter Entities to results that had Component/tag composition changes since `updatedComponents` or Component value changes since `updatedValues`.

```js
const query = world.createQuery().fromAll('Character', 'Sprite');
//only include entities that have been updated last tick or this tick
const entities = query.execute({
  updatedComponents: world.currentTick - 1,
  updatedValues: world.currentTick - 1
})
```

⚠ If you neglect to call [component.update()](./Component.md#update) when you update the values of a Component, then `component.updated` and `entity.updatedValues` will not be updated, and the query filter for `updatedValues` will not be accurate.
