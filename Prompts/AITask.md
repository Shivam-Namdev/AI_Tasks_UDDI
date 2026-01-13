# AI-Driven Test Case Generation Guide for UDDI Features

## Background Context

This document provides comprehensive prompts for AI-driven test automation. The goal is to create test cases with enough detail that AI can execute them independently while also serving as clear automation guides.

**Key Requirement:** Each test case should be written like a prompt to AI, enabling execution without prior knowledge, while referencing existing automation repository for context.

---

## Reference Test File

This guide is based on the existing test suite: `github/jan2025/qa-uddi-api-tests-sn/WAPI_PyTest/suites/RFE_UDDI/uddi_auth_zone_cases.py`

**Test Coverage:** 497 sequential test cases covering UDDI synchronization between NIOS and CSP for authoritative DNS zones.

---

## Master Context Prompt

**Use this first to establish AI understanding of the project:**

```
You are a QA automation engineer working on UDDI (Universal DDI) integration testing between NIOS (Network Identity Operating System) and CSP (Cloud Services Portal). 

**Project Context:**
- Technology Stack: Python, pytest, Infoblox WAPI, REST APIs
- System Under Test: Bidirectional synchronization between NIOS and CSP for DNS authoritative zones
- Framework: Custom framework with ib_NIOS, dhcp_ti_utils modules
- Authentication: Uses SSH for NIOS access, API keys for CSP

**Key Components:**
1. NIOS: On-premise DDI appliance (versions 9.0.6, 9.0.7+)
2. CSP: Cloud Services Portal for centralized management
3. Orpheus: Docker container agent for sync between NIOS and CSP
4. DNS Records: A, AAAA, CNAME, NS, TXT, SRV, CAA, NAPTR, PTR, DNAME, MX, Alias

**Test Repository Structure:**
- Config file: Contains grid_vip, grid_fqdn, grid_member1_fqdn, CSP_USER, ep_url
- Utilities: ib_NIOS.wapi_request(), dhcp_ti_utils methods, SSH class
- Audit logging: Uses log_action() and log_validation() functions

**Version-specific behavior:**
- NIOS < 9.0.7: Uses "member_order":"SIMULTANEOUSLY" for restart
- NIOS >= 9.0.7: Uses "mode":"SIMULTANEOUS" for restart

Generate test cases following pytest conventions with proper ordering, logging, assertions, and sleep intervals for sync operations.
```

---

## Prompt 1: Infrastructure Setup and Validation

```
Generate pytest test cases for infrastructure setup and validation:

**Test Suite:** Infrastructure and Container Health
**Order:** Tests 1-3
**Requirements:**
1. Test DNS service startup on all NIOS members
2. Validate DNS service is enabled on all members
3. Validate Orpheus container health monitoring

**Implementation Details:**

Test 1 - Start DNS Service:
- Use ib_NIOS.wapi_request('GET', object_type="member:dns") to get all members
- For each member, set enable_dns=True using PUT request
- Assert response is not a tuple (indicates success)
- Log appropriate messages

Test 2 - Validate DNS Started:
- Get member:dns with _return_fields=enable_dns
- Assert enable_dns == True for all members
- Handle failure cases

Test 3 - Validate Orpheus Container Health:
- Create SSH connection to grid_vip using RSA key authentication
- Start continuous docker logs monitoring in background: "nohup docker logs --follow orpheus > orpheus.log 2>&1 &"
- Poll every 5 minutes (300 seconds) for up to 70 minutes (14 attempts)
- Search for "Ehs state: success" in orpheus.log using grep
- If found: cleanup (kill background process, remove log file), return True
- If not found after 14 attempts: cleanup, call delete_on_prem_host_from_csp(), sys.exit()
- Use the SSH class with send_command() method

**Expected Behavior:**
- All DNS services should enable successfully
- Orpheus must reach "success" state before proceeding
- If Orpheus fails, cleanup hosts from CSP and exit gracefully

**Dependencies:**
- config.grid_vip, config.grid_fqdn
- SSH class with RSA key authentication
- delete_on_prem_host_from_csp() helper function

Generate complete Python code with proper error handling and logging.
```

---

## Prompt 2: Zone Creation from NIOS and CSP Validation

