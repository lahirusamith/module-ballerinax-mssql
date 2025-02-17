## Overview

This module provides the functionality required to access and manipulate data stored in a MSSQL database.

### Prerequisite
Add the MSSQL driver JAR as a native library dependency in your Ballerina project's `Ballerina.toml` file.
It is recommended to use a MSSQL driver version greater than 9.2.0 as this module uses the database properties
from the MSSQL driver version 9.2.0 onwards.

Follow one of the following ways to add the JAR in the file:

* Download the JAR and update the path.
    ```
    [[platform.java11.dependency]]
    path = "PATH"
    ```

* Add JAR with the maven dependency params.
    ```
    [[platform.java11.dependency]]
    groupId = "com.microsoft.sqlserver"
    artifactId = "mssql-jdbc"
    version = "9.2.0.jre11"
    ```

### Client
To access a database, you must first create a
[`mssql:Client`](https://docs.central.ballerina.io/ballerinax/mssql/latest/clients/Client) object.
The examples for creating a MSSQL client can be found below.

#### Creating a Client
This example shows the different ways of creating the `mssql:Client`.

The client can be created with an empty constructor, and thereby, the client will be initialized with the default properties.

```ballerina
mssql:Client|sql:Error dbClient = new ();
```

The `dbClient` receives the host, username, and password. Since the properties are passed in the same order as they are defined
in the `mssql:Client`, you can pass them without named params.

```ballerina
mssql:Client|sql:Error dbClient = new ("localhost", "rootUser", "rootPass", "information_schema", 1443);
```

The `dbClient` uses the named params to pass the attributes since it is skipping some params in the constructor.
Further, the [`mssql:Options`](https://docs.central.ballerina.io/ballerinax/mssql/latest/records/Options)
property is passed to configure the SSL and login timeout in the MSSQL client.

```ballerina
mssql:Options mssqlOptions = {
  secureSocket: {
    encrypt: true,
    cert: {
       path: "trustStorePath",
       password: "password"
    }
  },
  loginTimeout: 10
};

mssql:Client|sql:Error dbClient = new (user = "rootUser", password = "rootPass", options = mssqlOptions);
```

Similarly, the `dbClient` uses the named params and it provides an unshared connection pool of the
[`sql:ConnectionPool`](https://docs.central.ballerina.io/ballerina/sql/latest/records/ConnectionPool)
type to be used within the client.
For more details about connection pooling, see the [`sql` Module](https://docs.central.ballerina.io/ballerina/sql/latest).

```ballerina
mssql:Client|sql:Error dbClient = new (user = "rootUser", password = "rootPass", 
                                       connectionPool = {maxOpenConnections: 5});
```

#### Using SSL
To connect to the MSSQL database using an SSL connection, you must add the SSL configurations to the `mssql:Options` when creating the `dbClient`.
The value of `encrypt` must be set to `true`.
If `trustServerCertificate` is set to `true`, the client will not validate the server TLS/SSL certificate (used for testing in local environments).
For the key and cert files, you must provide the files in the `.p12` format.

```ballerina
string clientStorePath = "/path/to/keystore.p12";
string turstStorePath = "/path/to/truststore.p12";

mssql:Options mssqlOptions = {
  secureSocket: {
    encrypt: true,
    trustServerCertificate: false
    key: {
        path: clientStorePath,
        password: "password"
    },
    cert: {
        path: turstStorePath,
        password: "password"
    }
  }
};
```
#### Connection Pool Handling

All Ballerina database modules share the same connection pooling concept and there are three possible scenarios for
connection pool handling.  For its properties and possible values, see the [`sql:ConnectionPool`](https://docs.central.ballerina.io/ballerina/sql/latest/records/ConnectionPool).

1. Global, shareable default connection pool

   If you do not provide the `poolOptions` field when creating the database client, a globally-shareable pool will be
   created for your database unless a connection pool matching with the properties you provided already exists.

    ```ballerina
    mssql:Client|sql:Error dbClient = new ("localhost", "rootUser", "rootPass");
    ```

2. Client owned, unsharable connection pool

   If you define the `connectionPool` field inline when creating the database client with the `sql:ConnectionPool` type,
   an unsharable connection pool will be created.

    ```ballerina
    mssql:Client|sql:Error dbClient = new ("localhost", "rootUser", "rootPass", 
                                           connectionPool = { maxOpenConnections: 5 });
    ```

3. Local, shareable connection pool

   If you create a record of `sql:ConnectionPool` type and reuse that in the configuration of multiple clients,
   for each set of clients that connects to the same database instance with the same set of properties, a shared
   connection pool will be created.

    ```ballerina
    sql:ConnectionPool connPool = {maxOpenConnections: 5};
    
    mssql:Client|sql:Error dbClient1 = new ("localhost", "rootUser", "rootPass", connectionPool = connPool);
    mssql:Client|sql:Error dbClient2 = new ("localhost", "rootUser", "rootPass", connectionPool = connPool);
    mssql:Client|sql:Error dbClient3 = new ("localhost", "rootUser", "rootPass", connectionPool = connPool);
    ```

For more details about each property, see the [`mssql:Client`](https://docs.central.ballerina.io/ballerinax/mssql/latest/clients/Client) constructor.

The [`mssql:Client`](https://docs.central.ballerina.io/ballerinax/mssql/latest/clients/Client) references
[`sql:Client`](https://docs.central.ballerina.io/ballerina/sql/latest/clients/Client) and all the operations
defined by the `sql:Client` will be supported by the `mssql:Client` as well.

#### Closing the Client

Once all the database operations are performed, you can close the database client you have created by invoking the `close()`
operation. This will close the corresponding connection pool if it is not shared by any other database clients.

```ballerina
error? e = dbClient.close();
```
Or
```ballerina
check dbClient.close();
```

### Database Operations

Once the client is created, database operations can be executed through that client. This module defines the interface
and common properties that are shared among multiple database clients.  It also supports querying, inserting, deleting,
updating, and batch updating data.

#### Creating Tables

This sample creates a table with two columns. One column is of type `int` and the other is of type `varchar`.
The `CREATE` statement is executed via the `execute` remote function of the client.

```ballerina
// Create the ‘Students’ table with the  ‘id’, 'name', and ‘age’ fields.
sql:ExecutionResult result = check dbClient->execute("CREATE TABLE student(id INT AUTO_INCREMENT, " +
                         "age INT, name VARCHAR(255), PRIMARY KEY (id))");
//A value of the`sql:ExecutionResult` type is returned for the 'result'. 
```

#### Inserting Data

These samples show the data insertion by executing an `INSERT` statement using the `execute` remote function
of the client.

In this sample, the query parameter values are passed directly into the query statement of the `execute`
remote function.

```ballerina
sql:ExecutionResult result = check dbClient->execute("INSERT INTO student(age, name) VALUES (23, 'john')");
```

In this sample, the parameter values, which are in local variables are used to parameterize the SQL query in
the `execute` remote function. This type of parameterized SQL query can be used with any primitive Ballerina type
like `string`, `int`, `float`, or `boolean`. In that case, the corresponding SQL type of the parameter is derived
from the type of the Ballerina variable that is passed in.

```ballerina
string name = "Anne";
int age = 8;

sql:ParameterizedQuery query = `INSERT INTO student(age, name)
                                VALUES (${age}, ${name})`;
sql:ExecutionResult result = check dbClient->execute(query);
```

In this sample, the parameter values are passed as a `sql:TypedValue` to the `execute` remote function. Use the
corresponding subtype of the `sql:TypedValue` such as `sql:Varchar`, `sql:Char`, `sql:Integer`, etc. when you need to
provide more details such as the exact SQL type of the parameter.

```ballerina
sql:VarcharValue name = new ("James");
sql:IntegerValue age = new (10);

sql:ParameterizedQuery query = `INSERT INTO student(age, name)
                                VALUES (${age}, ${name})`;
sql:ExecutionResult result = check dbClient->execute(query);
```

#### Inserting Data With Auto-generated Keys

This sample demonstrates inserting data while returning the auto-generated keys. It achieves this by using the
`execute` remote function to execute the `INSERT` statement.

```ballerina
int age = 31;
string name = "Kate";

sql:ParameterizedQuery query = `INSERT INTO student(age, name)
                                VALUES (${age}, ${name})`;
sql:ExecutionResult result = check dbClient->execute(query);
//Number of rows affected by the execution of the query.
int? count = result.affectedRowCount;
//The integer or string generated by the database in response to a query execution.
string|int? generatedKey = result.lastInsertId;
}
```

#### Querying Data

These samples show how to demonstrate the different usages of the `query` operation and query the
database table and obtain the results.

This sample demonstrates querying data from a table in a database.
First, a type is created to represent the returned result set. This record can be defined as an open or a closed record
according to the requirement. If an open record is defined, the returned stream type will include both defined fields
in the record and additional database columns fetched by the SQL query, which are not defined in the record.

>**Note:** the mapping of the database column to the returned record's property is case-insensitive if it is defined in 
> the record(i.e., the `ID` column in the result can be mapped to the `id` property in the record). Additional column
> names are added to the returned record as in the SQL query. If the record is defined as a closed record, only the 
> defined fields in the record are returned or gives an error when additional columns are present in the SQL query. 

Next, the `SELECT` query is executed via the `query` remote function of the client. Once the query is executed, each data record 
can be retrieved by looping the result set. The `stream` returned by the `SELECT` operation holds a pointer to the
actual data in the database and it loads data from the table only when it is accessed. This stream can be iterated only
once.

```ballerina
// Define an open record type to represent the results.
type Student record {
    int id;
    int age;
    string name;
};

// Select the data from the database table. The query parameters are passed 
// directly. Similar to the `execute` samples, parameters can be passed as
// sub types of the `sql:TypedValue` as well.
int id = 10;
int age = 12;
sql:ParameterizedQuery query = `SELECT * FROM students
                                WHERE id < ${id} AND age > ${age}`;
stream<Student, sql:Error> resultStream = dbClient->query(query);

// Iterating the returned table.
error? e = resultStream.forEach(function(Student student) {
   //Can perform any operations using 'student' and can access any fields in the returned record of type Student.
});
```

Defining the return type is optional and you can query the database without providing the result type. Hence,
the above sample can be modified as follows with an open record type as the return type. The property name in the open record
type will be the same as how the column is defined in the database.

```ballerina
// Select the data from the database table. The query parameters are passed 
// directly. Similar to the `execute` samples, parameters can be passed as 
// sub types of `sql:TypedValue` as well.
int id = 10;
int age = 12;
sql:ParameterizedQuery query = `SELECT * FROM students
                                WHERE id < ${id} AND age > ${age}`;
stream<record{}, sql:Error> resultStream = dbClient->query(query);

// Iterating the returned table.
error? e = resultStream.forEach(function(record{} student) {
    //Can perform any operations using the 'student' and can access any fields in the returned record.
    io:println("Student name: ", student.value["name"]);
});
```

There are situations in which you may not want to iterate through the database and in that case, you may decide
to only use the `next()` operation in the result `stream` and retrieve the first record. In such cases, the returned
result stream will not be closed and you have to explicitly invoke the `close` operation on the
`sql:Client` to release the connection resources and avoid a connection leak as shown below.

```ballerina
stream<record{}, sql:Error> resultStream = 
            dbClient->query("SELECT count(*) as total FROM students");

record {|record {} value;|}? result = check resultStream.next();

if result is record {|record {} value;|} {
    // A valid result is returned.
    io:println("total students: ", result.value["total"]);
} else {
    // The `Student` table must be empty.
}

error? e = resultStream.close();
```

#### Updating Data

This sample demonstrates modifying data by executing an `UPDATE` statement via the `execute` remote function of
the client.

```ballerina
int age = 23;
sql:ParameterizedQuery query = `UPDATE students SET name = 'John' 
                                WHERE age = ${age}`;
sql:ExecutionResult result = check dbClient->execute(query);
```

#### Deleting Data

This sample demonstrates deleting data by executing a `DELETE` statement via the `execute` remote function of
the client.

```ballerina
string name = "John";
sql:ParameterizedQuery query = `DELETE from students WHERE name = ${name}`;
sql:ExecutionResult result = check dbClient->execute(query);
```

#### Batch Updating Data

This sample demonstrates how to insert multiple records with a single `INSERT` statement that is executed via the
`batchExecute` remote function of the client. This is done by creating a `table` with multiple records and
parameterized SQL query as same as the  above `execute` operations.

```ballerina
// Create the table with the records that need to be inserted.
var data = [
  { name: "John", age: 25  },
  { name: "Peter", age: 24 },
  { name: "jane", age: 22 }
];

// Do the batch update by passing the batches.
sql:ParameterizedQuery[] batch = from var row in data
                                 select `INSERT INTO students ('name', 'age')
                                 VALUES (${row.name}, ${row.age})`;
sql:ExecutionResult[] result = check dbClient->batchExecute(batch);
```

#### Execute SQL Stored Procedures

This sample demonstrates how to execute a stored procedure with a single `INSERT` statement that is executed via the
`call` remote function of the client.

```ballerina
int uid = 10;
sql:IntegerOutParameter insertId = new;

sql:ProcedureCallResult|sql:Error result = dbClient->call(`exec InsertPerson(${uid}, ${insertId})`);
if result is error {
    //An error returned
} else {
    stream<record{}, sql:Error>? resultStr = result.queryResult;
    if resultStr is stream<record{}, sql:Error> {
        sql:Error? e = resultStr.forEach(function(record{} result) {
        //can perform operations using 'result'.
      });
    }
    check result.close();
}
```
>**Note:** that you have to explicitly invoke the close operation on the `sql:ProcedureCallResult` to release the connection resources and avoid a connection leak as shown above.

>**Note:** The default thread pool size used in Ballerina is: `the number of processors available * 2`. You can configure the thread pool size by using the `BALLERINA_MAX_POOL_SIZE` environment variable.
