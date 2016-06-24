## Promised Dynamo DB Wrapper

A promise based fluent API around the Dynamo DB.  Operations are performed using simple Javascript objects which are translated 
by the service into DynamoDB's data structure.  Similarly, operation results are provided as Javascript objects translated from the 
 DynamoDB data structure.  See Data Types below for an explanation of the translation. 
 
This project is currently in alpha and as such does not expose all the functionality of the AWS SDK.  Known issues and 
limitations below call out some of current omissions and works in progress.

### Instantiation

```
DynamoDB( options, tables )
```

```
var dynamodb = new require( 'promised-dynamo' )( { accessKeyId: "ABCDEFGHIJK", secretAccessKey: "abc123", region: "us-east-1" }, [ 'tablename', 'anothertablename' ] );
```

Or

```
var AWS = require( 'aws-sdk' );
var dynamodb = new require( 'promised-dynamo' )( { db: new AWS.DynamoDB() }, [ 'tablename', 'anothertablename' ] );
```

#### Options

Option Name | Type | Description 
----------- | ---- | ----------- 
accessKeyId | String | AWS Access Key ID
secretAccessKey | String | AWS Secret Access Key associated with the provided Access Key ID
region | String | AWS Region

#### Table Configuration

The second argument to the DynamoDB factory function is an Array defining the tables which will be exposed by the 
returned object.  Table definitions may take two forms: strings and alias objects.

##### Table Definition Strings

If a string is provided as a table definition it is expected to be the name of a table in DynamoDB.  Operations on the 
table will be accessible at a property with the same name as the table.  For example, 

```
var dynamodb = require( 'promised-dynamo' )( { accessKeyId: "ABCDEFGHIJK", secretAccessKey: "abc123", region: "us-east-1" }, [ 'mytable' ] );

dynamodb.mytable.getItem( 'abc' )
    .then( function( item ) {
        console.log( item );
    } );
```

##### Table Alias Definitions

A table alias definition may be provided in place of a table name string in the Array of table definitions.  An alias 
definition indicates both the name of the DynamoDB table and the property name via which the table will be accessible 
on the resultant object.  For example,