```
Generate pytest test cases for authoritative zone creation from NIOS and validation in CSP:

**Test Suite:** NIOS to CSP Zone Synchronization
**Order:** Tests 4-43
**Scenario:** Create auth zone in NIOS, add all DNS record types, validate sync to CSP

**Phase 1: Zone Creation (Tests 4-15)**

Test 4 - Add Auth Zone with Primary Member from NIOS:
- Rename grid to config.grid_name first (use rename_grid() helper)
- Create zone_auth with fqdn='newzone.com', grid_primary=[{'name':config.grid_fqdn}]
- POST to zone_auth object
- Restart grid using restart_grid(config.build) - this handles version-specific restart logic
- Assert successful creation

Test 5-15 - Add DNS Records (one test per record type):
- A record: name='testing.newzone.com', ipv4addr='1.2.3.4'
- CNAME: name='cntest.newzone.com', canonical='help.com'
- NS: name='newzone.com', nameserver='ns.newzone.com', addresses=[{'address':'6.0.0.0'}]
- TXT: name='txt.newzone.com', text='text'
- SRV: name='srv.newzone.com', port=80, priority=20, target='test.newzone.com', weight=20
- CAA: name='caarec.newzone.com', ca_tag='issue', ca_flag=1, ca_value='one'
- NAPTR: name='nap.newzone.com', order=10, preference=10, replacement='tap'
- AAAA: name='aaaarec.newzone.com', ipv6addr='2001:db8:3333:4444:5555::'
- Alias: name='joy.newzone.com', target_name='abc', target_type='A'
- PTR: name='test_ptr_parvez.newzone.com', ptrdname='test_parvez.com'
- DNAME: name='test_dname_parvez.newzone.com', target='DNS_test'
- MX: name='test_mx_parvez.newzone.com', mail_exchanger='newzone.com', preference=10

After all records: sleep(450) for sync delay

**Phase 2: NIOS Validation (Tests 16-29)**
- Validate zone exists in NIOS (zone_auth GET)
- Validate grid_primary assignment
- For each record type: validate presence in NIOS, run dig command validation

Dig validation pattern:
```python
cmd_dig = f'dig @{config.grid_vip} <record_name> IN <RECORD_TYPE>'
res = subprocess.check_output(cmd_dig, shell=True).decode('utf-8')
assert re.search(r'.*QUERY, status: NOERROR.*', str(res))
assert re.search(r'.*newzone.com.*IN.*<TYPE>.*', str(res))
assert re.search(r'.*<expected_value>', str(res))
```

**Phase 3: CSP Validation (Tests 30-43)**
- Get DNS view: dns/view with name='default-'+config.grid_name
- Get auth_zone matching fqdn='newzone.com.' and view=dnsview_objectid
- Validate grid_primaries and grid_secondaries
- For each record: validate in dns/zone_child with parent filter
- Check absolute_name or protocol_absolute_name matches

**Sync Timing:**
- 450 seconds (7.5 min) after all records created
- Restart grid after zone creation

Generate complete test implementation with all validations, proper assertions, and error messages.
```

---

## Prompt 3: Update Operations from CSP with Audit Log Validation

```
Generate pytest test cases for update operations initiated from CSP with audit log validation:

**Test Suite:** CSP to NIOS Updates with Audit Tracking
**Order:** Tests 44-103
**Scenario:** Update zone and all records from CSP, capture audit logs, validate in NIOS

**Audit Log Pattern:**
- Start audit log: log("start", "/infoblox/var/audit.log", config.grid_vip)
- Perform operation
- Stop audit log: log("stop", "/infoblox/var/audit.log", config.grid_vip)
- Validate: logv(regex_pattern, "/infoblox/var/audit.log", config.grid_vip)
- Preprocessed CSP user: csp_user = preprocess_account(config.CSP_USER) - escapes '@' and '.'

**Test Pattern (repeat for each operation):**

Test N (odd) - Start Audit Log:
- Call log("start", "/infoblox/var/audit.log", config.grid_vip)

Test N+1 (even) - Perform Update from CSP:
1. Get dns/view for default-{config.grid_name}
2. Get dns/auth_zone for newzone.com
3. Get dns/host for primary/secondary hosts if needed
4. For zone update:
   - Update with comment='testing_zone', grid_secondaries=[{secondary_host}]
   - Use patch_object() with 'external_providers_metadata':{'ownership_type':'nios_ddi'}
5. For record update:
   - Get dns/zone_child with parent filter
   - Find specific record by absolute_name
   - Update comment='testing_record' and modify record-specific fields
   - Use patch_object()
6. sleep(10) after operation

Test N+2 - Stop Audit Log:
- Call log("stop", "/infoblox/var/audit.log", config.grid_vip)

Test N+3 - Validate Audit Log:
- Pattern: "\[federated\-user\]: \["+csp_user+"\] <Action> <RecordType> <name> DnsView=default..."
- Actions: Modified, Created, Deleted
- Include Changed fields in regex
- Assert True if logs found, False otherwise

Test N+4 - Validate in CSP:
- Query same object in CSP
- Verify changes persisted
- Check comment field exists and matches

Test N+5 - Validate in NIOS:
- Query object using WAPI
- Verify sync occurred
- restart_grid(config.build) after validation

**Update Operations to Cover:**
1. Zone: comment + grid_secondaries
2. A record: comment='testing_record', ipv4addr='1.2.3.5'
3. CNAME: comment='testing_record', canonical='newhelp.com'
4. NS: comment (read-only, should fail)
5. TXT: comment='testing_record', text='texthelp'
6. SRV: comment='testing_record', port=90, priority=10, weight=30
7. CAA: comment='testing_record', ca_value='testingone'
8. NAPTR: comment='testing', order=20, preference=20, replacement='test'
9. AAAA: comment='testing_record', ipv6addr='2001:db8:3303:4404:5556::'
10. PTR: comment='testing_record', ptrdname='test_record.com'
11. DNAME: comment='testing_record', target='DNAME_test'
12. MX: comment='testing_record', preference=30

**Audit Log Examples:**
- Zone: "Modified AuthZone newzone.com DnsView=default: Changed comment:.*testing_zone.*grid_secondaries:\[\]->\[\[grid_member=Member:.*\]\]"
- A Record: "Modified ARecord testing.newzone.com DnsView=default address=1.2.3.5.*Changed address:.*1.2.3.4.*->.*1.2.3.5.*comment:NULL->.*testing_record"

Generate all tests with complete audit log validation regex patterns specific to each record type and operation.
```

---

## Prompt 4: Record Updates from NIOS and Comment Deletion

