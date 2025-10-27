# POS System Implementation Guide

This comprehensive guide walks you through the complete process of implementing a Point of Sale system, from initial planning to deployment and maintenance.

## Table of Contents

1. [Pre-Implementation Planning](#pre-implementation-planning)
2. [System Requirements Analysis](#system-requirements-analysis)
3. [Technology Stack Selection](#technology-stack-selection)
4. [Development Setup](#development-setup)
5. [Database Design and Setup](#database-design-and-setup)
6. [Backend API Development](#backend-api-development)
7. [Frontend Development](#frontend-development)
8. [Payment Integration](#payment-integration)
9. [Hardware Integration](#hardware-integration)
10. [Testing Strategy](#testing-strategy)
11. [Deployment](#deployment)
12. [Training and Go-Live](#training-and-go-live)
13. [Maintenance and Updates](#maintenance-and-updates)

## Pre-Implementation Planning

### Business Requirements Gathering

#### Stakeholder Interviews
1. **Store Managers**
   - Daily operational workflows
   - Peak transaction periods
   - Staff training requirements
   - Reporting needs

2. **Cashiers/Sales Staff**
   - User interface preferences
   - Speed and efficiency requirements
   - Common transaction types
   - Error handling needs

3. **IT Department**
   - Infrastructure constraints
   - Security requirements
   - Integration needs
   - Maintenance capabilities

4. **Finance Department**
   - Accounting system integration
   - Tax reporting requirements
   - Payment processing needs
   - Budget constraints

#### Requirements Documentation Template
```markdown
# POS System Requirements Document

## Functional Requirements

### FR-001: Product Management
- **Description**: System must manage product catalog
- **Priority**: High
- **Acceptance Criteria**:
  - Add/edit/delete products
  - Barcode support
  - Category management
  - Price management
  - Inventory tracking

### FR-002: Transaction Processing
- **Description**: Process sales transactions
- **Priority**: High
- **Acceptance Criteria**:
  - Add items to cart
  - Apply discounts
  - Calculate tax
  - Process payments
  - Print receipts

[Continue for all requirements...]

## Non-Functional Requirements

### NFR-001: Performance
- Response time < 2 seconds for product lookup
- Support 50 concurrent users
- 99.9% uptime requirement

### NFR-002: Security
- PCI DSS compliance
- Data encryption
- User authentication
- Audit trails

[Continue for all NFRs...]
```

### Project Timeline Template

```markdown
# POS Implementation Timeline

## Phase 1: Planning and Design (Weeks 1-4)
- [ ] Requirements gathering and analysis
- [ ] System architecture design
- [ ] Database schema design
- [ ] UI/UX wireframes and mockups
- [ ] Technology stack finalization
- [ ] Risk assessment and mitigation planning

## Phase 2: Development Setup (Week 5)
- [ ] Development environment setup
- [ ] Version control system setup  
- [ ] CI/CD pipeline configuration
- [ ] Database setup and initial schema
- [ ] Basic project structure creation

## Phase 3: Core Development (Weeks 6-12)
- [ ] User authentication system
- [ ] Product management module
- [ ] Inventory management system
- [ ] Transaction processing engine
- [ ] Payment integration
- [ ] Basic reporting functionality

## Phase 4: Advanced Features (Weeks 13-16)
- [ ] Customer management
- [ ] Employee management
- [ ] Advanced reporting and analytics
- [ ] Multi-location support
- [ ] Loyalty program integration

## Phase 5: Integration and Testing (Weeks 17-20)
- [ ] Hardware integration
- [ ] Third-party system integration
- [ ] Unit testing completion
- [ ] Integration testing
- [ ] Performance testing
- [ ] Security testing
- [ ] User acceptance testing

## Phase 6: Deployment and Training (Weeks 21-24)
- [ ] Production environment setup
- [ ] Data migration
- [ ] Staff training program
- [ ] Soft launch with limited users
- [ ] Full deployment
- [ ] Post-deployment support
```

## System Requirements Analysis

### Performance Analysis

#### Transaction Volume Estimation
```python
# Example calculation for sizing requirements

class POSCapacityPlanner:
    def __init__(self, stores, avg_customers_per_day, avg_items_per_transaction):
        self.stores = stores
        self.avg_customers_per_day = avg_customers_per_day
        self.avg_items_per_transaction = avg_items_per_transaction
        
    def calculate_daily_transactions(self):
        return self.stores * self.avg_customers_per_day
    
    def calculate_peak_hour_transactions(self):
        # Assume 20% of daily transactions happen in peak hour
        daily_transactions = self.calculate_daily_transactions()
        return daily_transactions * 0.2
    
    def calculate_database_operations_per_day(self):
        daily_transactions = self.calculate_daily_transactions()
        # Each transaction: 1 insert transaction + N insert transaction_items + N update inventory
        operations_per_transaction = 1 + (2 * self.avg_items_per_transaction)
        return daily_transactions * operations_per_transaction
    
    def estimate_storage_requirements(self):
        # Rough estimation in MB per day
        daily_transactions = self.calculate_daily_transactions()
        # Transaction: ~1KB, Items: ~0.5KB each
        daily_storage_mb = daily_transactions * (1 + (self.avg_items_per_transaction * 0.5)) / 1024
        return {
            'daily_mb': daily_storage_mb,
            'monthly_mb': daily_storage_mb * 30,
            'yearly_gb': (daily_storage_mb * 365) / 1024
        }

# Example usage
planner = POSCapacityPlanner(stores=5, avg_customers_per_day=200, avg_items_per_transaction=3)
print(f"Daily transactions: {planner.calculate_daily_transactions()}")
print(f"Peak hour transactions: {planner.calculate_peak_hour_transactions()}")
print(f"Daily DB operations: {planner.calculate_database_operations_per_day()}")
print(f"Storage needs: {planner.estimate_storage_requirements()}")
```

### Infrastructure Requirements

#### Server Specifications
```yaml
# Production Environment Specifications

application_servers:
  count: 2  # For high availability
  specs:
    cpu: "4 cores, 3.0 GHz"
    memory: "16 GB RAM"
    storage: "500 GB SSD"
    network: "1 Gbps"
    os: "Linux Ubuntu 22.04 LTS"

database_server:
  primary:
    cpu: "8 cores, 3.0 GHz"
    memory: "32 GB RAM"
    storage: "1 TB NVMe SSD"
    network: "1 Gbps"
    os: "Linux Ubuntu 22.04 LTS"
  replica:
    cpu: "4 cores, 3.0 GHz"
    memory: "16 GB RAM"
    storage: "1 TB NVMe SSD"
    network: "1 Gbps"

load_balancer:
  type: "NGINX or AWS ALB"
  ssl_termination: true
  health_checks: true

backup_storage:
  type: "Network Attached Storage (NAS)"
  capacity: "5 TB"
  backup_retention: "90 days"
```

## Technology Stack Selection

### Recommended Technology Stacks

#### Option 1: Node.js + React Stack
```yaml
backend:
  runtime: "Node.js 18.x"
  framework: "Express.js"
  database: "MySQL 8.0 or PostgreSQL 14"
  orm: "Sequelize or TypeORM"
  authentication: "JWT with bcrypt"
  payment: "Stripe SDK"

frontend:
  framework: "React 18.x"
  state_management: "Redux Toolkit"
  ui_library: "Material-UI or Ant Design"
  build_tool: "Vite"
  styling: "Styled Components or Tailwind CSS"

infrastructure:
  containerization: "Docker"
  orchestration: "Docker Compose or Kubernetes"
  web_server: "NGINX"
  process_manager: "PM2"
  monitoring: "Winston + ELK Stack"
```

#### Option 2: Python + Django Stack
```yaml
backend:
  runtime: "Python 3.11"
  framework: "Django 4.2 + Django REST Framework"
  database: "PostgreSQL 14"
  orm: "Django ORM"
  authentication: "Django Authentication + JWT"
  payment: "Stripe Python SDK"

frontend:
  framework: "React or Vue.js"
  api_client: "Axios"
  build_tool: "Webpack or Vite"

infrastructure:
  web_server: "Gunicorn + NGINX"
  containerization: "Docker"
  task_queue: "Celery + Redis"
  monitoring: "Django-debug-toolbar + Sentry"
```

#### Option 3: Java Spring Boot Stack
```yaml
backend:
  runtime: "Java 17"
  framework: "Spring Boot 3.0"
  database: "MySQL 8.0"
  orm: "Spring Data JPA (Hibernate)"
  authentication: "Spring Security + JWT"
  payment: "Stripe Java SDK"

frontend:
  framework: "Angular or React"
  build_tool: "Angular CLI or Create React App"

infrastructure:
  application_server: "Embedded Tomcat"
  containerization: "Docker"
  monitoring: "Spring Boot Actuator + Micrometer"
```

## Development Setup

### Development Environment Setup

#### 1. Version Control Setup
```bash
# Initialize Git repository
git init pos-system
cd pos-system

# Create initial project structure
mkdir -p {backend,frontend,database,docs,scripts,tests}

# Create .gitignore
cat > .gitignore << EOF
# Dependencies
node_modules/
__pycache__/
*.pyc
.env
.env.local

# Build outputs
dist/
build/
*.jar
*.war

# IDE files
.vscode/
.idea/
*.swp
*.swo

# Logs
logs/
*.log

# Database
*.db
*.sqlite
EOF

# Initial commit
git add .
git commit -m "Initial project structure"
```

#### 2. Database Setup (Docker Compose)
```yaml
# docker-compose.yml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: pos_mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: pos_system
      MYSQL_USER: pos_user
      MYSQL_PASSWORD: pos_password
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./database/init:/docker-entrypoint-initdb.d
    command: --default-authentication-plugin=mysql_native_password

  redis:
    image: redis:7-alpine
    container_name: pos_redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  adminer:
    image: adminer
    container_name: pos_adminer
    ports:
      - "8080:8080"
    depends_on:
      - mysql

volumes:
  mysql_data:
  redis_data:
```

#### 3. Backend Setup (Node.js Example)
```bash
# Initialize Node.js project
cd backend
npm init -y

# Install dependencies
npm install express mysql2 bcrypt jsonwebtoken cors helmet morgan
npm install -D nodemon jest supertest

# Create basic server structure
mkdir -p {src,src/routes,src/models,src/middleware,src/controllers,src/services,tests}

# Package.json scripts
npm pkg set scripts.dev="nodemon src/app.js"
npm pkg set scripts.start="node src/app.js"
npm pkg set scripts.test="jest"
```

#### 4. Frontend Setup (React Example)
```bash
# Create React application
cd frontend
npx create-react-app pos-frontend --template typescript
cd pos-frontend

# Install additional dependencies
npm install @reduxjs/toolkit react-redux axios react-router-dom
npm install -D @testing-library/react @testing-library/jest-dom

# Create directory structure
mkdir -p src/{components,pages,store,services,utils,types}
```

## Database Design and Setup

### Complete Database Schema

```sql
-- Create database
CREATE DATABASE pos_system CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE pos_system;

-- Categories table
CREATE TABLE categories (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    parent_id INT,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (parent_id) REFERENCES categories(id)
);

-- Tax categories table
CREATE TABLE tax_categories (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    rate DECIMAL(5,4) NOT NULL,
    description TEXT,
    active BOOLEAN DEFAULT TRUE
);

-- Products table (enhanced)
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    cost DECIMAL(10,2),
    category_id INT,
    tax_category_id INT DEFAULT 1,
    barcode VARCHAR(50) UNIQUE,
    image_url VARCHAR(500),
    weight DECIMAL(8,3),
    dimensions VARCHAR(50),
    track_inventory BOOLEAN DEFAULT TRUE,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(id),
    FOREIGN KEY (tax_category_id) REFERENCES tax_categories(id),
    INDEX idx_sku (sku),
    INDEX idx_barcode (barcode),
    INDEX idx_category (category_id)
);

-- Locations table
CREATE TABLE locations (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    address TEXT,
    city VARCHAR(50),
    state VARCHAR(50),
    zip_code VARCHAR(20),
    country VARCHAR(50),
    phone VARCHAR(20),
    email VARCHAR(100),
    tax_rate DECIMAL(5,4) DEFAULT 0.0800,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Inventory table (enhanced)
CREATE TABLE inventory (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT NOT NULL,
    location_id INT NOT NULL,
    quantity_on_hand INT DEFAULT 0,
    quantity_committed INT DEFAULT 0,
    quantity_available INT GENERATED ALWAYS AS (quantity_on_hand - quantity_committed) STORED,
    reorder_point INT DEFAULT 0,
    reorder_quantity INT DEFAULT 0,
    last_counted_at TIMESTAMP,
    cost_average DECIMAL(10,4),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(id),
    FOREIGN KEY (location_id) REFERENCES locations(id),
    UNIQUE KEY unique_product_location (product_id, location_id),
    INDEX idx_low_stock (quantity_on_hand, reorder_point)
);

-- Employees table
CREATE TABLE employees (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    employee_number VARCHAR(20) UNIQUE NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    phone VARCHAR(20),
    role ENUM('cashier', 'supervisor', 'manager', 'admin') DEFAULT 'cashier',
    pin_code VARCHAR(255), -- encrypted
    active BOOLEAN DEFAULT TRUE,
    hire_date DATE,
    location_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (location_id) REFERENCES locations(id)
);

-- Customers table (enhanced)
CREATE TABLE customers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_number VARCHAR(50) UNIQUE,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255) UNIQUE,
    phone VARCHAR(20),
    address TEXT,
    city VARCHAR(100),
    state VARCHAR(50),
    zip_code VARCHAR(20),
    country VARCHAR(50) DEFAULT 'US',
    birth_date DATE,
    loyalty_points INT DEFAULT 0,
    total_spent DECIMAL(12,2) DEFAULT 0.00,
    visit_count INT DEFAULT 0,
    last_visit TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_email (email),
    INDEX idx_phone (phone),
    INDEX idx_customer_number (customer_number)
);

-- Transactions table (enhanced)
CREATE TABLE transactions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    transaction_number VARCHAR(50) UNIQUE NOT NULL,
    location_id INT NOT NULL,
    customer_id BIGINT,
    employee_id BIGINT NOT NULL,
    subtotal DECIMAL(12,2) NOT NULL,
    tax_amount DECIMAL(12,2) NOT NULL,
    discount_amount DECIMAL(12,2) DEFAULT 0.00,
    tip_amount DECIMAL(12,2) DEFAULT 0.00,
    total_amount DECIMAL(12,2) NOT NULL,
    payment_method ENUM('cash', 'credit_card', 'debit_card', 'mobile_payment', 'gift_card', 'store_credit') NOT NULL,
    payment_reference VARCHAR(100),
    transaction_type ENUM('sale', 'return', 'void', 'exchange') DEFAULT 'sale',
    status ENUM('pending', 'completed', 'cancelled', 'refunded') DEFAULT 'pending',
    notes TEXT,
    receipt_printed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (location_id) REFERENCES locations(id),
    FOREIGN KEY (customer_id) REFERENCES customers(id),
    FOREIGN KEY (employee_id) REFERENCES employees(id),
    INDEX idx_transaction_number (transaction_number),
    INDEX idx_location_date (location_id, created_at),
    INDEX idx_employee (employee_id),
    INDEX idx_customer (customer_id)
);

-- Transaction items table (enhanced)
CREATE TABLE transaction_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    transaction_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0.00,
    discount_percent DECIMAL(5,2) DEFAULT 0.00,
    tax_amount DECIMAL(10,2) NOT NULL,
    line_total DECIMAL(12,2) NOT NULL,
    cost_of_goods DECIMAL(10,2),
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (transaction_id) REFERENCES transactions(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id),
    INDEX idx_transaction (transaction_id),
    INDEX idx_product (product_id)
);

-- Payments table
CREATE TABLE payments (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    transaction_id BIGINT NOT NULL,
    payment_method ENUM('cash', 'credit_card', 'debit_card', 'mobile_payment', 'gift_card', 'store_credit') NOT NULL,
    amount DECIMAL(12,2) NOT NULL,
    tender_amount DECIMAL(12,2),
    change_amount DECIMAL(12,2) DEFAULT 0.00,
    payment_reference VARCHAR(100),
    gateway_transaction_id VARCHAR(100),
    status ENUM('pending', 'authorized', 'captured', 'failed', 'refunded') DEFAULT 'pending',
    processed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (transaction_id) REFERENCES transactions(id),
    INDEX idx_transaction (transaction_id),
    INDEX idx_reference (payment_reference)
);

-- Insert initial data
INSERT INTO tax_categories (name, rate, description) VALUES
('Standard', 0.0800, 'Standard tax rate'),
('Food', 0.0400, 'Reduced rate for food items'),
('Tax-Free', 0.0000, 'Tax-exempt items');

INSERT INTO categories (name, description) VALUES
('Electronics', 'Electronic devices and accessories'),
('Clothing', 'Apparel and fashion items'),
('Food & Beverage', 'Food and drink products'),
('Books', 'Books and reading materials'),
('Home & Garden', 'Home improvement and gardening supplies');

INSERT INTO locations (name, address, city, state, zip_code, tax_rate) VALUES
('Main Store', '123 Main Street', 'Anytown', 'CA', '12345', 0.0875),
('Mall Location', '456 Mall Drive', 'Anytown', 'CA', '12346', 0.0875);

-- Create indexes for performance
CREATE INDEX idx_products_active ON products (active);
CREATE INDEX idx_transactions_date_status ON transactions (created_at, status);
CREATE INDEX idx_inventory_low_stock ON inventory (quantity_on_hand) WHERE quantity_on_hand <= reorder_point;
```

### Database Migration Scripts

```javascript
// migrations/001_initial_schema.js
const { DataTypes } = require('sequelize');

module.exports = {
  up: async (queryInterface, Sequelize) => {
    // Create categories table
    await queryInterface.createTable('categories', {
      id: {
        type: DataTypes.INTEGER,
        primaryKey: true,
        autoIncrement: true
      },
      name: {
        type: DataTypes.STRING(100),
        allowNull: false
      },
      description: DataTypes.TEXT,
      parent_id: {
        type: DataTypes.INTEGER,
        references: {
          model: 'categories',
          key: 'id'
        }
      },
      active: {
        type: DataTypes.BOOLEAN,
        defaultValue: true
      },
      created_at: {
        type: DataTypes.DATE,
        defaultValue: Sequelize.literal('CURRENT_TIMESTAMP')
      },
      updated_at: {
        type: DataTypes.DATE,
        defaultValue: Sequelize.literal('CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP')
      }
    });

    // Add more table creations...
  },

  down: async (queryInterface, Sequelize) => {
    await queryInterface.dropTable('categories');
    // Drop other tables...
  }
};
```

## Backend API Development

### Project Structure
```
backend/
├── src/
│   ├── app.js                 # Main application entry point
│   ├── config/
│   │   ├── database.js        # Database configuration
│   │   ├── auth.js            # Authentication config
│   │   └── payment.js         # Payment gateway config
│   ├── controllers/
│   │   ├── authController.js
│   │   ├── productController.js
│   │   ├── transactionController.js
│   │   └── customerController.js
│   ├── middleware/
│   │   ├── auth.js            # Authentication middleware
│   │   ├── validation.js      # Input validation
│   │   └── errorHandler.js    # Error handling
│   ├── models/
│   │   ├── index.js           # Model definitions
│   │   ├── Product.js
│   │   ├── Transaction.js
│   │   └── Customer.js
│   ├── routes/
│   │   ├── api.js             # Main API router
│   │   ├── auth.js
│   │   ├── products.js
│   │   └── transactions.js
│   ├── services/
│   │   ├── paymentService.js
│   │   ├── inventoryService.js
│   │   └── reportingService.js
│   └── utils/
│       ├── logger.js
│       ├── helpers.js
│       └── constants.js
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── docs/
└── package.json
```

### Core API Implementation

```javascript
// src/app.js
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
const rateLimit = require('express-rate-limit');

const apiRoutes = require('./routes/api');
const errorHandler = require('./middleware/errorHandler');
const logger = require('./utils/logger');

const app = express();

// Security middleware
app.use(helmet());
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true
}));

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 1000 // limit each IP to 1000 requests per windowMs
});
app.use(limiter);

// Request parsing
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// Logging
app.use(morgan('combined', { stream: { write: msg => logger.info(msg.trim()) } }));

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({ status: 'OK', timestamp: new Date().toISOString() });
});

// API routes
app.use('/api/v1', apiRoutes);

// Error handling
app.use(errorHandler);

// 404 handler
app.use('*', (req, res) => {
  res.status(404).json({ error: 'Endpoint not found' });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  logger.info(`POS API server running on port ${PORT}`);
});

module.exports = app;
```

```javascript
// src/controllers/transactionController.js
const { Transaction, TransactionItem, Product, Customer } = require('../models');
const { sequelize } = require('../config/database');
const paymentService = require('../services/paymentService');
const inventoryService = require('../services/inventoryService');
const logger = require('../utils/logger');

class TransactionController {
  
  async createTransaction(req, res) {
    const transaction = await sequelize.transaction();
    
    try {
      const {
        customer_id,
        location_id,
        items,
        payment_method,
        payment_data,
        discount_amount = 0,
        tip_amount = 0
      } = req.body;
      
      const employee_id = req.user.id;
      
      // Validate inventory availability
      for (const item of items) {
        const available = await inventoryService.checkAvailability(
          item.product_id, 
          location_id, 
          item.quantity
        );
        
        if (!available) {
          throw new Error(`Insufficient inventory for product ${item.product_id}`);
        }
      }
      
      // Calculate totals
      let subtotal = 0;
      const taxRate = 0.08; // Should be fetched from location settings
      
      for (const item of items) {
        const product = await Product.findByPk(item.product_id);
        if (!product) {
          throw new Error(`Product ${item.product_id} not found`);
        }
        
        subtotal += item.quantity * product.price;
      }
      
      const taxAmount = (subtotal - discount_amount) * taxRate;
      const totalAmount = subtotal - discount_amount + taxAmount + tip_amount;
      
      // Process payment
      const paymentResult = await paymentService.processPayment({
        amount: totalAmount,
        payment_method,
        payment_data
      });
      
      if (!paymentResult.success) {
        throw new Error(`Payment failed: ${paymentResult.error}`);
      }
      
      // Generate transaction number
      const transactionNumber = await this.generateTransactionNumber();
      
      // Create transaction record
      const newTransaction = await Transaction.create({
        transaction_number: transactionNumber,
        location_id,
        customer_id,
        employee_id,
        subtotal,
        tax_amount: taxAmount,
        discount_amount,
        tip_amount,
        total_amount: totalAmount,
        payment_method,
        payment_reference: paymentResult.payment_id,
        status: 'completed'
      }, { transaction });
      
      // Create transaction items and update inventory
      for (const item of items) {
        const product = await Product.findByPk(item.product_id);
        const lineTotal = item.quantity * product.price;
        const lineTax = lineTotal * taxRate;
        
        await TransactionItem.create({
          transaction_id: newTransaction.id,
          product_id: item.product_id,
          quantity: item.quantity,
          unit_price: product.price,
          tax_amount: lineTax,
          line_total: lineTotal + lineTax,
          cost_of_goods: product.cost * item.quantity
        }, { transaction });
        
        // Update inventory
        await inventoryService.updateInventory(
          item.product_id,
          location_id,
          -item.quantity,
          transaction
        );
      }
      
      // Update customer statistics
      if (customer_id) {
        await Customer.increment(
          {
            total_spent: totalAmount,
            visit_count: 1
          },
          {
            where: { id: customer_id },
            transaction
          }
        );
        
        await Customer.update(
          { last_visit: new Date() },
          {
            where: { id: customer_id },
            transaction
          }
        );
      }
      
      await transaction.commit();
      
      // Load complete transaction with items
      const completeTransaction = await Transaction.findByPk(newTransaction.id, {
        include: [
          {
            model: TransactionItem,
            include: [Product]
          },
          Customer
        ]
      });
      
      logger.info(`Transaction ${transactionNumber} completed successfully`);
      
      res.status(201).json({
        status: 'success',
        data: {
          transaction: completeTransaction
        }
      });
      
    } catch (error) {
      await transaction.rollback();
      logger.error('Transaction creation failed:', error);
      
      res.status(400).json({
        status: 'error',
        error: error.message
      });
    }
  }
  
  async generateTransactionNumber() {
    const date = new Date();
    const year = date.getFullYear();
    const month = String(date.getMonth() + 1).padStart(2, '0');
    const day = String(date.getDate()).padStart(2, '0');
    
    // Get daily sequence number
    const today = `${year}-${month}-${day}`;
    const count = await Transaction.count({
      where: sequelize.literal(`DATE(created_at) = '${today}'`)
    });
    
    const sequence = String(count + 1).padStart(4, '0');
    return `TXN${year}${month}${day}${sequence}`;
  }
  
  async getTransaction(req, res) {
    try {
      const { id } = req.params;
      
      const transaction = await Transaction.findByPk(id, {
        include: [
          {
            model: TransactionItem,
            include: [Product]
          },
          Customer
        ]
      });
      
      if (!transaction) {
        return res.status(404).json({
          status: 'error',
          error: 'Transaction not found'
        });
      }
      
      res.json({
        status: 'success',
        data: { transaction }
      });
      
    } catch (error) {
      logger.error('Error fetching transaction:', error);
      res.status(500).json({
        status: 'error',
        error: 'Internal server error'
      });
    }
  }
  
  async processReturn(req, res) {
    // Implementation for processing returns
    // Similar structure to createTransaction but with negative quantities
  }
  
  async voidTransaction(req, res) {
    // Implementation for voiding transactions
  }
}

module.exports = new TransactionController();
```

## Testing Strategy

### Unit Tests Example

```javascript
// tests/unit/transactionController.test.js
const request = require('supertest');
const app = require('../../src/app');
const { Transaction, Product, Customer } = require('../../src/models');
const paymentService = require('../../src/services/paymentService');

jest.mock('../../src/services/paymentService');
jest.mock('../../src/models');

describe('Transaction Controller', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });
  
  describe('POST /api/v1/transactions', () => {
    it('should create a transaction successfully', async () => {
      // Mock data
      const mockProduct = {
        id: 1,
        name: 'Test Product',
        price: 29.99,
        cost: 15.00
      };
      
      const mockTransaction = {
        id: 1,
        transaction_number: 'TXN20240115001',
        total_amount: 32.39
      };
      
      // Mock service calls
      Product.findByPk.mockResolvedValue(mockProduct);
      paymentService.processPayment.mockResolvedValue({
        success: true,
        payment_id: 'pay_123456789'
      });
      Transaction.create.mockResolvedValue(mockTransaction);
      
      const transactionData = {
        location_id: 1,
        items: [{
          product_id: 1,
          quantity: 1
        }],
        payment_method: 'credit_card',
        payment_data: {
          card_number: '4111111111111111',
          exp_month: '12',
          exp_year: '2025',
          cvv: '123'
        }
      };
      
      const response = await request(app)
        .post('/api/v1/transactions')
        .set('Authorization', 'Bearer valid_jwt_token')
        .send(transactionData);
      
      expect(response.status).toBe(201);
      expect(response.body.status).toBe('success');
      expect(response.body.data.transaction).toBeDefined();
    });
    
    it('should handle insufficient inventory', async () => {
      // Test insufficient inventory scenario
    });
    
    it('should handle payment failures', async () => {
      // Test payment failure scenario
    });
  });
});
```

### Integration Tests

```javascript
// tests/integration/transaction.test.js
const request = require('supertest');
const app = require('../../src/app');
const { sequelize } = require('../../src/config/database');

describe('Transaction Integration Tests', () => {
  beforeAll(async () => {
    // Setup test database
    await sequelize.sync({ force: true });
    
    // Seed test data
    await seedTestData();
  });
  
  afterAll(async () => {
    await sequelize.close();
  });
  
  describe('Complete Transaction Flow', () => {
    it('should process a complete transaction end-to-end', async () => {
      // Login and get auth token
      const loginResponse = await request(app)
        .post('/api/v1/auth/login')
        .send({
          email: 'test@example.com',
          password: 'testpassword'
        });
      
      const token = loginResponse.body.data.token;
      
      // Create transaction
      const transactionResponse = await request(app)
        .post('/api/v1/transactions')
        .set('Authorization', `Bearer ${token}`)
        .send({
          location_id: 1,
          items: [{
            product_id: 1,
            quantity: 2
          }],
          payment_method: 'cash'
        });
      
      expect(transactionResponse.status).toBe(201);
      
      // Verify inventory was updated
      const inventoryResponse = await request(app)
        .get('/api/v1/inventory/1/location/1')
        .set('Authorization', `Bearer ${token}`);
      
      expect(inventoryResponse.body.data.quantity_on_hand).toBe(8); // Started with 10, sold 2
    });
  });
});
```

This implementation guide provides a comprehensive roadmap for building a POS system from start to finish. Each section includes practical examples, code snippets, and best practices to ensure a successful implementation.