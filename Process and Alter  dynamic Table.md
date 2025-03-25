<p><a target="_blank" href="https://app.eraser.io/workspace/ylRgySqBdTmgqTYY3HA6" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

```
const { success, failed } = require("../services/Response");
const { Parser } = require("node-sql-parser");
const db = require("../models");
const { queryExecuter } = require("./queryExecuter");
const constants = require("../utls/constants");
const { validate } = require("../validation/validate");
const { alterDynamicTablesSchema } = require("../validation/userGeneratedTableValidation");

async function processAlterTableQuery(rawQuery, userId, tableId) {
  const parser = new Parser();
  const parsedQuery = parser.astify(rawQuery, { database: "MySQL" });

  if (
    !parsedQuery ||
    parsedQuery.type !== "alter" ||
    parsedQuery.keyword !== "table"
  ) {
    throw new Error("Invalid ALTER TABLE query.");
  }

  const userGeneratedTable = await db.UserGeneratedModel.findByPk(tableId);
  if (!userGeneratedTable) {
    throw new Error(`User generated table with ID ${tableId} not found.`);
  }

  const tableNameFromQuery = parsedQuery.table[0].table;
  if (tableNameFromQuery !== userGeneratedTable.tableName) {
    throw new Error(
      `Table name in query (${tableNameFromQuery}) does not match the existing table name (${userGeneratedTable.tableName}).`
    );
  }

  let updatedSchema = [...userGeneratedTable.schema];
  let updatedDependencies = [...userGeneratedTable.dependencies];

  for (const specification of parsedQuery.alter_specification) {
    if (specification.keyword === "add") {
      if (specification.resource === "column") {
        const newColumn = {
          column: specification.column.column,
          type: specification.definition.dataType,
          isPrimaryKey: !!specification.primary_key,
          isAutoIncrement: !!specification.auto_increment,
          isNullable: specification.nullable ? specification.nullable.type !== "not null" : true,
          isUnique: !!specification.unique,
        };
        updatedSchema.push(newColumn);
      } else if (
        specification.resource === "constraint" &&
        specification.constraint_type === "FOREIGN KEY"
      ) {
        const newDependency = {
          referencedTable: specification.reference_definition.table[0].table,
          localColumns: specification.definition.map((col) => col.column),
          referencedColumns: specification.reference_definition.definition.map(
            (col) => col.column
          ),
          onDelete: specification.reference_definition.on_action.find(
            (action) => action.action === "delete"
          )?.option,
          onUpdate: specification.reference_definition.on_action.find(
            (action) => action.action === "update"
          )?.option,
        };
        updatedDependencies.push(newDependency);
      } else if (
        specification.resource === "constraint" &&
        specification.constraint_type === "PRIMARY KEY"
      ) {
        // Update existing schema to mark columns as primary key
        specification.index.columns.forEach((col) => {
          const columnIndex = updatedSchema.findIndex(
            (s) => s.column === col.column
          );
          if (columnIndex !== -1) {
            updatedSchema[columnIndex].isPrimaryKey = true;
          }
        });
      } else if (
        specification.resource === "constraint" &&
        specification.constraint_type === "UNIQUE"
      ) {
        specification.index.columns.forEach((col) => {
          const columnIndex = updatedSchema.findIndex(
            (s) => s.column === col.column
          );
          if (columnIndex !== -1) {
            updatedSchema[columnIndex].isUnique = true;
          }
        });
      }
    } else if (specification.keyword === "drop") {
      if (specification.resource === "column") {
        updatedSchema = updatedSchema.filter(
          (col) => col.column !== specification.column.column
        );
      } else if (specification.resource === "constraint") {
        if (specification.constraint_type === "FOREIGN KEY") {
          updatedDependencies = updatedDependencies.filter(
            (dep) =>
              !(
                dep.localColumns.length ===
                  specification.index.columns.length &&
                dep.localColumns.every(
                  (col, index) =>
                    col === specification.index.columns[index].column
                )
              )
          );
        } else if (specification.constraint_type === "PRIMARY KEY") {
          // Update existing schema to mark columns as not primary key
          const primaryKeyCols = userGeneratedTable.schema.filter(s => s.isPrimaryKey).map(s => s.column);
          if (specification.symbol && specification.symbol.value) {
            // Try to find based on constraint name (not standard in node-sql-parser for PK drop)
            console.warn("Dropping primary key by name is not fully supported by this parser for update.");
          } else if (specification.index && specification.index.columns) {
            specification.index.columns.forEach((col) => {
              const columnIndex = updatedSchema.findIndex(
                (s) => s.column === col.column
              );
              if (columnIndex !== -1) {
                updatedSchema[columnIndex].isPrimaryKey = false;
              }
            });
          } else {
            // If no specific columns mentioned, assume all PKs are being dropped (handle with caution)
            updatedSchema.forEach(col => col.isPrimaryKey = false);
          }
        } else if (specification.constraint_type === "UNIQUE") {
          const uniqueConstraintCols = specification.index.columns.map(c => c.column);
          updatedSchema = updatedSchema.map(col => {
            if (uniqueConstraintCols.includes(col.column)) {
              col.isUnique = false;
            }
            return col;
          });
        }
      }
    } else if (specification.keyword === "change" || specification.keyword === "modify") {
      const columnIndex = updatedSchema.findIndex(
        (col) => col.column === specification.column.column
      );
      if (columnIndex !== -1) {
        updatedSchema[columnIndex].column = specification.new_column.column;
        updatedSchema[columnIndex].type = specification.new_definition.dataType;
        updatedSchema[columnIndex].isPrimaryKey = !!specification.new_primary_key;
        updatedSchema[columnIndex].isAutoIncrement = !!specification.new_auto_increment;
        updatedSchema[columnIndex].isNullable = specification.new_nullable
          ? specification.new_nullable.type !== "not null"
          : true;
        updatedSchema[columnIndex].isUnique = !!specification.new_unique;
      }
    } else if (specification.keyword === "rename") {
      if (specification.resource === "column") {
        const columnIndex = updatedSchema.findIndex(
          (col) => col.column === specification.column.column
        );
        if (columnIndex !== -1) {
          updatedSchema[columnIndex].column = specification.new_column.column;
        }
      }
    }
  }

  const queryResults = await queryExecuter(rawQuery, userId);

  if (!queryResults.success) {
    throw queryResults.errors;
  }

  await db.UserGeneratedModel.update(
    {
      schema: updatedSchema,
      dependencies: updatedDependencies,
    },
    {
      where: { id: tableId },
    }
  );

  return queryResults.results;
}

async function alterDynamicTable(req, res) {
  const validationResult = await validate(alterDynamicTablesSchema, req);
  if (!validationResult || !validationResult.success) {
    return failed(null, validationResult.message, res);
  }

  const { rawQuery } = req.body;
  const { id: tableId } = req.params;
  const userId = req.identity.id;
  let results =;

  try {
    const queries = rawQuery.split(";").filter((q) => q.trim());
    for (const query of queries) {
      const parsedQuery = new Parser().astify(query, { database: "MySQL" });
      if (!parsedQuery || parsedQuery.type !== "alter" || parsedQuery.keyword !== "table") {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: "Only ALTER TABLE queries are allowed.",
          },
        });
      }
      const queryResult = await processAlterTableQuery(query, userId, tableId);
      results.push(queryResult);
    }
    return success(results, constants.COMMON.SUCCESS, res);
  } catch (error) {
    console.error("Error altering dynamic table:", error);
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
  processAlterTableQuery,
  alterDynamicTable,
};
```




<!--- Eraser file: https://app.eraser.io/workspace/ylRgySqBdTmgqTYY3HA6 --->