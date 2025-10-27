# POS System Compliance and Security Guide

This comprehensive guide covers security requirements, compliance standards, and best practices for Point of Sale systems.

## Table of Contents

1. [Payment Card Industry (PCI) Compliance](#payment-card-industry-pci-compliance)
2. [Data Security Standards](#data-security-standards)
3. [Authentication and Access Control](#authentication-and-access-control)
4. [Encryption Requirements](#encryption-requirements)
5. [Network Security](#network-security)
6. [Privacy Regulations](#privacy-regulations)
7. [Audit and Logging](#audit-and-logging)
8. [Incident Response](#incident-response)
9. [Security Assessment and Testing](#security-assessment-and-testing)
10. [Compliance Checklists](#compliance-checklists)

## Payment Card Industry (PCI) Compliance

### PCI DSS Requirements Overview

The Payment Card Industry Data Security Standard (PCI DSS) is mandatory for any organization that processes, stores, or transmits credit card information.

#### The 12 PCI DSS Requirements

1. **Install and maintain a firewall configuration to protect cardholder data**
2. **Do not use vendor-supplied defaults for system passwords and other security parameters**
3. **Protect stored cardholder data**
4. **Encrypt transmission of cardholder data across open, public networks**
5. **Protect all systems against malware and regularly update anti-virus software**
6. **Develop and maintain secure systems and applications**
7. **Restrict access to cardholder data by business need to know**
8. **Identify and authenticate access to system components**
9. **Restrict physical access to cardholder data**
10. **Track and monitor all access to network resources and cardholder data**
11. **Regularly test security systems and processes**
12. **Maintain a policy that addresses information security for all personnel**

### PCI DSS Implementation Guide

#### Requirement 1: Firewall Configuration

**Network Segmentation Strategy:**
```bash
# Example firewall rules for POS network segmentation
iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 3306 -j ACCEPT  # Allow POS terminals to DB
iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 443 -j ACCEPT   # HTTPS access
iptables -A INPUT -p tcp --dport 22 -s 192.168.100.0/24 -j ACCEPT  # SSH from admin network only
iptables -A INPUT -j DROP  # Default deny

# Network diagram
# Internet → Firewall → DMZ (Web Server) → Internal Firewall → POS Network → Database
```

**PCI Network Architecture:**
```yaml
network_zones:
  dmz:
    description: "Web servers and public-facing services"
    allowed_protocols: ["HTTP", "HTTPS"]
    access_from: ["Internet"]
    
  pos_network:
    description: "POS terminals and processing systems"
    allowed_protocols: ["HTTPS", "Database connections"]
    access_from: ["DMZ", "Admin network"]
    
  cardholder_data_environment:
    description: "Systems that store, process, or transmit CHD"
    allowed_protocols: ["Encrypted database connections"]
    access_from: ["POS network only"]
    
  admin_network:
    description: "System administration and monitoring"
    allowed_protocols: ["SSH", "SNMP", "RDP"]
    access_from: ["Authorized admin IPs only"]
```

#### Requirement 3: Protect Stored Cardholder Data

**Data Classification:**
```python
# data_classification.py
class CardholderData:
    """
    PCI DSS Data Classification
    """
    
    # Primary Account Number (PAN) - Must be protected
    PAN_REGEX = r'\b(?:\d{4}[-\s]?){3}\d{4}\b'
    
    # Sensitive Authentication Data - Must NOT be stored
    PROHIBITED_DATA = [
        'full_magnetic_stripe',
        'cav2_cvc2_cid',  # Card verification codes
        'pin_pin_block'   # PIN data
    ]
    
    # Data that can be stored (if business justified)
    ALLOWED_STORED_DATA = [
        'cardholder_name',
        'service_code',
        'expiration_date'
    ]
    
    @staticmethod
    def mask_pan(pan):
        """Mask PAN showing only first 6 and last 4 digits"""
        if len(pan) < 13:
            return pan
        return pan[:6] + '*' * (len(pan) - 10) + pan[-4:]
    
    @staticmethod
    def validate_storage_compliance(data_fields):
        """Validate that only allowed data is being stored"""
        violations = []
        
        for field in data_fields:
            if field in CardholderData.PROHIBITED_DATA:
                violations.append(f"VIOLATION: {field} must not be stored post-authorization")
        
        return violations

# Example usage
data_to_store = ['cardholder_name', 'expiration_date', 'cav2_cvc2_cid']
violations = CardholderData.validate_storage_compliance(data_to_store)
if violations:
    for violation in violations:
        print(violation)
```

**Data Encryption Implementation:**
```python
# encryption.py - PCI DSS compliant encryption
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
import base64
import os

class PCICompliantEncryption:
    """
    PCI DSS compliant encryption for cardholder data
    """
    
    def __init__(self, master_key=None):
        if master_key:
            self.key = master_key
        else:
            self.key = self.generate_key()
        self.cipher = Fernet(self.key)
    
    @staticmethod
    def generate_key():
        """Generate a new encryption key"""
        return Fernet.generate_key()
    
    @staticmethod
    def derive_key_from_password(password: str, salt: bytes = None):
        """Derive encryption key from password (for key derivation)"""
        if salt is None:
            salt = os.urandom(16)
        
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=100000,
        )
        key = base64.urlsafe_b64encode(kdf.derive(password.encode()))
        return key, salt
    
    def encrypt_cardholder_data(self, data: str) -> str:
        """Encrypt cardholder data"""
        if not data:
            return None
        
        encrypted_data = self.cipher.encrypt(data.encode())
        return base64.urlsafe_b64encode(encrypted_data).decode()
    
    def decrypt_cardholder_data(self, encrypted_data: str) -> str:
        """Decrypt cardholder data"""
        if not encrypted_data:
            return None
        
        try:
            decoded_data = base64.urlsafe_b64decode(encrypted_data.encode())
            decrypted_data = self.cipher.decrypt(decoded_data)
            return decrypted_data.decode()
        except Exception as e:
            raise ValueError(f"Decryption failed: {e}")
    
    def secure_delete(self, data: str):
        """Securely overwrite sensitive data in memory"""
        if data:
            # Overwrite the string data (Python limitation - strings are immutable)
            # In production, use specialized secure memory libraries
            pass

# Example usage
encryptor = PCICompliantEncryption()

# Encrypt PAN for storage (if business justified)
pan = "4111111111111111"
encrypted_pan = encryptor.encrypt_cardholder_data(pan)
print(f"Encrypted PAN: {encrypted_pan}")

# For display purposes, use masking instead
masked_pan = CardholderData.mask_pan(pan)
print(f"Masked PAN: {masked_pan}")
```

#### Requirement 4: Encrypt Transmission

**TLS Configuration:**
```nginx
# nginx.conf - PCI DSS compliant TLS configuration
server {
    listen 443 ssl http2;
    server_name pos.yourcompany.com;
    
    # SSL/TLS Configuration for PCI DSS compliance
    ssl_certificate /path/to/certificate.crt;
    ssl_certificate_key /path/to/private.key;
    
    # Use only strong protocols
    ssl_protocols TLSv1.2 TLSv1.3;
    
    # Strong cipher suites only
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256;
    ssl_prefer_server_ciphers on;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    # Disable server tokens
    server_tokens off;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name pos.yourcompany.com;
    return 301 https://$server_name$request_uri;
}
```

### PCI DSS Self-Assessment Questionnaire (SAQ)

```python
# pci_saq_validator.py
class PCISAQValidator:
    """
    PCI DSS Self-Assessment Questionnaire validator
    """
    
    def __init__(self):
        self.requirements = {
            "1.1": "Firewall configuration standards established",
            "1.2": "Firewall configurations restrict connections",
            "2.1": "Vendor defaults changed before installation",
            "2.2": "System components configured securely",
            "3.1": "CHD storage minimized",
            "3.2": "Sensitive authentication data not stored",
            "3.3": "PAN masked when displayed",
            "3.4": "PAN unreadable anywhere stored",
            "4.1": "Strong cryptography for CHD transmission",
            "4.2": "Unencrypted PANs not sent via end-user messaging",
            "6.1": "Security vulnerabilities identified and patched",
            "6.2": "All system components protected from malware",
            # ... continue for all requirements
        }
    
    def assess_compliance(self, responses):
        """
        Assess PCI DSS compliance based on responses
        responses: dict with requirement ID as key and boolean compliance as value
        """
        compliant_count = 0
        non_compliant = []
        
        for req_id, description in self.requirements.items():
            if responses.get(req_id, False):
                compliant_count += 1
            else:
                non_compliant.append((req_id, description))
        
        compliance_percentage = (compliant_count / len(self.requirements)) * 100
        
        return {
            'compliance_percentage': compliance_percentage,
            'compliant_requirements': compliant_count,
            'total_requirements': len(self.requirements),
            'non_compliant_items': non_compliant,
            'is_compliant': compliance_percentage == 100
        }
    
    def generate_compliance_report(self, assessment_results):
        """Generate compliance report"""
        report = f"""
PCI DSS COMPLIANCE ASSESSMENT REPORT
=====================================

Overall Compliance: {assessment_results['compliance_percentage']:.1f}%
Compliant Requirements: {assessment_results['compliant_requirements']}/{assessment_results['total_requirements']}

Status: {'COMPLIANT' if assessment_results['is_compliant'] else 'NON-COMPLIANT'}

"""
        if assessment_results['non_compliant_items']:
            report += "NON-COMPLIANT REQUIREMENTS:\n"
            report += "=" * 30 + "\n"
            for req_id, description in assessment_results['non_compliant_items']:
                report += f"{req_id}: {description}\n"
        
        return report

# Example usage
validator = PCISAQValidator()
sample_responses = {
    "1.1": True,
    "1.2": True,
    "2.1": False,  # Non-compliant
    "2.2": True,
    # ... more responses
}

results = validator.assess_compliance(sample_responses)
print(validator.generate_compliance_report(results))
```

## Data Security Standards

### Data Loss Prevention (DLP)

```python
# data_loss_prevention.py
import re
import hashlib
import logging
from typing import List, Dict, Any

class DataLossPrevention:
    """
    Data Loss Prevention system for POS environments
    """
    
    def __init__(self):
        self.patterns = {
            'credit_card': re.compile(r'\b(?:\d{4}[-\s]?){3}\d{4}\b'),
            'ssn': re.compile(r'\b\d{3}-\d{2}-\d{4}\b'),
            'phone': re.compile(r'\b\d{3}-\d{3}-\d{4}\b'),
            'email': re.compile(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b')
        }
        
        self.violation_threshold = {
            'credit_card': 0,  # Zero tolerance for credit cards
            'ssn': 0,         # Zero tolerance for SSN
            'phone': 10,      # Allow up to 10 phone numbers
            'email': 50       # Allow up to 50 email addresses
        }
        
        self.logger = logging.getLogger('dlp')
    
    def scan_content(self, content: str, context: str = "unknown") -> Dict[str, Any]:
        """
        Scan content for sensitive data patterns
        """
        findings = {}
        violations = []
        
        for pattern_name, pattern in self.patterns.items():
            matches = pattern.findall(content)
            if matches:
                findings[pattern_name] = len(matches)
                
                # Check against thresholds
                if len(matches) > self.violation_threshold[pattern_name]:
                    violation = {
                        'type': pattern_name,
                        'count': len(matches),
                        'context': context,
                        'severity': 'HIGH' if pattern_name in ['credit_card', 'ssn'] else 'MEDIUM'
                    }
                    violations.append(violation)
                    
                    # Log violation
                    self.logger.warning(f"DLP Violation: {pattern_name} detected in {context}")
        
        return {
            'findings': findings,
            'violations': violations,
            'clean': len(violations) == 0
        }
    
    def sanitize_content(self, content: str) -> str:
        """
        Remove or mask sensitive data from content
        """
        sanitized = content
        
        # Mask credit card numbers
        sanitized = self.patterns['credit_card'].sub(
            lambda m: self._mask_credit_card(m.group()), sanitized
        )
        
        # Mask SSNs
        sanitized = self.patterns['ssn'].sub('XXX-XX-XXXX', sanitized)
        
        return sanitized
    
    def _mask_credit_card(self, card_number: str) -> str:
        """Mask credit card number showing only last 4 digits"""
        clean_number = re.sub(r'[-\s]', '', card_number)
        if len(clean_number) >= 4:
            return '*' * (len(clean_number) - 4) + clean_number[-4:]
        return '*' * len(clean_number)

# Example usage
dlp = DataLossPrevention()

# Scan log content
log_content = """
Transaction processed for card 4111-1111-1111-1111
Customer email: john.doe@example.com
SSN: 123-45-6789 provided for verification
"""

scan_results = dlp.scan_content(log_content, "application_log")
if not scan_results['clean']:
    print("DLP violations detected!")
    for violation in scan_results['violations']:
        print(f"- {violation['type']}: {violation['count']} instances (Severity: {violation['severity']})")

# Sanitize content
sanitized_content = dlp.sanitize_content(log_content)
print("Sanitized content:")
print(sanitized_content)
```

### Secure Data Storage

```python
# secure_storage.py
import sqlite3
import bcrypt
import base64
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
import os

class SecureDataStorage:
    """
    Secure storage implementation for POS systems
    """
    
    def __init__(self, db_path: str, master_password: str):
        self.db_path = db_path
        self.key = self._derive_key(master_password)
        self.cipher = Fernet(self.key)
        self._init_database()
    
    def _derive_key(self, password: str) -> bytes:
        """Derive encryption key from master password"""
        # Use a fixed salt for simplicity (in production, use per-record salts)
        salt = b'stable_salt_for_demo'  # In production: use random salts
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=100000,
        )
        return base64.urlsafe_b64encode(kdf.derive(password.encode()))
    
    def _init_database(self):
        """Initialize secure database schema"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        # Create tables with encrypted sensitive data
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS secure_customer_data (
                id INTEGER PRIMARY KEY,
                customer_id TEXT NOT NULL,
                encrypted_name BLOB,
                encrypted_email BLOB,
                encrypted_phone BLOB,
                hash_verification TEXT,  -- Hash for integrity verification
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS audit_trail (
                id INTEGER PRIMARY KEY,
                action TEXT NOT NULL,
                table_name TEXT NOT NULL,
                record_id TEXT NOT NULL,
                user_id TEXT NOT NULL,
                timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                ip_address TEXT
            )
        ''')
        
        conn.commit()
        conn.close()
    
    def store_customer_data(self, customer_id: str, name: str, email: str, phone: str, user_id: str):
        """Store customer data with encryption"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        try:
            # Encrypt sensitive data
            encrypted_name = self.cipher.encrypt(name.encode())
            encrypted_email = self.cipher.encrypt(email.encode())
            encrypted_phone = self.cipher.encrypt(phone.encode())
            
            # Create hash for integrity verification
            data_hash = hashlib.sha256(
                f"{customer_id}{name}{email}{phone}".encode()
            ).hexdigest()
            
            # Store encrypted data
            cursor.execute('''
                INSERT INTO secure_customer_data 
                (customer_id, encrypted_name, encrypted_email, encrypted_phone, hash_verification)
                VALUES (?, ?, ?, ?, ?)
            ''', (customer_id, encrypted_name, encrypted_email, encrypted_phone, data_hash))
            
            # Log the action
            self._log_action('INSERT', 'secure_customer_data', customer_id, user_id, cursor)
            
            conn.commit()
            return True
            
        except Exception as e:
            conn.rollback()
            print(f"Error storing customer data: {e}")
            return False
        finally:
            conn.close()
    
    def retrieve_customer_data(self, customer_id: str, user_id: str) -> dict:
        """Retrieve and decrypt customer data"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        try:
            cursor.execute('''
                SELECT encrypted_name, encrypted_email, encrypted_phone, hash_verification
                FROM secure_customer_data
                WHERE customer_id = ?
            ''', (customer_id,))
            
            row = cursor.fetchone()
            if not row:
                return None
            
            # Decrypt data
            name = self.cipher.decrypt(row[0]).decode()
            email = self.cipher.decrypt(row[1]).decode()
            phone = self.cipher.decrypt(row[2]).decode()
            stored_hash = row[3]
            
            # Verify data integrity
            calculated_hash = hashlib.sha256(
                f"{customer_id}{name}{email}{phone}".encode()
            ).hexdigest()
            
            if calculated_hash != stored_hash:
                raise ValueError("Data integrity check failed")
            
            # Log the access
            self._log_action('SELECT', 'secure_customer_data', customer_id, user_id, cursor)
            conn.commit()
            
            return {
                'customer_id': customer_id,
                'name': name,
                'email': email,
                'phone': phone
            }
            
        except Exception as e:
            print(f"Error retrieving customer data: {e}")
            return None
        finally:
            conn.close()
    
    def _log_action(self, action: str, table_name: str, record_id: str, user_id: str, cursor, ip_address: str = None):
        """Log database actions for audit trail"""
        cursor.execute('''
            INSERT INTO audit_trail (action, table_name, record_id, user_id, ip_address)
            VALUES (?, ?, ?, ?, ?)
        ''', (action, table_name, record_id, user_id, ip_address))
    
    def get_audit_trail(self, days: int = 30) -> list:
        """Retrieve audit trail for specified number of days"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT action, table_name, record_id, user_id, timestamp, ip_address
            FROM audit_trail
            WHERE timestamp >= datetime('now', '-{} days')
            ORDER BY timestamp DESC
        '''.format(days))
        
        results = cursor.fetchall()
        conn.close()
        
        return [
            {
                'action': row[0],
                'table_name': row[1],
                'record_id': row[2],
                'user_id': row[3],
                'timestamp': row[4],
                'ip_address': row[5]
            }
            for row in results
        ]

# Example usage
storage = SecureDataStorage('secure_pos.db', 'strong_master_password_123!')

# Store customer data
storage.store_customer_data(
    customer_id='CUST001',
    name='John Doe',
    email='john.doe@example.com',
    phone='555-123-4567',
    user_id='admin'
)

# Retrieve customer data
customer_data = storage.retrieve_customer_data('CUST001', 'cashier1')
if customer_data:
    print(f"Retrieved: {customer_data['name']} - {customer_data['email']}")

# Check audit trail
audit_records = storage.get_audit_trail(7)  # Last 7 days
for record in audit_records[:5]:  # Show first 5 records
    print(f"{record['timestamp']}: {record['action']} on {record['table_name']} by {record['user_id']}")
```

## Authentication and Access Control

### Multi-Factor Authentication (MFA)

```python
# mfa_system.py
import pyotp
import qrcode
import io
import base64
import hashlib
import time
from typing import Optional, Dict, Any

class MFASystem:
    """
    Multi-Factor Authentication system for POS
    """
    
    def __init__(self):
        self.issuer_name = "SecurePOS System"
        self.user_secrets = {}  # In production, store in secure database
        
    def setup_mfa_for_user(self, user_id: str, username: str) -> Dict[str, Any]:
        """
        Set up MFA for a user
        Returns QR code and backup codes
        """
        # Generate a secret key for the user
        secret = pyotp.random_base32()
        
        # Create TOTP instance
        totp = pyotp.TOTP(secret)
        
        # Generate provisioning URI for QR code
        provisioning_uri = totp.provisioning_uri(
            name=username,
            issuer_name=self.issuer_name
        )
        
        # Generate QR code
        qr = qrcode.QRCode(version=1, box_size=10, border=5)
        qr.add_data(provisioning_uri)
        qr.make(fit=True)
        
        qr_image = qr.make_image(fill_color="black", back_color="white")
        
        # Convert QR code to base64 for web display
        buffer = io.BytesIO()
        qr_image.save(buffer, format='PNG')
        qr_code_base64 = base64.b64encode(buffer.getvalue()).decode()
        
        # Generate backup codes
        backup_codes = self._generate_backup_codes(user_id)
        
        # Store user's MFA configuration
        self.user_secrets[user_id] = {
            'secret': secret,
            'backup_codes': backup_codes,
            'setup_completed': False
        }
        
        return {
            'secret': secret,
            'qr_code': qr_code_base64,
            'backup_codes': backup_codes,
            'provisioning_uri': provisioning_uri
        }
    
    def verify_mfa_setup(self, user_id: str, token: str) -> bool:
        """
        Verify MFA setup with initial token
        """
        if user_id not in self.user_secrets:
            return False
        
        secret = self.user_secrets[user_id]['secret']
        totp = pyotp.TOTP(secret)
        
        if totp.verify(token):
            self.user_secrets[user_id]['setup_completed'] = True
            return True
        
        return False
    
    def verify_mfa_token(self, user_id: str, token: str) -> bool:
        """
        Verify MFA token for authentication
        """
        if user_id not in self.user_secrets:
            return False
        
        user_config = self.user_secrets[user_id]
        
        if not user_config['setup_completed']:
            return False
        
        # Check TOTP token
        secret = user_config['secret']
        totp = pyotp.TOTP(secret)
        
        if totp.verify(token):
            return True
        
        # Check backup codes
        if token in user_config['backup_codes']:
            # Remove used backup code
            user_config['backup_codes'].remove(token)
            return True
        
        return False
    
    def _generate_backup_codes(self, user_id: str, count: int = 10) -> list:
        """
        Generate backup codes for MFA
        """
        backup_codes = []
        for i in range(count):
            # Generate a unique code based on user_id, timestamp, and counter
            code_data = f"{user_id}{time.time()}{i}".encode()
            code_hash = hashlib.sha256(code_data).hexdigest()[:8].upper()
            backup_codes.append(code_hash)
        
        return backup_codes
    
    def disable_mfa(self, user_id: str) -> bool:
        """
        Disable MFA for a user
        """
        if user_id in self.user_secrets:
            del self.user_secrets[user_id]
            return True
        return False
    
    def get_remaining_backup_codes(self, user_id: str) -> int:
        """
        Get count of remaining backup codes
        """
        if user_id in self.user_secrets:
            return len(self.user_secrets[user_id]['backup_codes'])
        return 0

# Example usage
mfa = MFASystem()

# Setup MFA for a user
user_id = "user123"
username = "john.doe@company.com"
mfa_setup = mfa.setup_mfa_for_user(user_id, username)

print(f"Secret: {mfa_setup['secret']}")
print(f"Backup codes: {mfa_setup['backup_codes']}")

# Simulate user completing setup
setup_token = input("Enter token from authenticator app to complete setup: ")
if mfa.verify_mfa_setup(user_id, setup_token):
    print("MFA setup completed successfully!")
    
    # Verify subsequent login
    login_token = input("Enter token for login verification: ")
    if mfa.verify_mfa_token(user_id, login_token):
        print("Authentication successful!")
    else:
        print("Authentication failed!")
else:
    print("MFA setup failed!")
```

### Role-Based Access Control (RBAC)

```python
# rbac_system.py
from enum import Enum
from typing import Set, Dict, List, Optional
import json
from datetime import datetime, timedelta

class Permission(Enum):
    # Product Management
    VIEW_PRODUCTS = "view_products"
    ADD_PRODUCTS = "add_products"
    EDIT_PRODUCTS = "edit_products"
    DELETE_PRODUCTS = "delete_products"
    
    # Transaction Management
    PROCESS_SALES = "process_sales"
    PROCESS_RETURNS = "process_returns"
    VOID_TRANSACTIONS = "void_transactions"
    VIEW_TRANSACTIONS = "view_transactions"
    
    # Customer Management
    VIEW_CUSTOMERS = "view_customers"
    ADD_CUSTOMERS = "add_customers"
    EDIT_CUSTOMERS = "edit_customers"
    
    # Inventory Management
    VIEW_INVENTORY = "view_inventory"
    ADJUST_INVENTORY = "adjust_inventory"
    
    # Reporting
    VIEW_BASIC_REPORTS = "view_basic_reports"
    VIEW_DETAILED_REPORTS = "view_detailed_reports"
    VIEW_FINANCIAL_REPORTS = "view_financial_reports"
    
    # System Administration
    MANAGE_USERS = "manage_users"
    MANAGE_SETTINGS = "manage_settings"
    VIEW_SYSTEM_LOGS = "view_system_logs"
    
    # Cash Management
    OPEN_CASH_DRAWER = "open_cash_drawer"
    PERFORM_CASH_COUNT = "perform_cash_count"
    
    # Discounts and Overrides
    APPLY_DISCOUNTS = "apply_discounts"
    OVERRIDE_PRICES = "override_prices"

class Role:
    def __init__(self, name: str, permissions: Set[Permission], description: str = ""):
        self.name = name
        self.permissions = permissions
        self.description = description
        self.created_at = datetime.now()

class User:
    def __init__(self, user_id: str, username: str, email: str, roles: Set[str]):
        self.user_id = user_id
        self.username = username
        self.email = email
        self.roles = roles
        self.active = True
        self.created_at = datetime.now()
        self.last_login = None
        self.failed_login_attempts = 0
        self.locked_until = None

class RBACSystem:
    """
    Role-Based Access Control system for POS
    """
    
    def __init__(self):
        self.roles = self._initialize_default_roles()
        self.users = {}
        self.sessions = {}
        
    def _initialize_default_roles(self) -> Dict[str, Role]:
        """Initialize default roles for POS system"""
        roles = {}
        
        # Cashier role
        cashier_permissions = {
            Permission.VIEW_PRODUCTS,
            Permission.PROCESS_SALES,
            Permission.PROCESS_RETURNS,
            Permission.VIEW_CUSTOMERS,
            Permission.ADD_CUSTOMERS,
            Permission.APPLY_DISCOUNTS,
            Permission.OPEN_CASH_DRAWER
        }
        roles['cashier'] = Role('cashier', cashier_permissions, 'Basic cashier operations')
        
        # Supervisor role
        supervisor_permissions = cashier_permissions | {
            Permission.VOID_TRANSACTIONS,
            Permission.VIEW_TRANSACTIONS,
            Permission.EDIT_CUSTOMERS,
            Permission.VIEW_INVENTORY,
            Permission.VIEW_BASIC_REPORTS,
            Permission.OVERRIDE_PRICES,
            Permission.PERFORM_CASH_COUNT
        }
        roles['supervisor'] = Role('supervisor', supervisor_permissions, 'Supervisory functions')
        
        # Manager role
        manager_permissions = supervisor_permissions | {
            Permission.ADD_PRODUCTS,
            Permission.EDIT_PRODUCTS,
            Permission.ADJUST_INVENTORY,
            Permission.VIEW_DETAILED_REPORTS,
            Permission.VIEW_FINANCIAL_REPORTS,
            Permission.MANAGE_SETTINGS
        }
        roles['manager'] = Role('manager', manager_permissions, 'Store management')
        
        # Admin role
        admin_permissions = manager_permissions | {
            Permission.DELETE_PRODUCTS,
            Permission.MANAGE_USERS,
            Permission.VIEW_SYSTEM_LOGS
        }
        roles['admin'] = Role('admin', admin_permissions, 'System administration')
        
        return roles
    
    def create_user(self, user_id: str, username: str, email: str, roles: List[str]) -> bool:
        """Create a new user with specified roles"""
        # Validate roles exist
        for role_name in roles:
            if role_name not in self.roles:
                raise ValueError(f"Role '{role_name}' does not exist")
        
        user = User(user_id, username, email, set(roles))
        self.users[user_id] = user
        return True
    
    def has_permission(self, user_id: str, permission: Permission) -> bool:
        """Check if user has specific permission"""
        if user_id not in self.users:
            return False
        
        user = self.users[user_id]
        if not user.active:
            return False
        
        # Check if user is locked
        if user.locked_until and datetime.now() < user.locked_until:
            return False
        
        # Aggregate permissions from all user roles
        user_permissions = set()
        for role_name in user.roles:
            if role_name in self.roles:
                user_permissions.update(self.roles[role_name].permissions)
        
        return permission in user_permissions
    
    def get_user_permissions(self, user_id: str) -> Set[Permission]:
        """Get all permissions for a user"""
        if user_id not in self.users:
            return set()
        
        user = self.users[user_id]
        if not user.active:
            return set()
        
        user_permissions = set()
        for role_name in user.roles:
            if role_name in self.roles:
                user_permissions.update(self.roles[role_name].permissions)
        
        return user_permissions
    
    def assign_role(self, user_id: str, role_name: str) -> bool:
        """Assign role to user"""
        if user_id not in self.users or role_name not in self.roles:
            return False
        
        self.users[user_id].roles.add(role_name)
        return True
    
    def revoke_role(self, user_id: str, role_name: str) -> bool:
        """Revoke role from user"""
        if user_id not in self.users:
            return False
        
        self.users[user_id].roles.discard(role_name)
        return True
    
    def create_session(self, user_id: str, session_token: str, ip_address: str = None) -> bool:
        """Create user session"""
        if user_id not in self.users:
            return False
        
        user = self.users[user_id]
        user.last_login = datetime.now()
        user.failed_login_attempts = 0  # Reset on successful login
        
        self.sessions[session_token] = {
            'user_id': user_id,
            'created_at': datetime.now(),
            'ip_address': ip_address,
            'last_activity': datetime.now()
        }
        
        return True
    
    def validate_session(self, session_token: str, max_inactive_minutes: int = 30) -> Optional[str]:
        """Validate session and return user_id if valid"""
        if session_token not in self.sessions:
            return None
        
        session = self.sessions[session_token]
        
        # Check if session is expired
        if datetime.now() - session['last_activity'] > timedelta(minutes=max_inactive_minutes):
            del self.sessions[session_token]
            return None
        
        # Update last activity
        session['last_activity'] = datetime.now()
        return session['user_id']
    
    def lock_user_account(self, user_id: str, duration_minutes: int = 30):
        """Lock user account for specified duration"""
        if user_id in self.users:
            self.users[user_id].locked_until = datetime.now() + timedelta(minutes=duration_minutes)
    
    def record_failed_login(self, user_id: str, max_attempts: int = 5):
        """Record failed login attempt and lock if necessary"""
        if user_id in self.users:
            user = self.users[user_id]
            user.failed_login_attempts += 1
            
            if user.failed_login_attempts >= max_attempts:
                self.lock_user_account(user_id, 30)  # Lock for 30 minutes
    
    def audit_user_access(self, user_id: str, action: str, resource: str, ip_address: str = None):
        """Log user access for audit purposes"""
        audit_entry = {
            'user_id': user_id,
            'action': action,
            'resource': resource,
            'timestamp': datetime.now().isoformat(),
            'ip_address': ip_address
        }
        
        # In production, store in secure audit log
        print(f"AUDIT: {json.dumps(audit_entry)}")

# Example usage and testing
def test_rbac_system():
    rbac = RBACSystem()
    
    # Create users
    rbac.create_user('cashier1', 'alice', 'alice@company.com', ['cashier'])
    rbac.create_user('manager1', 'bob', 'bob@company.com', ['manager'])
    rbac.create_user('admin1', 'charlie', 'charlie@company.com', ['admin'])
    
    # Test permissions
    print("=== Permission Tests ===")
    
    # Cashier tests
    print(f"Cashier can process sales: {rbac.has_permission('cashier1', Permission.PROCESS_SALES)}")
    print(f"Cashier can delete products: {rbac.has_permission('cashier1', Permission.DELETE_PRODUCTS)}")
    
    # Manager tests
    print(f"Manager can view financial reports: {rbac.has_permission('manager1', Permission.VIEW_FINANCIAL_REPORTS)}")
    print(f"Manager can manage users: {rbac.has_permission('manager1', Permission.MANAGE_USERS)}")
    
    # Admin tests
    print(f"Admin can manage users: {rbac.has_permission('admin1', Permission.MANAGE_USERS)}")
    print(f"Admin can delete products: {rbac.has_permission('admin1', Permission.DELETE_PRODUCTS)}")
    
    # Test role assignment
    print("\n=== Role Assignment Test ===")
    rbac.assign_role('cashier1', 'supervisor')
    print(f"Cashier with supervisor role can void transactions: {rbac.has_permission('cashier1', Permission.VOID_TRANSACTIONS)}")
    
    # Test session management
    print("\n=== Session Management Test ===")
    rbac.create_session('cashier1', 'session123', '192.168.1.100')
    user_id = rbac.validate_session('session123')
    print(f"Session validation returned user_id: {user_id}")
    
    # Test audit logging
    rbac.audit_user_access('cashier1', 'process_sale', 'transaction_123', '192.168.1.100')

if __name__ == "__main__":
    test_rbac_system()
```

## Compliance Checklists

### PCI DSS Compliance Checklist

```yaml
# pci_dss_compliance_checklist.yaml
pci_dss_requirements:
  requirement_1:
    title: "Install and maintain a firewall configuration"
    controls:
      - id: "1.1"
        description: "Establish firewall and router configuration standards"
        status: "pending"  # pending, compliant, non-compliant
        evidence: ""
        responsible_party: "IT Security Team"
        
      - id: "1.2"
        description: "Build firewall configurations that restrict connections"
        status: "pending"
        evidence: ""
        responsible_party: "Network Administrator"
        
      - id: "1.3"
        description: "Prohibit direct public access between Internet and system components"
        status: "pending"
        evidence: ""
        responsible_party: "Network Administrator"

  requirement_2:
    title: "Do not use vendor-supplied defaults"
    controls:
      - id: "2.1"
        description: "Change vendor-supplied defaults before installing systems"
        status: "pending"
        evidence: ""
        responsible_party: "System Administrator"
        
      - id: "2.2"
        description: "Develop configuration standards for system components"
        status: "pending"
        evidence: ""
        responsible_party: "IT Security Team"
        
      - id: "2.3"
        description: "Encrypt all non-console administrative access"
        status: "pending"
        evidence: ""
        responsible_party: "System Administrator"

  requirement_3:
    title: "Protect stored cardholder data"
    controls:
      - id: "3.1"
        description: "Keep cardholder data storage to a minimum"
        status: "pending"
        evidence: ""
        responsible_party: "Development Team"
        
      - id: "3.2"
        description: "Do not store sensitive authentication data after authorization"
        status: "pending"
        evidence: ""
        responsible_party: "Development Team"
        
      - id: "3.3"
        description: "Mask PAN when displayed"
        status: "pending"
        evidence: ""
        responsible_party: "Development Team"
        
      - id: "3.4"
        description: "Render PAN unreadable anywhere it is stored"
        status: "pending"
        evidence: ""
        responsible_party: "Development Team"

# Continue for all 12 requirements...
```

### Security Assessment Checklist

```python
# security_assessment.py
class SecurityAssessment:
    """
    Comprehensive security assessment for POS systems
    """
    
    def __init__(self):
        self.assessment_categories = {
            'network_security': {
                'firewall_configuration': False,
                'network_segmentation': False,
                'wireless_security': False,
                'vpn_security': False,
                'intrusion_detection': False
            },
            'access_control': {
                'strong_authentication': False,
                'multi_factor_auth': False,
                'role_based_access': False,
                'session_management': False,
                'account_lockout': False
            },
            'data_protection': {
                'data_encryption': False,
                'secure_key_management': False,
                'data_classification': False,
                'data_loss_prevention': False,
                'secure_backup': False
            },
            'application_security': {
                'secure_coding': False,
                'input_validation': False,
                'sql_injection_protection': False,
                'xss_protection': False,
                'secure_session_handling': False
            },
            'physical_security': {
                'server_room_access': False,
                'workstation_security': False,
                'clean_desk_policy': False,
                'hardware_security': False,
                'environmental_controls': False
            },
            'monitoring_logging': {
                'audit_logging': False,
                'log_monitoring': False,
                'incident_detection': False,
                'security_alerting': False,
                'log_retention': False
            }
        }
    
    def assess_category(self, category: str, controls: dict) -> dict:
        """Assess a security category"""
        if category not in self.assessment_categories:
            raise ValueError(f"Unknown category: {category}")
        
        # Update assessment results
        self.assessment_categories[category].update(controls)
        
        # Calculate compliance percentage
        total_controls = len(self.assessment_categories[category])
        compliant_controls = sum(self.assessment_categories[category].values())
        compliance_percentage = (compliant_controls / total_controls) * 100
        
        return {
            'category': category,
            'total_controls': total_controls,
            'compliant_controls': compliant_controls,
            'compliance_percentage': compliance_percentage,
            'status': 'compliant' if compliance_percentage == 100 else 'needs_improvement'
        }
    
    def generate_security_report(self) -> str:
        """Generate comprehensive security assessment report"""
        report_lines = [
            "SECURITY ASSESSMENT REPORT",
            "=" * 50,
            f"Assessment Date: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}",
            ""
        ]
        
        overall_compliance = 0
        total_categories = len(self.assessment_categories)
        
        for category, controls in self.assessment_categories.items():
            total_controls = len(controls)
            compliant_controls = sum(controls.values())
            compliance_percentage = (compliant_controls / total_controls) * 100
            overall_compliance += compliance_percentage
            
            report_lines.extend([
                f"Category: {category.replace('_', ' ').title()}",
                f"Compliance: {compliance_percentage:.1f}% ({compliant_controls}/{total_controls})",
                f"Status: {'✓ COMPLIANT' if compliance_percentage == 100 else '✗ NEEDS IMPROVEMENT'}",
                ""
            ])
            
            # List non-compliant controls
            non_compliant = [control for control, status in controls.items() if not status]
            if non_compliant:
                report_lines.append("Non-compliant controls:")
                for control in non_compliant:
                    report_lines.append(f"  - {control.replace('_', ' ').title()}")
                report_lines.append("")
        
        overall_compliance = overall_compliance / total_categories
        report_lines.extend([
            "OVERALL ASSESSMENT",
            "=" * 20,
            f"Overall Compliance: {overall_compliance:.1f}%",
            f"Status: {'COMPLIANT' if overall_compliance >= 95 else 'NON-COMPLIANT'}",
            ""
        ])
        
        if overall_compliance < 95:
            report_lines.extend([
                "IMMEDIATE ACTION REQUIRED",
                "=" * 25,
                "The following security gaps must be addressed:",
                ""
            ])
            
            for category, controls in self.assessment_categories.items():
                non_compliant = [control for control, status in controls.items() if not status]
                if non_compliant:
                    report_lines.append(f"{category.replace('_', ' ').title()}:")
                    for control in non_compliant:
                        report_lines.append(f"  - {control.replace('_', ' ').title()}")
                    report_lines.append("")
        
        return '\n'.join(report_lines)

# Example usage
assessment = SecurityAssessment()

# Assess network security
network_results = assessment.assess_category('network_security', {
    'firewall_configuration': True,
    'network_segmentation': True,
    'wireless_security': False,  # Needs improvement
    'vpn_security': True,
    'intrusion_detection': False  # Needs improvement
})

# Assess access control
access_results = assessment.assess_category('access_control', {
    'strong_authentication': True,
    'multi_factor_auth': True,
    'role_based_access': True,
    'session_management': True,
    'account_lockout': True
})

# Generate report
print(assessment.generate_security_report())
```

This compliance and security guide provides a comprehensive framework for ensuring POS systems meet industry standards and security requirements. Regular assessments using these tools and checklists will help maintain compliance and protect sensitive cardholder data.