```
Generate pytest test cases for record updates from NIOS and comment deletion operations:

**Test Suite:** NIOS Updates and Comment Management
**Order:** Tests 104-155
**Scenario:** Update records from NIOS, validate in CSP, delete comments, validate deletion

**Phase 1: Update Records from NIOS (Tests 104-117)**

For each record type, create update test:
- Get record using WAPI with specific name
- Extract _ref from response
- PUT update with comment='testing_record'
- Assert response is not tuple
- After each update: restart_grid(config.build)

Record types to update:
1. Alias: joy.newzone.com
2. (Validation tests follow for all previously added records)

**Phase 2: Validate Updates in NIOS (Tests 105-117)**

Pattern:
```python
data = {"name": "<record_name>"}
get_ref = ib_NIOS.wapi_request('GET', object_type="record:<type>?_return_fields=comment", fields=json.dumps(data))
res_new = json.loads(get_ref)
assert res_new[0]['comment'] == "testing_record"
```

Include dig validation where applicable

**Phase 3: Delete Comments from NIOS (Tests 118-130)**

Pattern:
```python
data = {"name": "<record_name>"}  # or fqdn for zone
get_ref = ib_NIOS.wapi_request('GET', object_type="<object_type>", fields=json.dumps(data))
ref_data = json.loads(get_ref)
mem = {"comment": ""}
ref = ref_data[0]["_ref"]
get_ref = ib_NIOS.wapi_request('PUT', object_type=ref, fields=json.dumps(mem))
assert not isinstance(json.loads(get_ref), tuple)
```

Objects to clear comments:
- Zone: newzone.com
- All record types (A, CNAME, TXT, SRV, CAA, NAPTR, AAAA, Alias, PTR, DNAME, MX)
- Note: NS record is read-only, validate update fails

After all deletions: sleep(300)

**Phase 4: Validate Comment Deletion in NIOS (Tests 131-155)**

Pattern with try-except for missing comment key:
```python
try:
    if res[0]["comment"] == "":
        assert True
    else:
        assert False, f"Expected comment to be empty, but got: {res[0]['comment']}"
except KeyError:
    assert True, "Response does not contain 'comment' key"
```

**Phase 5: Validate Comment Deletion in CSP (Tests 132-155, interleaved)**

- Query dns/zone_child for each record
- Check comment field is absent or empty
- Assert True if comment properly deleted

**NS Record Special Handling:**
- Test 122: Update NS record addresses from NIOS (should update)
- Test 141-142: Validate NS update synced to CSP
- Test 108: Validate comment update fails for NS (read-only)

Generate complete implementation with proper error handling for missing keys and read-only scenarios.
```

---

## Prompt 5: Record Deletion from CSP with Audit Trail

```
Generate pytest test cases for deleting DNS records from CSP with complete audit trail:

**Test Suite:** CSP Record Deletion with Audit Validation
**Order:** Tests 156-228
**Scenario:** Delete all DNS records from CSP, capture audit logs, validate in NIOS

**Test Pattern (for each record type):**

Test N - Start Audit Log

Test N+1 - Delete Record from CSP:
1. Get dns/view, dns/auth_zone
2. Get dns/zone_child with parent filter
3. Find record by absolute_name
4. Call dhcp_ti_utils.delete_object(ObjectClass=record_id, ...)
5. sleep(10)

Test N+2 - Stop Audit Log

Test N+3 - Validate Audit Log:
- Pattern: "\[federated\-user\]: \["+csp_user+"\] Deleted <RecordType> <name> DnsView=default..."
- Include record-specific details (e.g., "address=", "dname=", "target=")

Test N+4 - Validate Deletion in CSP:
- Query dns/zone_child again
- Assert record with matching absolute_name not found

Test N+5 - Validate Deletion in NIOS:
- GET record using WAPI
- Assert record not in response
- restart_grid(config.build)

**Records to Delete (in order):**
1. A record (testing.newzone.com) - Tests 156-160
2. CNAME (cntest.newzone.com) - Tests 161-165
3. NS (ns.newzone.com) - Tests 164-167
4. TXT (txt.newzone.com) - Tests 168-171
5. SRV (srv.newzone.com) - Tests 172-175
6. CAA (caarec.newzone.com) - Tests 176-179
7. NAPTR (nap.newzone.com) - Tests 180-183
8. AAAA (aaaarec.newzone.com) - Tests 184-187
9. PTR (test_ptr_parvez.newzone.com) - Tests 188-191
10. DNAME (test_dname_parvez.newzone.com) - Tests 192-195
11. MX (test_mx_parvez.newzone.com) - Tests 196-199

**Audit Log Examples:**
- A: "Deleted ARecord testing.newzone.com DnsView=default.*"
- CNAME: "Deleted CnameRecord cntest.newzone.com DnsView=default"
- NS: "Deleted NsRecord newzone.com DnsView=default dname=ns.newzone.com"
- PTR: "Deleted PtrRecord test_ptr_parvez.newzone.com DnsView=default dname=test_record.com"

**Additional Tests (200-228):**
- Add new NS record (test.newzone.com) in NIOS
- Validate addition
- Delete NS record from NIOS
- Validate deletion in NIOS and CSP
- Delete Alias record (joy.newzone.com) from NIOS
- Validate all deletion syncs

**Zone Deletion (Tests 229-231):**
Test 229 - Delete zone from NIOS
Test 230 - Validate zone deletion in NIOS
Test 231 - Validate zone deletion in CSP (sleep(300) before validation)

Generate all tests with proper CSP API calls, audit log patterns, and NIOS validation.
```

---

## Prompt 6: Zone and Record Creation from CSP

