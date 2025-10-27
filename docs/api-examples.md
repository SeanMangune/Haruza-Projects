# POS System API Examples and Code Samples

This document provides practical code examples for implementing POS system functionality across different programming languages and frameworks.

## Table of Contents

1. [REST API Examples](#rest-api-examples)
2. [JavaScript/Node.js Examples](#javascriptnodejs-examples)
3. [Python Examples](#python-examples)
4. [Java Examples](#java-examples)
5. [Database Queries](#database-queries)
6. [Payment Gateway Integration](#payment-gateway-integration)
7. [Real-time Updates](#real-time-updates)

## REST API Examples

### Product Management API

#### Get All Products
```http
GET /api/v1/products?page=1&limit=20&category=electronics
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

Response:
```json
{
  "status": "success",
  "data": {
    "products": [
      {
        "id": 1,
        "sku": "ELEC-001",
        "name": "Wireless Headphones",
        "description": "High-quality wireless headphones with noise cancellation",
        "price": 199.99,
        "cost": 120.00,
        "category": "Electronics",
        "barcode": "1234567890123",
        "stock_quantity": 25,
        "active": true,
        "created_at": "2024-01-15T10:00:00Z",
        "updated_at": "2024-01-15T10:00:00Z"
      }
    ],
    "pagination": {
      "current_page": 1,
      "per_page": 20,
      "total": 150,
      "total_pages": 8
    }
  }
}
```

#### Create New Product
```http
POST /api/v1/products
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "sku": "CLOTH-001",
  "name": "Cotton T-Shirt",
  "description": "100% cotton comfortable t-shirt",
  "price": 29.99,
  "cost": 15.00,
  "category_id": 2,
  "barcode": "9876543210987",
  "stock_quantity": 100,
  "tax_category": "standard"
}
```

### Transaction Processing API

#### Create Transaction
```http
POST /api/v1/transactions
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "customer_id": 12345,
  "employee_id": 67890,
  "location_id": 1,
  "items": [
    {
      "product_id": 1,
      "quantity": 2,
      "unit_price": 199.99,
      "discount_percent": 10
    },
    {
      "product_id": 25,
      "quantity": 1,
      "unit_price": 29.99
    }
  ],
  "payment_method": "credit_card",
  "tax_rate": 0.08,
  "discount_amount": 39.98
}
```

Response:
```json
{
  "status": "success",
  "data": {
    "transaction": {
      "id": 789123,
      "transaction_number": "TXN-2024-001234",
      "subtotal": 389.97,
      "tax_amount": 28.00,
      "discount_amount": 39.98,
      "total_amount": 377.99,
      "status": "completed",
      "created_at": "2024-01-15T14:30:00Z"
    }
  }
}
```

## JavaScript/Node.js Examples

### Express.js POS API Server

```javascript
const express = require('express');
const mysql = require('mysql2/promise');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');

const app = express();
app.use(express.json());

// Database connection
const dbConfig = {
  host: 'localhost',
  user: 'pos_user',
  password: 'secure_password',
  database: 'pos_system'
};

// Authentication middleware
const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }
  
  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) return res.status(403).json({ error: 'Invalid token' });
    req.user = user;
    next();
  });
};

// Product management endpoints
app.get('/api/v1/products', authenticateToken, async (req, res) => {
  try {
    const connection = await mysql.createConnection(dbConfig);
    const { page = 1, limit = 20, category, search } = req.query;
    const offset = (page - 1) * limit;
    
    let query = 'SELECT * FROM products WHERE active = 1';
    let params = [];
    
    if (category) {
      query += ' AND category_id = ?';
      params.push(category);
    }
    
    if (search) {
      query += ' AND (name LIKE ? OR sku LIKE ? OR barcode LIKE ?)';
      params.push(`%${search}%`, `%${search}%`, `%${search}%`);
    }
    
    query += ' LIMIT ? OFFSET ?';
    params.push(parseInt(limit), offset);
    
    const [products] = await connection.execute(query, params);
    
    // Get total count for pagination
    let countQuery = 'SELECT COUNT(*) as total FROM products WHERE active = 1';
    let countParams = [];
    
    if (category) {
      countQuery += ' AND category_id = ?';
      countParams.push(category);
    }
    
    if (search) {
      countQuery += ' AND (name LIKE ? OR sku LIKE ? OR barcode LIKE ?)';
      countParams.push(`%${search}%`, `%${search}%`, `%${search}%`);
    }
    
    const [countResult] = await connection.execute(countQuery, countParams);
    const total = countResult[0].total;
    
    await connection.end();
    
    res.json({
      status: 'success',
      data: {
        products,
        pagination: {
          current_page: parseInt(page),
          per_page: parseInt(limit),
          total,
          total_pages: Math.ceil(total / limit)
        }
      }
    });
  } catch (error) {
    res.status(500).json({ error: 'Database error', details: error.message });
  }
});

