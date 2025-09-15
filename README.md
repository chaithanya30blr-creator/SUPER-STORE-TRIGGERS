# SUPER-STORE-TRIGGERS
-- Drop tables if they exist to start with a clean slate
DROP TABLE IF EXISTS sales_fact;
DROP TABLE IF EXISTS products_dim;

-- Create the products dimension table
CREATE TABLE products_dim (
    product_id VARCHAR(50) PRIMARY KEY,
    product_name VARCHAR(255),
    category VARCHAR(100),
    subcategory VARCHAR(100),
    stocklevel INTEGER
);

-- Create the sales fact table
CREATE TABLE sales_fact (
    order_id VARCHAR(50) PRIMARY KEY,
    product_id VARCHAR(50) REFERENCES products_dim(product_id),
    quantity_sold INTEGER,
    sale_date DATE
);

-- Populate the products_dim table with some sample data and initial stock levels
INSERT INTO products_dim (product_id, product_name, category, subcategory, stocklevel) VALUES
('OFF-PA-10001970', 'Easy-staple paper', 'Office Supplies', 'Paper', 50),
('FUR-CH-10000454', 'Hon Deluxe Fabric Upholstered Stacking Chairs', 'Furniture', 'Chairs', 10),
('TEC-AC-10003399', 'Memorex Mini Travel Drive 64 GB USB 2.0 Flash Drive', 'Technology', 'Accessories', 25);

CREATE OR REPLACE FUNCTION stocklevel_trigger_function()
RETURNS TRIGGER AS $$
DECLARE
    current_stock INTEGER;
BEGIN
    -- Get the current stock level for the product being sold
    SELECT stocklevel INTO current_stock
    FROM products_dim
    WHERE product_id = NEW.product_id;

    -- Check if there is enough stock to fulfill the order
    IF current_stock >= NEW.quantity_sold THEN
        -- If stock is sufficient, update the stock level
        UPDATE products_dim
        SET stocklevel = stocklevel - NEW.quantity_sold
        WHERE product_id = NEW.product_id;
        
        -- Allow the INSERT to proceed
        RETURN NEW;
    ELSE
        -- If stock is insufficient, raise an exception to prevent the transaction
        RAISE EXCEPTION 'Insufficient stock for product %! Current stock: %, Order quantity: %',
            NEW.product_id, current_stock, NEW.quantity_sold;
        
        -- The transaction is rolled back, so no data is inserted
        RETURN NULL;
    END IF;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER stocklevel_trigger
BEFORE INSERT ON sales_fact
FOR EACH ROW
EXECUTE FUNCTION stocklevel_trigger_function();


-- Initial stock level for the product
SELECT * FROM products_dim WHERE product_id = 'OFF-PA-10001970';

-- Fictitious transaction: Insert a new sales record
INSERT INTO sales_fact (order_id, product_id, quantity_sold, sale_date) VALUES
('CA-2016-152156', 'OFF-PA-10001970', 5, '2016-11-08');

-- Check the updated stock level for the product
SELECT * FROM products_dim WHERE product_id = 'OFF-PA-10001970';

-- Initial stock level for the product
SELECT * FROM products_dim WHERE product_id = 'FUR-CH-10000454';

-- Fictitious transaction: Attempt to insert a new sales record that exceeds stock
INSERT INTO sales_fact (order_id, product_id, quantity_sold, sale_date) VALUES
('CA-2016-152157', 'FUR-CH-10000454', 30, '2016-11-08');

-- Check the stock level for the product (it should be unchanged)
SELECT * FROM products_dim WHERE product_id = 'FUR-CH-10000454';

-- Increase the stock level for the product that was out of stock
UPDATE products_dim
SET stocklevel = 30 -- Or any number greater than the quantity sold
WHERE product_id = 'FUR-CH-10000454';

-- Now, re-run the transaction that previously failed
INSERT INTO sales_fact (order_id, product_id, quantity_sold, sale_date) VALUES
('CA-2016-152157', 'FUR-CH-10000454', 30, '2016-11-08');

UPDATE products_dim
SET stocklevel = 50 -- Or any number greater than 30
WHERE product_id = 'FUR-CH-10000454';

INSERT INTO sales_fact (order_id, product_id, quantity_sold, sale_date) VALUES
('CA-2016-152158', 'FUR-CH-10000454', 30, '2016-11-08');

-- Now, check the updated stock level
SELECT * FROM products_dim WHERE product_id = 'FUR-CH-10000454';
