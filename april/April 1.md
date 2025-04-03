<p><a target="_blank" href="https://app.eraser.io/workspace/0rb2ltUSXMZezSBc0VVA" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

so what i have to do today April 1st

- [x] Make a centralized service to execute multiple queries
Those can be of any type such as 
- Create Table Queries
- Alter Table Queries
- Drop Table Queries

I have to parse each of the SQL(s) and check how many number of tables need to be created 
and all 

I will have to test for the case when there are high number of tables to be dropped

- [x] Make the admin query execute service to be capable of handling all kind of queries not just limited to the DML (select | update | delete | truncate | insert) only
- [x] Complete all the cases in the SQL file execution
- [x] Make an API to download the sql file report execution
- [x] Unit test all of the above features and make sure they don't create any problems
create two tables 

```

-- Table 1 creation
CREATE TABLE table1 (
  id INT PRIMARY KEY AUTO_INCREMENT,
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL
);

-- Inserting values into table1
INSERT INTO table1 (first_name, last_name) 
VALUES 
  ('shivam', 'shukla'),
  ('Amit', 'sir'),
  ('Nishant', 'Sir');

-- Table 2 creation
CREATE TABLE table2 (
  id INT PRIMARY KEY AUTO_INCREMENT,
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL
);

-- Table 3 creation (fixed the duplicate CREATE statement)
CREATE TABLE table3 (
  id INT PRIMARY KEY AUTO_INCREMENT,
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL
);

-- Table 4 creation
CREATE TABLE table4 (
  id INT PRIMARY KEY AUTO_INCREMENT,
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL
);

-- Table 5 creation
CREATE TABLE table5 (
  id INT PRIMARY KEY AUTO_INCREMENT,
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL
);
```




<!--- Eraser file: https://app.eraser.io/workspace/0rb2ltUSXMZezSBc0VVA --->