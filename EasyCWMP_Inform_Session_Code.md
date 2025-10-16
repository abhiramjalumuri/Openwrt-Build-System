# EasyCWMP Inform Message and Session Management Code

This document shows the key code sections for TR-069 Inform message handling and session management in EasyCWMP.

## 1. Main Session Flow (`cwmp_inform()`)

### Location: `/package/easycwmp/src/cwmp.c:268-340`

```c
int cwmp_inform(void)
{
    mxml_node_t *node;
    int method_id;

    log_message(NAME, L_NOTICE, "start session\n");
    
    // Initialize HTTP client
    if (http_client_init()) {
        log_message(NAME, L_DEBUG, "initializing http client failed\n");
        goto error;
    }
    
    // Initialize external scripts
    if (external_init()) {
        log_message(NAME, L_DEBUG, "external scripts initialization failed\n");
        goto error;
    }

    // Send Inform message and receive InformResponse
    if(rpc_inform()) {
        log_message(NAME, L_NOTICE, "sending Inform failed\n");
        goto error;
    }
    log_message(NAME, L_NOTICE, "receive InformResponse from the ACS\n");

    // Clean up events after successful Inform
    cwmp_remove_event(EVENT_REMOVE_AFTER_INFORM, 0);
    cwmp_clear_notifications();

    // Handle pending transfers and other RPC methods
    do {
        // Process TransferComplete messages
        while((node = backup_check_transfer_complete()) && !cwmp->hold_requests) {
            if(rpc_transfer_complete(node, &method_id)) {
                log_message(NAME, L_NOTICE, "sending TransferComplete failed\n");
                goto error;
            }
            log_message(NAME, L_NOTICE, "receive TransferCompleteResponse from the ACS\n");

            backup_remove_transfer_complete(node);
            cwmp_remove_event(EVENT_REMOVE_AFTER_TRANSFER_COMPLETE, method_id);
            if (!backup_check_transfer_complete())
                cwmp_remove_event(EVENT_REMOVE_AFTER_TRANSFER_COMPLETE, 0);
        }
        
        // Handle GetRPCMethods if requested
        if(cwmp->get_rpc_methods && !cwmp->hold_requests) {
            if(rpc_get_rpc_methods()) {
                log_message(NAME, L_NOTICE, "sending GetRPCMethods failed\n");
                goto error;
            }
            log_message(NAME, L_NOTICE, "receive GetRPCMethodsResponse from the ACS\n");
            cwmp->get_rpc_methods = false;
        }

        // Handle incoming messages from ACS
        if (cwmp_handle_messages()) {
            log_message(NAME, L_DEBUG, "handling xml message failed\n");
            goto error;
        }
        cwmp->hold_requests = false;
    } while (cwmp->get_rpc_methods || backup_check_transfer_complete());
    
    // Clean up and end session successfully
    http_client_exit();
    xml_exit();
    cwmp_handle_end_session();
    external_exit();
    cwmp->retry_count = 0;
    log_message(NAME, L_NOTICE, "end session success\n");
    return 0;

error:
    // Handle session failure
    http_client_exit();
    xml_exit();
    cwmp_handle_end_session();
    external_exit();
    log_message(NAME, L_NOTICE, "end session failed\n");
    cwmp_retry_session();
    return -1;
}
```

## 2. Inform RPC Implementation (`rpc_inform()`)

### Location: `/package/easycwmp/src/cwmp.c:203-237`

```c
static inline int rpc_inform()
{
    char *msg_in = NULL, *msg_out = NULL;
    int error = 0, count = 0;

    // Prepare the Inform XML message
    if (xml_prepare_inform_message(&msg_out)) {
        log_message(NAME, L_DEBUG, "Inform xml message creating failed\n");
        return -1;
    }

    log_message(NAME, L_NOTICE, "send Inform\n");

    // Send Inform and handle potential retries
    do {
        FREE(msg_in);

        // Send HTTP message to ACS
        if (http_send_message(msg_out, &msg_in)) {
            log_message(NAME, L_DEBUG, "sending Inform http message failed\n");
            error = -1; 
            break;
        }

        // Check for empty response
        if (!msg_in) {
            log_message(NAME, L_DEBUG, "parse Inform xml message from ACS: Empty message\n");
            error = -1; 
            break;
        }

        // Parse InformResponse from ACS
        error = xml_parse_inform_response_message(msg_in);
        if (error && (error != FAULT_ACS_8005)) {
            log_message(NAME, L_DEBUG, "parse Inform xml message from ACS failed\n");
            error = -1; 
            break;
        }
    } while(error && (count++)<10);  // Retry up to 10 times

    FREE(msg_out);
    FREE(msg_in);
    return error;
}
```

## 3. Inform Message Preparation (`xml_prepare_inform_message()`)

### Location: `/package/easycwmp/src/xml.c:568-671`