// Transaction processing
app.post('/api/v1/transactions', authenticateToken, async (req, res) => {
  const connection = await mysql.createConnection(dbConfig);
  
  try {
    await connection.beginTransaction();
    
    const { customer_id, employee_id, location_id, items, payment_method, tax_rate } = req.body;
    
    // Calculate totals
    let subtotal = 0;
    let total_discount = 0;
    
    for (const item of items) {
      const line_total = item.quantity * item.unit_price;
      const line_discount = line_total * (item.discount_percent || 0) / 100;
      subtotal += line_total;
      total_discount += line_discount;
    }
    
    const tax_amount = (subtotal - total_discount) * tax_rate;
    const total_amount = subtotal - total_discount + tax_amount;
    
    // Generate transaction number
    const transaction_number = `TXN-${new Date().getFullYear()}-${Date.now()}`;
    
    // Insert transaction
    const [transactionResult] = await connection.execute(
      `INSERT INTO transactions (transaction_number, customer_id, employee_id, location_id, 
       subtotal, tax_amount, discount_amount, total_amount, payment_method, status) 
       VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, 'completed')`,
      [transaction_number, customer_id, employee_id, location_id, 
       subtotal, tax_amount, total_discount, total_amount, payment_method]
    );
    
    const transaction_id = transactionResult.insertId;
    
    // Insert transaction items and update inventory
    for (const item of items) {
      const line_total = item.quantity * item.unit_price;
      const line_discount = line_total * (item.discount_percent || 0) / 100;
      const line_tax = (line_total - line_discount) * tax_rate;
      
      // Insert transaction item
      await connection.execute(
        `INSERT INTO transaction_items (transaction_id, product_id, quantity, 
         unit_price, discount_amount, tax_amount, line_total) 
         VALUES (?, ?, ?, ?, ?, ?, ?)`,
        [transaction_id, item.product_id, item.quantity, item.unit_price, 
         line_discount, line_tax, line_total - line_discount]
      );
      
      // Update inventory
      await connection.execute(
        'UPDATE inventory SET quantity_on_hand = quantity_on_hand - ? WHERE product_id = ? AND location_id = ?',
        [item.quantity, item.product_id, location_id]
      );
    }
    
    await connection.commit();
    
    res.json({
      status: 'success',
      data: {
        transaction: {
          id: transaction_id,
          transaction_number,
          subtotal,
          tax_amount,
          discount_amount: total_discount,
          total_amount,
          status: 'completed'
        }
      }
    });
    
  } catch (error) {
    await connection.rollback();
    res.status(500).json({ error: 'Transaction failed', details: error.message });
  } finally {
    await connection.end();
  }
});

// Start server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`POS API server running on port ${PORT}`);
});
```

### Frontend JavaScript (Vanilla JS)

```javascript
class POSSystem {
  constructor(apiBaseUrl, authToken) {
    this.apiBaseUrl = apiBaseUrl;
    this.authToken = authToken;
    this.currentTransaction = {
      items: [],
      subtotal: 0,
      tax: 0,
      total: 0
    };
  }
  
  // API helper method
  async apiCall(endpoint, method = 'GET', data = null) {
    const config = {
      method,
      headers: {
        'Authorization': `Bearer ${this.authToken}`,
        'Content-Type': 'application/json'
      }
    };
    
    if (data) {
      config.body = JSON.stringify(data);
    }
    
    const response = await fetch(`${this.apiBaseUrl}${endpoint}`, config);
    return await response.json();
  }
  
  // Product search
  async searchProducts(query) {
    const result = await this.apiCall(`/products?search=${encodeURIComponent(query)}`);
    return result.data.products;
  }
  
