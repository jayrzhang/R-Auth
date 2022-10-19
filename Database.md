```go
Billing Duration:

      Represents the bill cycle. For phase 1, it will be value 30. The billing duration details can be added or updated. 

Database table name: lookup_billing_durations

Database Table Structure:
 id | created_by | create_time | updated_by | update_time | version | value | description | active

Unique column: value (duplicated continent name not allowed, case sensitive)

Sample record:
e6509a58-3d19-4533-ae43-987bb625d872 | user | 2022-09-21 09:32:56.609609 |  | 2022-09-21 09:32:56.609685 | 0 | 30 | Monthly basis  | t

REST supported operations:

post          - Creates new billing duration record, URI: <base context>/billing-durations
list all      - Retrieves all billing durations, URI: <base-context>/billing-durations/
list          - List specific billing duration, URI: <base-context>/billing-durations/{id}
update        - Updates specific billing duration record, URI: <base-context>/billing-durations/{id}
update status - Updates specific billing duration record status, URI: <base-context>/billing-durations/status/{id}

```
