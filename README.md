# node-mssql [![Dependency Status](https://david-dm.org/patriksimek/node-mssql.png)](https://david-dm.org/patriksimek/node-mssql) [![NPM version](https://badge.fury.io/js/mssql.png)](http://badge.fury.io/js/mssql)

An easy-to-use MSSQL database connector for Node.js.

There are some TDS modules which offer functionality to communicate with MSSQL databases but none of them does offer enough comfort - implementation takes a lot of lines of code. So I decided to create this module, that make work as easy as it could without losing any important functionality. node-mssql uses other TDS modules as drivers and offer easy to use unified interface. It also add extra features and bug fixes.

There is also [co](https://github.com/visionmedia/co) wrapper available - [co-mssql](https://github.com/patriksimek/co-mssql).

**Extra features:**
- Unified interface for multiple MSSQL modules
- Connection pooling with Transactions and Prepared statements
- Parametrized Stored Procedures in [node-tds](https://github.com/cretz/node-tds) and [Microsoft Driver for Node.js for SQL Server](https://github.com/WindowsAzure/node-sqlserver)
- Serialization of Geography and Geometry CLR types
- Injects original TDS modules with enhancements and bug fixes

At the moment it support three TDS modules:
- [Tedious](https://github.com/pekim/tedious) by Mike D Pilsbury (pure javascript - windows/osx/linux)
- [Microsoft Driver for Node.js for SQL Server](https://github.com/WindowsAzure/node-sqlserver) by Microsoft Corporation (native - windows only)
- [node-tds](https://github.com/cretz/node-tds) by Chad Retz (pure javascript - windows/osx/linux)

## What's new in 0.5.3 (stable, npm)

- Support for [Prepared Statements](#prepared-statement)
- Fixed order of output parameters
- Minor fixes in node-tds driver

## What's new in 0.5.1

- Updated to new Tedious 0.2.2
    - Added support for TDS 7.4
    - Added request cancelation
    - Added support for UDT, TVP, Time, Date, DateTime2 and DateTimeOffset data types
    - Numeric, Decimal, SmallMoney and Money are now supported as input parameters
    - Fixed compatibility with TDS 7.1 (SQL Server 2000)
    - Minor fixes
- You can now easily set up types' length/scale (`sql.VarChar(50)`)
- Serialization of [Geography and Geometry](#geography) CLR types
- Support for creating [Table-Value Parameters](#tvp) (`var tvp = new sql.Table()`)
- Output parameters are now Input-Output and can handle initial value
- Option to choose whether to pass/receive times in UTC or local time
- Connecting to named instances simplified
- Default SQL data type for JS String type is now NVarChar (was VarChar)

## Comming soon in 0.5.4 (unstable, git)

- Multiple errors handling (`err.precedingErrors`)

## Installation

    npm install mssql

## Quick Example

```javascript
var sql = require('mssql'); 

var config = {
    user: '...',
    password: '...',
    server: 'localhost', // You can use 'localhost\\instance' to connect to named instance
    database: '...',
    
    options: {
        encrypt: true // Use this if you're on Windows Azure
    }
}

var connection = new sql.Connection(config, function(err) {
    // ... error checks
    
    // Query
	
    var request = new sql.Request(connection); // or: var request = connection.request();
    request.query('select 1 as number', function(err, recordset) {
        // ... error checks
        
        console.dir(recordset);
    });
	
    // Stored Procedure
	
    var request = new sql.Request(connection);
    request.input('input_parameter', sql.Int, 10);
    request.output('output_parameter', sql.VarChar(50));
    request.execute('procedure_name', function(err, recordsets, returnValue) {
        // ... error checks
        
        console.dir(recordsets);
    });
	
});
```

## Quick Example with one global connection

```javascript
var sql = require('mssql'); 

var config = {
    user: '...',
    password: '...',
    server: 'localhost', // You can use 'localhost\\instance' to connect to named instance
    database: '...',
    
    options: {
        encrypt: true // Use this if you're on Windows Azure
    }
}

sql.connect(config, function(err) {
    // ... error checks
	
    // Query
	
    var request = new sql.Request();
    request.query('select 1 as number', function(err, recordset) {
        // ... error checks

        console.dir(recordset);
    });
	
    // Stored Procedure
	
    var request = new sql.Request();
    request.input('input_parameter', sql.Int, value);
    request.output('output_parameter', sql.VarChar(50));
    request.execute('procedure_name', function(err, recordsets, returnValue) {
        // ... error checks

        console.dir(recordsets);
    });
	
});
```

## Documentation

### Configuration

* [Basic](#cfg-basic)
* [Tedious](#cfg-tedious)
* [Microsoft Driver for Node.js for SQL Server](#cfg-msnodesql)
* [node-tds](#cfg-node-tds)

### Connections

* [Connection](#connection)
* [connect](#connect)
* [close](#close)

### Requests

* [Request](#request)
* [execute](#execute)
* [input](#input)
* [output](#output)
* [query](#query)
* [cancel](#cancel)

### Transactions

* [Transaction](#transaction)
* [begin](#begin)
* [commit](#commit)
* [rollback](#rollback)

### Prepared Statements

* [PreparedStatement](#prepared-statement)
* [input](#prepared-statement-input)
* [output](#prepared-statement-output)
* [prepare](#prepare)
* [execute](#prepared-statement-execute)
* [unprepare](#unprepare)

### Other

* [Geography and Geometry](#geography)
* [Table-Valued Parameter](#tvp)
* [Errors](#errors)
* [Metadata](#meta)
* [Data Types](#data-types)
* [Verbose Mode](#verbose)
* [Known Issues](#issues)

## Configuration

```javascript
var config = {
    user: '...',
    password: '...',
    server: 'localhost',
    database: '...',
    pool: {
        max: 10,
        min: 0,
        idleTimeoutMillis: 30000
    }
}
```

<a name="cfg-basic" />
### Basic configuration is same for all drivers.

- **driver** - Driver to use (default: `tedious`). Possible values: `tedious`, `msnodesql` or `tds`.
- **user** - User name to use for authentication.
- **password** - Password to use for authentication.
- **server** - Server to connect to. You can use 'localhost\\instance' to connect to named instance.
- **port** - Port to connect to (default: `1433`). Don't set when connecting to named instance.
- **database** - Database to connect to (default: dependent on server configuration).
- **timeout** - Connection timeout in ms (default: `15000`).
- **pool.max** - The maximum number of connections there can be in the pool (default: `10`).
- **pool.min** - The minimun of connections there can be in the pool (default: `0`).
- **pool.idleTimeoutMillis** - The Number of milliseconds before closing an unused connection (default: `30000`).

<a name="cfg-tedious" />
### Tedious

- **options.instanceName** - The instance name to connect to. The SQL Server Browser service must be running on the database server, and UDP port 1444 on the database server must be reachable.
- **options.useUTC** - A boolean determining whether or not use UTC time for values without time zone offset (default: `true`).
- **options.encrypt** - A boolean determining whether or not the connection will be encrypted (default: `false`) Encryption support is experimental.
- **options.tdsVersion** - The version of TDS to use (default: `7_4`, available: `7_1`, `7_2`, `7_3_A`, `7_3_B`, `7_4`).

More information about Tedious specific options: http://pekim.github.io/tedious/api-connection.html

<a name="cfg-msnodesql" />
### Microsoft Driver for Node.js for SQL Server

This driver is not part of the default package and must be installed separately by `npm install msnodesql`. If you are looking for compiled binaries, see [node-sqlserver-binary](https://github.com/jorgeazevedo/node-sqlserver-binary).

- **options.instanceName** - The instance name to connect to. The SQL Server Browser service must be running on the database server, and UDP port 1444 on the database server must be reachable.
- **connectionString** - Connection string (default: see below).
- **options.trustedConnection** - Use Windows Authentication (default: `false`).
- **options.useUTC** - A boolean determining whether or not to use UTC time for values without time zone offset (default: `true`).

Default connection string when connecting to port:
```
Driver={SQL Server Native Client 11.0};Server={#{server},#{port}};Database={#{database}};Uid={#{user}};Pwd={#{password}};Trusted_Connection={#{trusted}};
```

Default connection string when connecting to named instance:
```
Driver={SQL Server Native Client 11.0};Server={#{server}\\#{instance}};Database={#{database}};Uid={#{user}};Pwd={#{password}};Trusted_Connection={#{trusted}};
```

<a name="cfg-node-tds" />
### node-tds

This driver is not part of the default package and must be installed separately by `npm install tds`.

_This module update node-tds driver with extra features and bug fixes by overriding some of its internal functions. If you want to disable this, require module with `var sql = require('mssql/nofix')`._

<a name="connection" />
## Connections

```javascript
var connection = new sql.Connection({ /* config */ });
```

### Events

- **connect** - Dispatched after connection has established.
- **close** - Dispatched after connection has closed a pool (by calling `close`).

---------------------------------------

<a name="connect" />
### connect([callback])

Create connection to the server.

__Arguments__

- **callback(err)** - A callback which is called after connection has established, or an error has occurred. Optional.

__Example__

```javascript
var connection = new sql.Connection({
    user: '...',
    password: '...',
    server: 'localhost',
    database: '...'
});

connection.connect(function(err) {
    // ...
});
```

---------------------------------------

<a name="close" />
### close()

Close connection to the server.

__Example__

```javascript
connection.close();
```

<a name="request" />
## Requests

```javascript
var request = new sql.Request(/* [connection] */);
```

If you omit connection argument, global connection is used instead.

### Events

- **recordset(recordset)** - Dispatched when new recordset is parsed (and all its rows).
- **row(row)** - Dispatched when new row is parsed.
- **done(err, recordsets)** - Dispatched when request is complete.

---------------------------------------

<a name="execute" />
### execute(procedure, [callback])

Call a stored procedure.

__Arguments__

- **procedure** - Name of the stored procedure to be executed.
- **callback(err, recordsets, returnValue)** - A callback which is called after execution has completed, or an error has occurred. `returnValue` is also accessible as property of recordsets.

__Example__

```javascript
var request = new sql.Request();
request.input('input_parameter', sql.Int, value);
request.output('output_parameter', sql.Int);
request.execute('procedure_name', function(err, recordsets, returnValue) {
    // ... error checks
    
    console.log(recordsets.length); // count of recordsets returned by the procedure
    console.log(recordsets[0].length); // count of rows contained in first recordset
    console.log(returnValue); // procedure return value
    console.log(recordsets.returnValue); // same as previous line
	
    console.log(request.parameters.output_parameter.value); // output value
	
    // ...
});
```

---------------------------------------

<a name="input" />
### input(name, [type], value)

Add an input parameter to the request.

__Arguments__

- **name** - Name of the input parameter without @ char.
- **type** - SQL data type of input parameter. If you omit type, module automaticaly decide which SQL data type should be used based on JS data type.
- **value** - Input parameter value. `undefined` ans `NaN` values are automatically converted to `null` values.

__Example__

```javascript
request.input('input_parameter', value);
request.input('input_parameter', sql.Int, value);
```

__JS Data Type To SQL Data Type Map__

- `String` -> `sql.NVarChar`
- `Number` -> `sql.Int`
- `Boolean` -> `sql.Bit`
- `Date` -> `sql.DateTime`
- `Buffer` -> `sql.VarBinary`
- `sql.Table` -> `sql.TVP`

Default data type for unknown object is `sql.NVarChar`.

You can define your own type map.

```javascript
sql.map.register(MyClass, sql.Text);
```

You can also overwrite the default type map.

```javascript
sql.map.register(Number, sql.BigInt);
```

---------------------------------------

<a name="output" />
### output(name, type, [value])

Add an output parameter to the request.

__Arguments__

- **name** - Name of the output parameter without @ char.
- **type** - SQL data type of output parameter.
- **value** - Output parameter value initial value. `undefined` and `NaN` values are automatically converted to `null` values. Optional.

__Example__

```javascript
request.output('output_parameter', sql.Int);
request.output('output_parameter', sql.VarChar(50), 'abc');
```

---------------------------------------

<a name="query" />
### query(command, [callback])

Execute the SQL command.

__Arguments__

- **command** - T-SQL command to be executed.
- **callback(err, recordset)** - A callback which is called after execution has completed, or an error has occurred.

__Example__

```javascript
var request = new sql.Request();
request.query('select 1 as number', function(err, recordset) {
    // ... error checks
    
    console.log(recordset[0].number); // return 1
	
    // ...
});
```

You can enable multiple recordsets in queries with the `request.multiple = true` command.

```javascript
var request = new sql.Request();
request.multiple = true;

request.query('select 1 as number; select 2 as number', function(err, recordsets) {
    // ... error checks
    
    console.log(recordsets[0][0].number); // return 1
    console.log(recordsets[1][0].number); // return 2
});
```

---------------------------------------

<a name="cancel" />
### cancel()

Cancel currently executing request. Return `true` if cancellation packet was send successfully. Not available in `msnodesql` and `tds` drivers.

__Example__

```javascript
var request = new sql.Request();
request.query('waitfor delay \'00:00:05\'; select 1 as number', function(err, recordset) {
    console.log(err instanceof sql.RequestError);  // true
    console.log(err.message);                      // Canceled.
    console.log(err.code);                         // ECANCEL
	
    // ...
});

request.cancel();
```

<a name="transaction" />
## Transactions

**Important:** always use `Transaction` class to create transactions - it ensures that all your requests are executed on one connection. Once you call `begin`, a single connection is acquired from the connection pool and all subsequent requests (initialized with the `Transaction` object) are executed exclusively on this connection. Transaction also contains a queue to make sure your requests are executed in series. After you call `commit` or `rollback`, connection is then released back to the connection pool.

```javascript
var transaction = new sql.Transaction(/* [connection] */);
```

If you omit connection argument, global connection is used instead.

__Example__

```javascript
var transaction = new sql.Transaction(/* [connection] */);
transaction.begin(function(err) {
    // ... error checks

    var request = new sql.Request(transaction);
    request.query('insert into mytable (mycolumn) values (12345)', function(err, recordset) {
        // ... error checks

        transaction.commit(function(err, recordset) {
            // ... error checks
            
            console.log("Transaction commited.");
        });
    });
});
```

Transaction can also be created by `var transaction = connection.transaction();`. Requests can also be created by `var request = transaction.request();`.

### Events

- **begin** - Dispatched when transaction begin.
- **commit** - Dispatched on successful commit.
- **rollback** - Dispatched on successful rollback.

---------------------------------------

<a name="begin" />
### begin([isolationLevel], [callback])

Begin a transaction.

__Arguments__

- **isolationLevel** - Controls the locking and row versioning behavior of TSQL statements issued by a connection. Optional. `READ_COMMITTED` by default. For possible values see `sql.ISOLATION_LEVEL`.
- **callback(err)** - A callback which is called after transaction has began, or an error has occurred. Optional.

__Example__

```javascript
var transaction = new sql.Transaction();
transaction.begin(function(err) {
    // ... error checks
});
```

---------------------------------------

<a name="commit" />
### commit([callback])

Commit a transaction.

__Arguments__

- **callback(err)** - A callback which is called after transaction has committed, or an error has occurred. Optional.

__Example__

```javascript
var transaction = new sql.Transaction();
transaction.begin(function(err) {
    // ... error checks
    
    transaction.commit(function(err) {
        // ... error checks
    })
});
```

---------------------------------------

<a name="rollback" />
### rollback([callback])

Rollback a transaction.

__Arguments__

- **callback(err)** - A callback which is called after transaction has rolled back, or an error has occurred. Optional.

__Example__

```javascript
var transaction = new sql.Transaction();
transaction.begin(function(err) {
    // ... error checks
    
    transaction.rollback(function(err) {
        // ... error checks
    })
});
```

<a name="prepared-statement" />
## PreparedStatement

**Important:** always use `PreparedStatement` class to create prepared statements - it ensures that all your executions of prepared statement are executed on one connection. Once you call `prepare`, a single connection is aquired from the connection pool and all subsequent executions are executed exclusively on this connection. Prepared Statement also contains a queue to make sure your executions are executed in series. After you call `unprepare`, the connection is then released back to the connection pool.

```javascript
var ps = new sql.PreparedStatement(/* [connection] */);
```

If you omit the connection argument, the global connection is used instead.

__Example__

```javascript
var ps = new sql.PreparedStatement(/* [connection] */);
ps.input('param', sql.Int);
ps.prepare('select @param as value', function(err) {
    // ... error checks

    ps.execute({param: 12345}, function(err, recordset) {
        // ... error checks

        ps.unprepare(function(err) {
            // ... error checks
            
        });
    });
});
```

**IMPORTANT**: Remember that each prepared statement means one reserved connection from the pool. Don't forget to unprepare a prepared statement!

**TIP**: You can also create prepared statements in transactions (`new sql.PreparedStatement(transaction)`), but keep in mind you can't execute other requests in the transaction until you call `unprepare`.

---------------------------------------

<a name="prepared-statement-input" />
### input(name, type)

Add an input parameter to the prepared statement.

__Arguments__

- **name** - Name of the input parameter without @ char.
- **type** - SQL data type of input parameter.

__Example__

```javascript
ps.input('input_parameter', sql.Int);
ps.input('input_parameter', sql.VarChar(50));
```

---------------------------------------

<a name="prepared-statement-output" />
### output(name, type)

Add an output parameter to the prepared statement.

__Arguments__

- **name** - Name of the output parameter without @ char.
- **type** - SQL data type of output parameter.

__Example__

```javascript
ps.output('output_parameter', sql.Int);
ps.output('output_parameter', sql.VarChar(50));
```

---------------------------------------

<a name="prepare" />
### prepare(statement, [callback])

Prepare a statement.

__Arguments__

- **statement** - T-SQL statement to prepare.
- **callback(err)** - A callback which is called after preparation has completed, or an error has occurred. Optional.

__Example__

```javascript
var ps = new sql.PreparedStatement();
ps.prepare('select @param as value', function(err) {
    // ... error checks
});
```

---------------------------------------

<a name="prepared-statement-execute" />
### execute(values, [callback])

Execute a prepared statement.

__Arguments__

- **values** - An object whose names correspond to the names of parameters that were added to the prepared statement before it was prepared.
- **callback(err)** - A callback which is called after execution has completed, or an error has occurred. Optional.

__Example__

```javascript
var ps = new sql.PreparedStatement();
ps.input('param', sql.Int);
ps.prepare('select @param as value', function(err) {
    // ... error checks
    
    ps.execute({param: 12345}, function(err, recordset) {
        // ... error checks
        
        console.log(recordset[0].value); // return 12345
    })
});
```

You can enable multiple recordsets by `ps.multiple = true` command.

```javascript
var ps = new sql.PreparedStatement();
ps.input('param', sql.Int);
ps.prepare('select @param as value', function(err) {
    // ... error checks
    
    ps.multiple = true;
    ps.execute({param: 12345}, function(err, recordsets) {
        // ... error checks
        
        console.log(recordsets[0][0].value); // return 12345
    })
});
```

---------------------------------------

<a name="unprepare" />
### unprepare([callback])

Unprepare a prepared statement.

__Arguments__

- **callback(err)** - A callback which is called after unpreparation has completed, or an error has occurred. Optional.

__Example__

```javascript
var ps = new sql.PreparedStatement();
ps.input('param', sql.Int);
ps.prepare('select @param as value', function(err, recordsets) {
    // ... error checks

    ps.unprepare(function(err) {
        // ... error checks
        
    });
});
```

<a name="geography" />
## Geography and Geometry

node-mssql has built-in serializer for Geography and Geometry CLR data types.

```sql
select geography::STGeomFromText('LINESTRING(-122.360 47.656, -122.343 47.656 )', 4326)
select geometry::STGeomFromText('LINESTRING (100 100 10.3 12, 20 180, 180 180)', 0)
```

Results in:

```javascript
{ srid: 4326,
  version: 1,
  points: [ { x: 47.656, y: -122.36 }, { x: 47.656, y: -122.343 } ],
  figures: [ { attribute: 1, pointOffset: 0 } ],
  shapes: [ { parentOffset: -1, figureOffset: 0, type: 2 } ],
  segments: [] }
  
{ srid: 0,
  version: 1,
  points: 
   [ { x: 100, y: 100, z: 10.3, m: 12 },
     { x: 20, y: 180, z: NaN, m: NaN },
     { x: 180, y: 180, z: NaN, m: NaN } ],
  figures: [ { attribute: 1, pointOffset: 0 } ],
  shapes: [ { parentOffset: -1, figureOffset: 0, type: 2 } ],
  segments: [] }
```

<a name="tvp" />
## Table-Valued Parameter (TVP)

Supported on SQL Server 2008 and later. Not supported by optional drivers `msnodesql` and `tds`. You can pass a data table as a parameter to stored procedure. First, we have to create custom type in our database.

```sql
CREATE TYPE TestType AS TABLE ( a VARCHAR(50), b INT );
```

Next we will need a stored procedure.

```sql
CREATE PROCEDURE MyCustomStoredProcedure (@tvp TestType readonly) AS SELECT * FROM @tvp
```

Now let's go back to our Node.js app.

```javascript
var tvp = new sql.Table()

// Columns must correspond with type we have created in database.
tvp.columns.add('a', sql.VarChar(50));
tvp.columns.add('b', sql.Int);

// Add rows
tvp.rows.add('hello tvp', 777); // Values are in same order as columns.
```

You can send table as a parameter to stored procedure.

```javascript
var request = new sql.Request();
request.input('tvp', tvp);
request.execute('MyCustomStoredProcedure', function(err, recordsets, returnValue) {
    // ... error checks
    
    console.dir(recordsets[0][0]); // {a: 'hello tvp', b: 777}
});
```

**TIP**: You can also create Table variable from any recordset with `recordset.toTable()`.

<a name="errors" />
## Errors

There are three type of errors you can handle:

- **ConnectionError** - Errors related to connections and connection pool.
- **TransactionError** - Errors related to creating, commiting and rolling back transactions.
- **RequestError** - Errors related to queries and stored procedures execution.
- **PreparedStatementError** - Errors related to prepared statements.

Those errors are initialized in node-mssql module and its original stack can be cropped. You can always access original error with `err.originalError`.

SQL Server may generate more than one error for one request so you can access preceding errors with `err.precedingErrors`.

<a name="meta" />
## Metadata

Recordset metadata are accessible through the `recordset.columns` property.

```javascript
var request = new sql.Request();
request.query('select 1 as first, \'asdf\' as second', function(err, recordset) {
    console.dir(recordset.columns);
	
    console.log(recordset.columns.first.type === sql.Int); // true
    console.log(recordset.columns.second.type === sql.VarChar); // true
});
```

Columns structure for example above:

```javascript
{ first: { name: 'first', length: 10, type: [sql.Int] },
  second: { name: 'second', length: 4, type: [sql.VarChar] } }
```

<a name="data-types" />
## Data Types

You can define data types with length/precision/scale:

```javascript
request.input("name", sql.VarChar, "abc");               // varchar(3)
request.input("name", sql.VarChar(50), "abc");           // varchar(50)
request.input("name", sql.VarChar(sql.MAX), "abc");      // varchar(MAX)
request.output("name", sql.VarChar);                     // varchar(8000)
request.output("name", sql.VarChar, "abc");              // varchar(3)

request.input("name", sql.Decimal, 155.33);              // decimal(18, 0)
request.input("name", sql.Decimal(10), 155.33);          // decimal(10, 0)
request.input("name", sql.Decimal(10, 2), 155.33);       // decimal(10, 2)

request.input("name", sql.DateTime2, new Date());        // datetime2(7)
request.input("name", sql.DateTime2(5), new Date());     // datetime2(5)
```

List of supported data types:

```
sql.Bit
sql.BigInt
sql.Decimal ([precision], [scale])
sql.Float
sql.Int
sql.Money
sql.Numeric ([precision], [scale])
sql.SmallInt
sql.SmallMoney
sql.Real
sql.TinyInt

sql.Char ([length])
sql.NChar ([length])
sql.Text
sql.NText
sql.VarChar ([length])
sql.NVarChar ([length])
sql.Xml

sql.Time ([scale])
sql.Date
sql.DateTime
sql.DateTime2 ([scale])
sql.DateTimeOffset ([scale])
sql.SmallDateTime

sql.UniqueIdentifier

sql.Binary
sql.VarBinary ([length])
sql.Image

sql.UDT
sql.Geography
sql.Geometry
```

To setup MAX length for `VarChar`, `NVarChar` and `VarBinary` use `sql.MAX` length.

<a name="verbose" />
## Verbose Mode

You can enable verbose mode by `request.verbose = true` command.

```javascript
var request = new sql.Request();
request.verbose = true;
request.input('username', 'patriksimek');
request.input('password', 'dontuseplaintextpassword');
request.input('attempts', 2);
request.execute('my_stored_procedure');
```

Output for the example above could look similar to this.

```
---------- sql execute --------
     proc: my_stored_procedure
    input: @username, varchar, patriksimek
    input: @password, varchar, dontuseplaintextpassword
    input: @attempts, bigint, 2
---------- response -----------
{ id: 1,
  username: 'patriksimek',
  password: 'dontuseplaintextpassword',
  email: null,
  language: 'en',
  attempts: 2 }
---------- --------------------
   return: 0
 duration: 5ms
---------- completed ----------
```

<a name="issues" />
## Known issues

### Tedious

- If you're facing problems with connecting SQL Server 2000, try setting the default TDS version to 7.1 with `config.options.tdsVersion = '7_1'` ([issue](https://github.com/patriksimek/node-mssql/issues/36))

### msnodesql

- msnodesql 0.2.1 contains bug in DateTimeOffset ([reported](https://github.com/WindowsAzure/node-sqlserver/issues/160))

### node-tds

- If you're facing problems with date, try changing your tsql language `set language 'English';`.
- node-tds 0.1.0 doesn't support connecting to named instances.
- node-tds 0.1.0 contains bug and return same value for columns with same name.
- node-tds 0.1.0 doesn't support codepage of input parameters.
- node-tds 0.1.0 contains bug in selects that doesn't return any values *(select @param = 'value')*.
- node-tds 0.1.0 doesn't support Binary, VarBinary and Image as parameters.
- node-tds 0.1.0 always return date/time values in local time.

<a name="license" />
## License

Copyright (c) 2013-2014 Patrik Simek

The MIT License

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
