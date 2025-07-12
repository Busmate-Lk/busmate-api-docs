## General Info

- **Title:** NTC Planning Section Official Portal API
- **Version:** 1.0.0
- **Description:** API for managing routes, schedules, stops, permits, operators, buses, and dashboard metrics for the NTC (Sri Lanka) Planning Section.
- **Authentication:** JWT Bearer

---

## API Endpoints Overview

**Total Endpoints:** 56

### By Resource

| Resource                  | Endpoints | CRUD/Other Operations                                                                                 |
|---------------------------|-----------|-------------------------------------------------------------------------------------------------------|
| Route Groups              | 5         | List, Create, Get, Update, Delete, Audit Logs                                                         |
| Routes                    | 2         | List, Get                                                                                            |
| Schedules                 | 7         | List, Create, Get, Update, Delete, Bulk Create, Audit Logs                                            |
| Stops                     | 7         | List, Create, Get, Update, Delete, Search, Audit Logs                                                 |
| Permits                   | 7         | List, Create, Get, Update, Delete, Audit Logs                                                         |
| Permit Bus Change Requests| 6         | List, Create, Get, Approve, Reject                                                                    |
| Operators                 | 7         | List, Create, Get, Update, Delete, Audit Logs                                                         |
| Buses                     | 7         | List, Create, Get, Update, Delete, Audit Logs                                                         |
| Schedule Assignments      | 7         | List, Create, Get, Update, Delete, Bulk Create, Audit Logs                                            |

---

## Key Features

- **Pagination, Filtering, Sorting:** Most list endpoints support these features.
- **Bulk Operations:** Schedules and Schedule Assignments support bulk creation.
- **Audit Logs:** Available for all major resources.
- **Security:** All endpoints require JWT authentication and enforce permissions.
- **Reference Checks:** Delete operations check for references (e.g., active schedules, assignments).

---

## Schema Coverage

- **Entities:** Route, RouteGroup, Schedule, Stop, Permit, PermitBusChangeRequest, Operator, Bus, ScheduleAssignment, AuditLog, Error, Meta.
- **Standardized Error Handling:** All endpoints return structured error responses.

---

## Example: Endpoint Count by HTTP Method

- **GET:** 28
- **POST:** 17
- **PUT:** 7
- **DELETE:** 7

---

## Notable Patterns

- **CRUD for all major entities**
- **Audit log retrieval for all major entities**
- **Bulk creation for schedules and assignments**
- **Approval/rejection workflow for permit-bus change requests**

---

**In summary:**  
This API is comprehensive, RESTful, and covers all aspects of NTC planning section operations, with robust security, audit, and bulk operation support. There are 56 endpoints covering all major resources and their management.