```
Generate pytest test cases for creating authoritative zone and records from CSP:

**Test Suite:** CSP to NIOS Zone Creation
**Order:** Tests 232-287
**Scenario:** Create zone from CSP, add all record types, validate sync to NIOS

**Phase 1: Zone Creation from CSP (Tests 232-237)**

Test 232 - Start Audit Log

Test 233 - Add Auth Zone with Member from CSP:
1. Get dns/view for 'default-'+config.grid_name
2. Get dns/host for grid primary
3. Create zone using POST to dns/auth_zone:
   ```python
   mem = {
       'fqdn': 'newzonetest.com',
       "grid_primaries": [{"host": primary_host_objectid}],
       "disabled": False,
       "view": dnsview_objectid,
       'external_providers_metadata': {'ownership_type': 'nios_ddi', "sync_read_only": False},
       'primary_type': 'cloud'
   }
   ```
4. sleep(10)

Test 234 - Stop Audit Log

Test 235 - Validate Audit Log:
- Pattern: "\[federated\-user\]: \["+csp_user+"\] Created AuthZone newzonetest.com DnsView=default:.*fqdn=.*newzonetest.com.*grid_primaries=\[\[grid_member=Member:"+config.grid_fqdn+".*\]\]"

Test 236 - Validate Zone in CSP
Test 237 - Validate Zone with Primary Member in CSP

**Phase 2: Add Records from CSP (Tests 238-287)**

For each record type, create test group:

Test N - Start Audit Log

Test N+1 - Add Record from CSP:
1. Get dns/view and dns/auth_zone
2. Get zone_ref for newzonetest.com
3. POST to dns/record with:
   ```python
   mem = {
       "name_in_zone": "<record_name>",
       "rdata": {<record_specific_data>},
       "type": "<RECORD_TYPE>",
       "zone": zone_ref
   }
   ```
4. sleep(10)

Test N+2 - Stop Audit Log

Test N+3 - Validate Audit Log:
- Pattern specific to record type created

Test N+4 - Validate Record in CSP (dns/zone_child query)

**Records to Create:**
1. A: name_in_zone='testrecord', address='1.2.3.6'
2. TXT: name_in_zone='txt', text='text'
3. CNAME: name_in_zone='cname', cname='test'
4. SRV: name_in_zone='srv', port=80, priority=20, target='test', weight=20
5. AAAA: name_in_zone='kate', address='2001:db8:3333:4444:5555::'
6. NAPTR: name_in_zone='truce', order=10, preference=10, replacement='rec'
7. CAA: name_in_zone='test', flags=64, tag='issue', value='two'
8. MX: name_in_zone='test_mx_parvez', exchange='newzonetest.com', preference=10
9. DNAME: name_in_zone='test_dname_parvez', target='test'
10. PTR: name_in_zone='test_ptr_parvez', dname='test_parvez.com.'

**Audit Log Examples:**
- A: "Created ARecord testrecord.newzonetest.com DnsView=default address=1.2.3.6:*"
- TXT: "Created TxtRecord txt.newzonetest.com view=default text=text:*"
- MX: "Created MxRecord test_mx_parvez.newzonetest.com DnsView=default exchanger=newzonetest.com.newzonetest.com:.*"

**Phase 3: Validate in NIOS (Tests 288-321, interleaved)**

After each record creation, validate:
1. Record exists in NIOS (WAPI GET)
2. Run dig command validation
3. restart_grid(config.build)

Generate complete implementation with proper CSP record creation API calls and NIOS validation including dig commands.
```

---

## Prompt 7: NIOS Updates and CSP Comment Deletion

```
Generate pytest test cases for updating records from NIOS and deleting comments from CSP:

**Test Suite:** NIOS Record Updates and CSP Comment Management
**Order:** Tests 288-378
**Scenario:** Add comments to records in NIOS, validate in CSP, delete comments from CSP, validate in NIOS

**Phase 1: Update Zone and Records from NIOS (Tests 288-321)**

Test 288-289 - Validate Zone in NIOS:
- Validate newzonetest.com exists
- Validate grid_primary assignment
- restart_grid(config.build)

Test 290-291 - Update Zone with Comment from NIOS:
- Add comment='testing_zone'
- Add grid_secondaries=[{"name": config.grid_member1_fqdn, "stealth": False}]
- Validate update in NIOS
- restart_grid(config.build)

For each record type (Tests 292-321):
1. Validate record exists in NIOS
2. Run dig validation
3. Update with comment='testing_record'
4. Validate comment added
5. restart_grid(config.build)

Records: testrecord (A), txt (TXT), cname (CNAME), srv (SRV), kate (AAAA), truce (NAPTR), test (CAA), test_mx_parvez (MX), test_dname_parvez (DNAME), test_ptr_parvez (PTR)

sleep(300) after all updates

**Phase 2: Validate Updates in CSP (Tests 322-378)**

Test 322 - Validate Zone Comment in CSP:
- Query dns/auth_zone
- Verify comment='testing_zone' and grid_secondaries present

For each record (Tests 323-378):

Test N - Validate Comment in CSP:
- Get dns/zone_child
- Verify comment='testing_record' exists

Test N+1 - Start Audit Log

Test N+2 - Delete Comment from CSP:
- Get record from dns/zone_child
- PATCH with comment='' and 'external_providers_metadata':{'ownership_type':'nios_ddi'}
- sleep(10)

Test N+3 - Stop Audit Log

Test N+4 - Validate Audit Log:
- Pattern: "\[federated\-user\]: \["+csp_user+"\] Modified <RecordType> <name> DnsView=default.*"

Test N+5 - Validate Comment Deleted in CSP:
- Query record again
- Assert comment field empty or absent

**Comment Deletion Operations:**
1. Zone: comment='' (Tests 323-327)
2. A record (testrecord) - Tests 328-333
3. TXT (txt) - Tests 334-339
4. CNAME (cname) - Tests 340-345
5. SRV (srv) - Tests 346-351
6. AAAA (kate) - Tests 352-354
7. NAPTR (truce) - Tests 355-357
8. CAA (test) - Tests 358-360
9. MX (test_mx_parvez) - Tests 361-366
10. DNAME (test_dname_parvez) - Tests 367-372
11. PTR (test_ptr_parvez) - Tests 373-378

sleep(20) after all deletions

**Phase 3: Validate Comment Deletion in NIOS (Tests 379-425)**

For each object:
- GET from NIOS with _return_fields=comment
- Use try-except to handle missing comment key:
  ```python
  try:
      if res[0]["comment"] == "":
          assert True
      else:
          assert False
  except KeyError:
      assert True  # Comment key doesn't exist
  ```

Delete all records from NIOS and validate deletion in both NIOS and CSP

Generate complete implementation with all CSP PATCH operations for comment deletion and NIOS validation.
```