  // Add item to current transaction
  addItem(product, quantity = 1) {
    const existingItem = this.currentTransaction.items.find(item => item.product_id === product.id);
    
    if (existingItem) {
      existingItem.quantity += quantity;
    } else {
      this.currentTransaction.items.push({
        product_id: product.id,
        product_name: product.name,
        quantity: quantity,
        unit_price: product.price,
        discount_percent: 0
      });
    }
    
    this.calculateTotals();
    this.updateDisplay();
  }
  
  // Remove item from transaction
  removeItem(productId) {
    this.currentTransaction.items = this.currentTransaction.items.filter(
      item => item.product_id !== productId
    );
    this.calculateTotals();
    this.updateDisplay();
  }
  
  // Calculate transaction totals
  calculateTotals(taxRate = 0.08) {
    let subtotal = 0;
    let totalDiscount = 0;
    
    this.currentTransaction.items.forEach(item => {
      const lineTotal = item.quantity * item.unit_price;
      const lineDiscount = lineTotal * (item.discount_percent / 100);
      subtotal += lineTotal;
      totalDiscount += lineDiscount;
    });
    
    this.currentTransaction.subtotal = subtotal;
    this.currentTransaction.discount = totalDiscount;
    this.currentTransaction.tax = (subtotal - totalDiscount) * taxRate;
    this.currentTransaction.total = subtotal - totalDiscount + this.currentTransaction.tax;
  }
  
  // Process payment
  async processPayment(paymentMethod, customerId = null) {
    const transactionData = {
      customer_id: customerId,
      employee_id: this.getCurrentEmployeeId(),
      location_id: this.getCurrentLocationId(),
      items: this.currentTransaction.items,
      payment_method: paymentMethod,
      tax_rate: 0.08
    };
    
    try {
      const result = await this.apiCall('/transactions', 'POST', transactionData);
      
      if (result.status === 'success') {
        // Print receipt
        this.printReceipt(result.data.transaction);
        
        // Clear current transaction
        this.clearTransaction();
        
        // Show success message
        this.showMessage('Transaction completed successfully!', 'success');
        
        return result.data.transaction;
      } else {
        throw new Error(result.error || 'Transaction failed');
      }
    } catch (error) {
      this.showMessage(`Transaction failed: ${error.message}`, 'error');
      throw error;
    }
  }
  
  // Update display
  updateDisplay() {
    const itemsContainer = document.getElementById('transaction-items');
    const subtotalElement = document.getElementById('subtotal');
    const taxElement = document.getElementById('tax');
    const totalElement = document.getElementById('total');
    
    // Update items list
    itemsContainer.innerHTML = '';
    this.currentTransaction.items.forEach(item => {
      const itemElement = document.createElement('div');
      itemElement.className = 'transaction-item';
      itemElement.innerHTML = `
        <div class="item-details">
          <span class="item-name">${item.product_name}</span>
          <span class="item-price">$${item.unit_price.toFixed(2)} x ${item.quantity}</span>
        </div>
        <div class="item-total">$${(item.unit_price * item.quantity).toFixed(2)}</div>
        <button class="remove-item" onclick="pos.removeItem(${item.product_id})">×</button>
      `;
      itemsContainer.appendChild(itemElement);
    });
    
    // Update totals
    subtotalElement.textContent = `$${this.currentTransaction.subtotal.toFixed(2)}`;
    taxElement.textContent = `$${this.currentTransaction.tax.toFixed(2)}`;
    totalElement.textContent = `$${this.currentTransaction.total.toFixed(2)}`;
  }
  
  // Clear transaction
  clearTransaction() {
    this.currentTransaction = {
      items: [],
      subtotal: 0,
      tax: 0,
      total: 0
    };
    this.updateDisplay();
  }
  
  // Utility methods
  getCurrentEmployeeId() {
    return parseInt(localStorage.getItem('current_employee_id') || '1');
  }
  
  getCurrentLocationId() {
    return parseInt(localStorage.getItem('current_location_id') || '1');
  }
  
  showMessage(message, type = 'info') {
    const messageElement = document.getElementById('message');
    messageElement.textContent = message;
    messageElement.className = `message ${type}`;
    messageElement.style.display = 'block';
    
    setTimeout(() => {
      messageElement.style.display = 'none';
    }, 3000);
  }
  
