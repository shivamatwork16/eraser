<p><a target="_blank" href="https://app.eraser.io/workspace/R4jqkSxLLL03XK6U4IrA" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

only generating the single table for now 

```
const { success, failed } = require("../services/Response");
const { Parser } = require("node-sql-parser");
const constants = require("../utls/constants");
const db = require("../models");
const {
  generateDynamicTablesSchema,
} = require("../validation/userGeneratedTableValidation");
const { getConnection } = require("../config/mysql.config");
const { validate } = require("../validation/validate");

async function generateDynamicTables(req, res) {
  const validationResult = await validate(generateDynamicTablesSchema, req);
  if (!validationResult || !validationResult.success) {
    return failed(null, validationResult.message, res);
  }

  const { title, rawQuery } = req.body;
  const userId = req.identity.id;

  try {
    // Verify this is a CREATE TABLE query
    if (
      !rawQuery.toLowerCase().trim().startsWith("create table if not exists")
    ) {
      return failed(null, constants.DYNAMIC_TABLE.INVALID_START, res);
    }

    // Parse the query first
    const parser = new Parser();
    const parsedQuery = parser.astify(rawQuery, { databse: "MySQL" });

    if (parsedQuery.length !== 1 || parsedQuery[0].type !== "create") {
      return failed(null, constants.DYNAMIC_TABLE.ONLY_ONE_TABLE, res);
    }
    const parsedCreateTable = parsedQuery[0];

    // Extract table name and schema
    const tableName = parsedCreateTable.table[0].table;
    const schema = parsedCreateTable.create_definitions
      .map((def) => {
        if (def.resource === "column") {
          return {
            column: def.column.column,
            type: def.definition.dataType,
          };
        }
        return null;
      })
      .filter((item) => item);

    // Extract dependencies
    let dependencies = [];
    const foreignKeys = parsedCreateTable.create_definitions.filter(
      (def) => def.resource === "foreign key"
    );
    if (foreignKeys.length > 0) {
      dependencies = foreignKeys.map((fk) => fk.definition.reference.table);
    }

    const connection = await getConnection();
    const [results] = await connection.execute(rawQuery);

    // Only save successful table creation
    await db.UserGeneratedModel.create({
      addedBy: userId,
      title: title,
      tableName: tableName,
      rawQuery,
      schema: schema,
      dependencies: dependencies,
      queryType: "create",
      ast: parsedCreateTable, // Store the AST
      isDeleted: false,
    });

    return success(results, constants.COMMON.SUCCESS, res);
  } catch (error) {
    console.error("Error creating dynamic table:", error);
    return res.status(400).json({
      success: false,
      error: {
        code: error.code || 400,
        message: error.message,
      },
    });
  }
}

module.exports = {
  generateDynamicTables,
};
```
dynamically handling them

```
const { success, failed } = require("../services/Response");
const {
  processCreateTableQuery,
} = require("../services/userGeneratedTableService");
const constants = require("../utls/constants");
const {
  generateDynamicTablesSchema,
} = require("../validation/userGeneratedTableValidation");
const { validate } = require("../validation/validate");

async function generateDynamicTables(req, res) {
  const validationResult = await validate(generateDynamicTablesSchema, req);
  if (!validationResult || !validationResult.success) {
    return failed(null, validationResult.message, res);
  }

  const { title, rawQuery } = req.body;
  const userId = req.identity.id;

  let result = [];

  try {
    const queries = rawQuery.split(";").filter((q) => q.trim());
    for (const query of queries) {
      if (query.trim()) {
        const queryResult = await processCreateTableQuery(query, userId, title);
        result.push(queryResult);
      }
    }
    return success(result, constants.COMMON.SUCCESS, res);
  } catch (error) {
    console.error("Error creating dynamic tables:", error);
    return res.status(400).json({
      success: false,
      error: {
        code: error.code || 400,
        message: error.message,
      },
    });
  }
}

module.exports = {
  generateDynamicTables,
};
```




<!--- Eraser file: https://app.eraser.io/workspace/R4jqkSxLLL03XK6U4IrA --->