[This project](https://github.com/valhuber/OptLocking) is the sample app, created by API Logic Server with optimistic locking (not generally released), using 2 patches to work around a bug I discovered in testing.

&nbsp;

## Problem Statement

Optimistic locking is a valuable feature for interactive systems with typical constraints:

1. Rows cannot be locked (pessimistically) on read, _in case_ they are updated
   * This would drastically decrease performance by making users wait on locks, so undesirable
2. Database design cannot tolerate new `VersionNumber` columns
3. Minimize network traffic and keep client coding simple
   * E.g., unwieldy to send all "old" values back  

Consider the scenario:

| Time | User | Action |
|:----- |:-------|:----|
| T0 | U1 | Reads Row.Column with value V1  |
| T1 | U2 | Reads same row |
| T2 | U1 | Updates row with value V2 |
| T3 | U2 | Updates row with value V3 - V2 value overwritten, U1 not happy |

A widely accepted solution is **optimistic locking:** 

1. No read locks
2. On update, ensure the row has not changed since the user read it

&nbsp;

## Approach: `CheckSum` to detect changes

Before summarizing the approach, we note some key elements provided by architectural components.

&nbsp;

### Key Architectural Elements

&nbsp;

#### 1. SAFRS `@jsonapi_attr`

SAFRS API provides add derived attribute: [`@jsonapi_attr`](https://github.com/thomaxxl/safrs/blob/master/examples/demo_pythonanywhere_com.py)
   * This enables the server to compute unstored values, here, `CheckSum`
   * SAFRS supports sending such values on client `patch` operations, so it is visible in logic

&nbsp;

#### 2. SQLAlchemy `loaded_as_persistent`

SQLAlchemy provides the `loaded_as_persistent` [event](https://docs.sqlalchemy.org/en/20/orm/events.html#sqlalchemy.orm.SessionEvents.loaded_as_persistent), enabling us to compute the `CheckSum`, store it in the row, so can later check it on update.

&nbsp;

#### 3. The rules engine supports generic `before_logic`

This enables us to check the row compare `CheckSum` values; see [`logic/declare_logic](/logic/declare_logic.py).

&nbsp;

&nbsp;

### Configurable options

Opt Locking is configured on server startup, from the Config file with Env overrides. The options are:

| Option | Included on `Get` | Checked on `Patch` |
|:----- |:-------|:----|
| **ignored** | Never | Never |
| **optional** | Always | Yes - but no error if omitted |
| **required** | Always | Yes - error if omitted |

&nbsp;

### Approach

The approach is summarized in the table below.  See the the code in [`api/system/opt_locking/opt_locking.py`](/api/system/opt_locking/opt_locking.py) for details.

&nbsp;

| Phase | Responsibility | Action | Notes |
|:-----|:-------|:-------|:----|
| Design Time | API Logic Server CLI | Declare Checksum | `models.py` - json_attr |
| Runtime - Read | System | Compute Checksum | `opt_locking#loaded_as` (from api_logic_server_run.py) |
| Runtime - Call Patch | Custom App Code<br>Admin App | Return as-read-Checksum | See examples below |
| Runtime - Process Patch | System | Compare CheckSums: as-read vs. current | `opt_locking#opt_locking_patch`, via `logic/declare_logic.py`: generic before event |

&nbsp;

### Bug: Failing on Patch without CheckSum

This approach works with client supply the `CheckSum` in the `Patch`.

It ***fails*** on `Patch` with no `CheckSum`, improperly reporting _Sorry, row altered by another user..._.

The cause: 

1. **patch retrieves the row**
2. This invokes `opt_locking#loaded_as` which sets the CheckSum
    * Aside: we *expected* this to get overwritten from client `ChecksSum` in the patched data, but that's wrong - in this case, it's *missing* in the patched data...
3. This gets seen in `opt_locking#opt_lock_patch`
    * It _thinks_ it's the `as_read_checksum`, but in fact reflects the proposed update values
    * Which is different from `current_checksum = checksum_old_row(logic_row.old_row)`

One solution is for *safrs patch* to set the `CheckSum` to a special value, which `opt_locking#opt_lock_patch` recognizes and skips the opt locking check.  I patched my local version as shown below, and this approach works:

![experiment](/images/patch_exp_test.png)

&nbsp;

### Other Discussions

Comments:

1. Best name for `CheckSum` (e.g. `S_CheckSum_` - see Category) - Thomas: will be configurable
    * Thomas will provide guidance on setting SAFRS config options
2. Admin App needs to include this in Patch...
3. Admin App reporting "Data Error Logging Disabled" - incorrect excp?  
    * Due to safrs logging level - Val ToDo

&nbsp;

&nbsp;

----
&nbsp;

## Test with cURL

You can explore this using the sample database with the cURL commands below.

&nbsp;

### `Get`

Observe that `CheckSum` is returned [6785985870086950264]:

```
curl -X 'GET' \
  'http://localhost:5656/api/Employee/5/?fields%5BEmployee%5D=Id%2CLastName%2CFirstName%2CTitle%2CTitleOfCourtesy%2CBirthDate%2CHireDate%2CAddress%2CCity%2CRegion%2CPostalCode%2CCountry%2CHomePhone%2CExtension%2CNotes%2CReportsTo%2CPhotoPath%2CEmployeeType%2CSalary%2CWorksForDepartmentId%2COnLoanDepartmentId%2CUnionId%2CDues%2C_check_sum_%2CCheckSum%2C__proper_salary__%2CProperSalary' \
  -H 'accept: application/vnd.api+json' \
  -H 'Content-Type: application/vnd.api+json'
```

&nbsp;

### `Patch`

**Important:** Admin App is not sending unchanged attributes; we must convince it to send the CheckSum.

To simulate the client:
1. Use cURL (note: this should fail with constraint violation):

```curl
curl -X 'PATCH' \
  'http://localhost:5656/api/Employee/5/' \
  -H 'accept: application/vnd.api+json' \
  -H 'Content-Type: application/json' \
  -d '{
    "data": {
        "attributes": {
            "Salary": 97000,
            "CheckSum": 6785985870086950264,
            "Proper_Salary": 50000,
            "Id": 5},
        "type": "Employee",
        "id": 5
    }
}'
```
&nbsp;

### `Patch` no `CheckSum`

**Important:** Admin App is not sending unchanged attributes; we must convince it to send the CheckSum.

To simulate the client (same issue occurs in Admin App, which incidentally *should* return the `CheckSum`)
1. Use cURL (note: this should fail with constraint violation):

```curl
curl -X 'PATCH' \
  'http://localhost:5656/api/Employee/5/' \
  -H 'accept: application/vnd.api+json' \
  -H 'Content-Type: application/json' \
  -d '{
    "data": {
        "attributes": {
            "Salary": 97000,
            "Proper_Salary": 50000,
            "Id": 5},
        "type": "Employee",
        "id": 5
    }
}'
```
&nbsp;