  // Print receipt (placeholder)
  printReceipt(transaction) {
    console.log('Printing receipt for transaction:', transaction.transaction_number);
    // Implement actual receipt printing logic here
  }
}

// Initialize POS system
const pos = new POSSystem('/api/v1', localStorage.getItem('auth_token'));

// Event listeners
document.addEventListener('DOMContentLoaded', function() {
  // Barcode scanner input
  document.getElementById('barcode-input').addEventListener('keypress', async function(e) {
    if (e.key === 'Enter') {
      const barcode = this.value.trim();
      if (barcode) {
        const products = await pos.searchProducts(barcode);
        if (products.length > 0) {
          pos.addItem(products[0]);
          this.value = '';
        } else {
          pos.showMessage('Product not found', 'error');
        }
      }
    }
  });
  
  // Payment buttons
  document.getElementById('pay-cash').addEventListener('click', () => {
    pos.processPayment('cash');
  });
  
  document.getElementById('pay-card').addEventListener('click', () => {
    pos.processPayment('credit_card');
  });
  
  // Clear transaction button
  document.getElementById('clear-transaction').addEventListener('click', () => {
    pos.clearTransaction();
  });
});
```

## Python Examples

### Flask POS API

```python
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_jwt_extended import JWTManager, create_access_token, jwt_required, get_jwt_identity
from werkzeug.security import generate_password_hash, check_password_hash
from datetime import datetime, timedelta
import os

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = os.getenv('DATABASE_URL', 'mysql://user:password@localhost/pos_system')
app.config['JWT_SECRET_KEY'] = os.getenv('JWT_SECRET_KEY', 'your-secret-key')
app.config['JWT_ACCESS_TOKEN_EXPIRES'] = timedelta(hours=24)

db = SQLAlchemy(app)
jwt = JWTManager(app)

