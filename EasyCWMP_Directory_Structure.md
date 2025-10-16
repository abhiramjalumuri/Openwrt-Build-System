# EasyCWMP Inform Message Code Directory Structure Guide

## 📁 Complete Directory Structure for Inform Message Code

```
/home/abhiram-jalumuri/openwrt/package/easycwmp/
├── 📁 src/                              # Main source code directory
│   ├── 🔧 cwmp.c                        # Core CWMP session management
│   ├── 🔧 cwmp.h                        # CWMP function declarations
│   ├── 🔧 xml.c                         # XML message processing (KEY FILE)
│   ├── 🔧 xml.h                         # XML function declarations
│   ├── 🔧 messages.h                    # SOAP message templates (KEY FILE)
│   ├── 🔧 easycwmp.c                    # Main daemon entry point
│   ├── 🔧 easycwmp.h                    # Main header file
│   ├── 🔧 http.c                        # HTTP client implementation
│   ├── 🔧 http.h                        # HTTP function declarations
│   ├── 🔧 external.c                    # External script interface
│   ├── 🔧 external.h                    # External script declarations
│   ├── 🔧 config.c                      # Configuration management
│   ├── 🔧 config.h                      # Configuration declarations
│   ├── 🔧 json.c                        # JSON parameter handling
│   ├── 🔧 json.h                        # JSON function declarations
│   ├── 🔧 backup.c                      # Backup/restore functionality
│   ├── 🔧 backup.h                      # Backup function declarations
│   ├── 🔧 log.c                         # Logging implementation
│   ├── 🔧 log.h                         # Logging declarations
│   ├── 🔧 time.c                        # Time utilities
│   ├── 🔧 time.h                        # Time function declarations
│   ├── 🔧 ubus.c                        # OpenWrt ubus integration
│   ├── 🔧 ubus.h                        # ubus declarations
│   ├── 🔧 base64.c                      # Base64 encoding/decoding
│   ├── 🔧 base64.h                      # Base64 declarations
│   ├── 🔧 basicauth.c                   # HTTP Basic authentication
│   ├── 🔧 basicauth.h                   # Basic auth declarations
│   ├── 🔧 digestauth.c                  # HTTP Digest authentication
│   ├── 🔧 digestauth.h                  # Digest auth declarations
│   ├── 🔧 md5.c                         # MD5 hash implementation
│   └── 🔧 md5.h                         # MD5 declarations
│
├── 📁 ext/                              # External files and templates
│   ├── 📁 soap_msg_templates/           # SOAP message templates (KEY DIR)
│   │   ├── 📋 cwmp_inform_message.xml   # Inform message template (KEY FILE)
│   │   └── 📋 cwmp_response_message.xml # Response message template
│   │
│   └── 📁 openwrt/                      # OpenWrt-specific files
│       ├── 📁 config/                   # Configuration files
│       │   └── 📋 easycwmp              # UCI configuration template
│       ├── 📁 init.d/                   # Init scripts
│       │   └── 📋 easycwmpd             # Service startup script
│       ├── 📁 scripts/                  # Runtime scripts
│       │   ├── 📋 easycwmp.sh           # Main script interface
│       │   ├── 📋 defaults              # Default parameter values
│       │   └── 📁 functions/            # Data model functions
│       │       ├── 📁 common/           # Common functions
│       │       ├── 📁 tr098/            # TR-098 data model
│       │       └── 📁 tr181/            # TR-181 data model (KEY DIR)
│       └── 📁 build/                    # Build configuration
│           └── 📋 Config.in             # Build options
│
├── 📋 Makefile                          # OpenWrt package build file
├── 📋 Config.in                         # Package configuration
├── 📋 configure.ac                      # Autoconf configuration
├── 📋 Makefile.am                       # Automake configuration
└── 📁 bin/                              # Build output directory
```

## 🔍 Key Files for Inform Message Code Exploration

### 1. 🎯 **Core Inform Message Files** (START HERE)

#### `/src/messages.h` - SOAP Message Templates
```bash
# View the Inform message template
head -50 /home/abhiram-jalumuri/openwrt/package/easycwmp/src/messages.h
```
- Contains `CWMP_INFORM_MESSAGE` macro
- Defines XML structure for Inform messages
- **Key Content**: SOAP envelope template

#### `/src/xml.c` - XML Message Processing
```bash
# View Inform message preparation
sed -n '568,671p' /home/abhiram-jalumuri/openwrt/package/easycwmp/src/xml.c
```
- **Function**: `xml_prepare_inform_message()`
- **Function**: `xml_parse_inform_response_message()`
- **Function**: `xml_prepare_events_inform()`
- **Key Content**: XML generation and parsing logic

