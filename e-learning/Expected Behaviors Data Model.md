<p><a target="_blank" href="https://app.eraser.io/workspace/kpRJDQ5PoMjqjtoylBhc" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

## we will make test cases and check for the expected behavior and mach with actual behaviour


1. add data model
```
-- Step 1: Create a database for the store
CREATE DATABASE fruit_store;

-- Step 2: Use the newly created database
USE fruit_store;

-- Step 3: Create the Categories table
CREATE TABLE Categories (
    category_id INT AUTO_INCREMENT PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL,
    description TEXT
);

-- Step 4: Create the Suppliers table
CREATE TABLE Suppliers (
    supplier_id INT AUTO_INCREMENT PRIMARY KEY,
    supplier_name VARCHAR(100) NOT NULL,
    contact_info VARCHAR(255)
);

-- Step 5: Create the Fruits table
CREATE TABLE Fruits (
    fruit_id INT AUTO_INCREMENT PRIMARY KEY,
    fruit_name VARCHAR(100) NOT NULL,
    category_id INT,
    price DECIMAL(10, 2) NOT NULL,
    supplier_id INT,
    description TEXT,
    FOREIGN KEY (category_id) REFERENCES Categories(category_id),
    FOREIGN KEY (supplier_id) REFERENCES Suppliers(supplier_id)
);

-- Step 6: Create the Inventory table
CREATE TABLE Inventory (
    inventory_id INT AUTO_INCREMENT PRIMARY KEY,
    fruit_id INT,
    stock_quantity INT NOT NULL,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (fruit_id) REFERENCES Fruits(fruit_id)
);

-- Step 7: Create the Orders table (for customers)
CREATE TABLE Orders (
    order_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_name VARCHAR(100),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(10, 2)
);

-- Step 8: Create the Order_Items table (linking orders and fruits)
CREATE TABLE Order_Items (
    order_item_id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT,
    fruit_id INT,
    quantity INT,
    price DECIMAL(10, 2),
    FOREIGN KEY (order_id) REFERENCES Orders(order_id),
    FOREIGN KEY (fruit_id) REFERENCES Fruits(fruit_id)
);

-- Step 9: Create a simple sample entry for categories
INSERT INTO Categories (category_name, description) VALUES
('Citrus', 'Fruits like oranges, lemons, and grapefruits'),
('Tropical', 'Fruits like bananas, mangoes, and pineapples'),
('Berries', 'Fruits like strawberries, blueberries, and raspberries');

-- Step 10: Create a sample entry for suppliers
INSERT INTO Suppliers (supplier_name, contact_info) VALUES
('Fresh Farms', '1234 Farm Road, Phone: 555-1234'),
('Tropical Supply Co.', '5678 Tropic Avenue, Phone: 555-5678');

-- Step 11: Create some sample fruits
INSERT INTO Fruits (fruit_name, category_id, price, supplier_id, description) VALUES
('Orange', 1, 1.25, 1, 'Sweet and tangy citrus fruit'),
('Mango', 2, 1.50, 2, 'Juicy tropical fruit'),
('Strawberry', 3, 3.00, 1, 'Sweet and flavorful berry');

-- Step 12: Add inventory data
INSERT INTO Inventory (fruit_id, stock_quantity) VALUES
(1, 100),
(2, 50),
(3, 200);

-- Step 13: Sample Order Entry
INSERT INTO Orders (customer_name, total_amount) VALUES
('John Doe', 10.00);

-- Step 14: Sample Order Item Entry
INSERT INTO Order_Items (order_id, fruit_id, quantity, price) VALUES
(1, 1, 4, 1.25),   -- 4 Oranges
(1, 2, 2, 1.50);   -- 2 Mangoes
```




<!--- Eraser file: https://app.eraser.io/workspace/kpRJDQ5PoMjqjtoylBhc --->