# Database Models
class Product(db.Model):
    __tablename__ = 'products'
    
    id = db.Column(db.BigInteger, primary_key=True)
    sku = db.Column(db.String(50), unique=True, nullable=False)
    name = db.Column(db.String(255), nullable=False)
    description = db.Column(db.Text)
    price = db.Column(db.Numeric(10, 2), nullable=False)
    cost = db.Column(db.Numeric(10, 2))
    category_id = db.Column(db.Integer)
    barcode = db.Column(db.String(50))
    active = db.Column(db.Boolean, default=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    def to_dict(self):
        return {
            'id': self.id,
            'sku': self.sku,
            'name': self.name,
            'description': self.description,
            'price': float(self.price),
            'cost': float(self.cost) if self.cost else None,
            'category_id': self.category_id,
            'barcode': self.barcode,
            'active': self.active,
            'created_at': self.created_at.isoformat(),
            'updated_at': self.updated_at.isoformat()
        }

class Transaction(db.Model):
    __tablename__ = 'transactions'
    
    id = db.Column(db.BigInteger, primary_key=True)
    transaction_number = db.Column(db.String(50), unique=True, nullable=False)
    location_id = db.Column(db.Integer, nullable=False)
    customer_id = db.Column(db.BigInteger)
    employee_id = db.Column(db.BigInteger)
    subtotal = db.Column(db.Numeric(10, 2), nullable=False)
    tax_amount = db.Column(db.Numeric(10, 2), nullable=False)
    discount_amount = db.Column(db.Numeric(10, 2), default=0)
    total_amount = db.Column(db.Numeric(10, 2), nullable=False)
    payment_method = db.Column(db.String(50))
    status = db.Column(db.Enum('pending', 'completed', 'cancelled', name='transaction_status'), default='pending')
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    def to_dict(self):
        return {
            'id': self.id,
            'transaction_number': self.transaction_number,
            'location_id': self.location_id,
            'customer_id': self.customer_id,
            'employee_id': self.employee_id,
            'subtotal': float(self.subtotal),
            'tax_amount': float(self.tax_amount),
            'discount_amount': float(self.discount_amount),
            'total_amount': float(self.total_amount),
            'payment_method': self.payment_method,
            'status': self.status,
            'created_at': self.created_at.isoformat()
        }

# API Routes
@app.route('/api/v1/products', methods=['GET'])
@jwt_required()
def get_products():
    page = request.args.get('page', 1, type=int)
    per_page = min(request.args.get('per_page', 20, type=int), 100)
    search = request.args.get('search', '')
    category = request.args.get('category', type=int)
    
    query = Product.query.filter(Product.active == True)
    
    if search:
        query = query.filter(
            db.or_(
                Product.name.contains(search),
                Product.sku.contains(search),
                Product.barcode.contains(search)
            )
        )
    
    if category:
        query = query.filter(Product.category_id == category)
    
    products = query.paginate(
        page=page, 
        per_page=per_page, 
        error_out=False
    )
    
    return jsonify({
        'status': 'success',
        'data': {
            'products': [product.to_dict() for product in products.items],
            'pagination': {
                'current_page': page,
                'per_page': per_page,
                'total': products.total,
                'total_pages': products.pages
            }
        }
    })

@app.route('/api/v1/products', methods=['POST'])
@jwt_required()
def create_product():
    data = request.get_json()
    
    # Validate required fields
    required_fields = ['sku', 'name', 'price']
    for field in required_fields:
        if field not in data:
            return jsonify({'error': f'Missing required field: {field}'}), 400
    
    # Check if SKU already exists
    if Product.query.filter_by(sku=data['sku']).first():
        return jsonify({'error': 'SKU already exists'}), 400
    
    try:
        product = Product(
            sku=data['sku'],
            name=data['name'],
            description=data.get('description'),
            price=data['price'],
            cost=data.get('cost'),
            category_id=data.get('category_id'),
            barcode=data.get('barcode')
        )
        
        db.session.add(product)
        db.session.commit()
        
        return jsonify({
            'status': 'success',
            'data': {'product': product.to_dict()}
        }), 201
        
    except Exception as e:
        db.session.rollback()
        return jsonify({'error': 'Failed to create product', 'details': str(e)}), 500

@app.route('/api/v1/transactions', methods=['POST'])
@jwt_required()
def create_transaction():
    data = request.get_json()
    employee_id = get_jwt_identity()
    
    try:
        # Calculate totals
        subtotal = 0
        total_discount = 0
        
        for item in data['items']:
            line_total = item['quantity'] * item['unit_price']
            line_discount = line_total * (item.get('discount_percent', 0) / 100)
            subtotal += line_total
            total_discount += line_discount
        
        tax_amount = (subtotal - total_discount) * data.get('tax_rate', 0.08)
        total_amount = subtotal - total_discount + tax_amount
        
        # Generate transaction number
        transaction_number = f"TXN-{datetime.now().year}-{int(datetime.now().timestamp())}"
        
        # Create transaction
        transaction = Transaction(
            transaction_number=transaction_number,
            location_id=data['location_id'],
            customer_id=data.get('customer_id'),
            employee_id=employee_id,
            subtotal=subtotal,
            tax_amount=tax_amount,
            discount_amount=total_discount,
            total_amount=total_amount,
            payment_method=data['payment_method'],
            status='completed'
        )
        
        db.session.add(transaction)
        db.session.flush()  # Get the transaction ID
        
        # Add transaction items (simplified - would need TransactionItem model)
        # ... transaction items logic ...
        
        db.session.commit()
        
        return jsonify({
            'status': 'success',
            'data': {'transaction': transaction.to_dict()}
        }), 201
        
    except Exception as e:
        db.session.rollback()
        return jsonify({'error': 'Transaction failed', 'details': str(e)}), 500

# Payment processing service
class PaymentProcessor:
    def __init__(self):
        self.gateway_url = os.getenv('PAYMENT_GATEWAY_URL')
        self.merchant_id = os.getenv('MERCHANT_ID')
        
    def process_payment(self, amount, payment_method, card_data=None):
        """Process payment through payment gateway"""
        import requests
        
        payload = {
            'merchant_id': self.merchant_id,
            'amount': amount,
            'payment_method': payment_method,
            'transaction_id': f"txn_{int(datetime.now().timestamp())}"
        }
        
        if payment_method == 'credit_card' and card_data:
            payload.update({
                'card_number': card_data['number'],
                'expiry_month': card_data['exp_month'],
                'expiry_year': card_data['exp_year'],
                'cvv': card_data['cvv']
            })
        
        try:
            response = requests.post(
                f"{self.gateway_url}/process",
                json=payload,
                timeout=30
            )
            
            return response.json()
            
        except requests.RequestException as e:
            return {'status': 'error', 'message': str(e)}

if __name__ == '__main__':
    app.run(debug=True)
```

## Database Queries

### Common POS Queries

```sql
-- Daily sales report
SELECT 
    DATE(created_at) as sale_date,
    COUNT(*) as transaction_count,
    SUM(subtotal) as gross_sales,
    SUM(discount_amount) as total_discounts,
    SUM(tax_amount) as total_tax,
    SUM(total_amount) as net_sales
FROM transactions 
WHERE DATE(created_at) = CURDATE()
    AND status = 'completed'
GROUP BY DATE(created_at);

-- Top selling products
SELECT 
    p.name,
    p.sku,
    SUM(ti.quantity) as total_sold,
    SUM(ti.line_total) as revenue
FROM transaction_items ti
JOIN products p ON ti.product_id = p.id
JOIN transactions t ON ti.transaction_id = t.id
WHERE t.created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
    AND t.status = 'completed'
GROUP BY p.id, p.name, p.sku
ORDER BY total_sold DESC
LIMIT 10;

-- Low stock alert
SELECT 
    p.name,
    p.sku,
    i.quantity_on_hand,
    i.reorder_point,
    i.reorder_quantity
FROM inventory i
JOIN products p ON i.product_id = p.id
WHERE i.quantity_on_hand <= i.reorder_point
    AND p.active = 1
ORDER BY i.quantity_on_hand ASC;

-- Customer purchase history
SELECT 
    c.first_name,
    c.last_name,
    COUNT(t.id) as transaction_count,
    SUM(t.total_amount) as total_spent,
    MAX(t.created_at) as last_purchase
FROM customers c
LEFT JOIN transactions t ON c.id = t.customer_id
WHERE t.status = 'completed'
GROUP BY c.id, c.first_name, c.last_name
ORDER BY total_spent DESC;

-- Hourly sales analysis
SELECT 
    HOUR(created_at) as hour_of_day,
    COUNT(*) as transaction_count,
    AVG(total_amount) as avg_transaction_value,
    SUM(total_amount) as total_sales
FROM transactions
WHERE DATE(created_at) = CURDATE()
    AND status = 'completed'
GROUP BY HOUR(created_at)
ORDER BY hour_of_day;
```

## Payment Gateway Integration

### Stripe Integration Example

```javascript
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);

