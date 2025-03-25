<p><a target="_blank" href="https://app.eraser.io/workspace/2A1qUBDlXhdb4sfOcdqW" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

## goal is to make a data import fucntion which will insert the data in the database from a given csv


features - 

it must work with both csv and xlsx (excell file)

data insertion report -> how many records has been inserted and how many errored and they have errored



## input 
the admin will specify the id of the table in which they want to insert 

and the csv ( `csv means both csv and xlsx` )itself

 so we have already defined the schema 

extracted the schema from the table create query using the sql parser the simple create query gets translated into this 

---

```
CREATE TABLE tes (id int PRIMARY KEY AUTO_INCREMENT, name VARCHAR(100), email VARCHAR(100) not null)
```
The Abstract Syntax Tree for the above query will be

```

{
  "type": "create",
  "keyword": "table",
  "temporary": null,
  "if_not_exists": null,
  "table": [
    {
      "db": null,
      "table": "tes"
    }
  ],
  "ignore_replace": null,
  "as": null,
  "query_expr": null,
  "create_definitions": [
    {
      "column": {
        "type": "column_ref",
        "table": null,
        "column": "id",
        "collate": null
      },
      "definition": {
        "dataType": "INT",
        "suffix": []
      },
      "resource": "column",
      "primary_key": "primary key",
      "auto_increment": "auto_increment"
    },
    {
      "column": {
        "type": "column_ref",
        "table": null,
        "column": "name",
        "collate": null
      },
      "definition": {
        "dataType": "VARCHAR",
        "length": 100,
        "parentheses": true,
        "suffix": null
      },
      "resource": "column"
    },
    {
      "column": {
        "type": "column_ref",
        "table": null,
        "column": "email",
        "collate": null
      },
      "definition": {
        "dataType": "VARCHAR",
        "length": 100,
        "parentheses": true,
        "suffix": null
      },
      "resource": "column",
      "nullable": {
        "type": "not null",
        "value": "not null"
      }
    }
  ],
  "table_options": null
}
```


since we will only perform limited number of queries so we will ignore others and focus to build the schema and the dependencies so for the above table the simplified schema will be exactly

```
[
    {
        "type": "INT",
        "column": "id",
        "isUnique": true,
        "isNullable": false,
        "isPrimaryKey": true,
        "isAutoIncrement": true
    },
    {
        "type": "VARCHAR",
        "column": "name",
        "isUnique": false,
        "isNullable": true,
        "isPrimaryKey": false,
        "isAutoIncrement": false
    },
    {
        "type": "VARCHAR",
        "column": "email",
        "isUnique": false,
        "isNullable": false,
        "isPrimaryKey": false,
        "isAutoIncrement": false
    }
]
```
in our simplified schema we are just saving the columns and some basic details about it 



to create this we are using a function which returs the simplified schema from the given parsed sql array 