---

## Prompt 8: Complete Record Deletion and Zone Cleanup

```
Generate pytest test cases for deleting all records from NIOS and zone cleanup:

**Test Suite:** Record Deletion and Cleanup
**Order:** Tests 379-425
**Scenario:** Validate comment deletions, delete all records from NIOS, delete zone from CSP

**Phase 1: Validate Comment Deletions in NIOS (Tests 379-409)**

For each object (zone and all records):
```python
data = {"name": "<record_name>"}  # or {"fqdn": "..."} for zone
get_req = ib_NIOS.wapi_request('GET', object_type="<object_type>?_return_fields=comment", fields=json.dumps(data))
res = json.loads(get_req)

try:
    if res[0]["comment"] == "":
        assert True
    else:
        assert False, f"Expected empty comment, got: {res[0]['comment']}"
except KeyError:
    assert True  # Comment doesn't exist
```

Objects to validate:
- Zone (newzonetest.com)
- A, TXT, CNAME, SRV, AAAA, NAPTR, CAA, MX, DNAME, PTR records

**Phase 2: Delete Records from NIOS (Tests 380-409, interleaved)**

For each record:
1. GET record to find _ref
2. DELETE using _ref
3. Validate deletion in NIOS (record not in GET response)
4. sleep(300) after all deletions

**Phase 3: Validate Deletions in CSP (Tests 410-419)**

For each deleted record:
- Get dns/view and dns/auth_zone
- Query dns/zone_child with parent filter
- Assert record with specific absolute_name not found

Records to validate deletion:
- testrecord.newzonetest.com (A)
- txt.newzonetest.com (TXT)
- cname.newzonetest.com (CNAME)
- srv.newzonetest.com (SRV)
- kate.newzonetest.com (AAAA)
- truce.newzonetest.com (NAPTR)
- test.newzonetest.com (CAA)
- test_mx_parvez.newzonetest.com (MX)
- test_dname_parvez.newzonetest.com (DNAME)
- test_ptr_parvez.newzonetest.com (PTR)

**Phase 4: Delete Zone from CSP (Tests 420-425)**

Test 420 - Start Audit Log

Test 421 - Delete Auth Zone from CSP:
1. Get dns/view
2. Get dns/auth_zone for newzonetest.com
3. Call dhcp_ti_utils.delete_object(ObjectClass=authzone_objectid, ...)
4. sleep(10)

Test 422 - Stop Audit Log

Test 423 - Validate Audit Log:
- Pattern: "\[federated\-user\]: \["+csp_user+"\] Deleted AuthZone newzonetest.com DnsView=default exclude_subobj=False"

Test 424 - Validate Zone Deleted in CSP:
- Query dns/auth_zone
- Assert newzonetest.com not found

Test 425 - Validate Zone Deleted in NIOS:
- GET zone_auth
- Assert newzonetest.com not in response

Generate complete deletion workflow with proper validation in both systems.
```

---

## Prompt 9: Read-Only User Validation Tests

```
Generate pytest test cases for validating read-only user permissions after federation disable/enable:

**Test Suite:** Read-Only User Permission Validation
**Order:** Tests 426-480
**Scenario:** Create zone/records from NIOS, disable federation, validate read-only mode in CSP, re-enable federation, validate write mode

**Phase 1: Create Zone and All Records from NIOS (Tests 426-458)**

Test 426 - Create Zone newzone.com with primary member
Test 427-436 - Add all record types:
- A (test_a_parvez)
- AAAA (test_aaaa_parvez)
- CAA (test_caa_parvez)
- CNAME (test_cname_parvez)
- DNAME (test_dname_parvez)
- MX (test_mx_parvez)
- NAPTR (test_naptr_parvez)
- PTR (test_ptr_parvez)
- SRV (test_srv_parvez)
- TXT (test_txt_parvez)

After all records: restart_grid(), sleep(300)

Tests 437-458 - Validate in NIOS and CSP:
- Confirm all records exist in NIOS
- Confirm all records synced to CSP

**Phase 2: Disable Federation (Tests 459-460)**

Test 459 - Disable Federation:
```python
get_refs = ib_NIOS.wapi_request('GET', object_type="grid")
res = json.loads(get_refs)
for ref in res:
    r = ref["_ref"]
    data = {'enable_federation': False}
    response = ib_NIOS.wapi_request('PUT', ref=r, fields=json.dumps(data))
