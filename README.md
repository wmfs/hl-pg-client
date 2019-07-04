# hl-pg-client
[![Tymly Package](https://img.shields.io/badge/tymly-package-blue.svg)](https://tymly.io/)
[![npm (scoped)](https://img.shields.io/npm/v/@wmfs/hl-pg-client.svg)](https://www.npmjs.com/package/@wmfs/hl-pg-client)
[![CircleCI](https://circleci.com/gh/wmfs/hl-pg-client.svg?style=svg)](https://circleci.com/gh/wmfs/hl-pg-client)
[![codecov](https://codecov.io/gh/wmfs/hl-pg-client/branch/master/graph/badge.svg)](https://codecov.io/gh/wmfs/hl-pg-client)
[![CodeFactor](https://www.codefactor.io/repository/github/wmfs/hl-pg-client/badge)](https://www.codefactor.io/repository/github/wmfs/hl-pg-client)
[![Dependabot badge](https://img.shields.io/badge/Dependabot-active-brightgreen.svg)](https://dependabot.com/)
[![Commitizen friendly](https://img.shields.io/badge/commitizen-friendly-brightgreen.svg)](http://commitizen.github.io/cz-cli/)
[![JavaScript Style Guide](https://img.shields.io/badge/code_style-standard-brightgreen.svg)](https://standardjs.com)
[![license](https://img.shields.io/github/license/mashape/apistatus.svg)](https://github.com/wmfs/tymly/blob/master/packages/pg-concat/LICENSE)


> Provides a slightly higher level PostgreSQL client, that builds on pg-pool

## Usage

### Create
```
const HlPgClient = require('@hl-pg-client')

const client = new HlPgClient(connectionString)
```

### Query
To just run a simple query, just use the ```query``` method:

```
const time = await client.query('SELECT NOW()')
const name = await client.query('select $1::text as name', ['brianc'])
console.log(name.rows[0].name, 'says hello at', time.rows[0].name)
```

You can also use a callback here if you'd like:

```
client.query('SELECT $1::text as name', ['brianc'], function (err, res) {
  console.log(res.rows[0].name) // brianc 
})
```

Internally, ```query``` uses pg-pool.query which ensures that a connection is acquired from the pool
beforehand, and then properly released again afterwards.

### Larger Transactions
To run multiple SQL statements within the same transaction, use the ```run``` method:
```
await client.run(statementArray)
```
or using callbacks
```
client.run(statementArray, (err, res) => { ... })
```

Each item in the statement array should be an object with the form 
```
{
  sql: 'SQL statement to execute',
  params: parameter array
    // optional
  preStatementHook: (item, ctx) => { ... },
    // optional, called before the SQL is executed
  postStatementHook: (result, ctx) => { ... },
    // optional, called after the SQL is executed
  action: (sql, params, client) => { ... }
    // alternative action to perform instead of calling query
}
```

In the above the ```ctx``` parameter is initially the empty object, ```{}```, 
and it is passed to each postStatementHook in turn. 

The return value of ```run``` is ```ctx.returnValue```.

The run method ensures that a client is properly acquired from and released back to the pool, whether the transaction 
succeeds or fails.  In the event of a statement failure, it will attempt to rollback the transaction.

#### Run example

```
const statements = [
  { "sql": "BEGIN;" },
  { "sql": "CREATE TEMP TABLE delete_supercopy_test_adults ON COMMIT DROP AS SELECT * FROM supercopy_test.adults WITH NO DATA;" },
  { 
    "sql": "COPY delete_supercopy_test_adults(adult_no) FROM '/some/path/to/the/deletes/adults.csv' CSV HEADER;",
    "action": copyStream
  },
  { "sql": "DELETE FROM supercopy_test.adults WHERE (adult_no) IN (SELECT adult_no FROM delete_supercopy_test_adults);" },
  { "sql": "DROP TABLE delete_supercopy_test_adults;" }
  { "sql": "COMMIT;" }
]
await client.run(statements)

...

function copyStream(statement, params, client) {
  // use pg-copy-stream to send local file to remote PostgreSQL server
  return new Promise((resolve, reject) => {
    const components = statement.match(/COPY (.*?) FROM '([^']*)'/)
    const tableAndCols = components[1]
    const filename = components[2]
    const newStatement = `COPY ${tableAndCols} FROM STDIN CSV HEADER;`

    const stream = client.query(
      copyFrom(newStatement)
    )
    stream.on('end', function () {
      resolve()
    }).on('error', function (err) {
      reject(err)
    })

    const fileStream = fs.createReadStream(filename)
    fileStream.on('error', function (err) {
      reject(err)
    })

    fileStream.pipe(stream)
  })
} // copyStream
```

### SQL Files

The ```runFile``` method reads a file of SQL statements and executes them within a single transaction.

### Testing

Ensure a PostgreSQL database is setup for testing and add a PG_CONNECTION_STRING environment variable when establishing a pool of PostgreSQL connections, for example:

```
PG_CONNECTION_STRING=postgres://postgres:postgres@localhost:5432/my_test_db
```