# Mini Case — 商业零售/订货系统（改进版）

以下为对原始 Mini Case 文档的整理与规范化版本，目标是让 ERD 更清晰、便于生成 SQL DDL 并可直接用于建模工具。

## 1. 实体与属性（已标注主键 key）

- Customer (customer_id: INTEGER key, name: VARCHAR, email: VARCHAR, customer_type: VARCHAR, total_purchase: DECIMAL)
- Invoice (invoice_id: INTEGER key, invoice_date: DATE, customer_id: INTEGER, total_amount: DECIMAL, shipping_address: VARCHAR, shipping_fee: DECIMAL, status: VARCHAR, payment_status: VARCHAR)
- InvoiceLine (invoice_line_id: INTEGER key, invoice_id: INTEGER, product_id: INTEGER, quantity: INTEGER, unit_price: DECIMAL)
- Product (product_id: INTEGER key, name: VARCHAR, product_type: VARCHAR, scale: VARCHAR, unit_price: DECIMAL, quantity_on_hand: INTEGER, min_inventory: INTEGER, manufacturer_id: INTEGER, last_sale_date: DATE)
- Manufacturer (manufacturer_id: INTEGER key, name: VARCHAR, website: VARCHAR)
- PurchaseOrder (purchase_order_id: INTEGER key, order_date: DATE, manufacturer_id: INTEGER, total_cost: DECIMAL, order_status: VARCHAR)
- PurchaseOrderLine (po_line_id: INTEGER key, purchase_order_id: INTEGER, product_id: INTEGER, quantity: INTEGER, unit_cost: DECIMAL)

> 说明：我为关联表（InvoiceLine、PurchaseOrderLine）补充了单独的主键（surrogate key）以利于扩展、审计与引用；也可以选择使用复合主键 (invoice_id, product_id) / (purchase_order_id, product_id) 代替。

## 2. 业务规则（精简与编号）

1. Customer 与 Invoice：
   - 一个 Customer 可有 0..N 张 Invoice。
   - 每张 Invoice 只属于一个 Customer。

2. Invoice 与 InvoiceLine：
   - 每张 Invoice 必须包含 1..N 个 InvoiceLine。
   - 每个 InvoiceLine 必属一张 Invoice。

3. Product 与 InvoiceLine：
   - 一个 Product 可出现在 0..N 个 InvoiceLine 中。
   - 每个 InvoiceLine 对应一个 Product。

4. Manufacturer 与 Product：
   - 一个 Manufacturer 可供应 1..N 个 Product。
   - 每个 Product 必须由一个 Manufacturer 提供。

5. Manufacturer 与 PurchaseOrder：
   - 一个 Manufacturer 可接收 0..N 个 PurchaseOrder。
   - 每个 PurchaseOrder 只针对一个 Manufacturer。

6. PurchaseOrder 与 PurchaseOrderLine：
   - 每个 PurchaseOrder 必须包含 1..N 个 PurchaseOrderLine。
   - 每个 PurchaseOrderLine 对应一个 Product。

## 3. 关键假设（已澄清）

1. backorder（缺货）状态以 `Invoice.status` 字段表示（例如："backorder"、"shipped" 等）。
2. 支付网关（信用卡银行）为外部系统，不在本数据库建模范围内。支付结果通过 `payment_status` 字段记录（例如："paid","pending","failed"）。
3. 自动补货规则：当 `Product.quantity_on_hand <= Product.min_inventory` 时，系统会触发下达 PurchaseOrder。
4. 产品报废判定：若 `Product.last_sale_date` 超过预定阈值（例如 4 周）且业务规则决定报废，产品记录仍保留（软删除或增加 `is_scrapped` 标志）。
5. 业务中允许保留历史销售/订单记录用于审计，因此对相关表采用 INSERT-only 模式（不物理删除）。

## 4. 建议的约束与索引