sleep(400)  # Wait for federation to fully disable
```

Test 460 - Validate Federation Disabled:
- GET grid?_return_fields=enable_federation
- Assert enable_federation == False

**Phase 3: Validate Read-Only Mode in CSP (Tests 461-470)**

For each record type:
1. Get dns/view, dns/auth_zone
2. Get dns/zone_child for specific record
3. Attempt to PATCH with comment='testing_read_only'
4. Expected result: Operation should fail (403 Forbidden or 404 Not Found)
5. Validate record remains unchanged

Records to test:
- A, AAAA, CAA, CNAME, DNAME, MX, NAPTR, PTR, SRV, TXT

**Phase 4: Re-enable Federation (Tests 471-472)**

Test 471 - Enable Federation:
```python
data = {'enable_federation': True}
response = ib_NIOS.wapi_request('PUT', ref=r, fields=json.dumps(data))
sleep(450)
```

Test 472 - Validate Federation Enabled:
- Assert enable_federation == True

**Phase 5: Validate Write Mode Restored (Tests 473-478)**

Test 473 - Start Audit Log

Test 474 - Update A Record from CSP:
- PATCH test_a_parvez with comment='testing_read_only'
- Should succeed now

Test 475 - Stop Audit Log

Test 476 - Validate Audit Log:
- Pattern: "\[federated\-user\]: \["+csp_user+"\] Modified ARecord test_a_parvez.newzone.com DnsView=default address=10.0.0.1:.*"

Test 477 - Validate Update in CSP
Test 478 - Validate Update in NIOS:
- Verify comment='testing_read_only' synced

**Phase 6: Cleanup (Tests 479-480)**

Test 479 - Delete Zone from NIOS
Test 480 - Validate Zone Deleted
- restart_grid(config.build)

Generate complete read-only validation tests showing permission enforcement when federation is disabled.
```

---

## Prompt 10: Member Assignment and Configuration Tests

```
Generate pytest test cases for grid member assignment operations:

**Test Suite:** Member Assignment and Configuration
**Order:** Tests 481-497
**Scenario:** Test grid primary, grid secondary, and external secondary assignments

**Phase 1: Zone Creation and Secondary Assignment (Tests 481-484)**

Test 481 - Create Zone newzone.com:
- POST zone_auth with grid_primary=[{'name': config.grid_fqdn}]

Test 482 - Update Secondary Member:
```python
get_ref = ib_NIOS.wapi_request('GET', object_type="zone_auth")
# Find newzone.com
mem = {"grid_secondaries": [{"name": config.grid_member1_fqdn}]}
resp = ib_NIOS.wapi_request('PUT', object_type="zone_auth", ref=reference, fields=json.dumps(mem))
sleep(350)
restart_grid(config.build)
```

Test 483 - Validate Primary and Secondary in CSP:
- Get dns/auth_zone
- Verify grid_primaries contains primary_host_objectid
- Verify grid_secondaries contains secondary_host_objectid

Test 484 - Validate in NIOS:
- GET zone_auth?_return_fields=fqdn,grid_primary,grid_secondaries
- Assert grid_primary=[{'name': config.grid_fqdn, "stealth": False}]
- Assert grid_secondaries[0]["name"] == config.grid_member1_fqdn

**Phase 2: Duplicate Secondary Test (Test 485)**

Test 485 - Attempt to Add Duplicate Secondary from CSP:
```python
mem = {
    "grid_primaries": [{"host": primary_host_objectid}],
    "grid_secondaries": [
        {"host": primary_host_objectid},  # Same as primary - should fail
        {"host": secondary_host_objectid}
    ],
    'external_providers_metadata': {'ownership_type': 'nios_ddi'}
}
dns_code, dns_response = dhcp_ti_utils.patch_object(...)
# Expect this to fail or be rejected
```

**Phase 3: Remove Secondary (Tests 486-487)**

Test 486 - Delete Grid Secondary from NIOS:
```python
mem = {"grid_secondaries": []}
get_ref = ib_NIOS.wapi_request('PUT', object_type=ref, fields=json.dumps(mem))
```

Test 487 - Validate Secondary Removed:
- GET zone_auth?_return_fields=fqdn,grid_secondaries
- Assert grid_secondaries == [] or is empty

**Phase 4: External Secondary Tests (Tests 488-494)**

Test 488 - Add External Secondary from NIOS:
```python
mem = {
    'grid_primary': [{'name': config.grid_fqdn}],
    "external_secondaries": [{"name": "test.com", "address": "12.0.0.0"}]
}
resp = ib_NIOS.wapi_request('PUT', object_type="zone_auth", ref=reference, fields=json.dumps(mem))
restart_grid(config.build)
sleep(300)
```

Test 489 - Validate External Secondary in NIOS:
- GET with _return_fields=external_secondaries
- Assert external_secondaries[0]["name"] == "test.com"
- Assert external_secondaries[0]["address"] == "12.0.0.0"

Test 490 - Validate Grid Secondary Deletion in CSP:
- Query dns/auth_zone
- Assert grid_secondaries array is empty or absent

Test 491 - Validate External Secondary in CSP:
- Query dns/auth_zone
- Verify external_secondaries contains {"address": "12.0.0.0", "fqdn": "test.com"}

Test 492 - Delete External Secondary from CSP:
```python
mem = {
    'grid_primaries': [{"host": primary_host_objectid}],
    "external_secondaries": [],
    'external_providers_metadata': {'ownership_type': 'nios_ddi'}
}
dhcp_ti_utils.patch_object(...)
```

Test 493 - Validate Deletion in CSP
Test 494 - Validate Deletion in NIOS:
- Assert external_secondaries == []

**Phase 5: Cleanup (Tests 495-497)**

Test 495 - Delete Zone from NIOS
Test 496 - Validate Zone Deleted in NIOS
Test 497 - Validate Zone Deleted in CSP

Generate complete member assignment tests including grid members and external secondaries with proper validation.
```

---

## Prompt 11: Helper Functions and Utilities