```
var dynamodb = require( 'promised-dynamo' )( 
    { accessKeyId: "ABCDEFGHIJK", secretAccessKey: "abc123", region: "us-east-1" }, 
    [ { name: "dev-myTable", alias: "mytable" } ] );
    
dynamodb.mytable.getItem( 'abc' )
    .then( function( item ) {
        console.log( item );
    } );
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

##### Returns

A promise which, upon success, resolves to a Javascript object representing the found item.  If no item matched the 
hash and range specified the promise will resolve to `null`.

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

Limits the evaluated rows to the number provided.  WARNING, Limit does not limit results, it mirrors DynamoDB's Limit option and as such limits the number of records evaluated during the query.

```
dynamodb.tablename.query( '12345' ).filter( { created: { ">": 1427517440482 } } ).then( function( results ) {
    results.forEach( console.log );
} );
```

Filters the query results using the provided filter object.  See Filter Object below for a detailed description of the filter object.

```
dynamodb.tablename.query( '12345' ).reverse().then( function( results ) {
    results.forEach( console.log );
} );
```

Reverses the order in which the items in the results are sorted based on the Sort Key.  

##### Returns 

A promise which, upon success, resolves to a Javascript Array of the Javascript objects representing the results 
of the query.  If no records match the query parameters the promise resolves to an empty Array.

##### Known Issues / Limitations

* Currently if the size of the result set exceeds the maximum return record number there is no mechanism for requesting the next "page" of results.

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

##### Returns 

A promise which, upon success, resolves to a Javascript Array of the Javascript objects representing the results 
of the scan.  If no records match the scan criteria the promise resolves to an empty Array.

##### Known Issues / Limitations

* Currently if the size of the result set exceeds the maximum return record number there is no mechanism for requesting the next "page" of results.

#### putItem(item, options)

```
dynamodb.tablename.putItem( 
    { id: '3a', email: 'john@johnson.com', someNumbers: [ 1, 3883, 2983 ] }, 
    { returnValues: "ALL_OLD" } )
    .then( function( result ) {
        console.log( JSON.stringify( result.item );
    } );
```

Executes a putItem request which will either add the new item or replace the existing item if an item already exists with the same key(s).
Upon success the promise will resolve to the data object returned from the AWS SDK.

##### Options

Option Name | Type | Description 
----------- | ---- | ----------- 
conditionExpression | Object | Allows for the definition of an expression to be evaluated in determining whether to execute the put request.  See Filter Object below for a definition of the form of a condition expression.
returnConsumedCapacity | String | Indicates which aspects of capacity should be reported as part of the results.  See Consumed Capacity constants below for valid options.   
returnItemCollectionMetrics | String | Indicates which metrics should be provided as part of the results.  See Item Collection Metrics constants below for valid options. 
returnValues | String | Defines whether values should be provided as part of the put result.  The only supported options for putItem are `NONE` and `ALL_OLD`.  When set to an option other than `NONE` the resolved data object will include an `item` property containing the returned attributes mapped to a Javascript object.

##### Returns

A promise which, upon success, resolves to the `data` object returned from the AWS SDK during a put.  See Options above for 
considerations concerning the information in this result.

#### deleteItem(hash, range)

```
dynamodb.tablename.deleteItem( '2a', '123' ).then( function() {
    console.log( 'success' );
} );
```

Executes a deleteItem request removing the identified item if it exists.  

##### Returns

A promise which resolves upon successful deletion of the identified item.

#### updateItem(hash, range, updateExpression, options)

```
dynamodb.tablename.updateItem( '2a', null, { newProp: 6 } ).then( function() {
    console.log( 'success' );
} );
```

Executes an updateItem request updating the identified item or creating a new item if one does not exist.  See Update Expression 
below for details concerning the form of the `updateExpression` parameter. 

##### Options

Option Name | Type | Description 
----------- | ---- | ----------- 
conditionExpression | Object | Allows for the definition of an expression to be evaluated in determining whether to execute the update request.  See Filter Object below for a definition of the form of a condition expression.
returnConsumedCapacity | String | Indicates which aspects of capacity should be reported as part of the results.  See Consumed Capacity constants below for valid options.   
returnItemCollectionMetrics | String | Indicates which metrics should be provided as part of the results.  See Item Collection Metrics constants below for valid options. 
returnValues | String | Defines whether values should be provided as part of the put result.  See Return Values constants below for valid options.  When set to an option other than `NONE` the resolved data object will include an `item` property containing the returned attributes mapped to a Javascript object.

##### Returns

A promise which, upon success, resolves to the `data` object returned from the AWS SDK during an update.  See Options above for 
considerations concerning the information in this result.

### Update Expression

```
{
    SET: {
        type: "Tacos"
    },
    ADD: {
        coolness: 5
    },
    REMOVE: [ "rating", "tightness" ],
    DELETE: {
        favorites: [ "burgers" ]
    }
}
```

The `updateItem` function takes as one of its parameters an Update Expression object.  Such an object defines the 
updates to be made and allows for the functionality exposed by the [AWS Update Expression](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.Modifying.html). 
The keys of the Update Expression object represent the operation being undertaken.  Any number of expression types may 
be set in tandem and any number of expressions may be declared under an expression type.

#### SET Expressions

SET expressions indicate that a particular property should be set to a particular value.  The value may be any of the 
types listed in Data Types below.  It is not actually necessary to indicate the SET expression and keys of the Update Expression 
object which are not an operator identifier (`SET`, `ADD`, or `REMOVE`) will be treated as though they are declared 
under the `SET` operator.  This simplifies the common case of setting properties on an object.

```
{ 
    SET: {
        type: "Tacos"
    }
}
```

Is equivalent to

```
{
    type: "Tacos"
}
```

When setting nested properties use a string key to identify the fully qualified nested property name.

```
{ 
    SET: {
        "a.b": "c"
    }
}
```

#### ADD Expressions

ADD operations are supported for Number and Set data types.  As such it can be used to add a value to a Number or to add 
an item to a Set. 

```
{
    ADD: {
        someNumber: 7.3,
        someStringSet: "D"
    }
}
```

Note: The official [Update Expression documentation](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.Modifying.html#Expressions.Modifying.UpdateExpressions.ADD) 
discourages the use of the `ADD` operator.

#### REMOVE Expressions

REMOVE expressions are used to denote that a property should be completely removed from a record.  Multiple properties 
may be specified for removal by providing an Array to the REMOVE operation.

```
{
    REMOVE: "property1"
}
```

```
{
    REMOVE: [ "property1", "property2", "property3" ]
}
```

#### DELETE Expressions

DELETE expressions are used to delete items from String Sets or Number Sets.  The value deleted must be an appropriately 
typed Set.

```
{
    DELETE: { letters: [ "d", "e" ] }
}
```

### Filter Object

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

#### Simple Expression Definitions

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

#### Composite Expression Definitions

Composite expressions represent multiple expressions which are ANDed or ORed together.  These expressions take the following form.

```
AND|OR: [
    expression,
    expression,
    ...
]
```

The AND or OR expression is paired with an Array of expressions.  Each of these contained expressions may be either a simple expression or a composite expression.


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

Lists are only partially supported and the API around this is likely to change in the future.  Lists coming from Dynamo 
will be converted into an Array of the contained values.  Lists being fed to Dynamo have the following limitations 
currently: 

* Only lists of objects are supported
* Empty lists are not supported


```
{
  propertyName: [
    {
      key: value
    },
    {
      key: value
    }
  ]
}
```

### Utilities

#### DynamoDB.mapDynamoObjectToJavascriptObject( object )

Takes a Dynamo DB Attributes object and maps it to a Javascript object. 

#### DynamoDB.mapJavascriptObjectToDynamoObject( object ) 

Takes a Javascript object and transforms it into a Dynamo DB Attributes object.

### Constants

#### Consumed Capacity Options

Enumeration of valid option values for the `returnConsumedCapacity` option.

* `dynamoDb.consumedCapacityOptions.INEXES`
* `dynamoDb.consumedCapacityOptions.TOTAL`
* `dynamoDb.consumedCapacityOptions.NONE`

#### Item Collection Metrics Options

Enumeration of valid option values for the `returnItemCollectionMetrics` option.

* `dynamoDb.itemCollectionMetricsOptions.SIZE`
* `dynamoDb.itemCollectionMetricsOptions.NONE`

#### Return Values Options

Enumeration of valid option values for the `returnValues` option.

* `dynamoDb.valuesOptions.NONE`
* `dynamoDb.valuesOptions.ALLOLD`
* `dynamoDb.valuesOptions.UPDATEDOLD`
* `dynamoDb.valuesOptions.ALLNEW`
* `dynamoDb.valuesOptions.UPDATEDNEW`
