# R-Auth High Level overview
## Overview
R-Auth is an Authentication system for integrating with all systems.
```go
R-Auth is a containerized application
R-Auth runs on port: `8092`
R-Auth is orchestrated by kubernetes
R-Auth deployment life cycle is managed by helm
R-Auth is responsible for integrating with all external systems in "Solution R"
R-Auth REST endpoints(Rel 1.0) starts with base context 'v1/rauth/'
```
## Capability
```go
 - Accesses authentication server via Ldap
 - Integrates with external Solution R microservices. Supported microservices are,
   - R-Data
 - Provides endpoints for CRUD operations on certain tables in R-Auth database, 
   - Roles
   - Authorities
```
## Architecture Flow Diagram
```mermaid
classDiagram
REST Endpoint --|> Authorize
Authorize --|> APIFilter
APIFilter --|> LdapUserAuthoritiesPopulator
LdapUserAuthoritiesPopulator --|> AuthUserDetailsServiceImpl
AuthUserDetailsServiceImpl --|> WebSecurityConfiguration
WebSecurityConfiguration --|> Principal
Principal --|> ResponseEntity
```
---
# R-Auth Documents
## Features
### Overview
R-Auth provides below importatnt features:
1. Authorization service
### Authorize flow
``` mermaid
    sequenceDiagram
    autonumber
    participant AuthorizationController
    participant APIFilter
    participant LdapUserAuthoritiesPopulator
    participant AuthUserDetailsServiceImpl
    participant WebSecurityConfiguration
    AuthorizationController->>APIFilter: Invoke http interceptor to check if user is superuser or normal user
    alt Normaluser
        APIFilter->>LdapUserAuthoritiesPopulator: Fetch granted authorities
        LdapUserAuthoritiesPopulator->>AuthUserDetailsServiceImpl: Load user by username
        alt User exist
            AuthUserDetailsServiceImpl->>AuthUserDetailsServiceImpl: update granted authorities if any
        else User not exist
            AuthUserDetailsServiceImpl->>AuthUserDetailsServiceImpl: insert new user information into database
            end
        AuthUserDetailsServiceImpl->>WebSecurityConfiguration: Build AuthenticationManagerBuilder
        WebSecurityConfiguration->>WebSecurityConfiguration: Validate given username and password
        alt Validate success
            WebSecurityConfiguration->>AuthorizationController: Provide granted authorities
        else Validate failed
            WebSecurityConfiguration->>AuthorizationController: Exception thrown InvalidGrantException
            end
    else Superuser
        APIFilter->>R-Integration Vault Endpoint: Validate given username and password
        alt Validate success
            WebSecurityConfiguration->>AuthorizationController: Provide granted authorities
        else Validate failed
            WebSecurityConfiguration->>AuthorizationController: Exception thrown InvalidGrantException
            end
    end
```
