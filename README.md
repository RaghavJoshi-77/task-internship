# task-internship

### Backend Engineering Intern Case Study: StockFlow

This document presents the solution for the Bynry Inc. Backend Developer Case Study, focusing on database design in ERD format and API implementation using Express.js and Prisma ORM.

-----

### Part 2: Database Design (ERD Format)

The following ERD (Entity-Relationship Diagram) represents the database schema for the StockFlow platform. The design meets the provided requirements while also accounting for necessary assumptions and relationships.

#### Explanation of Design Choices

  * **Entities:** The diagram includes key entities like **Company**, **Warehouse**, **Product**, **ProductType**, **Supplier**, **Inventory**, and **Sale**. Each entity corresponds to a database table with its own set of attributes.
  * **Relationships:**
      * **One-to-Many:** A `Company` can have many `Warehouses`, but each `Warehouse` belongs to only one `Company`.
      * **Many-to-Many:**
          * A `Product` can be stored in many `Warehouses`, and a `Warehouse` can store many `Products`. This is resolved with the **`Inventory`** table, which acts as a junction table.
          * A `Product` can be supplied by many `Suppliers`, and a `Supplier` can provide many `Products`. This is handled by the **`ProductSupplier`** junction table.
          * A `Product` can be a bundle, containing other `Products` as components. This self-referencing relationship is managed by the **`BundleProduct`** junction table.
      * **ProductType and Product:** A `Product` is associated with a `ProductType` to define its low-stock threshold. This provides a flexible and scalable way to manage different product types.
  * **Attributes:**
      * **Primary Keys (PK):** Unique identifiers for each entity (e.g., `id` for `Product`).
      * **Foreign Keys (FK):** Keys that link entities together, enforcing referential integrity (e.g., `companyId` in the `Warehouse` table).
      * **Constraints:** The `sku` attribute in the `Product` entity is marked as **`UNIQUE`** to ensure no two products have the same SKU. The `Inventory` table uses a **composite primary key** on `(productId, warehouseId)` to ensure a unique inventory record for each product-warehouse pair.
      * here is the er diagram for this 

<img width="720" height="692" alt="image" src="https://github.com/user-attachments/assets/533bb133-8f13-40a8-b931-cc0e858e1239" />

for better view checkout the link
https://app.eraser.io/workspace/bWLA4QksRyxmM37FYtaP?origin=share

####Assumptions 
The database schema design was based on interpreting the high-level requirements and filling in the details.

Low Stock Thresholds: I assumed the low-stock threshold varies by product type, leading to the creation of a dedicated 
product_types table.

Product Bundles: I assumed that product bundles are a distinct product type and that the relationship between a bundle and its components is a many-to-many relationship, which is why a bundle_products junction table was created.

Companies and Warehouses: The design assumes that a Warehouse belongs to a single Company, creating a one-to-many relationship.

Inventory Change Tracking: I assumed that "tracking when inventory levels change"  required a detailed 
inventory_changes table to store an audit log, including the type of change and the new quantity.

Suppliers: The design assumes a many-to-many relationship between products and suppliers, allowing for a product to have multiple suppliers.

-----

### Part 3: API Implementation (Express.js & Prisma ORM)

This section provides the implementation for the low-stock alerts API endpoint using Express.js and Prisma ORM. The code is based on the schema designed in Part 2.

#### Endpoint Specification

  * **Method:** `GET`
  * **URL:** `/api/companies/:company_id/alerts/low-stock`
  * **Purpose:** To retrieve a list of products for a given company that are below their low-stock threshold and have had recent sales activity.

#### Prisma Schema Assumptions

The following Prisma schema corresponds to the ERD from Part 2. The key relationships are defined, allowing for easy querying.