#### `/src/cwmp.c` - Session Management
```bash
# View session and RPC handling
sed -n '200,340p' /home/abhiram-jalumuri/openwrt/package/easycwmp/src/cwmp.c
```
- **Function**: `cwmp_inform()`
- **Function**: `rpc_inform()`
- **Function**: `cwmp_handle_end_session()`
- **Key Content**: Session flow control

### 2. 📋 **Message Templates**

#### `/ext/soap_msg_templates/cwmp_inform_message.xml`
```bash
# View the actual XML template
cat /home/abhiram-jalumuri/openwrt/package/easycwmp/ext/soap_msg_templates/cwmp_inform_message.xml
```
- Raw XML template for Inform messages
- SOAP envelope structure
- Parameter placeholders

### 3. 🔧 **Supporting Files**

#### `/src/http.c` - HTTP Communication
```bash
# View HTTP message sending
grep -n "http_send_message" /home/abhiram-jalumuri/openwrt/package/easycwmp/src/http.c
```
- HTTP client implementation
- Message transmission to ACS
- Response handling

#### `/src/external.c` - Script Interface
```bash
# View external script calls
grep -n "inform" /home/abhiram-jalumuri/openwrt/package/easycwmp/src/external.c
```
- Parameter collection via scripts
- Event handling
- Data model interface

#### `/src/config.c` - Configuration Management
```bash
# View configuration loading
grep -n "device\|acs" /home/abhiram-jalumuri/openwrt/package/easycwmp/src/config.c
```
- Device information loading
- ACS configuration
- UCI integration

### 4. 🗂️ **Data Model Scripts**

#### `/ext/openwrt/scripts/functions/tr181/`
```bash
# List TR-181 functions
ls -la /home/abhiram-jalumuri/openwrt/package/easycwmp/ext/openwrt/scripts/functions/tr181/
```
- Device parameter implementations
- TR-181 data model functions
- Parameter get/set operations

#### `/ext/openwrt/scripts/functions/common/`
```bash
# List common functions
ls -la /home/abhiram-jalumuri/openwrt/package/easycwmp/ext/openwrt/scripts/functions/common/
```
- Shared utility functions
- Common parameter operations
- System interface functions

## 🔍 Code Exploration Commands

### Quick File Content Inspection:
```bash
# Navigate to source directory
cd /home/abhiram-jalumuri/openwrt/package/easycwmp/src

# View all Inform-related functions
grep -n "inform" *.c *.h

# View message templates
grep -n "CWMP_" messages.h

# View XML processing functions
grep -n "xml_.*inform" xml.c

# View session management
grep -n "session" cwmp.c

# View HTTP communication
grep -n "http_send" http.c
```

### Deep Code Analysis:
```bash
# Find all Inform message references
find /home/abhiram-jalumuri/openwrt/package/easycwmp -name "*.c" -o -name "*.h" | xargs grep -n -i "inform"

# Find all XML processing functions
grep -n "xml_.*(" /home/abhiram-jalumuri/openwrt/package/easycwmp/src/xml.h

# Find all CWMP session functions
grep -n "cwmp_.*(" /home/abhiram-jalumuri/openwrt/package/easycwmp/src/cwmp.h

# View all log messages related to Inform
grep -n "log_message.*[Ii]nform" /home/abhiram-jalumuri/openwrt/package/easycwmp/src/*.c
```

## 📊 **File Analysis Priority**

### **Priority 1 - Core Logic** (Must Read):
1. `src/messages.h` - Message templates
2. `src/xml.c` - XML processing (lines 568-700)
3. `src/cwmp.c` - Session management (lines 200-340)

### **Priority 2 - Supporting Logic** (Important):
4. `src/http.c` - HTTP communication
5. `src/external.c` - Script interface
6. `ext/soap_msg_templates/cwmp_inform_message.xml` - Template

### **Priority 3 - Configuration** (Reference):
7. `src/config.c` - Configuration loading
8. `ext/openwrt/scripts/` - Data model scripts
9. `ext/openwrt/config/easycwmp` - UCI config

## 🎯 **Recommended Exploration Path**

1. **Start with**: `/src/messages.h` to see message structure
2. **Then explore**: `/src/xml.c` functions 568-700 for message creation
3. **Follow with**: `/src/cwmp.c` functions 200-340 for session flow
4. **Check**: `/src/http.c` for transmission details
5. **Review**: `/ext/soap_msg_templates/` for actual templates
6. **Understand**: `/ext/openwrt/scripts/` for parameter collection

This structure gives you complete visibility into how EasyCWMP handles Inform messages from template to transmission!