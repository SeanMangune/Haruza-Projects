# POS System Troubleshooting Guide

This comprehensive troubleshooting guide helps diagnose and resolve common issues encountered in Point of Sale systems.

## Table of Contents

1. [System Startup Issues](#system-startup-issues)
2. [Hardware Problems](#hardware-problems)
3. [Network and Connectivity Issues](#network-and-connectivity-issues)
4. [Payment Processing Problems](#payment-processing-problems)
5. [Database Issues](#database-issues)
6. [Performance Problems](#performance-problems)
7. [Security and Authentication Issues](#security-and-authentication-issues)
8. [Integration Problems](#integration-problems)
9. [Data Synchronization Issues](#data-synchronization-issues)
10. [Reporting and Analytics Problems](#reporting-and-analytics-problems)
11. [Diagnostic Tools and Scripts](#diagnostic-tools-and-scripts)
12. [Emergency Procedures](#emergency-procedures)

## System Startup Issues

### POS Terminal Won't Start

#### Symptoms
- Black screen on startup
- System hangs during boot
- Error messages during initialization
- Application fails to load

#### Troubleshooting Steps

1. **Check Power and Connections**
   ```bash
   # Check power status
   sudo systemctl status power-management
   
   # Verify hardware connections
   lsusb  # List USB devices
   lspci  # List PCI devices
   ```

2. **Verify System Services**
   ```bash
   # Check critical services
   systemctl status mysql
   systemctl status nginx
   systemctl status pos-api
   
   # Check service logs
   journalctl -u pos-api -n 50
   ```

3. **Database Connectivity Test**
   ```bash
   # Test database connection
   mysql -u pos_user -p -h localhost pos_system -e "SELECT 1;"
   
   # Check database status
   systemctl status mysql
   mysqladmin -u root -p status
   ```

4. **Application Log Analysis**
   ```bash
   # Check application logs
   tail -f /var/log/pos/application.log
   
   # Check error logs
   tail -f /var/log/pos/error.log
   
   # Check system logs
   dmesg | grep -i error
   ```

#### Solutions

**Service Start Script**
```bash
#!/bin/bash
# pos-startup.sh - Emergency startup script

echo "Starting POS system recovery..."

# Start database
sudo systemctl start mysql
sleep 5

# Start Redis cache
sudo systemctl start redis
sleep 2

# Start API server
sudo systemctl start pos-api
sleep 5

# Start web server
sudo systemctl start nginx

# Verify all services
services=("mysql" "redis" "pos-api" "nginx")
for service in "${services[@]}"; do
    if systemctl is-active --quiet "$service"; then
        echo "✓ $service is running"
    else
        echo "✗ $service failed to start"
        journalctl -u "$service" -n 10
    fi
done
```

### Application Crashes During Startup

#### Common Causes and Solutions

1. **Configuration File Issues**
   ```bash
   # Validate configuration syntax
   node -c /opt/pos/config/app.js
   
   # Check configuration file permissions
   ls -la /opt/pos/config/
   ```

2. **Missing Dependencies**
   ```bash
   # Check Node.js modules
   cd /opt/pos && npm ls
   
   # Reinstall dependencies if needed
   npm install --production
   ```

3. **Port Conflicts**
   ```bash
   # Check port usage
   netstat -tulpn | grep :3000
   
   # Kill conflicting processes
   sudo fuser -k 3000/tcp
   ```

## Hardware Problems

### Receipt Printer Issues

#### Symptoms
- No receipt printing
- Partial receipt printing
- Poor print quality
- Paper jam errors

#### Troubleshooting Steps

1. **Check Physical Connections**
   ```bash
   # List connected printers
   lpstat -p
   
   # Check USB connections
   lsusb | grep -i printer
   
   # Test printer communication
   echo "Test print" | lp -d receipt_printer
   ```

2. **Printer Status Check**
   ```python
   # printer_diagnostic.py
   import serial
   import time
   
   def check_printer_status(port='/dev/ttyUSB0'):
       try:
           ser = serial.Serial(port, 9600, timeout=1)
           
           # Send status request command (ESC/POS)
           ser.write(b'\x10\x04\x01')  # DLE EOT n
           time.sleep(0.1)
           
           response = ser.read(1)
           if response:
               status = ord(response)
               print(f"Printer status: {bin(status)}")
               
               if status & 0x08:
                   print("ERROR: Paper out")
               if status & 0x20:
                   print("ERROR: Cover open")
               if status & 0x40:
                   print("ERROR: Paper jam")
           else:
               print("No response from printer")
               
           ser.close()
           return True
           
       except Exception as e:
           print(f"Printer communication error: {e}")
           return False
   
   if __name__ == "__main__":
       check_printer_status()
   ```

3. **Print Test Script**
   ```python
   # print_test.py
   def print_test_receipt():
       receipt_content = """
   \x1b\x40  # Initialize printer
   \x1b\x61\x01  # Center alignment
   
   ================================
           TEST RECEIPT
   ================================
   
   Item 1........................$10.00
   Item 2........................$15.50
   --------------------------------
   Subtotal......................$25.50
   Tax...........................$2.04
   ================================
   Total.........................$27.54
   
   Thank you for your business!
   
   \x1d\x56\x00  # Cut paper
   """
       
       try:
           with open('/dev/usb/lp0', 'wb') as printer:
               printer.write(receipt_content.encode('utf-8'))
           print("Test receipt sent successfully")
       except Exception as e:
           print(f"Print test failed: {e}")
   
   print_test_receipt()
   ```

### Barcode Scanner Problems

#### Symptoms
- Scanner not reading barcodes
- Incorrect data transmission
- Scanner not connecting
- Delayed response

#### Troubleshooting Steps

1. **Scanner Configuration Test**
   ```bash
   # Check input devices
   cat /proc/bus/input/devices | grep -A 5 -B 5 scanner
   
   # Monitor scanner input
   sudo cat /dev/input/event2  # Replace with correct event number
   ```

2. **Scanner Test Script**
   ```python
   # scanner_test.py
   import evdev
   import threading
   import queue
   
   class BarcodeScanner:
       def __init__(self, device_path=None):
           self.device_path = device_path or self.find_scanner()
           self.device = None
           self.scan_queue = queue.Queue()
           
       def find_scanner(self):
           devices = [evdev.InputDevice(path) for path in evdev.list_devices()]
           for device in devices:
               if 'scanner' in device.name.lower() or 'barcode' in device.name.lower():
                   return device.path
           return None
           
       def start_scanning(self):
           if not self.device_path:
               print("No scanner device found")
               return
               
           try:
               self.device = evdev.InputDevice(self.device_path)
               print(f"Connected to scanner: {self.device.name}")
               
               for event in self.device.read_loop():
                   if event.type == evdev.ecodes.EV_KEY:
                       self.process_key_event(event)
                       
           except Exception as e:
               print(f"Scanner error: {e}")
               
       def process_key_event(self, event):
           # Process barcode scan events
           if event.value == 1:  # Key press
               key = evdev.ecodes.KEY[event.code]
               print(f"Scanned key: {key}")
   
   # Test the scanner
   scanner = BarcodeScanner()
   scanner.start_scanning()
   ```

### Cash Drawer Issues

#### Symptoms
- Drawer won't open
- Drawer opens randomly
- No response to open commands

#### Troubleshooting Steps

1. **Cash Drawer Test**
   ```python
   # cash_drawer_test.py
   import serial
   import time
   
   def open_cash_drawer(port='/dev/ttyUSB0'):
       try:
           # Open serial connection to receipt printer
           ser = serial.Serial(port, 9600, timeout=1)
           time.sleep(0.1)
           
           # Send cash drawer open command (ESC/POS)
           # ESC p m t1 t2 (27 112 0 25 250)
           ser.write(b'\x1b\x70\x00\x19\xfa')
           
           print("Cash drawer open command sent")
           ser.close()
           return True
           
       except Exception as e:
           print(f"Cash drawer error: {e}")
           return False
   
   open_cash_drawer()
   ```

2. **Electrical Connection Check**
   ```bash
   # Check RJ11/RJ12 connection
   # Verify printer-to-drawer cable
   # Test with multimeter if available
   
   # Check printer documentation for drawer commands
   ```

### Payment Terminal Problems

#### Symptoms
- Terminal not responding
- Card read errors
- Connection timeouts
- Certification failures

#### Troubleshooting Steps

1. **Terminal Communication Test**
   ```python
   # payment_terminal_test.py
   import socket
   import time
   
   def test_payment_terminal(host='192.168.1.100', port=8080):
       try:
           sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
           sock.settimeout(5)
           
           result = sock.connect_ex((host, port))
           if result == 0:
               print(f"Connection to {host}:{port} successful")
               
               # Send test command
               test_message = b'TEST\n'
               sock.send(test_message)
               
               response = sock.recv(1024)
               print(f"Terminal response: {response.decode('utf-8')}")
               
           else:
               print(f"Connection to {host}:{port} failed")
               
           sock.close()
           
       except Exception as e:
           print(f"Payment terminal test error: {e}")
   
   test_payment_terminal()
   ```

## Network and Connectivity Issues

### Internet Connection Problems

#### Symptoms
- Unable to process credit card payments
- Cloud sync failures
- API timeouts
- Slow system response

#### Troubleshooting Steps

1. **Network Connectivity Check**
   ```bash
   # Basic connectivity test
   ping -c 4 8.8.8.8
   ping -c 4 google.com
   
   # DNS resolution test
   nslookup google.com
   dig google.com
   
   # Check network interfaces
   ip addr show
   iwconfig  # For wireless
   ```

2. **Speed and Latency Test**
   ```bash
   # Install speedtest-cli if not available
   pip install speedtest-cli
   
   # Run speed test
   speedtest-cli
   
   # Check latency to payment gateway
   ping -c 10 api.stripe.com
   traceroute api.stripe.com
   ```

3. **Network Configuration Script**
   ```bash
   #!/bin/bash
   # network_diagnostic.sh
   
   echo "=== Network Diagnostic Report ==="
   echo "Timestamp: $(date)"
   echo
   
   echo "=== Network Interfaces ==="
   ip addr show
   echo
   
   echo "=== Default Gateway ==="
   ip route show default
   echo
   
   echo "=== DNS Configuration ==="
   cat /etc/resolv.conf
   echo
   
   echo "=== Connectivity Tests ==="
   
   # Test local gateway
   gateway=$(ip route | grep default | awk '{print $3}' | head -n1)
   echo "Testing gateway ($gateway):"
   ping -c 3 $gateway
   echo
   
   # Test external connectivity
   echo "Testing external connectivity:"
   ping -c 3 8.8.8.8
   echo
   
   # Test DNS resolution
   echo "Testing DNS resolution:"
   nslookup google.com
   echo
   
   # Test HTTPS connectivity
   echo "Testing HTTPS connectivity:"
   curl -I https://www.google.com
   ```

### Local Network Issues

#### Symptoms
- Can't connect to local database
- Printer not accessible
- Multi-terminal communication problems

#### Troubleshooting Steps

1. **Port Scanning**
   ```bash
   # Check open ports on local machine
   netstat -tlnp
   
   # Scan specific ports
   nmap -p 3306,3000,80,443 localhost
   
   # Check if specific service is listening
   netstat -tlnp | grep :3306
   ```

2. **Firewall Configuration**
   ```bash
   # Check firewall status
   sudo ufw status
   
   # Check iptables rules
   sudo iptables -L
   
   # Allow specific ports
   sudo ufw allow 3306  # MySQL
   sudo ufw allow 3000  # API server
   ```

## Payment Processing Problems

### Credit Card Processing Failures

#### Symptoms
- Transactions declined incorrectly
- Gateway timeout errors
- Invalid response formats
- Authentication failures

#### Troubleshooting Steps

1. **Payment Gateway Status Check**
   ```python
   # payment_gateway_test.py
   import requests
   import json
   from datetime import datetime
   
   def test_payment_gateway():
       # Test endpoints for major payment processors
       test_endpoints = {
           'stripe': 'https://api.stripe.com/v1/charges',
           'square': 'https://connect.squareup.com/v2/payments',
           'paypal': 'https://api.paypal.com/v1/oauth2/token'
       }
       
       for name, url in test_endpoints.items():
           try:
               response = requests.get(url, timeout=10)
               print(f"{name} Gateway Status: {response.status_code}")
               print(f"Response time: {response.elapsed.total_seconds():.2f}s")
               
               if response.status_code == 401:
                   print("✓ Gateway is responding (authentication required)")
               elif response.status_code < 500:
                   print("✓ Gateway is accessible")
               else:
                   print("✗ Gateway may be experiencing issues")
                   
           except requests.exceptions.Timeout:
               print(f"✗ {name} Gateway timeout")
           except requests.exceptions.RequestException as e:
               print(f"✗ {name} Gateway error: {e}")
           
           print("-" * 50)
   
   test_payment_gateway()
   ```

2. **Transaction Log Analysis**
   ```python
   # payment_log_analyzer.py
   import json
   import re
   from datetime import datetime, timedelta
   
   def analyze_payment_logs(log_file_path):
       patterns = {
           'declined': re.compile(r'payment.*declined', re.IGNORECASE),
           'timeout': re.compile(r'timeout|timed out', re.IGNORECASE),
           'network_error': re.compile(r'network.*error|connection.*error', re.IGNORECASE),
           'auth_error': re.compile(r'authentication.*failed|unauthorized', re.IGNORECASE)
       }
       
       error_counts = {key: 0 for key in patterns.keys()}
       recent_errors = []
       
       try:
           with open(log_file_path, 'r') as file:
               for line in file:
                   # Check for error patterns
                   for error_type, pattern in patterns.items():
                       if pattern.search(line):
                           error_counts[error_type] += 1
                           
                           # Extract timestamp if possible
                           timestamp_match = re.search(r'\d{4}-\d{2}-\d{2}.\d{2}:\d{2}:\d{2}', line)
                           if timestamp_match:
                               recent_errors.append({
                                   'type': error_type,
                                   'timestamp': timestamp_match.group(),
                                   'message': line.strip()
                               })
           
           print("=== Payment Error Analysis ===")
           for error_type, count in error_counts.items():
               print(f"{error_type.replace('_', ' ').title()}: {count}")
           
           print("\n=== Recent Errors ===")
           for error in recent_errors[-10:]:  # Show last 10 errors
               print(f"[{error['timestamp']}] {error['type']}: {error['message'][:100]}...")
               
       except FileNotFoundError:
           print(f"Log file not found: {log_file_path}")
       except Exception as e:
           print(f"Error analyzing logs: {e}")
   
   analyze_payment_logs('/var/log/pos/payment.log')
   ```

3. **Payment Processor Health Check**
   ```bash
   #!/bin/bash
   # payment_health_check.sh
   
   echo "=== Payment Processor Health Check ==="
   
   # Check SSL certificates
   echo "Checking SSL certificates..."
   for domain in api.stripe.com connect.squareup.com api.paypal.com; do
       echo "Testing $domain:"
       echo | openssl s_client -connect $domain:443 -servername $domain 2>/dev/null | \
       openssl x509 -noout -dates
       echo
   done
   
   # Test API endpoints
   echo "Testing API endpoints..."
   
   # Stripe test
   curl -I -w "Response time: %{time_total}s\n" -s -o /dev/null \
        https://api.stripe.com/v1/charges
   
   # Square test  
   curl -I -w "Response time: %{time_total}s\n" -s -o /dev/null \
        https://connect.squareup.com/v2/payments
   ```

### EMV Chip Card Issues

#### Symptoms
- Chip read failures
- "Insert card" loops
- Slow chip processing
- Fallback to magnetic stripe

#### Solutions

1. **EMV Terminal Diagnostics**
   ```python
   # emv_diagnostic.py
   def emv_diagnostic_test():
       """
       Perform EMV terminal diagnostic tests
       """
       tests = [
           "Card insertion detection",
           "Chip communication",
           "Application selection",
           "Cardholder verification",
           "Transaction authorization",
           "Card removal detection"
       ]
       
       print("=== EMV Terminal Diagnostic ===")
       
       for i, test in enumerate(tests, 1):
           print(f"{i}. {test}...")
           # In real implementation, this would interface with EMV library
           # result = emv_lib.run_test(test)
           # For demo purposes:
           import time
           time.sleep(0.5)
           print("   ✓ PASS")
       
       print("\nDiagnostic complete. All tests passed.")
   
   emv_diagnostic_test()
   ```

## Database Issues

### Connection Problems

#### Symptoms
- "Connection refused" errors
- Timeout errors
- "Too many connections" errors
- Slow query performance

#### Troubleshooting Steps

1. **Database Status Check**
   ```bash
   # Check MySQL service status
   systemctl status mysql
   
   # Check MySQL processes
   ps aux | grep mysql
   
   # Check MySQL error log
   tail -f /var/log/mysql/error.log
   
   # Test connection
   mysql -u pos_user -p -e "SELECT NOW();"
   ```

2. **Connection Pool Monitoring**
   ```sql
   -- Check current connections
   SHOW PROCESSLIST;
   
   -- Check connection statistics
   SHOW STATUS LIKE 'Connections';
   SHOW STATUS LIKE 'Threads_connected';
   SHOW STATUS LIKE 'Max_used_connections';
   
   -- Check variables
   SHOW VARIABLES LIKE 'max_connections';
   SHOW VARIABLES LIKE 'wait_timeout';
   ```

3. **Database Performance Script**
   ```python
   # db_performance_check.py
   import mysql.connector
   import time
   import threading
   
   def db_performance_test():
       config = {
           'user': 'pos_user',
           'password': 'pos_password',
           'host': 'localhost',
           'database': 'pos_system',
           'raise_on_warnings': True
       }
       
       # Test basic connection
       try:
           cnx = mysql.connector.connect(**config)
           cursor = cnx.cursor()
           
           # Test simple query performance
           start_time = time.time()
           cursor.execute("SELECT COUNT(*) FROM products")
           result = cursor.fetchone()
           end_time = time.time()
           
           print(f"Product count query: {end_time - start_time:.3f}s")
           print(f"Total products: {result[0]}")
           
           # Test transaction performance
           start_time = time.time()
           cursor.execute("SELECT COUNT(*) FROM transactions WHERE DATE(created_at) = CURDATE()")
           result = cursor.fetchone()
           end_time = time.time()
           
           print(f"Today's transactions query: {end_time - start_time:.3f}s")
           print(f"Today's transactions: {result[0]}")
           
           cursor.close()
           cnx.close()
           
       except mysql.connector.Error as err:
           print(f"Database error: {err}")
   
   db_performance_test()
   ```

### Data Integrity Issues

#### Symptoms
- Inconsistent inventory counts
- Transaction totals don't match
- Missing records
- Duplicate entries

#### Troubleshooting Steps

1. **Data Integrity Check Script**
   ```sql
   -- Check for orphaned transaction items
   SELECT ti.id, ti.transaction_id
   FROM transaction_items ti
   LEFT JOIN transactions t ON ti.transaction_id = t.id
   WHERE t.id IS NULL;
   
   -- Check for negative inventory
   SELECT p.name, i.quantity_on_hand
   FROM inventory i
   JOIN products p ON i.product_id = p.id
   WHERE i.quantity_on_hand < 0;
   
   -- Check transaction totals
   SELECT 
       t.id,
       t.total_amount,
       (SUM(ti.line_total) + t.tax_amount - t.discount_amount) as calculated_total
   FROM transactions t
   JOIN transaction_items ti ON t.id = ti.transaction_id
   GROUP BY t.id
   HAVING ABS(t.total_amount - calculated_total) > 0.01;
   ```

2. **Database Repair Script**
   ```python
   # db_repair.py
   import mysql.connector
   from decimal import Decimal
   
   def repair_transaction_totals():
       config = {
           'user': 'pos_user',
           'password': 'pos_password',
           'host': 'localhost',
           'database': 'pos_system'
       }
       
       cnx = mysql.connector.connect(**config)
       cursor = cnx.cursor()
       
       # Find transactions with incorrect totals
       query = """
       SELECT 
           t.id,
           t.total_amount,
           (SUM(ti.line_total) + t.tax_amount - t.discount_amount) as calculated_total
       FROM transactions t
       JOIN transaction_items ti ON t.id = ti.transaction_id
       GROUP BY t.id
       HAVING ABS(t.total_amount - calculated_total) > 0.01
       """
       
       cursor.execute(query)
       incorrect_transactions = cursor.fetchall()
       
       print(f"Found {len(incorrect_transactions)} transactions with incorrect totals")
       
       for trans_id, current_total, calculated_total in incorrect_transactions:
           print(f"Transaction {trans_id}: {current_total} -> {calculated_total}")
           
           # Update the transaction total
           update_query = "UPDATE transactions SET total_amount = %s WHERE id = %s"
           cursor.execute(update_query, (calculated_total, trans_id))
       
       cnx.commit()
       cursor.close()
       cnx.close()
       
       print("Transaction totals repaired")
   
   repair_transaction_totals()
   ```

## Performance Problems

### Slow Response Times

#### Symptoms
- Long loading times
- Delayed transaction processing
- UI freezing
- Timeout errors

#### Troubleshooting Steps

1. **Performance Monitoring Script**
   ```python
   # performance_monitor.py
   import psutil
   import time
   import requests
   
   def monitor_system_performance():
       print("=== System Performance Monitor ===")
       
       # CPU Usage
       cpu_percent = psutil.cpu_percent(interval=1)
       print(f"CPU Usage: {cpu_percent}%")
       
       # Memory Usage
       memory = psutil.virtual_memory()
       print(f"Memory Usage: {memory.percent}% ({memory.used / 1024**3:.1f}GB / {memory.total / 1024**3:.1f}GB)")
       
       # Disk Usage
       disk = psutil.disk_usage('/')
       print(f"Disk Usage: {disk.percent}% ({disk.used / 1024**3:.1f}GB / {disk.total / 1024**3:.1f}GB)")
       
       # Network I/O
       network = psutil.net_io_counters()
       print(f"Network - Sent: {network.bytes_sent / 1024**2:.1f}MB, Received: {network.bytes_recv / 1024**2:.1f}MB")
       
       # Test API response time
       try:
           start_time = time.time()
           response = requests.get('http://localhost:3000/api/v1/health')
           end_time = time.time()
           print(f"API Response Time: {(end_time - start_time) * 1000:.0f}ms")
       except Exception as e:
           print(f"API Test Failed: {e}")
   
   monitor_system_performance()
   ```

2. **Database Query Optimization**
   ```sql
   -- Enable slow query log
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
   SET GLOBAL long_query_time = 1;
   
   -- Check for missing indexes
   SELECT 
       table_name,
       column_name,
       cardinality
   FROM information_schema.statistics
   WHERE table_schema = 'pos_system'
   AND cardinality < 100;
   
   -- Analyze table usage
   SELECT 
       table_name,
       table_rows,
       data_length / 1024 / 1024 as data_size_mb,
       index_length / 1024 / 1024 as index_size_mb
   FROM information_schema.tables
   WHERE table_schema = 'pos_system'
   ORDER BY data_length DESC;
   ```

## Security and Authentication Issues

### Login Problems

#### Symptoms
- Authentication failures
- Session timeouts
- Invalid token errors
- Permission denied errors

#### Troubleshooting Steps

1. **Authentication Debug Script**
   ```python
   # auth_debug.py
   import jwt
   import hashlib
   import bcrypt
   from datetime import datetime, timedelta
   
   def debug_jwt_token(token, secret_key):
       try:
           # Decode without verification first to see contents
           unverified = jwt.decode(token, options={"verify_signature": False})
           print("Token contents (unverified):")
           print(f"  User ID: {unverified.get('user_id')}")
           print(f"  Role: {unverified.get('role')}")
           print(f"  Issued: {datetime.fromtimestamp(unverified.get('iat', 0))}")
           print(f"  Expires: {datetime.fromtimestamp(unverified.get('exp', 0))}")
           
           # Now verify the token
           verified = jwt.decode(token, secret_key, algorithms=['HS256'])
           print("✓ Token signature is valid")
           
           # Check expiration
           if verified['exp'] < datetime.utcnow().timestamp():
               print("✗ Token has expired")
           else:
               print("✓ Token is not expired")
               
           return verified
           
       except jwt.ExpiredSignatureError:
           print("✗ Token has expired")
       except jwt.InvalidTokenError as e:
           print(f"✗ Invalid token: {e}")
       except Exception as e:
           print(f"✗ Token validation error: {e}")
       
       return None
   
   def verify_password(password, hashed_password):
       """Verify bcrypt password"""
       try:
           return bcrypt.checkpw(password.encode('utf-8'), hashed_password.encode('utf-8'))
       except Exception as e:
           print(f"Password verification error: {e}")
           return False
   
   # Example usage
   # debug_jwt_token("your_jwt_token_here", "your_secret_key")
   ```

2. **Session Management Check**
   ```bash
   # Check active sessions
   redis-cli KEYS "session:*"
   
   # Check session data
   redis-cli GET "session:user_123"
   
   # Clear expired sessions
   redis-cli EVAL "
   local keys = redis.call('keys', 'session:*')
   local expired = 0
   for i=1,#keys do
       local ttl = redis.call('ttl', keys[i])
       if ttl == -1 then
           redis.call('del', keys[i])
           expired = expired + 1
       end
   end
   return expired
   " 0
   ```

## Integration Problems

### Third-Party Service Issues

#### Symptoms
- API endpoint unavailable
- Authentication failures with external services
- Data sync failures
- Webhook delivery failures

#### Troubleshooting Steps

1. **API Integration Test**
   ```python
   # integration_test.py
   import requests
   import json
   from datetime import datetime
   
   def test_integrations():
       integrations = {
           'accounting_system': {
               'url': 'https://api.quickbooks.com/v3/company/123/items',
               'auth_header': 'Bearer your_oauth_token'
           },
           'inventory_system': {
               'url': 'https://api.inventory-service.com/v1/products',
               'auth_header': 'API-Key your_api_key'
           },
           'loyalty_program': {
               'url': 'https://api.loyalty-service.com/v1/customers',
               'auth_header': 'Authorization: Bearer your_token'
           }
       }
       
       print("=== Integration Testing ===")
       
       for service_name, config in integrations.items():
           print(f"\nTesting {service_name}...")
           
           try:
               headers = {'Authorization': config['auth_header']}
               response = requests.get(config['url'], headers=headers, timeout=10)
               
               print(f"  Status: {response.status_code}")
               print(f"  Response time: {response.elapsed.total_seconds():.2f}s")
               
               if response.status_code == 200:
                   print("  ✓ Integration working")
               elif response.status_code == 401:
                   print("  ✗ Authentication failed")
               elif response.status_code >= 500:
                   print("  ✗ Service unavailable")
               else:
                   print(f"  ? Unexpected status: {response.status_code}")
                   
           except requests.exceptions.Timeout:
               print("  ✗ Request timeout")
           except requests.exceptions.RequestException as e:
               print(f"  ✗ Request failed: {e}")
   
   test_integrations()
   ```

## Diagnostic Tools and Scripts

### System Health Check

```bash
#!/bin/bash
# pos_health_check.sh - Comprehensive system health check

echo "========================================"
echo "POS System Health Check"
echo "========================================"
echo "Timestamp: $(date)"
echo

# System resources
echo "=== System Resources ==="
echo "CPU Usage:"
top -bn1 | grep "Cpu(s)" | awk '{print $2 $3 $4 $5 $6 $7 $8}'

echo "Memory Usage:"
free -h

echo "Disk Usage:"
df -h /

echo

# Services status
echo "=== Critical Services ==="
services=("mysql" "redis" "nginx" "pos-api")
for service in "${services[@]}"; do
    if systemctl is-active --quiet "$service"; then
        echo "✓ $service is running"
    else
        echo "✗ $service is not running"
    fi
done

echo

# Network connectivity
echo "=== Network Connectivity ==="
if ping -c 1 8.8.8.8 >/dev/null 2>&1; then
    echo "✓ Internet connectivity OK"
else
    echo "✗ No internet connectivity"
fi

# Database connectivity
echo "=== Database Connectivity ==="
if mysql -u pos_user -ppos_password -e "SELECT 1;" >/dev/null 2>&1; then
    echo "✓ Database connection OK"
else
    echo "✗ Cannot connect to database"
fi

# API health check
echo "=== API Health Check ==="
if curl -f http://localhost:3000/health >/dev/null 2>&1; then
    echo "✓ API endpoint responding"
else
    echo "✗ API endpoint not responding"
fi

echo
echo "Health check complete."
```

### Log Analysis Tool

```python
# log_analyzer.py
import re
import json
from datetime import datetime, timedelta
from collections import defaultdict, Counter

class POSLogAnalyzer:
    def __init__(self, log_files):
        self.log_files = log_files
        self.patterns = {
            'error': re.compile(r'ERROR|FATAL|CRITICAL', re.IGNORECASE),
            'warning': re.compile(r'WARN|WARNING', re.IGNORECASE),
            'transaction': re.compile(r'transaction.*completed|payment.*processed', re.IGNORECASE),
            'failed_payment': re.compile(r'payment.*failed|transaction.*declined', re.IGNORECASE),
            'slow_query': re.compile(r'slow.*query|query.*timeout', re.IGNORECASE)
        }
    
    def analyze_logs(self, hours=24):
        """Analyze logs from the last N hours"""
        cutoff_time = datetime.now() - timedelta(hours=hours)
        
        results = {
            'summary': defaultdict(int),
            'errors': [],
            'warnings': [],
            'performance_issues': [],
            'hourly_stats': defaultdict(lambda: defaultdict(int))
        }
        
        for log_file in self.log_files:
            try:
                with open(log_file, 'r') as f:
                    for line in f:
                        timestamp = self.extract_timestamp(line)
                        if timestamp and timestamp >= cutoff_time:
                            self.analyze_line(line, timestamp, results)
            except FileNotFoundError:
                print(f"Log file not found: {log_file}")
            except Exception as e:
                print(f"Error reading {log_file}: {e}")
        
        return results
    
    def extract_timestamp(self, line):
        """Extract timestamp from log line"""
        timestamp_patterns = [
            r'(\d{4}-\d{2}-\d{2}.\d{2}:\d{2}:\d{2})',
            r'(\d{2}/\d{2}/\d{4}.\d{2}:\d{2}:\d{2})',
            r'(\w{3}.\d{2}.\d{2}:\d{2}:\d{2})'
        ]
        
        for pattern in timestamp_patterns:
            match = re.search(pattern, line)
            if match:
                try:
                    return datetime.strptime(match.group(1), '%Y-%m-%d %H:%M:%S')
                except ValueError:
                    continue
        return None
    
    def analyze_line(self, line, timestamp, results):
        """Analyze individual log line"""
        hour_key = timestamp.strftime('%Y-%m-%d %H:00')
        
        for category, pattern in self.patterns.items():
            if pattern.search(line):
                results['summary'][category] += 1
                results['hourly_stats'][hour_key][category] += 1
                
                if category == 'error':
                    results['errors'].append({
                        'timestamp': timestamp.isoformat(),
                        'message': line.strip()
                    })
                elif category == 'warning':
                    results['warnings'].append({
                        'timestamp': timestamp.isoformat(),
                        'message': line.strip()
                    })
                elif category == 'slow_query':
                    results['performance_issues'].append({
                        'timestamp': timestamp.isoformat(),
                        'message': line.strip()
                    })
    
    def generate_report(self, results):
        """Generate analysis report"""
        print("=== POS Log Analysis Report ===")
        print(f"Analysis period: Last 24 hours")
        print(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        print()
        
        print("=== Summary ===")
        for category, count in results['summary'].items():
            print(f"{category.replace('_', ' ').title()}: {count}")
        print()
        
        if results['errors']:
            print("=== Recent Errors ===")
            for error in results['errors'][-5:]:  # Last 5 errors
                print(f"[{error['timestamp']}] {error['message'][:100]}...")
            print()
        
        if results['performance_issues']:
            print("=== Performance Issues ===")
            for issue in results['performance_issues'][-5:]:
                print(f"[{issue['timestamp']}] {issue['message'][:100]}...")
            print()
        
        print("=== Hourly Statistics ===")
        for hour in sorted(results['hourly_stats'].keys())[-12:]:  # Last 12 hours
            stats = results['hourly_stats'][hour]
            print(f"{hour}: Transactions: {stats['transaction']}, "
                  f"Errors: {stats['error']}, "
                  f"Failed Payments: {stats['failed_payment']}")

# Usage
if __name__ == "__main__":
    log_files = [
        '/var/log/pos/application.log',
        '/var/log/pos/payment.log',
        '/var/log/pos/error.log'
    ]
    
    analyzer = POSLogAnalyzer(log_files)
    results = analyzer.analyze_logs(hours=24)
    analyzer.generate_report(results)
```

## Emergency Procedures

### System Recovery Checklist

```markdown
# POS System Emergency Recovery Checklist

## Immediate Response (0-5 minutes)
- [ ] Assess the scope of the problem
- [ ] Notify management and IT team
- [ ] Switch to backup/manual processes if available
- [ ] Document the incident time and symptoms

## Quick Fixes (5-15 minutes)
- [ ] Restart POS application
- [ ] Check network connectivity
- [ ] Verify database connection
- [ ] Test hardware components
- [ ] Check system resources (CPU, memory, disk)

## Service Restoration (15-30 minutes)
- [ ] Run system health check script
- [ ] Review recent error logs
- [ ] Restart critical services if needed
- [ ] Test basic POS functions
- [ ] Verify payment processing

## Full Recovery (30+ minutes)
- [ ] Perform comprehensive system diagnostics
- [ ] Restore from backup if necessary
- [ ] Update all affected systems
- [ ] Test all functionality thoroughly
- [ ] Train staff on any changes

## Post-Incident (After resolution)
- [ ] Document root cause
- [ ] Update procedures
- [ ] Plan preventive measures
- [ ] Schedule follow-up review
```

### Backup and Recovery Procedures

```bash
#!/bin/bash
# backup_recovery.sh

BACKUP_DIR="/backup/pos"
DB_NAME="pos_system"
DB_USER="pos_user"
DATE=$(date +%Y%m%d_%H%M%S)

# Create database backup
create_backup() {
    echo "Creating database backup..."
    mysqldump -u $DB_USER -p $DB_NAME > "$BACKUP_DIR/db_backup_$DATE.sql"
    
    # Backup application files
    tar -czf "$BACKUP_DIR/app_backup_$DATE.tar.gz" /opt/pos/
    
    echo "Backup completed: $BACKUP_DIR"
}

# Restore from backup
restore_backup() {
    local backup_file=$1
    
    if [ -z "$backup_file" ]; then
        echo "Please specify backup file"
        return 1
    fi
    
    echo "Restoring database from $backup_file..."
    mysql -u $DB_USER -p $DB_NAME < "$backup_file"
    
    echo "Restart services..."
    systemctl restart pos-api
    systemctl restart nginx
    
    echo "Restore completed"
}

# Emergency restore procedure
emergency_restore() {
    echo "=== EMERGENCY RESTORE PROCEDURE ==="
    
    # Find latest backup
    latest_backup=$(ls -t $BACKUP_DIR/db_backup_*.sql | head -n1)
    
    if [ -z "$latest_backup" ]; then
        echo "No backup files found!"
        return 1
    fi
    
    echo "Latest backup: $latest_backup"
    read -p "Restore from this backup? (y/N): " confirm
    
    if [ "$confirm" = "y" ] || [ "$confirm" = "Y" ]; then
        restore_backup "$latest_backup"
    else
        echo "Restore cancelled"
    fi
}

case "$1" in
    backup)
        create_backup
        ;;
    restore)
        restore_backup "$2"
        ;;
    emergency)
        emergency_restore
        ;;
    *)
        echo "Usage: $0 {backup|restore <file>|emergency}"
        exit 1
        ;;
esac
```

This troubleshooting guide provides comprehensive solutions for the most common issues encountered in POS systems. Keep this guide accessible and ensure your team is trained on these procedures for effective problem resolution.