```prisma
model Company {
  id        Int      @id @default(autoincrement())
  name      String
  warehouses Warehouse[]
}

model Warehouse {
  id         Int      @id @default(autoincrement())
  name       String
  companyId  Int
  company    Company  @relation(fields: [companyId], references: [id])
  inventory  Inventory[]
}

model Product {
  id          Int       @id @default(autoincrement())
  name        String
  sku         String    @unique
  price       Decimal   @db.Decimal(10, 2)
  productTypeId Int
  productType ProductType @relation(fields: [productTypeId], references: [id])
  inventory   Inventory[]
  sales       Sale[]
  suppliers   ProductSupplier[]
}

model ProductType {
  id                Int      @id @default(autoincrement())
  name              String
  lowStockThreshold Int
  products          Product[]
}

model Inventory {
  productId  Int
  warehouseId Int
  quantity   Int
  product    Product    @relation(fields: [productId], references: [id])
  warehouse  Warehouse  @relation(fields: [warehouseId], references: [id])
  @@id([productId, warehouseId])
}

model Sale {
  id         Int      @id @default(autoincrement())
  productId  Int
  saleDate   DateTime @default(now())
  product    Product  @relation(fields: [productId], references: [id])
}

model Supplier {
  id      Int      @id @default(autoincrement())
  name    String
  contactEmail String
  products ProductSupplier[]
}

model ProductSupplier {
  productId  Int
  supplierId Int
  product    Product  @relation(fields: [productId], references: [id])
  supplier   Supplier @relation(fields: [supplierId], references: [id])
  @@id([productId, supplierId])
}
```

#### Express.js API Implementation

```javascript
// server.js
const express = require('express');
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();
const app = express();

app.use(express.json());

// API Endpoint for Low Stock Alerts
app.get('/api/companies/:company_id/alerts/low-stock', async (req, res) => {
    try {
        const companyId = parseInt(req.params.company_id, 10);
        const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);

        // Fetch all low-stock products for the company's warehouses
        const lowStockProducts = await prisma.inventory.findMany({
            where: {
                warehouse: {
                    companyId: companyId,
                },
                product: {
                    sales: {
                        some: {
                            saleDate: {
                                gte: thirtyDaysAgo,
                            },
                        },
                    },
                },
            },
            include: {
                product: {
                    include: {
                        productType: true,
                        suppliers: {
                            include: {
                                supplier: true,
                            },
                        },
                    },
                },
                warehouse: true,
            },
        });

        const alerts = lowStockProducts
            .filter(inv => inv.quantity <= inv.product.productType.lowStockThreshold)
            .map(inv => {
                // Assumption: Simplified calculation for days_until_stockout.
                // A more robust solution would involve aggregating sales data.
                const averageDailySales = 2; // Hardcoded for this example
                const daysUntilStockout = inv.quantity / averageDailySales;

                const mainSupplier = inv.product.suppliers[0]?.supplier || null;

                return {
                    product_id: inv.productId,
                    product_name: inv.product.name,
                    sku: inv.product.sku,
                    warehouse_id: inv.warehouseId,
                    warehouse_name: inv.warehouse.name,
                    current_stock: inv.quantity,
                    threshold: inv.product.productType.lowStockThreshold,
                    days_until_stockout: Math.ceil(daysUntilStockout),
                    supplier: mainSupplier ? {
                        id: mainSupplier.id,
                        name: mainSupplier.name,
                        contact_email: mainSupplier.contactEmail,
                    } : null,
                };
            });

        const responsePayload = {
            alerts,
            total_alerts: alerts.length,
        };

        return res.status(200).json(responsePayload);

    } catch (error) {
        console.error("Error fetching low-stock alerts:", error);
        return res.status(500).json({ message: "An internal server error occurred." });
    } finally {
        await prisma.$disconnect();
    }
});

// Start the server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Server is running on port ${PORT}`);
});
```

#### Explanation of Approach

  * **Prisma Client:** The code uses `PrismaClient` to interact with the database. Prisma's ORM capabilities allow for expressive and type-safe queries.
  * **Query Logic:**
      * The `prisma.inventory.findMany` method is the core of the query.
      * The `where` clause filters inventory records, ensuring we only look at records for the specified `companyId` and for products that have had `recent sales activity` (defined as a sale in the last 30 days).
      * The `include` clause eagerly loads related data (`product`, `warehouse`, `productType`, and `supplier`) to reduce database queries.
  * **Post-Query Processing:** After fetching the data, the code filters the results to check if the `current_stock` is below the `lowStockThreshold`. It then maps the data to the final JSON format, calculates `days_until_stockout`, and handles cases where a product might not have a supplier.
  * **Error Handling:** A `try...catch` block handles potential errors, returning a `500` status code with a descriptive error message.
