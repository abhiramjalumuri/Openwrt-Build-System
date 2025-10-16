# EasyCWMP HTTPS Inform Failure Debug Guide

## 🚨 **Common "Inform Failed" Issues with HTTPS**

### **Issue 1: SSL Certificate Verification Failure**

**Symptoms:**
- "sending Inform failed"
- "LibCurl Error: SSL certificate problem"
- "Peer certificate cannot be authenticated"

**Debug Commands:**
```bash
# On device (172.29.128.211):
# 1. Check current SSL settings
uci show easycwmp.acs | grep ssl

# 2. Test ACS connectivity
curl -v https://your-acs-server:7547/

# 3. Check if SSL verification is causing issues
easycwmpd -f -d 2>&1 | grep -i ssl
```

**Quick Fix - Disable SSL Verification (for testing):**
```bash
# Temporarily disable SSL verification
uci set easycwmp.acs.ssl_verify='0'
uci commit easycwmp
/etc/init.d/easycwmpd restart
```

### **Issue 2: HTTP Connection Problems**

**Symptoms:**
- "LibCurl Error: Couldn't connect to server"
- "LibCurl Error: Timeout was reached"
- HTTP 302/307 redirect loops

**Debug Commands:**
```bash
# Check network connectivity
ping your-acs-server.com

# Test HTTP vs HTTPS
curl -v http://your-acs-server:7547/
curl -v https://your-acs-server:7547/

# Check firewall
iptables -L | grep 7547
```

**Fix:**
```bash
# Check ACS URL configuration
uci show easycwmp.acs.url

# Ensure correct protocol and port
uci set easycwmp.acs.url='https://your-acs-server:7547/'
uci commit easycwmp
```

### **Issue 3: Authentication Problems**

**Symptoms:**
- HTTP 401 Unauthorized
- HTTP 403 Forbidden
- "LibCurl Error: The requested URL returned error: 401"

**Debug Commands:**
```bash
# Check credentials
uci show easycwmp.acs.username
uci show easycwmp.acs.password

# Test manual authentication
curl -u username:password https://your-acs-server:7547/
```

**Fix:**
```bash
# Set correct credentials
uci set easycwmp.acs.username='your-username'
uci set easycwmp.acs.password='your-password'
uci commit easycwmp
```

## 🔍 **Step-by-Step Debug Process**

### **Step 1: Enable Debug Logging**
```bash
# Stop service
/etc/init.d/easycwmpd stop

# Run with full debug
easycwmpd -f -d > /tmp/easycwmp_debug.log 2>&1 &

# Watch logs in real-time
tail -f /tmp/easycwmp_debug.log
```

### **Step 2: Analyze Key Log Sections**

**Look for these patterns:**
```bash
# HTTP request being sent
"+++ SEND HTTP REQUEST +++"

# SSL configuration
"ssl_cert:", "ssl_cacert:", "ssl_verify:"

# LibCurl errors
"LibCurl Error:"

# HTTP response codes
"RECEIVED HTTP RESPONSE"

# Session results
"end session success" or "end session failed"
```

### **Step 3: Common Configuration Fixes**

**For SSL Issues:**
```bash
# Option 1: Disable SSL verification (testing only)
uci set easycwmp.acs.ssl_verify='0'

# Option 2: Add CA certificate
uci set easycwmp.acs.ssl_cacert='/etc/ssl/certs/ca-certificates.crt'

# Option 3: Use HTTP instead of HTTPS (if supported)
uci set easycwmp.acs.url='http://your-acs-server:7547/'

uci commit easycwmp
```

**For Connection Issues:**
```bash
# Check and fix URL format
uci set easycwmp.acs.url='https://your-acs-server:7547/'

# Set reasonable timeout
uci set easycwmp.acs.timeout='30'

# Disable HTTP 100-continue if problematic
uci set easycwmp.acs.http100continue_disable='1'

uci commit easycwmp
```

## 🎯 **Files to Check for HTTPS Issues**

### **1. HTTP Client Code: `/package/easycwmp/src/http.c`**
- Lines 116-200: `http_send_message()` function
- Lines 73-78: SSL configuration setup
- Lines 153-189: Error handling and response processing

### **2. Configuration: `/etc/config/easycwmp`**
```bash
# Check these settings:
config acs 'acs'
    option url 'https://...'
    option ssl_verify '0'        # Try setting to 0 for testing
    option ssl_cert ''           # Client certificate if required
    option ssl_cacert ''         # CA certificate path
```

### **3. System Logs:**
```bash
# Check system logs for additional errors
logread | grep -i easycwmp
logread | grep -i ssl
logread | grep -i curl
```

## ⚡ **Quick Resolution Commands**

**Run these on your device to fix common HTTPS issues:**

```bash
# 1. Disable SSL verification temporarily
uci set easycwmp.acs.ssl_verify='0'
uci commit easycwmp

# 2. Restart service with debug
/etc/init.d/easycwmpd stop
easycwmpd -f -d

# 3. If it works, you have SSL cert issues
# If it still fails, check URL/credentials
```

This should help you identify and fix the specific HTTPS/SSL issue causing your Inform failures!