class StripePaymentProcessor {
  async processPayment(amount, paymentMethodId, customerId = null) {
    try {
      const paymentIntent = await stripe.paymentIntents.create({
        amount: Math.round(amount * 100), // Convert to cents
        currency: 'usd',
        payment_method: paymentMethodId,
        customer: customerId,
        confirm: true,
        return_url: 'https://yourpos.com/payment-return'
      });
      
      return {
        success: true,
        payment_intent_id: paymentIntent.id,
        status: paymentIntent.status,
        amount: paymentIntent.amount / 100
      };
      
    } catch (error) {
      return {
        success: false,
        error: error.message,
        code: error.code
      };
    }
  }
  
  async processRefund(paymentIntentId, amount = null) {
    try {
      const refund = await stripe.refunds.create({
        payment_intent: paymentIntentId,
        amount: amount ? Math.round(amount * 100) : undefined
      });
      
      return {
        success: true,
        refund_id: refund.id,
        status: refund.status,
        amount: refund.amount / 100
      };
      
    } catch (error) {
      return {
        success: false,
        error: error.message
      };
    }
  }
}
```

## Real-time Updates

### WebSocket Implementation

```javascript
const WebSocket = require('ws');
const jwt = require('jsonwebtoken');

class POSWebSocketServer {
  constructor(server) {
    this.wss = new WebSocket.Server({ 
      server,
      verifyClient: this.verifyClient.bind(this)
    });
    
    this.clients = new Map(); // locationId -> Set of clients
    
    this.wss.on('connection', this.handleConnection.bind(this));
  }
  
  verifyClient(info) {
    const token = info.req.url.split('token=')[1];
    try {
      jwt.verify(token, process.env.JWT_SECRET);
      return true;
    } catch (error) {
      return false;
    }
  }
  