```
Generate helper functions and utility code for the test suite:

**Required Helper Functions:**

1. restart_grid(version):
```python
def restart_grid(version):
    """
    Restart grid services with version-specific handling
    Args:
        version: Build version string (e.g., "9.0.7", "NIOS_9.0.7_53718")
    
    Logic:
    - Parse version (handle NIOS_X.Y.Z_BUILD format)
    - If version >= 9.0.7: use restart_grid_907()
    - Else: use restart_grid_below_907()
    """
    # Extract version from NIOS_ format if present
    # Compare major.minor.patch
    # Call appropriate restart function
```

2. restart_grid_below_907():
```python
def restart_grid_below_907():
    grid = ib_NIOS.wapi_request('GET', object_type="grid", grid_vip=config.grid_vip)
    ref = json.loads(grid)[0]['_ref']
    data = {
        "member_order": "SIMULTANEOUSLY",
        "restart_option": "FORCE_RESTART",
        "service_option": "ALL"
    }
    request_restart = ib_NIOS.wapi_request('POST', 
        object_type=ref + "?_function=restartservices",
        fields=json.dumps(data),
        grid_vip=config.grid_vip)
    sleep(15)
```

3. restart_grid_907():
```python
def restart_grid_907():
    # Same as above but with different data format:
    data = {
        "mode": "SIMULTANEOUS",
        "restart_option": "FORCE_RESTART",
        "services": ["ALL"]
    }
```

4. preprocess_account(email):
```python
def preprocess_account(email):
    """Escape special characters for regex matching in audit logs"""
    preprocessed = email.replace('@', '\@').replace('.', '\.')
    return preprocessed
```

5. SSH Class:
```python
class SSH:
    """SSH connection manager using paramiko"""
    client = None
    
    def __init__(self, address):
        # Initialize SSH client
        # Use RSA key from ~/.ssh/id_rsa
        # Connect to address
        
    def send_command(self, command):
        # Execute command
        # Return stdout.readlines()
```

6. validate_orpheus_container_in_success_state():
```python
def validate_orpheus_container_in_success_state():
    """
    Monitor Orpheus container until it reaches success state
    - Start docker logs in background
    - Poll every 5 minutes for up to 70 minutes
    - Search for "Ehs state: success"
    - Cleanup log files and processes
    - Exit and cleanup hosts if not successful
    """
```

7. delete_on_prem_host_from_csp():
```python
def delete_on_prem_host_from_csp():
    """
    Remove all grid members from CSP
    - Get all member VIP addresses
    - Get all member management IPs
    - Delete each from CSP using dhcp_ti_utils.delete_host()
    """
```

8. rename_grid():
```python
def rename_grid():
    """Rename grid to config.grid_name"""
    get_refs = ib_NIOS.wapi_request('GET', object_type="grid")
    res = json.loads(get_refs)
    for ref in res:
        r = ref["_ref"]
        data = {'name': config.grid_name}
        response = ib_NIOS.wapi_request('PUT', ref=r, fields=json.dumps(data))
    sleep(300)
```

Generate all helper functions with proper error handling, logging, and documentation.
```

---

## Test Case Analysis Summary

### 1. Purpose:
Validate **UDDI (Universal DDI) synchronization** between **NIOS (on-premise)** and **CSP (Cloud Services Portal)** for authoritative DNS zones and their DNS records. Ensure bidirectional data consistency when creating, updating, and deleting DNS zones and various record types.

### 2. Pre-requisites:
- **NIOS Grid** configured and operational (version detection for 9.0.7+ compatibility)
- **CSP (Cloud Services Portal)** integration enabled
- **Orpheus container** running in success state (EHS state validation)
- **SSH access** to the grid master with RSA key authentication
- **DNS services** enabled on grid members
- **Audit logging** capability enabled
- **Grid name** configured (uses `config.grid_name`)
- **CSP user credentials** configured (uses `config.CSP_USER`)

### 3. Steps to Reproduce:

#### Phase 1: NIOS → CSP Synchronization (Tests 1-43)
- Start DNS services on NIOS
- Create authoritative zone with primary member on NIOS
- Add various DNS records (A, AAAA, CNAME, NS, TXT, SRV, CAA, NAPTR, PTR, DNAME, MX, Alias)
- Validate records appear in both NIOS and CSP

#### Phase 2: Update Operations from CSP (Tests 44-103)
- Update zone properties (comments, grid secondaries) from CSP
- Update each record type with comments from CSP
- Validate audit logs for each operation
- Verify changes reflect in both NIOS and CSP

#### Phase 3: Update Validation from NIOS (Tests 104-130)
- Update records from NIOS side
- Delete comments from records in NIOS
- Validate synchronization to CSP

#### Phase 4: Delete Comment Validation (Tests 131-155)
- Verify comment deletions in both NIOS and CSP

#### Phase 5: Delete Records from CSP (Tests 156-228)
- Delete individual records from CSP
- Capture and validate audit logs
- Verify deletions sync to NIOS
- Delete entire zone and validate removal

#### Phase 6: CSP → NIOS Synchronization (Tests 232-287)
- Create zone from CSP
- Add records from CSP
- Validate in both NIOS and CSP
- Capture audit logs for all operations

#### Phase 7: NIOS Updates & CSP Validation (Tests 288-321)
- Add and update records from NIOS
- Validate updates appear in CSP

#### Phase 8: CSP Updates & Deletions (Tests 322-425)
- Update records from CSP
- Delete comments from CSP
- Delete all records from NIOS
- Validate deletions in CSP
- Delete zone and validate complete removal

#### Phase 9: Read-Only User Validation (Tests 426-480)
- Create zone and all record types from NIOS
- Attempt updates/deletes from CSP with read-only user
- Verify operations fail with appropriate error (403/404)

