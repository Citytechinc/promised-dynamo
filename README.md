## Promised Dynamo DB Wrapper

A promise based fluent API around the Dynamo DB.  Operations are done using simple JSON objects which are translated by the service into DynamoDB's API.

### Instantiation

```
var dynamodb = require( './services/dynamodb' )( { accessKeyId: "ABCDEFGHIJK", secretAccessKey: "abc123", region: "us-east-1" }, [ 'tablename', 'anothertablename' ] );
```

### Usage

#### getItem(hash[, range])

```
dynamodb.tablename.getItem('129833448').then( function(result) {
    console.log(result);
} );
```

Executes a getItem request against the provided hash index.  This method will error if your table has a hash and range primary index and only a hash is provided.

```
dynamodb.tablename.getItem('john', 'tacos').then( function(result) {
    console.log(result);
} );
```

Executes a getItem request against the provided hash and range values.

#### query(hash[, range[, index]])

```
dynamodb.tablename.query( '12345' ).then( function( results ) {
    results.forEach( console.log );
} );
```

Executes a query against the provided hash index.  If multiple items match this hash all of them will be returned.

```
dynamodb.tablename.query( '12345', 'abcde' ).then( function( results ) {
    results.forEach( console.log );
} );
```

Executes a query against the provided hash and range.  This form is essentially the same as getItem(hash, range) but will return results in an Array.

```
dynamodb.tablename.query( '12345', '2015-05-06', 'id-and-date' ).then( function( results ) {
    results.forEach( console.log );
} );
```

Executes a query against the provided hash and range using the identified local or secondary index.

```
dynamodb.tablename.query( '12345', null, 'id-and-date' ).then( function( results ) {
    results.forEach( console.log );
} );
```

Executes a query against the provided hash using the identified local or secondary index.  If multiple results are returned they will be sorted by the range index of the secondary index.

```
dynamodb.tablename.query( '12345' ).limit( 3 ).then( function( results ) {
    results.forEach( console.log );
} );
```

Limits the results of a query to the number provided.

```
dynamodb.tablename.query( '12345' ).filter( { created: { ">": 1427517440482 } } ).then( function( results ) {
    results.forEach( console.log );
} );
```

Filters the query results using the provided filter object.  See filtering below for a detailed description of the filter object.

##### Filter Object

```
{
  dayOfWeek: { IN: [ 1, 2, 3, 4, 5 ] },
  time: { BETWEEN: [ 9, 20 ] },
  type: "Tacos",
  rating: { ">=": 3 },
  externalId: { "CONTAINS": 1189839 },
  OR: [
    {
      coolness: { ">": 2 }
    },
    {
      AND: [
             { tightness: { ">": 5 } },
             { yelpRated: true }
           ]
    }
  ]
}
```

A query's index based results may be filtered based on a filter object.  A filter object is made up of a series of expression definitions which are ANDed together.  A single expression definition is either a simple expression definition or a composite expression definition.

###### Simple Expression Definitions

A simple expression applies an operator to a named property and a value.  Such an expression may  take one of two forms.

```
propertyName: { OPERATOR: value }
```

```
propertyName: value
```

The later of these forms implicitly uses the "=" operator.  The following are valid operators:

* = - Expects a single value
* \> - Expects a single value
* \>= - Expects a single value
* < - Expects a single value
* <= - Expects a single value
* <> - Expects a single value
* IN - Expects an array of values
* BETWEEN - Expects an array containing exactly two values where the first value is the bottom of the range and the later value is the top of the range
* CONTAINS - Expects a single value

###### Composite Expression Definitions

Composite expressions represent multiple expressions which are ANDed or ORed together.  These expressions take the following form.

```
AND|OR: [
    expression,
    expression,
    ...
]
```

The AND or OR expression is paired with an Array of expressions.  Each of these contained expressions may be either a simple expression or a composite expression.

#### scan(filterDefinition)

```
dynamodb.tablename.scan( { someNumbers: { CONTAINS: 17 } } ).then( function( results ) {
    results.forEach( console.log );
} );
```

Executes a scan using the provided filter definition.  See Filter Objects above for an explanation of the form of a filter definition object.

```
dynamodb.tablename.scan( { someNumbers: { CONTAINS: 17 } } ).limit( 5 ).then( function( results ) {
    results.forEach( console.log );
} );
```

Limits the results of the scan to the limit number provided. 

#### putItem(item)

```
dynamodb.tablename.putItem( { id: '3a', email: 'john@johnson.com', someNumbers: [ 1, 3883, 2983 ] } ).then( function() {
    console.log( 'success' );
} );
```

Executes a putItem request which will either add the new item or replace the existing item if an item already exists with the same key(s).

#### updateItem(hash, range, itemUpdates)

```
dynamodb.tablename.updateItem( '2a', null, { newProp: 6 } ).then( function() {
    console.log( 'success' );
} );
```

Executes an updateItem request updating the identified item or creating a new item if one does not exist.

### Data Types

The following data types are currently supported

#### Strings - S

```
{ propertyName: "string value" }
```

#### String Sets - SS

```
{ propertyName: [ "string value 1", "string value 2" ] }
```

#### Numbers - N

```
{ propertyName: 123 }
```

#### Number Sets - NS

```
{ propertyName: [ 123, 456 ] }
```

#### Boolean - B

```
{ propertyName: true }
```

#### Maps - M

Nested objects are treated as Maps.  Maps may be nested to an arbitrary depth.  Lists may also be nested in Maps.

```
{ 
  propertyName: {
    subPropertyOne: "abc",
    subPropertyTwo: "def"
  }
}
```

#### Lists - L