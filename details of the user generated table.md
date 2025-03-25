<p><a target="_blank" href="https://app.eraser.io/workspace/Mvm5is67kNcQv67WAmjS" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

```
async function getDetailsOfModel(req, res) {
  const { tableName } = req.query;
  const userId = req.identity.id;

  if (!tableName) {
    return res.status(400).send({
      success: false,
      message: "Table name is required.",
    });
  }

  try {
    const model = await db.UserGeneratedModel.findOne({
      where: { tableName: tableName, addedBy: userId, isDeleted: false },
    });

    if (!model) {
      return res.status(404).send({
        success: false,
        message: "Model not found.",
      });
    }

    return res.send({ success: true, schema: model.schema });
  } catch (error) {
    console.error("Error fetching model details:", error);
    return res.status(500).send({
      success: false,
      message: "Failed to fetch model details.",
    });
  }
}
```


// for updating the model 



```
async function updateModel(req, res) {
  const { title, schema, label } = req.body;
  const { tableName } = req.params; // Get tableName from URL parameters
  const userId = req.identity.id;

  if (!title && !schema && !label) {
    return res.status(400).json({
      success: false,
      error: {
        code: 400,
        message: constants.USER_GENERATED_MODEL.DATA_REQUIRED,
      },
    });
  }

  const validTitle = /^[a-zA-Z_]\w{0,39}$/;

  if (title && !title.match(validTitle)) {
    return res.status(400).send({
      success: false,
      message: constants.USER_GENERATED_MODEL.INAVLID_SCHEMA,
    });
  }

  try {
    const existingModel = await db.UserGeneratedModel.findOne({
      where: { tableName: tableName, addedBy: userId, isDeleted: false },
    });

    if (!existingModel) {
      return res.status(404).send({
        success: false,
        message: "Model not found.",
      });
    }

    let modelContent = existingModel.rawQuery; // Default to existing query if no schema provided
    let validatedSchema = existingModel.schema; // Default to existing schema if no schema provided

    if (schema) {
      modelContent = `module.exports = (sequelize, DataTypes) => { return sequelize.define('${tableName}', { id: { type: DataTypes.INTEGER, autoIncrement: true, primaryKey: true }, `;
      validatedSchema = [];

      for (let col of schema) {
        if (!col.column.match(validTitle)) {
          return res.status(400).send({
            success: false,
            message: `Invalid column name: ${col.column}`,
          });
        }

        const dataType = helper.typeMapping[col.type.toLowerCase()];
        if (!dataType) {
          return res.status(400).send({
            success: false,
            message: `Invalid data type: ${col.type}`,
          });
        }

        validatedSchema.push({
          column: col.column,
          type: col.type.toLowerCase(),
        });
      }

      validatedSchema.forEach((col, index) => {
        const dataType = helper.typeMapping[col.type];
        modelContent += `${col.column}: { type: ${dataType} }`;
        if (index < validatedSchema.length - 1) {
          modelContent += ", ";
        }
      });
      modelContent += ` }, { tableName: '${tableName}', timestamps: true }); };`;
      validatedSchema.push({
        column: "id",
        type: "integer",
        primaryKey: true,
        autoIncrement: true,
      });

    }

    await existingModel.update({
      title: label || existingModel.title, // Update label if provided
      schema: validatedSchema,
      rawQuery: modelContent,
    });

    res.send({
      success: true,
      message: "Model updated successfully.",
      tableName,
    });

    if (schema) {
      try {
        await modelServices.handleModelGeneration(
          tableName,
          modelContent,
          `${tableName}.model.js`
        );
        console.log("✅ Database Synced");
      } catch (err) {
        console.error("❌ Database Sync Error:", err);
      }
    }

  } catch (error) {
    console.error("Error updating UserGeneratedModel entry:", error);
    return res.status(500).send({
      success: false,
      message: "Failed to update model.",
    });
  }
}
```




<!--- Eraser file: https://app.eraser.io/workspace/Mvm5is67kNcQv67WAmjS --->