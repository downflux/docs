# Pathing API

## Pathfinding API

### Hierarchical Path Finding (Botea 2004)
### Flow Field Tiles (Emerson 2020)

## Map Editing

* `AddStructure(timestamp, coordinate, structure_type) -> None`
* `RemoveStructure(timestamp, coordinate) -> None`
* `GetMap(timestamp) -> Map`
* `stream GetTileUpdates() -> []{[]Tile, timestamp}`
