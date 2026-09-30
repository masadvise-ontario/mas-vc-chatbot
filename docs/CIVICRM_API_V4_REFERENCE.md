# CiviCRM API v4 Reference for VC Chatbot

**Created**: 2026-01-21
**Purpose**: Document API v4 patterns discovered during VC chatbot development
**Environment**: masadvise.org (production), masdemo.localhost (dev)

---

## Authentication

### Required Headers

```bash
X-Civi-Auth: Bearer <api_key>
X-Civi-Key: <site_key>
Content-Type: application/x-www-form-urlencoded
```

**Critical:**
- Both `X-Civi-Auth` AND `X-Civi-Key` required for non-admin users
- Admin users can use just `X-Civi-Auth` (but including both is safer)

### API Keys

**Production:**
- Site Key: Stored in n8n "CiviCRM Custom Auth" credential
- User API Keys: Stored in n8n "CiviCRM Custom Auth" credential

**Dev (masdemo.localhost):**
- Generate via CiviCRM → Contacts → API Key tab

---

## API Endpoint

### Correct URL Pattern
```
https://www.masadvise.org/civicrm/ajax/api4/{Entity}/{Action}
```

**Common Entities:**
- `Contact`
- `Case` (note: entity is "Case" but in PHP it's CiviCase)
- `Relationship`
- `RelationshipCache` (preferred for queries)
- `Activity`

**Common Actions:**
- `get` - Retrieve records
- `getFields` - Get field metadata
- `create` - Create new record
- `update` - Update existing record
- `delete` - Delete record

### ❌ Wrong URLs (Don't Use)
```
❌ https://www.masadvise.org/civicrm/api4/Contact/get (missing /ajax/)
❌ https://www.masadvise.org//civicrm/ajax/api4/Contact/get (double slash)
❌ https://www.masadvise.org/civicrm/rest/api4/Contact/get (wrong path)
```

---

## Request Format

### Body Structure

**Content-Type:** `application/x-www-form-urlencoded`

**Parameter Name:** `params` (exactly)

**Value:** URL-encoded JSON string

### curl Example
```bash
curl --location 'https://www.masadvise.org/civicrm/ajax/api4/Contact/get' \
--header 'X-Civi-Auth: Bearer <api_key>' \
--header 'X-Civi-Key: <site_key>' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'params={
  "select": ["id", "display_name", "email_primary.email"],
  "where": [["id", "=", 123]],
  "limit": 1
}'
```

### n8n HTTP Request Node
```javascript
// Body Parameters
{
  "name": "params",
  "value": "={{ JSON.stringify({
    select: ['id', 'display_name'],
    where: [['id', '=', $json.contact_id]],
    limit: 1
  }) }}"
}
```

---

## Query Structure

### Basic Query
```json
{
  "select": ["field1", "field2"],
  "where": [["field", "operator", "value"]],
  "limit": 25
}
```

### With Joins
```json
{
  "select": [
    "id",
    "subject",
    "client.sort_name",
    "vc.near_contact_id.sort_name"
  ],
  "where": [["id", "=", 1]],
  "join": [
    ["Contact AS client", "LEFT", "CaseContact", ["id", "=", "client.case_id"]],
    ["Contact AS vc", "LEFT", "RelationshipCache",
      ["client.id", "=", "vc.far_contact_id"],
      ["vc.near_relation:name", "=", "\"Case Coordinator is\""],
      ["vc.case_id", "=", "id"]
    ]
  ],
  "limit": 1
}
```

### Complex WHERE Clauses
```json
{
  "where": [
    ["OR", [
      ["display_name", "LIKE", "%Smith%"],
      ["email_primary.email", "LIKE", "%smith@%"]
    ]],
    ["AND", [
      ["contact_type:name", "=", "Individual"],
      ["is_deleted", "=", false]
    ]]
  ]
}
```

---

## Relationship Patterns

### Use RelationshipCache (Not Relationship)

**Why?**
- Faster for queries (pre-computed bidirectional data)
- Better for filtering
- Includes both directions of relationship

### Key Fields

| Field | Description | Example |
|-------|-------------|---------|
| `near_contact_id` | One side of relationship | VC (Case Coordinator) |
| `far_contact_id` | Other side of relationship | Client contact |
| `near_relation:name` | Relationship type (near side) | "Case Coordinator is" |
| `far_relation:name` | Relationship type (far side) | "Case Coordinator" |
| `case_id` | Linked case (if applicable) | 123 |

### Join Pattern for Cases with VC

```json
{
  "join": [
    [
      "Contact AS client",
      "LEFT",
      "CaseContact",
      ["id", "=", "client.case_id"]
    ],
    [
      "Contact AS vc",
      "LEFT",
      "RelationshipCache",
      ["client.id", "=", "vc.far_contact_id"],
      ["vc.near_relation:name", "=", "\"Case Coordinator is\""],
      ["vc.case_id", "=", "id"]
    ]
  ]
}
```

**Access VC Data:**
- `vc.near_contact_id` - VC contact ID
- `vc.near_contact_id.sort_name` - VC name
- `vc.near_contact_id.email_primary.email` - VC email

**Access Client Data:**
- `client.id` - Client contact ID
- `client.sort_name` - Client name
- `client.employer_id.display_name` - Client organization

---

## Custom Fields

### Naming Convention
```
{CustomGroupName}.{CustomFieldName}
```

### MAS Case Custom Fields (Cases_SR_Projects_)

| Field Name | Type | Example |
|------------|------|---------|
| `Cases_SR_Projects_.MAS_SR_Case_Code` | Text | "SR-2024-001" |
| `Cases_SR_Projects_.Notes` | Textarea | Internal notes |
| `Cases_SR_Projects_.Practice_Area:label` | Option Value | "Finance" |
| `Cases_SR_Projects_.Requested_Start_Date` | Date | "2024-01-15" |
| `Cases_SR_Projects_.Referral:label` | Option Value | "Website" |
| `Cases_SR_Projects_.Virtual_Work:label` | Option Value | "Yes" |

### Using `:label` Suffix
- For option value fields, use `:label` to get the display label
- Without `:label`, you get the numeric option value
- Example: `Practice_Area:label` → "Finance" vs `Practice_Area` → "3"

---

## Contact Queries

### Get Active VCs
```json
{
  "select": [
    "id",
    "display_name",
    "email_primary.email",
    "employer_id.display_name"
  ],
  "where": [
    ["contact_sub_type:name", "CONTAINS", "MAS_Rep"],
    ["MAS_Rep.VC_Status:name", "=", "Active"]
  ]
}
```

### Search by Name or Email
```json
{
  "select": ["id", "display_name", "email_primary.email"],
  "where": [
    ["OR", [
      ["display_name", "LIKE", "%Smith%"],
      ["email_primary.email", "LIKE", "%smith%"]
    ]]
  ]
}
```

### Get Organization Employees
```json
{
  "select": [
    "id",
    "display_name",
    "email_primary.email",
    "employer_id.display_name"
  ],
  "where": [
    ["employer_id", "=", 456]
  ]
}
```

---

## Case Queries

### Get Unassigned Cases
```json
{
  "select": [
    "id",
    "subject",
    "case_type_id:label",
    "status_id:name",
    "Cases_SR_Projects_.MAS_SR_Case_Code"
  ],
  "where": [
    ["status_id:name", "=", "Sent for Assignment"]
  ]
}
```

**Key Insight:** Unassigned cases have status "Sent for Assignment" (not a missing VC relationship)

### Get Cases for Specific VC
```json
{
  "select": [
    "id",
    "subject",
    "status_id:name",
    "vc.near_contact_id.sort_name"
  ],
  "where": [
    ["vc.near_contact_id", "=", 123]
  ],
  "join": [
    ["Contact AS client", "LEFT", "CaseContact", ["id", "=", "client.case_id"]],
    ["Contact AS vc", "LEFT", "RelationshipCache",
      ["client.id", "=", "vc.far_contact_id"],
      ["vc.near_relation:name", "=", "\"Case Coordinator is\""],
      ["vc.case_id", "=", "id"]
    ]
  ]
}
```

### Get Cases for Client Organization
```json
{
  "select": [
    "id",
    "subject",
    "client.sort_name",
    "client.employer_id.display_name"
  ],
  "where": [
    ["client.employer_id", "=", 456]
  ],
  "join": [
    ["Contact AS client", "LEFT", "CaseContact", ["id", "=", "client.case_id"]]
  ]
}
```

### Get Full Case Details
```json
{
  "select": [
    "id",
    "subject",
    "case_type_id:label",
    "status_id:name",
    "start_date",
    "end_date",
    "details",
    "Cases_SR_Projects_.MAS_SR_Case_Code",
    "Cases_SR_Projects_.Notes",
    "Cases_SR_Projects_.Practice_Area:label",
    "client.id",
    "client.sort_name",
    "client.email_primary.email",
    "client.employer_id.display_name",
    "vc.near_contact_id",
    "vc.near_contact_id.sort_name",
    "vc.near_contact_id.email_primary.email"
  ],
  "where": [["id", "=", 1]],
  "join": [
    ["Contact AS client", "LEFT", "CaseContact", ["id", "=", "client.case_id"]],
    ["Contact AS vc", "LEFT", "RelationshipCache",
      ["client.id", "=", "vc.far_contact_id"],
      ["vc.near_relation:name", "=", "\"Case Coordinator is\""],
      ["vc.case_id", "=", "id"]
    ]
  ],
  "limit": 1
}
```

---

## Case Statuses

### Service Request Statuses

**Opened (Active):**
- Ongoing
- Request RCS
- RCS Completed
- **Sent for Assignment** (unassigned cases)
- Active
- On Hold

**Closed:**
- Help provided - no project
- No VC Response
- No Client Response
- Project Created
- Cancelled
- Closed - Not Completed
- Completed
- Closed
- Resolved

### Project Statuses

**Opened (Active):**
- Active
- Cancelled
- Closed - Not Completed
- Completed
- On Hold

**Closed:**
- Help provided - no project
- No VC Response
- No Client Response
- Project Created
- Cancelled
- Closed - Not Completed
- Completed
- Closed
- Resolved

---

## Common Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `=` | Equals | `["id", "=", 123]` |
| `!=` | Not equals | `["status", "!=", "Closed"]` |
| `LIKE` | Pattern match (case-insensitive) | `["name", "LIKE", "%Smith%"]` |
| `IN` | Value in list | `["id", "IN", [1, 2, 3]]` |
| `NOT IN` | Value not in list | `["status", "NOT IN", ["Closed"]]` |
| `IS NULL` | Field is null | `["employer_id", "IS NULL"]` |
| `IS NOT NULL` | Field is not null | `["email", "IS NOT NULL"]` |
| `>`, `<`, `>=`, `<=` | Comparisons | `["created_date", ">", "2024-01-01"]` |
| `CONTAINS` | Multi-value field contains | `["contact_sub_type:name", "CONTAINS", "MAS_Rep"]` |

---

## Response Format

### Success Response
```json
{
  "values": [
    {
      "id": 123,
      "display_name": "John Smith",
      "email_primary.email": "john@example.com"
    }
  ],
  "count": 1
}
```

### Error Response
```json
{
  "error_code": "unauthorized",
  "error_message": "API permission check failed for Contact/get"
}
```

---

## Best Practices

### 1. Always Set checkPermissions
```json
{
  "checkPermissions": false
}
```
Use `false` when you control permissions via the API user itself (recommended).

### 2. Always Set a Limit
```json
{
  "limit": 25  // Default to prevent large result sets
}
```

### 3. Use Specific Field Selection
```json
{
  "select": ["id", "display_name", "email_primary.email"]  // Don't use "*"
}
```

### 4. Test in API Explorer First
1. Go to https://www.masadvise.org/civicrm/api4
2. Build query visually
3. Copy generated code to curl or n8n

### 5. Use :name Suffix for IDs
```json
{
  "select": [
    "status_id:name",      // "Active" (readable)
    "case_type_id:label"   // "Service Request" (readable)
  ]
}
```

---

## Troubleshooting

### Authentication Error (401)
**Cause:** Missing or invalid API key/site key
**Fix:** Verify both `X-Civi-Auth` and `X-Civi-Key` headers

### HTML Response Instead of JSON
**Cause:** Wrong URL (missing `/ajax/`)
**Fix:** Use `/civicrm/ajax/api4/` not `/civicrm/api4/`

### Empty WHERE Clause Returns All Records
**Cause:** No filters applied
**Fix:** Add appropriate WHERE conditions or adjust limit

### Custom Field Not Found
**Cause:** Wrong field name or group name
**Fix:** Check exact name in CiviCRM → Customize Data → Custom Fields

### Join Returns Null
**Cause:** Relationship doesn't exist or wrong join condition
**Fix:** Verify relationship exists, check join syntax (especially quoted strings)

### Form Encoding Error
**Cause:** Body sent as raw JSON instead of form-urlencoded
**Fix:** Use `Content-Type: application/x-www-form-urlencoded` and `params=<json>`

---

## Security Considerations

### API User Permissions
Create dedicated API user with minimal permissions:
- ✅ Access CiviCRM
- ✅ View all contacts
- ✅ Access all cases and activities
- ❌ Edit/delete contacts
- ❌ Administer CiviCRM

### Rate Limiting
- No built-in rate limiting in CiviCRM
- Implement in n8n or API gateway if needed

### Data Filtering
- Always filter by VC assignment for case queries
- Don't expose sensitive fields (API keys, custom sensitive data)
- Use `checkPermissions: false` with care

---

## Related Resources

- **CiviCRM API v4 Documentation**: https://docs.civicrm.org/dev/en/latest/api/v4/
- **API Explorer**: https://www.masadvise.org/civicrm/api4
- **MAS API v4 Protocol**: `/home/brian/workspace/development/mascode/docs/API4-PATTERNS.md` (canonical: https://github.com/masadvise-ontario/mascode/blob/master/docs/API4-PATTERNS.md) — moved to mascode, its owner, when `mas-claude-context` was retired (klaus PRs #95/#97/#98). The old `protocols/api4.md` copy is byte-identical but sits in a directory that is being wound down.
- **n8n Workflow** (historical): `/home/brian/workspace/workflows/personal/mas-vc-chatbot/workflows/civicrm-tool-handler.json` — **archive only.** n8n was decommissioned 2026-07-07; that repo is the historical JSON archive and the instance has zero active workflows. This project is code-first.

---

**Document Version**: 1.0
**Last Updated**: 2026-01-21
**Maintained By**: Brian Flett
