# RDF Cube Schema

## Core Schema

The _RDF Cube Schema_ defines a minimal set of classes and properties necessary to represent multi-dimensional arrays of data in RDF.

The core schema is very simple and almost unconstrained from an RDF perspective. This ensures it can be used in many ways and does not restrict its usefulness due to too rigorous definitions.

Prefix: `cube`
Namespace: `http://ns.bergnet.org/cube/`

### Classes

* `cube:Cube`: Represents the entry point for a collection of observations, conforming to some common dimensional structure.
* `cube:Observation`: A single observation in the cube, may have one or more associated dimensions.
* `cube:ObservationSet`: A set of observations.
* `cube:Constraint`: Specifies constraints that need to be met on the Cube. Used for metadata and validation. (Optional)

`Cube` and `Observation` are pretty much self-describing. All `Observation`s somehow linked with a `Cube` need to adhere to the same dimensional structure.

An `ObservationSet` is a structure that acts as a container for multiple `Observation`s. It can be used to group any set of `Observation`s, as long as they use the same dimensions. There is on purpose no stronger semantics attached to this set, to make sure it can be used in almost any scenario. A cube can have one or more.

### Properties

* `cube:observationSet`: Connects a cube with a set of observations.
* `cube:observationConstraint`: Connects a cube with a constraint for metadata and validation.
* `cube:observation`: Connects a set of observations with a single observation. 
* `cube:observedBy`: Indicates how this observation was gathered. This can be a description of the method, link to a description of the sensor, etc.

### Additional features not part of the core

A `Constraint` for a cube. A Constraint is optional but recommended, it is used to:
* Define how data (`Observation`s) in a `Cube` can be validated.
* Add Cube-specific metadata (custom labels, translation to other languages, etc).

A `View` can be used to combine data from multiple `Cube`s, see its own section for details.

Annotations are not part of the core-model of `Cube`s, see its own section for details.

### Dimensions

> A dimension is a structure that categorizes facts and measures to enable users to answer business questions. Commonly used dimensions are people, products, place, and time ([Source: Wikidata](https://en.wikipedia.org/wiki/Dimension_(data_warehouse))).

In _RDF Cube Schema_, facts, measures, and categories are all considered a dimension.

All `Observation`s need to provide the same set of dimensions, they cannot be optional. This ensures that cubes can be queried efficiently.

Unlike other RDF vocabularies in that domain, there is no specific class for a dimension. Creating a new _RDF Cube Schema_ dimension would be the same as defining a new [RDF Property](https://www.w3.org/TR/rdf-schema/#ch_property). 

This encourages re-use and makes it much easier to cherrypick on existing RDF properties and use them as dimensions. Obvious examples are temporal properties like `schema:validFrom`, `dcterms:date`, `dcterms:temporal`, etc.

In general, any RDF Property can be considered for describing a dimension, except for heavily constrained properties that might lead to unwanted conclusions (inference) by [reasoners](http://www.w3.org/TR/sparql11-entailment/).

> Note that choosing a particular dimension will have implications. For querying cubes via SPARQL, spatial and temporal dimensions might only be filtered properly if the datatype used (i.e. `xsd:date`, `geo:asWKT`) is supported/optimized by the SPARQL endpoint. This is for example mostly _not_ the case for fragment datatypes like `xsd:gYear` etc.
>
> It has to be ensured that properties are not attached at the wrong level. Spatial dimensions for example are most likely *not* attached to the observation directly but to an instance of a dimension referenced in the observation.

## Metadata and Validation (Constraint)

_RDF Cube Schema_ supports attaching a "shape", or "constraint" to a cube. The constraints itself are expressed using the [Shapes Constraint Language (SHACL)](https://www.w3.org/TR/shacl/).

Providing shape and constraints for a cube facilitates the interpretation and validation of the cube for tooling and libraries.

From a pure publishing point of view and according to the [Open World Model](http://linked-data-training.zazuko.com/Ontologies/index.html#14), publishing observations using the core _RDF Cube Schema_ is enough. However, in reality one might want to use the data for other purposes as well, like visualizing it in web applications and other publications. To be useful, such tools might require additional metadata and cubes that adhere to certain constraints.

It is up to the consumer of the data to decide how the shape is used. If provided, it should be possible to validate the observations in the cube with it. 

### Shapes

A `cube:Constraint` is also a `sh:NodeShape`. SHACL is used to restrict the cube to a particular structure. The shape presented in this section can be used to validate all `Observation`s in the cube.

If additional restrictions are needed, additional restrictions and shapes could be provided.

The following snipped defines a new shape, it applies to all `cube:Observation`:

```turtle
<example-cube/cube/shape> a sh:NodeShape, cube:Constraint ;
  sh:closed true ;
  sh:property [
    sh:path rdf:type ;
    sh:nodeKind sh:IRI ;
    sh:in ( cube:Observation )
  ]
```

For each dimension (facts, measures, and categories) an additional `sh:property` should be provided. The dimension/property is referenced using `sh:path`.

Additional metadata like labels can be attached to the `sh:property` node.

Additional constraints can be added according to the SHACL specification, for example datatypes, min/max values, etc.
The following snippet defines a dimension (property) `dc:date` with a literal value (`xsd:dateTime`):

```turtle
[
  schema:name "Date and Time"@en, "Datum und Zeit"@de, "Date et heure"@fr; # optional, label of the vocab can be used
  sh:path dc:date ;
  sh:datatype xsd:dateTime ;
  sh:minCount 1 ;
  sh:maxCount 1 ;
]
```

Dimensions that point to objects like code lists (i.e taxonomies represented in vocabularies like [SKOS](https://www.w3.org/TR/skos-primer/)) can be expressed as well:

```turtle
[
  schema:name "Room"@en, "Raum"@de , "Pièce"@fr ;
  sh:path ex:room ;
  sh:maxCount 1 ;
  sh:in ( <building1/level1/room1> <building1/level1/room2> ) ;
]
```

### Generating Shapes

It is possible to generate a minimal SHACL shape given a `Cube` and a set of `Observation`s.

A SPARQL query to generate it is provided.

## Validate the cube

You can validate the cube with a cli tool written in Node.js.

Install the package dependencies: `npm i`

Validate `cube.ttl` by using the SHACL shape in `shape.ttl`: 

    ./bin/rdf-cube-schema.js validate cube.ttl shape.ttl