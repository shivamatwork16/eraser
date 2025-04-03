<p><a target="_blank" href="https://app.eraser.io/workspace/KPXGwgsmpwekD1yZ5pgJ" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

today i did

## project e learning
1. Provide proper validations in the  sub-category management service now no extra data will be used - done
2. provide comment support in the queries had to do a lot of hacking around the \n and some regex tricks as well - done
3. admin query execution service with full support of DML - done
4. do a testing for the join inner join outer join and all of its related things  - done
5. provide a way to execute the sql files as well 
6. update the query listing and make it multi category - done
7. Add new sub category id in the query question management service due to this i had to make changes in the {add query questino , edit query question , listing query question , details query question and all}
8.  fixed the course update api
```
const constants = require("../utls/constants");
const { admin } = require("./mysqlServices");

class QueryService {
  constructor(parser) {
    this.parser = parser;
  }

  async executeQueries({
    sql,
    userId,
    whiteListPattern = null,
    tableName = null,
    customValidation = null,
  }) {
    // Process queries
    const queries = sql
      .split(";")
      .map((q) => q.trim())
      .filter((q) => q.length > 0);

    const results = [];
    const errors = [];

    for (const query of queries) {
      try {
        // Run custom validation if provided
        if (customValidation) {
          await customValidation(query);
        }

        // Validate against whitelist if provided
        if (whiteListPattern) {
          this.parser.whiteListCheck(query, [whiteListPattern]);
        }

        // Execute query
        const result = await admin.execute(query, userId);

        if (!result?.success) {
          throw new Error(result?.errors?.message || "Query execution failed");
        }

        results.push({
          sql: query,
          data: result.results,
        });
      } catch (error) {
        const formattedError = this.formatError(error, query, tableName);
        errors.push(formattedError);
      }
    }

    return {
      success: errors.length === 0,
      results,
      errors,
    };
  }

  formatError(error, query, tableName) {
    let errorMessage = constants.DATABASE.ONLY_DML;
    let errorCode = 422;

    // console.log(this.parser);

    if (error instanceof this.parser.SyntaxError) {
      errorMessage = `Syntax error: ${error.message}`;
    } else if (error.message.includes("White list check fail")) {
      const [detectedTable] = this.parser.tableList(query);
      errorMessage = detectedTable
        ? `Table '${detectedTable}' is not allowed${tableName ? `. Only '${tableName}' is permitted` : ""}`
        : "Invalid table reference in query";
    } else if (error.message.includes("Query execution failed")) {
      errorCode = 400;
      errorMessage = error.message;
    } else if (error.message.includes("Invalid SQL input")) {
      errorCode = 400;
      errorMessage = error.message;
    }

    return {
      sql: query,
      error: {
        code: errorCode,
        message: errorMessage,
      },
    };
  }
}

module.exports = QueryService;
```


```
-- Customers Table
CREATE TABLE Customers (
    CustomerID INT PRIMARY KEY,
    FirstName VARCHAR(50),
    LastName VARCHAR(50),
    City VARCHAR(50),
    Country VARCHAR(50)
);

INSERT INTO Customers (CustomerID, FirstName, LastName, City, Country) VALUES
(1, 'John', 'Doe', 'New York', 'USA'),
(2, 'Jane', 'Smith', 'London', 'UK'),
(3, 'Robert', 'Jones', 'Paris', 'France'),
(4, 'Maria', 'Garcia', 'Madrid', 'Spain'),
(5, 'Ken', 'Yamamoto', 'Tokyo', 'Japan');

-- Orders Table
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY,
    CustomerID INT,
    OrderDate DATE,
    ProductID INT,
    Quantity INT,
    FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID)
);

INSERT INTO Orders (OrderID, CustomerID, OrderDate, ProductID, Quantity) VALUES
(101, 1, '2024-01-15', 1, 2),
(102, 2, '2024-02-01', 2, 1),
(103, 1, '2024-02-10', 3, 3),
(104, 3, '2024-02-15', 1, 1),
(105, 4, '2024-03-01', 4, 4),
(106, 2, '2024-03-10', 2, 2);

-- Products Table
CREATE TABLE Products (
    ProductID INT PRIMARY KEY,
    ProductName VARCHAR(50),
    Price DECIMAL(10, 2)
);

INSERT INTO Products (ProductID, ProductName, Price) VALUES
(1, 'Laptop', 1200.00),
(2, 'Mouse', 25.00),
(3, 'Keyboard', 75.00),
(4, 'Monitor', 300.00);
```




<!--- Eraser file: https://app.eraser.io/workspace/KPXGwgsmpwekD1yZ5pgJ --->