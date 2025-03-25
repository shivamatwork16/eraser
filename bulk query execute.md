<p><a target="_blank" href="https://app.eraser.io/workspace/4cWnKs1DpRdit4zzrhl6" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

so it depends on multiple files



1. the controller file
2. the parsing file
3. the validation file
## controller file
```
const db = require("../models");
const {
  parseCsv,
  parseXlsx,
  executeDataImportQuery,
} = require("../services/dataImportServices");
const constants = require("../utls/constants");
const {
  dataImportSchema,
  validateParsedData,
} = require("../validation/dataImportValidation");
const { validate } = require("../validation/validate");
const path = require("path");
const fs = require("fs");
const axios = require("axios");

async function DataImportController(req, res) {
  const validationResult = await validate(dataImportSchema, req);
  if (!validationResult || !validationResult.success) {
    return res.status(400).json({
      success: false,
      error: { code: 400, message: validationResult.message },
    });
  }

  const { id, fileLink, fileType } = req.body;
  const userId = req.identity.id;

  try {
    const model = await db.UserGeneratedModel.findOne({
      where: { id: id, addedBy: userId, isDeleted: false },
    });

    if (!model) {
      return res.status(404).json({
        success: false,
        error: { code: 404, message: constants.USER_GENERATED_MODEL.NOT_FOUND },
      });
    }

    const tempFilePath = path.join(__dirname, `temp_${Date.now()}.${fileType}`);

    try {
      const response = await axios({
        url: fileLink,
        method: "GET",
        responseType: "stream",
      });

      const writer = fs.createWriteStream(tempFilePath);
      response.data.pipe(writer);

      await new Promise((resolve, reject) => {
        writer.on("finish", resolve);
        writer.on("error", reject);
      });

      let parsedData;
      if (fileType === "csv") {
        parsedData = await parseCsv(tempFilePath);
      } else if (fileType === "xlsx") {
        parsedData = await parseXlsx(tempFilePath);
      }

      const validatedData = await validateParsedData(parsedData, model.schema);

      if (!validatedData.success) {
        fs.unlinkSync(tempFilePath);
        return res.status(400).json({
          success: false,
          error: { code: 400, message: validatedData.message },
        });
      }

      await executeDataImportQuery(model, validatedData.data, db);

      fs.unlinkSync(tempFilePath);

      return res.json({
        success: true,
        message: constants.COMMON.SUCCESS,
      });
    } catch (downloadError) {
      console.error("Error downloading or processing file:", downloadError);
      fs.unlinkSync(tempFilePath); // clean up the temp file
      return res.status(500).json({
        success: false,
        error: { code: 500, message: constants.DATA_IMPORT.DOWNLOAD_FAILED },
      });
    }
  } catch (error) {
    console.error("an error occurred during data import", error);
    return res.status(500).json({
      success: false,
      error: {
        code: 500,
        message: constants.COMMON.SERVER_ERROR,
      },
    });
  }
}

module.exports = {
  DataImportController,
};
```
## the validation file
```
async function validateParsedData(parsedData, schema) {
  try {
    const joiSchema = {};
    schema.forEach((col) => {
      if (col.column !== "id") {
        const sequelizeType = col.type.toLowerCase();
        switch (sequelizeType) {
          case "string":
          case "text":
            joiSchema[col.column] = Joi.string().allow("").allow(null);
            break;
          case "integer":
          case "bigint":
            joiSchema[col.column] = Joi.number().integer().allow(null);
            break;
          case "float":
          case "double":
          case "decimal":
            joiSchema[col.column] = Joi.number().allow(null);
            break;
          case "boolean":
            joiSchema[col.column] = Joi.boolean().allow(null);
            break;
          case "date":
          case "dateonly":
          case "datetime":
          case "time":
            joiSchema[col.column] = Joi.date().allow(null);
            break;
          case "json":
            joiSchema[col.column] = Joi.object().allow(null);
            break;
          case "uuid":
            joiSchema[col.column] = Joi.string().guid().allow(null);
            break;
          case "enum":
            joiSchema[col.column] = Joi.string().allow(null);
            break;
          case "array":
            joiSchema[col.column] = Joi.array().allow(null);
            break;
          default:
            joiSchema[col.column] = Joi.any().allow(null);
        }
      }
    });

    const schemaObject = Joi.object(joiSchema);

    const validatedData = parsedData.map((item) => {
      const { value, error } = schemaObject.validate(item);
      if (error) {
        throw new Error(`Validation error: ${error.message}`);
      }
      return value;
    });

    return { success: true, data: validatedData };
  } catch (error) {
    return { success: false, message: error.message };
  }
}
```




<!--- Eraser file: https://app.eraser.io/workspace/4cWnKs1DpRdit4zzrhl6 --->