```javascript
function extractTableSchema(parsedQuery) {
  return parsedQuery.create_definitions.reduce((schema, def) => {
    if (def.resource === "column") {
      const columnInfo = {
        column: def.column.column,
        type: def.definition.dataType,
        isPrimaryKey: !!def.primary_key,
        isAutoIncrement: !!def.auto_increment,
        isNullable: def.primary_key
          ? false
          : def.nullable
            ? def.nullable.type !== "not null"
            : true,
        isUnique: !!def.unique,
      };
      return [...schema, columnInfo];
    }
    return schema;
  }, []);
}
```
```javascript
async function prepareInsertManyQuery(parsedData, table) {
  const schema = table.schema;
  const tableName = table.tableName;
  const columns = schema.map((col) => col.column);
  const values = [];
  const erroredRows = [];

  for (const rowData of parsedData) {
    const rowValues = [];
    const rowErrors = [];
    let isValidRow = true;

    for (const columnSchema of schema) {
      const columnName = columnSchema.column;
      const columnType = columnSchema.type.toUpperCase();
      const expectedJsType = helper.mysql.mysqlToJsTypes[columnType];
      const actualValue = Object.keys(rowData).reduce((acc, key) => {
        if (key.toLowerCase() === columnName.toLowerCase()) {
          return rowData[key];
        }
        return acc;
      }, undefined); // Case-insensitive lookup

      if (actualValue !== undefined) {
        const actualJsType = typeof actualValue;

        // Basic type checking
        let typeMatch = false;
        if (expectedJsType === "number") {
          typeMatch =
            actualJsType === "number" ||
            (!isNaN(actualValue) && isFinite(actualValue));
          if (typeMatch) {
            rowValues.push(actualValue);
          }
        } else if (expectedJsType === "bigint") {
          typeMatch = actualJsType === "string" && /^\d+$/.test(actualValue); // Assuming bigint comes as string from CSV
          if (typeMatch) {
            rowValues.push(actualValue);
          }
        } else if (expectedJsType === "boolean") {
          typeMatch =
            actualJsType === "boolean" ||
            actualValue?.toLowerCase() === "true" ||
            actualValue?.toLowerCase() === "false";
          if (typeMatch) {
            rowValues.push(
              actualValue?.toLowerCase() === "true" ? "TRUE" : "FALSE"
            );
          }
        } else if (expectedJsType === "string") {
          rowValues.push(`'${String(actualValue).replace(/'/g, "''")}'`); // Escape single quotes
          typeMatch = true; // Assume any value can be stringified
        } else if (expectedJsType === "object" && columnType === "JSON") {
          try {
            JSON.parse(actualValue);
            rowValues.push(`'${String(actualValue).replace(/'/g, "''")}'`);
            typeMatch = true;
          } catch (e) {
            rowErrors.push(
              `Column '${columnName}': Expected JSON format, got '${actualValue}'`
            );
            isValidRow = false;
          }
        } else if (
          expectedJsType === "string" &&
          (columnType === "DATE" ||
            columnType === "DATETIME" ||
            columnType === "TIMESTAMP")
        ) {
          // Basic check if it looks like a date - you might need more robust date parsing
          if (
            actualJsType === "string" &&
            (/\d{4}-\d{2}-\d{2}/.test(actualValue) ||
              /\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}/.test(actualValue))
          ) {
            rowValues.push(`'${actualValue}'`);
            typeMatch = true;
          } else {
            rowErrors.push(
              `Column '${columnName}': Expected ${columnType} format, got '${actualValue}'`
            );
            isValidRow = false;
          }
        } else if (expectedJsType === "Buffer") {
          // Handle Buffer/BLOB types if needed - this might require specific encoding/decoding
          rowErrors.push(
            `Column '${columnName}': Buffer/BLOB type is not directly supported in this example.`
          );
          isValidRow = false;
        } else if (expectedJsType && actualJsType !== expectedJsType) {
          // Fallback for other types
          rowErrors.push(
            `Column '${columnName}': Expected type ${expectedJsType}, got ${actualJsType} for value '${actualValue}'`
          );
          isValidRow = false;
        } else if (!expectedJsType) {
          console.warn(
            `No JavaScript type mapping found for MySQL type: ${columnType}`
          );
          rowValues.push(`'${String(actualValue).replace(/'/g, "''")}'`);
          typeMatch = true; // If no mapping, try to insert as string
        } else if (expectedJsType === actualJsType) {
          rowValues.push(actualValue); // For direct matches like number to number
          typeMatch = true;
        }

        if (
          !typeMatch &&
          expectedJsType !== "string" &&
          expectedJsType !== "object" &&
          expectedJsType !== "Buffer"
        ) {
          if (
            !rowErrors.some((err) => err.startsWith(`Column '${columnName}'`))
          ) {
            rowErrors.push(
              `Column '${columnName}': Expected type ${expectedJsType}, got ${actualJsType} for value '${actualValue}'`
            );
            isValidRow = false;
          }
        }
      } else if (
        !columnSchema.isNullable &&
        !columnSchema.isAutoIncrement &&
        !columnSchema.isPrimaryKey
      ) {
        rowErrors.push(
          `Column '${columnName}': Value is missing and column is not nullable.`
        );
        isValidRow = false;
      } else {
        rowValues.push("NULL"); // If value is missing but nullable, insert NULL
      }
    }

    if (isValidRow) {
      values.push(`(${rowValues.join(", ")})`);
    } else {
      erroredRows.push({ data: rowData, errors: rowErrors });
    }
  }

  const query =
    values.length > 0
      ? `INSERT INTO ${tableName} (${columns.join(", ")}) VALUES ${values.join(", ")};`
      : null;

  return { query, erroredRows };
}
```
---

Well we decided to change the approach so now we will use the schema given by the mysql descrbie



[﻿www.npmjs.com/package/node-sql-parser](https://www.npmjs.com/package/node-sql-parser) 

check here for making the execute function even more robust



<!--- Eraser file: https://app.eraser.io/workspace/2A1qUBDlXhdb4sfOcdqW --->