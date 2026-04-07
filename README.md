# bynry-case-study
Backend case study solution for Bynry. This project includes API design, database schema, and low-stock alert implementation for a scalable inventory management system.
Inventory Management System for B2B SaaS
Overview
You're joining a team building "StockFlow" - a B2B inventory management platform. Small businesses use it to track products across multiple warehouses and manage supplier relationships.
Time Allocation
Take-Home Portion: 90 minutes maximum
Live Discussion: 30-45 minutes (scheduled separately)

Part 1: Code Review & Debugging (30 minutes)
A previous intern wrote this API endpoint for adding new products. Something is wrong - the code compiles but doesn't work as expected in production.
@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json
    
    # Create new product
    product = Product(
        name=data['name'],
        sku=data['sku'],
        price=data['price'],
        warehouse_id=data['warehouse_id']
    )
    
    db.session.add(product)
    db.session.commit()
    
    # Update inventory count
    inventory = Inventory(
        product_id=product.id,
        warehouse_id=data['warehouse_id'],
        quantity=data['initial_quantity']
    )
    
    db.session.add(inventory)
    db.session.commit()
    
    return {"message": "Product created", "product_id": product.id}

Your Tasks:
Identify Issues:
1. No input validation for request data
2. No transaction handling (multiple commits)
3. SKU uniqueness not enforced
4. Incorrect data model (product tied to one warehouse)
5. No error handling (try-catch missing)
6. Price not validated (can be negative or wrong type)
7. Optional fields not handled properly
8. No HTTP status codes in response
Explain Impact: 
1. No input validation:
If required fields are missing, the API will crash with errors, making it unreliable.
2. No transaction handling:
If product is created but inventory fails, inconsistent data will be stored in the database.
3. SKU not unique:
Duplicate SKUs can cause confusion in inventory tracking and reporting.
4. Incorrect data model:
Product is linked to only one warehouse, but requirement says multiple warehouses, so system will not scale.
5. No error handling:
Any database or runtime error will crash the application instead of returning a proper response.
6. Price not validated:
Invalid or negative prices may be stored, leading to incorrect business data.

7. Optional fields not handled:
Missing optional fields like initial_quantity may cause runtime errors.
8. No HTTP status codes:
Client cannot properly understand success or failure of API request.

Provide Fixes: 
@app.route('/api/products', methods=['POST'])
def create_product():
    try:
        data = request.json

        # Validate required fields
        required_fields = ['name', 'sku', 'price']
        for field in required_fields:
            if field not in data:
                return {"error": f"{field} is required"}, 400

        # Validate price
        if float(data['price']) < 0:
            return {"error": "Price must be positive"}, 400

        # Check SKU uniqueness
        existing_product = Product.query.filter_by(sku=data['sku']).first()
        if existing_product:
            return {"error": "SKU already exists"}, 400

        # Use transaction
        with db.session.begin():

            # Create product (no warehouse_id here)
            product = Product(
                name=data['name'],
                sku=data['sku'],
                price=float(data['price'])
            )
            db.session.add(product)

            # Create inventory if warehouse info provided
            if 'warehouse_id' in data:
                inventory = Inventory(
                    product=product,
                    warehouse_id=data['warehouse_id'],
                    quantity=data.get('initial_quantity', 0)
                )
                db.session.add(inventory)

        return {
            "message": "Product created",
            "product_id": product.id
        }, 201

    except Exception as e:
        db.session.rollback()
        return {"error": str(e)}, 500
Used proper HTTP status codes:
- 201 for successful product creation
- 400 for client errors (invalid input, duplicate SKU)
- 500 for server errors
Additional Context (you may need to ask for more):
Products can exist in multiple warehouses
SKUs must be unique across the platform
Price can be decimal values
Some fields might be optional

Part 2: Database Design (25 minutes)
Based on the requirements below, design a database schema. Note: These requirements are intentionally incomplete - you should identify what's missing.
Given Requirements:
Companies can have multiple warehouses
Products can be stored in multiple warehouses with different quantities
Track when inventory levels change
Suppliers provide products to companies
Some products might be "bundles" containing other products
Your Tasks:

Design Schema: 
1. Companies Table
id (INT, PRIMARY KEY)
name (VARCHAR)
created_at (TIMESTAMP)


2. Warehouses Table
id (INT, PRIMARY KEY)
company_id (INT, FOREIGN KEY → Companies.id)
name (VARCHAR)
location (VARCHAR)
One Company → Many Warehouses 

3. Products Table
id (INT, PRIMARY KEY)
company_id (INT, FOREIGN KEY → Companies.id)
name (VARCHAR)
sku (VARCHAR, UNIQUE)
price DECIMAL(10,2)
product_type (ENUM: 'normal', 'bundle')
threshold (INT)
One Company → Many Products 

        4. Inventory Table
       id INT PRIMARY KEY
       product_id INT (FK → Products.id)
       warehouse_id INT (FK → Warehouses.id)
       quantity INT
       UNIQUE (product_id, warehouse_id)
Same product in multiple warehouses
Different quantity per warehouse

   5. Suppliers Table
  id INT PRIMARY KEY
  name VARCHAR(255)
  contact_email VARCHAR(255)


 6. Product_Suppliers
product_id INT (FK → Products.id)
supplier_id INT (FK → Suppliers.id)
PRIMARY KEY (product_id, supplier_id)
Many-to-many relationship

7.Inventory_Logs
id INT PRIMARY KEY
product_id INT (FK → Products.id)
warehouse_id INT (FK → Warehouses.id)
change_quantity INT
timestamp TIMESTAMP
reason VARCHAR(255)
Tracks stock changes over time

