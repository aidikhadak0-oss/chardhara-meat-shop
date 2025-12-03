-- products: मासु/आइटम
CREATE TABLE products (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  cut_type TEXT,
  unit TEXT DEFAULT 'kg',
  qty REAL DEFAULT 0, -- kg
  purchase_price_per_kg REAL DEFAULT 0,
  sell_price_per_kg REAL DEFAULT 0,
  barcode TEXT,
  low_stock_threshold REAL DEFAULT 5.0,
  notes TEXT
);

-- stock_transactions: आया वा गएको स्टक र बिक्रीले घट्दा ट्र्याक गर्न
CREATE TABLE stock_transactions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  product_id INTEGER,
  type TEXT, -- 'in' or 'out'
  qty REAL,
  unit_price REAL, -- per kg (for cost or sale price)
  total_amount REAL, -- qty * unit_price
  date TEXT, -- ISO
  reference TEXT,
  related_sale_id INTEGER, -- nullable, FK to sales.id if 'out' from sale
  FOREIGN KEY(product_id) REFERENCES products(id)
);

-- sales: हरेक checkout को सार
CREATE TABLE sales (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  date TEXT,
  customer_id INTEGER, -- nullable (if credit sale, link customer)
  total_amount REAL,
  total_cost REAL, -- sum of purchase cost (for profit calc) [optional]
  profit REAL,
  payment_type TEXT, -- 'cash' or 'credit'
  paid_amount REAL, -- amount paid now (for partial credit)
  due_amount REAL, -- remaining if credit
  items_json TEXT, -- JSON array [{product_id, qty, price_per_kg, subtotal}]
  reference TEXT
);

-- customers: उधारो दिन/लिनेको दर्ता
CREATE TABLE customers (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT,
  phone TEXT,
  address TEXT,
  notes TEXT
);

-- ledgers (customer balances / transactions) — per customer ledger entries
CREATE TABLE customer_ledger (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  customer_id INTEGER,
  type TEXT, -- 'credit' (sale on credit), 'payment' (customer paid), 'adjustment'
  amount REAL, -- positive for credit (customer owes), negative for payment? (we'll standardize)
  date TEXT,
  reference TEXT,
  sale_id INTEGER, -- if linked to sale
  notes TEXT,
  FOREIGN KEY(customer_id) REFERENCES customers(id)
);dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter
  intl: ^0.18.0
  {
  "appTitle": "Meat Shop Manager",
  "dashboard": "Dashboard",
  "todaySales": "Today's Sales",
  "newSale": "New Sale",
  "stockIn": "Stock In",
  "products": "Products",
  "lowStock": "Low Stock"
}
  "appTitle": "मासु पसल व्यवस्थापक",
  "dashboard": "ड्यासबोर्ड",
  "todaySales": "आजको बिक्री",
  "newSale": "नयाँ बिक्री",
  "stockIn": "आएको सामान",
  "products": "सामानहरू",
  "lowStock": "कम स्टक"
}
Future<void> completeSale({
  required List<CartItem> items, // each: productId, qty, pricePerKg
  required String paymentType, // 'cash' or 'credit'
  required double paidNow,
  int? customerId,
}) async {
  final db = await DBHelper.db;
  final totalAmount = items.fold(0.0, (s, it) => s + it.qty * it.pricePerKg);
  // estimate total cost using purchase price from product table
  double totalCost = 0.0;
  for (var it in items) {
    final p = await DBHelper.getProductById(it.productId);
    totalCost += it.qty * p.purchasePrice;
  }
  final profit = totalAmount - totalCost;
  final due = (paymentType == 'credit') ? (totalAmount - paidNow) : 0.0;

  final saleId = await db.insert('sales', {
    'date': DateTime.now().toIso8601String(),
    'customer_id': customerId,
    'total_amount': totalAmount,
    'total_cost': totalCost,
    'profit': profit,
    'payment_type': paymentType,
    'paid_amount': paidNow,
    'due_amount': due,
    'items_json': jsonEncode(items.map((i)=>i.toMap()).toList())
  });

  // create stock_transactions and update product qty
  for (var it in items) {
    await db.insert('stock_transactions', {
      'product_id': it.productId,
      'type': 'out',
      'qty': it.qty,
      'unit_price': it.pricePerKg,
      'total_amount': it.qty * it.pricePerKg,
      'date': DateTime.now().toIso8601String(),
      'related_sale_id': saleId
    });
    // decrement product qty
    final p = await DBHelper.getProductById(it.productId);
    p.qty -= it.qty;
    await DBHelper.updateProduct(p);
  }

  // if credit, create ledger entry (customer owes due)
  if (paymentType == 'credit') {
    await db.insert('customer_ledger', {
      'customer_id': customerId,
      'type': 'credit',
      'amount': due,
      'date': DateTime.now().toIso8601String(),
      'sale_id': saleId
    });
    if (paidNow > 0) {
      // record payment part
      await db.insert('customer_ledger', {
        'customer_id': customerId,
        'type': 'payment',
        'amount': -paidNow, // or store positive with type distinguishing
        'date': DateTime.now().toIso8601String(),
        'sale_id': saleId
      });
    }
  }

  // optionally: print receipt / show success
}
SELECT
  c.id,
  c.name,
  COALESCE(SUM(CASE WHEN cl.type='credit' THEN cl.amount ELSE 0 END),0)
  + COALESCE(SUM(CASE WHEN cl.type='adjustment' THEN cl.amount ELSE 0 END),0)
  - COALESCE(SUM(CASE WHEN cl.type='payment' THEN cl.amount ELSE 0 END),0) AS balance
FROM customers c
LEFT JOIN customer_ledger cl ON c.id = cl.customer_id
GROUP BY c.id;