#### Phase 10: Member Assignment Tests (Tests 481-497)
- Test zone member assignment changes
- Test grid member secondary configurations
- Validate member assignment updates

### 4. Expected Outcome:
- All CRUD operations (Create, Read, Update, Delete) on authoritative zones **sync bidirectionally** between NIOS and CSP
- **Audit logs** capture all modification events with correct user attribution (CSP user email)
- All DNS record types supported: **A, AAAA, CNAME, NS, TXT, SRV, CAA, NAPTR, PTR, DNAME, MX, Alias**
- **Read-only users** cannot perform write operations (403 Forbidden)
- Zone and record **metadata** (comments, member assignments) sync correctly
- **Orpheus container** maintains success state throughout testing
- Changes propagate within expected timeframes
- **HTTP 200** status codes for successful operations

### 5. Category of Test:
- **Functional Testing**: Core CRUD operations and synchronization
- **Integration Testing**: NIOS ↔ CSP bidirectional sync
- **Security Testing**: Read-only user access control (Tests 426-480)
- **Audit Testing**: Log validation for compliance tracking
- **Regression Testing**: Comprehensive record type coverage
- **State Validation**: Container health monitoring

### 6. Server Detail:
- **Primary Platform**: **NIOS (Network Identity Operating System)**
- **Version Support**: NIOS 9.0.6 and below, **NIOS 9.0.7+** (with version-specific restart logic)
- **Integration**: **CSP (Cloud Services Portal)** - Infoblox's cloud management platform
- **Architecture**: Hybrid deployment (on-premise NIOS + cloud CSP)
- **Container**: **Orpheus** - synchronization agent running in Docker on NIOS
- **Grid Configuration**: Master-member topology with HA support
- **API**: WAPI (Web API) for NIOS operations, CSP REST APIs for cloud operations

---

## Usage Instructions for AI Code Generation

### Step 1: Establish Context
Start with the **Master Context Prompt** to provide AI with complete understanding of:
- Project structure and technology stack
- System architecture (NIOS, CSP, Orpheus)
- Available utilities and frameworks
- Version-specific behaviors

### Step 2: Sequential Prompt Usage
Use prompts 1-11 in order, as each builds upon previous tests:
1. **Prompt 1**: Infrastructure setup and container health
2. **Prompt 2**: Zone creation from NIOS → CSP validation
3. **Prompt 3**: Updates from CSP with audit logs
4. **Prompt 4**: Updates from NIOS and comment deletion
5. **Prompt 5**: Record deletion from CSP with audit trail
6. **Prompt 6**: Zone and record creation from CSP
7. **Prompt 7**: NIOS updates and CSP comment deletion
8. **Prompt 8**: Complete record deletion and cleanup
9. **Prompt 9**: Read-only user permission validation
10. **Prompt 10**: Member assignment operations
11. **Prompt 11**: Helper functions and utilities

### Step 3: Customization
Replace configuration placeholders with actual values:
- `config.grid_vip` → Actual grid VIP address
- `config.grid_fqdn` → Grid FQDN
- `config.grid_member1_fqdn` → Member FQDN
- `config.CSP_USER` → CSP user email
- `config.ep_url` → CSP endpoint URL
- `config.grid_name` → Grid name
- `config.build` → NIOS version

### Step 4: Code Validation
Review generated code for:
- ✓ Proper pytest decorators: `@pytest.mark.run(order=N)`
- ✓ Correct assertions and error handling
- ✓ Appropriate sleep intervals (sync timing)
- ✓ Valid audit log regex patterns
- ✓ Proper cleanup in teardown phases
- ✓ Version-specific logic handling

### Step 5: Execution Guidelines
- **Run tests sequentially** - Order dependencies exist
- **Monitor Orpheus health** - Must be in success state
- **Check audit logs** - Validate user attribution
- **Verify sync timing** - Allow adequate propagation time
- **Handle cleanup** - Ensure proper resource removal

---

## AI Contribution Metrics

To measure AI vs Engineer contribution as per requirements:

### Tracking Metrics:
1. **Lines of Code Generated by AI**: Count automated test generation
2. **Manual Code Modifications**: Track engineer edits post-generation
3. **Bug Fixes by AI**: Issues resolved through AI suggestions
4. **Time Saved**: Compare manual vs AI-assisted development time
5. **Test Coverage**: AI-generated vs manually written test cases

### Success Criteria:
- **>70% code generation** by AI prompts
- **<30% manual intervention** for corrections
- **100% test execution success** after AI generation
- **Reduced development time** by 50%+

---

## Reference Repository Structure

```
qa-uddi-api-tests-sn/
├── WAPI_PyTest/
│   ├── suites/
│   │   └── RFE_UDDI/
│   │       ├── uddi_auth_zone_cases.py (497 tests)
│   │       └── uddi_IPAM_automation_part1.py
│   ├── ib_utils/
│   │   ├── start_stop_logs.py (log_action)
│   │   └── file_content_validation.py (log_validation)
│   └── common/
│       ├── ib_NIOS.py (wapi_request)
│       └── dhcp_ti_utils.py (CSP API methods)
└── config.py (grid configuration)
```

---

## Conclusion

This guide provides comprehensive, AI-ready test case prompts that:

1. ✅ **Enable AI execution** - Each prompt contains complete context
2. ✅ **Require no prior knowledge** - Full implementation details provided
3. ✅ **Reference existing code** - Points to repository structure
4. ✅ **Ensure reproducibility** - Clear steps and expected outcomes
5. ✅ **Support metrics tracking** - AI vs Engineer contribution measurable

**Total Coverage**: 497 test cases across 11 functional areas with complete audit trail validation and bidirectional synchronization testing.