8.Bundles 
bundle_id INT (FK → Products.id)
product_id INT (FK → Products.id)
quantity INT
PRIMARY KEY (bundle_id, product_id)
A bundle contains multiple products

Identify Gaps: 
1. Is SKU unique globally or per company?
2. Can a product have multiple suppliers?
3. What defines "recent sales activity"?
4. Can inventory quantity go negative?
5. Is threshold per product or per warehouse?
6. Are warehouses shared between companies?

Explain Decisions:.
- Used Inventory table to support multiple warehouses per product.
- Enforced unique SKU to avoid duplicate products.
- Used Inventory_Logs to track all stock changes.
- Used many-to-many mapping for product-supplier flexibility.
- Used normalized schema to improve scalability and avoid redundancy.
Format: Use any notation (SQL DDL, ERD, text description, etc.)

Part 3: API Implementation (35 minutes)
Implement an endpoint that returns low-stock alerts for a company.
Business Rules (discovered through previous questions):
Low stock threshold varies by product type
Only alert for products with recent sales activity
Must handle multiple warehouses per company
Include supplier information for reordering
Endpoint Specification:
GET /api/companies/{company_id}/alerts/low-stock

Expected Response Format:
{
  "alerts": [
    {
      "product_id": 123,
      "product_name": "Widget A",
      "sku": "WID-001",
      "warehouse_id": 456,
      "warehouse_name": "Main Warehouse",
      "current_stock": 5,
      "threshold": 20,
      "days_until_stockout": 12,
      "supplier": {
        "id": 789,
        "name": "Supplier Corp",
        "contact_email": "orders@supplier.com"
      }
    }
  ],
  "total_alerts": 1
}

Your Tasks:
Write Implementation
- Threshold is stored in the Product table.
- Recent sales activity means sales in last 30 days.
- Each product has at least one supplier.
- One product can exist in multiple warehouses.
Assumptions:
- Threshold value is stored at the product level in the Products table.
- Recent sales activity is defined as sales within the last 30 days.
- Each product has at least one associated supplier for reordering.
- A product can exist in multiple warehouses with different inventory quantities.
- If threshold is not defined, a default value of 10 is used.
- Inventory quantity is assumed to be non-negative.


Handle Edge Cases: 
@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def get_low_stock_alerts(company_id):
    try:
        alerts = []

        # Fetch all products for the company
        products = Product.query.filter_by(company_id=company_id).all()

        for product in products:
            # Get inventory records for each product
            inventories = Inventory.query.filter_by(product_id=product.id).all()

            for inv in inventories:
                threshold = product.threshold if product.threshold else 10

                # Assume function to get recent sales count
                recent_sales = get_recent_sales(product.id, days=30)

                # Check low stock condition
                if inv.quantity < threshold and recent_sales > 0:

                    # Get supplier info (assume one supplier)
                    supplier = get_supplier(product.id)

                    alerts.append({
                        "product_id": product.id,
                        "product_name": product.name,
                        "sku": product.sku,
                        "warehouse_id": inv.warehouse_id,
                        "warehouse_name": get_warehouse_name(inv.warehouse_id),
                        "current_stock": inv.quantity,
                        "threshold": threshold,
                        "days_until_stockout": estimate_stockout(product.id),
                        "supplier": {
                            "id": supplier.id,
                            "name": supplier.name,
                            "contact_email": supplier.contact_email
                        }
                    })

        return {
            "alerts": alerts,
            "total_alerts": len(alerts)
        }, 200

    except Exception as e:
        return {"error": str(e)}, 500

Note: Helper functions such as get_recent_sales(), get_supplier(), get_warehouse_name(), and estimate_stockout() are assumed to be implemented as part of the system.

Explain Approach:
- First, fetch all products for the given company.
- Then, retrieve inventory data for each product across warehouses.
- Check if stock is below the threshold.
- Filter only products with recent sales activity.
- Fetch supplier details for reordering.
- Build response in required format.
     Edge Cases:
         1. Product has no inventory → skip or return empty
         2. No recent sales → no alert generated
        3. No supplier assigned → handle null safely
        4. Threshold not defined → use default value
        5. Large data → performance issues (can optimize using JOIN queries)
    Improvements:
      Use JOIN queries instead of loops for better performance
      Add pagination for large datasets
      Use caching for frequent API calls
      Add indexing on product_id and warehouse_id

Hints: You'll need to make assumptions about the database schema and business logic. Document these assumptions.
      Conclusion:
This solution ensures scalability, data consistency, and efficient inventory management while handling real-world business scenarios such as multi-warehouse support, supplier integration, and low-stock alerting.

Submission Instructions
Create a document with your responses to all three parts
Include reasoning for each decision you made
List assumptions you had to make due to incomplete requirements
Submit within 90 minutes of receiving this case study
Be prepared to walk through your solutions in the live session

Live Session Topics (Preview)
During our video call, we'll discuss:
Your debugging approach and thought process
Database design trade-offs and scalability considerations
How you'd handle edge cases in your API implementation
Questions about missing requirements and how you'd gather more info
Alternative approaches you considered

Evaluation Criteria
Technical Skills:
Code quality and best practices
Database design principles
Understanding of API design
Problem-solving approach
Communication:
Ability to identify and ask about ambiguities
Clear explanation of technical decisions
Professional collaboration style
Business Understanding:
Recognition of real-world constraints
Consideration of user experience
Scalability and maintenance thinking

Note: This is designed to assess your current skill level and learning potential. We don't expect perfect solutions - we're more interested in your thought process and ability to work with incomplete information.