```c
int xml_prepare_inform_message(char **msg_out)
{
    mxml_node_t *tree, *b, *n, *parameter_list;
    struct external_parameter *external_parameter;
    char *c;
    int counter = 0;

    // Load the Inform message template
    tree = mxmlLoadString(NULL, CWMP_INFORM_MESSAGE, MXML_OPAQUE_CALLBACK);
    if (!tree) goto error;

    // Add CWMP ID to message
    if(xml_add_cwmpid(tree)) goto error;

    // Set retry count
    b = mxmlFindElement(tree, tree, "RetryCount", NULL, NULL, MXML_DESCEND);
    if (!b) goto error;
    b = mxmlNewInteger(b, cwmp->retry_count);
    if (!b) goto error;

    // Add device identification information
    b = mxmlFindElement(tree, tree, "Manufacturer", NULL, NULL, MXML_DESCEND);
    if (!b) goto error;
    b = mxmlNewOpaque(b, cwmp->deviceid.manufacturer);
    if (!b) goto error;

    b = mxmlFindElement(tree, tree, "OUI", NULL, NULL, MXML_DESCEND);
    if (!b) goto error;
    b = mxmlNewOpaque(b, cwmp->deviceid.oui);
    if (!b) goto error;

    b = mxmlFindElement(tree, tree, "ProductClass", NULL, NULL, MXML_DESCEND);
    if (!b) goto error;
    b = mxmlNewOpaque(b, cwmp->deviceid.product_class);
    if (!b) goto error;

    b = mxmlFindElement(tree, tree, "SerialNumber", NULL, NULL, MXML_DESCEND);
    if (!b) goto error;
    b = mxmlNewOpaque(b, cwmp->deviceid.serial_number);
    if (!b) goto error;
   
    // Add events that triggered this Inform
    if (xml_prepare_events_inform(tree))
        goto error;

    // Add current timestamp
    b = mxmlFindElement(tree, tree, "CurrentTime", NULL, NULL, MXML_DESCEND);
    if (!b) goto error;
    b = mxmlNewOpaque(b, mix_get_time());
    if (!b) goto error;

    // Get parameters to include in Inform
    external_action_simple_execute("inform", "parameter", NULL);
    if (external_action_handle(json_handle_get_parameter_value))
        goto error;

    // Build parameter list...
    // [Additional parameter building code continues...]

    // Convert XML tree to string
    *msg_out = mxmlSaveAllocString(tree, xml_format_cb);
    mxmlDelete(tree);
    
    return 0;

error:
    if (tree) mxmlDelete(tree);
    return -1;
}
```

## 4. Inform Response Parsing (`xml_parse_inform_response_message()`)

### Location: `/package/easycwmp/src/xml.c:671-700`

```c
int xml_parse_inform_response_message(char *msg_in)
{
    mxml_node_t *tree, *b;
    char *c;
    int fault = 0;

    // Parse the XML response from ACS
    tree = mxmlLoadString(NULL, msg_in, MXML_OPAQUE_CALLBACK);
    if (!tree) goto error;
    
    // Recreate XML namespace
    if(xml_recreate_namespace(tree)) goto error;

    // Check for SOAP faults
    b = xml_mxml_find_node_by_env_type(tree, "Fault");
    if (b) {
        // Check for specific fault 8005 (retry required)
        b = mxmlFindElementOpaque(b, b, "8005", MXML_DESCEND);
        if (b) {
            fault = FAULT_ACS_8005;
            goto out;
        }
        goto error;
    }

    // Get hold request information
    xml_get_hold_request(tree);
    
    // Get MaxEnvelopes value
    b = mxmlFindElement(tree, tree, "MaxEnvelopes", NULL, NULL, MXML_DESCEND);
    if (!b) goto error;

    b = mxmlWalkNext(b, tree, MXML_DESCEND_FIRST);
    if (!b || !b->value.opaque)
        goto error;

out:
    mxmlDelete(tree);
    return fault;

error:
    mxmlDelete(tree);
    return -1;
}
```

## 5. Session End Handling (`cwmp_handle_end_session()`)

### Location: `/package/easycwmp/src/cwmp.c:245-266`

```c
static void cwmp_handle_end_session(void)
{
    // Apply any pending service changes
    external_action_simple_execute("apply", "service", NULL);
    external_action_handle(NULL);
    
    // Handle different session end types
    if (cwmp->end_session & ENDS_FACTORY_RESET) {
        log_message(NAME, L_NOTICE, "end session: factory reset\n");
        external_action_simple_execute("factory_reset", NULL, NULL);
        external_action_handle(NULL);
        exit(EXIT_SUCCESS);
    }
    
    if (cwmp->end_session & ENDS_REBOOT) {
        log_message(NAME, L_NOTICE, "end session: reboot\n");
        external_action_simple_execute("reboot", NULL, NULL);
        external_action_handle(NULL);
        exit(EXIT_SUCCESS);
    }
    
    if (cwmp->end_session & ENDS_RELOAD_CONFIG) {
        log_message(NAME, L_NOTICE, "end session: configuration reload\n");
        config_load();
    }
    
    // Clear session end flags
    cwmp->end_session = 0;
}

void cwmp_add_handler_end_session(int handler)
{
    cwmp->end_session |= handler;
}
```

## 6. Key Log Messages for Monitoring

### Successful Session Flow:
1. `"start session"` - Session initialization
2. `"send Inform"` - Inform message sent to ACS
3. `"receive InformResponse from the ACS"` - Inform successful
4. `"end session success"` - Session completed successfully

### Error Conditions:
1. `"sending Inform failed"` - Inform transmission failed
2. `"end session failed"` - Session ended with errors
3. `"Inform xml message creating failed"` - XML generation error
4. `"sending Inform http message failed"` - HTTP transmission error

### Session Types:
1. `"end session: factory reset"` - Factory reset requested
2. `"end session: reboot"` - Device reboot requested  
3. `"end session: configuration reload"` - Config reload requested

## 7. Event Flags and Constants

### End Session Types:
```c
#define ENDS_FACTORY_RESET  1
#define ENDS_REBOOT        2  
#define ENDS_RELOAD_CONFIG 4
```

### Fault Codes:
```c
#define FAULT_ACS_8005     8005  // Retry required
```

This code structure provides comprehensive TR-069 session management with proper error handling, logging, and state management for successful Inform exchanges with the ACS.