- 为每个外键建立 FK 约束并指定删除/更新行为（例如：ON DELETE RESTRICT 或 ON DELETE CASCADE，根据业务决定）。
- 常用查询字段（如 `customer_id`, `product_id`, `invoice_date`）建议建立索引。
- 对 `Customer.email` 添加 UNIQUE 约束（若业务要求邮箱唯一）。

## 5. 示例 SQL DDL（MySQL 风格，供参考）

```sql
CREATE TABLE Customer (
  customer_id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  email VARCHAR(200),
  customer_type VARCHAR(50),
  total_purchase DECIMAL(12,2) DEFAULT 0
);

CREATE TABLE Manufacturer (
  manufacturer_id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  website VARCHAR(255)
);

CREATE TABLE Product (
  product_id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  product_type VARCHAR(100),
  scale VARCHAR(50),
  unit_price DECIMAL(10,2),
  quantity_on_hand INT DEFAULT 0,
  min_inventory INT DEFAULT 0,
  manufacturer_id INT NOT NULL,
  last_sale_date DATE,
  FOREIGN KEY (manufacturer_id) REFERENCES Manufacturer(manufacturer_id) ON DELETE RESTRICT
);

CREATE TABLE Invoice (
  invoice_id INT PRIMARY KEY AUTO_INCREMENT,
  invoice_date DATE NOT NULL,
  customer_id INT NOT NULL,
  total_amount DECIMAL(12,2),
  shipping_address VARCHAR(500),
  shipping_fee DECIMAL(10,2),
  status VARCHAR(50),
  payment_status VARCHAR(50),
  FOREIGN KEY (customer_id) REFERENCES Customer(customer_id) ON DELETE RESTRICT
);

CREATE TABLE InvoiceLine (
  invoice_line_id INT PRIMARY KEY AUTO_INCREMENT,
  invoice_id INT NOT NULL,
  product_id INT NOT NULL,
  quantity INT NOT NULL,
  unit_price DECIMAL(10,2),
  FOREIGN KEY (invoice_id) REFERENCES Invoice(invoice_id) ON DELETE CASCADE,
  FOREIGN KEY (product_id) REFERENCES Product(product_id) ON DELETE RESTRICT
);

CREATE TABLE PurchaseOrder (
  purchase_order_id INT PRIMARY KEY AUTO_INCREMENT,
  order_date DATE NOT NULL,
  manufacturer_id INT NOT NULL,
  total_cost DECIMAL(12,2),
  order_status VARCHAR(50),
  FOREIGN KEY (manufacturer_id) REFERENCES Manufacturer(manufacturer_id) ON DELETE RESTRICT
);

CREATE TABLE PurchaseOrderLine (
  po_line_id INT PRIMARY KEY AUTO_INCREMENT,
  purchase_order_id INT NOT NULL,
  product_id INT NOT NULL,
  quantity INT NOT NULL,
  unit_cost DECIMAL(10,2),
  FOREIGN KEY (purchase_order_id) REFERENCES PurchaseOrder(purchase_order_id) ON DELETE CASCADE,
  FOREIGN KEY (product_id) REFERENCES Product(product_id) ON DELETE RESTRICT
);
```

## 6. 建模/交付建议（后续可选）

1. 若你要我直接同步修改 `.erd` 文件（`university_crowsfoot.erd` 或另建 `MiniCase_Team23.erd`），我可以生成相应的 bigER 语法模型并保存为 `.erd` 文件。  
2. 我也可以把当前 DDL 转为种子数据脚本（小量样本）用于本地验证。  
3. 若你偏好使用复合主键（例如 InvoiceLine 的 (invoice_id, product_id)），我可以按该约定调整 DDL 与 ERD。

---

如果你同意这些改动，我可以：
- 1) 生成並写入一个 `.erd` 文件（基于上面实体/关系），
- 2) 或继续把 `Mini Case_Team 23.md` 再细化为课堂提交格式（含图像和简短摘要）。

请选择你要的下一步（生成 `.erd`、生成 SQL 种子、或只需要文档即可）。