  handleConnection(ws, req) {
    const token = req.url.split('token=')[1];
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const locationId = decoded.location_id;
    
    // Add client to location group
    if (!this.clients.has(locationId)) {
      this.clients.set(locationId, new Set());
    }
    this.clients.get(locationId).add(ws);
    
    ws.on('close', () => {
      this.clients.get(locationId).delete(ws);
      if (this.clients.get(locationId).size === 0) {
        this.clients.delete(locationId);
      }
    });
    
    ws.on('message', (data) => {
      try {
        const message = JSON.parse(data);
        this.handleMessage(ws, message, locationId);
      } catch (error) {
        ws.send(JSON.stringify({ error: 'Invalid message format' }));
      }
    });
  }
  
  handleMessage(ws, message, locationId) {
    switch (message.type) {
      case 'inventory_update':
        this.broadcastToLocation(locationId, {
          type: 'inventory_updated',
          product_id: message.product_id,
          new_quantity: message.quantity
        });
        break;
        
      case 'price_change':
        this.broadcastToLocation(locationId, {
          type: 'price_updated',
          product_id: message.product_id,
          new_price: message.price
        });
        break;
    }
  }
  
  broadcastToLocation(locationId, message) {
    const clients = this.clients.get(locationId);
    if (clients) {
      const messageStr = JSON.stringify(message);
      clients.forEach(client => {
        if (client.readyState === WebSocket.OPEN) {
          client.send(messageStr);
        }
      });
    }
  }
  
  // Method to send updates from other parts of the application
  notifyInventoryUpdate(locationId, productId, newQuantity) {
    this.broadcastToLocation(locationId, {
      type: 'inventory_updated',
      product_id: productId,
      new_quantity: newQuantity,
      timestamp: new Date().toISOString()
    });
  }
}

module.exports = POSWebSocketServer;
```

### Frontend WebSocket Client

```javascript
class POSWebSocketClient {
  constructor(url, token) {
    this.url = `${url}?token=${token}`;
    this.ws = null;
    this.reconnectInterval = 5000;
    this.maxReconnectAttempts = 5;
    this.reconnectAttempts = 0;
    
    this.connect();
  }
  
  connect() {
    this.ws = new WebSocket(this.url);
    
    this.ws.onopen = () => {
      console.log('Connected to POS WebSocket server');
      this.reconnectAttempts = 0;
    };
    
    this.ws.onmessage = (event) => {
      try {
        const message = JSON.parse(event.data);
        this.handleMessage(message);
      } catch (error) {
        console.error('Error parsing WebSocket message:', error);
      }
    };
    
    this.ws.onclose = () => {
      console.log('Disconnected from POS WebSocket server');
      this.attemptReconnect();
    };
    
    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };
  }
  
  attemptReconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      console.log(`Attempting to reconnect (${this.reconnectAttempts}/${this.maxReconnectAttempts})...`);
      
      setTimeout(() => {
        this.connect();
      }, this.reconnectInterval);
    } else {
      console.error('Max reconnection attempts reached');
    }
  }
  
  handleMessage(message) {
    switch (message.type) {
      case 'inventory_updated':
        this.updateInventoryDisplay(message.product_id, message.new_quantity);
        break;
        
      case 'price_updated':
        this.updatePriceDisplay(message.product_id, message.new_price);
        break;
        
      default:
        console.log('Unknown message type:', message.type);
    }
  }
  
  updateInventoryDisplay(productId, newQuantity) {
    const inventoryElement = document.querySelector(`[data-product-id="${productId}"] .inventory-count`);
    if (inventoryElement) {
      inventoryElement.textContent = newQuantity;
      
      // Show low stock warning
      if (newQuantity <= 5) {
        inventoryElement.classList.add('low-stock');
      } else {
        inventoryElement.classList.remove('low-stock');
      }
    }
  }
  
  updatePriceDisplay(productId, newPrice) {
    const priceElements = document.querySelectorAll(`[data-product-id="${productId}"] .price`);
    priceElements.forEach(element => {
      element.textContent = `$${newPrice.toFixed(2)}`;
    });
  }
  
  send(message) {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(message));
    } else {
      console.error('WebSocket is not connected');
    }
  }
}

// Initialize WebSocket client
const wsClient = new POSWebSocketClient('ws://localhost:3000', localStorage.getItem('auth_token'));
```

This comprehensive set of examples provides practical implementations for various aspects of a POS system, from basic CRUD operations to real-time updates and payment processing. These examples can be adapted and extended based on specific business requirements and technology stacks.