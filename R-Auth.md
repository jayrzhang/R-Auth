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
R-Auth is a microservice that handles login authorization and authentication for the resource servers 
of R-Auth, R-Data, R-Report, Roaming360. Any other Java springboot microservices can be easily configured to 
connect with R-Auth for authentication service. Its authorization server is connected with Ldap server 
and Vault.

### OAuth2 flow

``` mermaid
    sequenceDiagram
    autonumber
    participant Client
    participant Authorization_Server
    participant Resource_Server
    Client->>Authorization_Server: Client requests access token(with parameters of clientId, clientSecret, username, password...)
    Authorization_Server->>Client: Return an access token
    Client->>Resource_Server: Issue API calls with access token to get the resource
    Resource_Server->>Client: Return the requested resource
```
  Note 1: Authorization_Server is what R-Auth provides for authentication, Resource_Server is the protected services 
  that we want to get access to. A resource server can be R-Data, R-Report and even R-Auth itself.
  Note 2: When client requests access token, the clientId is the basic authentication to be matched with each resource 
  server's resourceId. In our symcarrier solution, clientId can be symcarrier, roaming360 or rauth.
    
### Authorize flow
``` mermaid
    sequenceDiagram
    autonumber
    participant AuthorizationController
    participant R_Integration
    participant Database
    participant VaultController
    participant VaultServiceImpl
    participant LdapUserAuthoritiesPopulator
    participant AuthUserDetailsServiceImpl
    participant WebSecurityConfiguration
    VaultController->>VaultServiceImpl: Invoke initialize() at application start-up
    VaultServiceImpl->>R_Integration: Send request to R-Integration Vault endpoint to fetch superuser username and password
    VaultServiceImpl->>Database: Update db with superuser username and password
    VaultServiceImpl->>WebSecurityConfiguration: Update InMemoryUserDetailsManager with superuser username and password
    WebSecurityConfiguration->>WebSecurityConfiguration: Connect spring security with Ldap server and InMemoryUserDetailsManager to validate given username and password
    alt Validate success
        WebSecurityConfiguration->>LdapUserAuthoritiesPopulator: Fetch granted authorities
        LdapUserAuthoritiesPopulator->>AuthUserDetailsServiceImpl: Load user by username
        alt User exist
            AuthUserDetailsServiceImpl->>AuthUserDetailsServiceImpl: Update granted authorities if any
        else User not exist
            AuthUserDetailsServiceImpl->>Database: Insert new user information into db
            end
        WebSecurityConfiguration->>AuthorizationController: Return granted authorities
    else Validate failed
        WebSecurityConfiguration->>AuthorizationController: Exception thrown InvalidGrantException
        end
```

### Keycloak flow
``` mermaid
    sequenceDiagram
    autonumber
    participant R-Portal
    participant R-Data
    participant R-Report
    participant Keycloak
    participant API-GW
    participant Keycloak-Redis
    participant UM-Redis
    R-Portal->>Keycloak: Client redirected to Keycloak login page for authentication
    Keycloak->>Keycloak-Redis: Store session with scopes
    Keycloak-Redis->>Keycloak: Session stored
    Keycloak->>R-Portal: Return an access token and redirect client to R-Portal default dashboard page when authentication succeeds
    R-Portal->>Keycloak-Redis: Validate session and scopes for roles
    Keycloak-Redis->>R-Portal: Return list of roles
    R-Portal->>API-GW: Issue API calls with access token to API-GW to get the resource
    API-GW->>Keycloak: Validate user in Keycloak for access token
    Keycloak->>API-GW: Return true if validation succeeds
    API-GW->>UM: Validate session and scopes for access token and validate API permissions based on session id
    UM->>API-GW: Session and API permissions validated
    API-GW->>R-Data: Send request for requested resource
    R-Data->>API-GW: Return the requested resource
    API-GW->>R-Report: Send request for requested resource
    R-Report->>API-GW: Return the requested resource
    API-GW->>R-Portal: Return the requested resource
```


## Database
### Roles
```go
Roles:
  Represents all the available roles for users. It will be matched from the Ldap groups and generated at the beginning of the API start-up using query script. 
  
Database table name: roles

Database Table Structure:

                                                     Table "rauth.roles"
   Column   |            Type             | Collation | Nullable | Default | Storage  | Stats target | Description 
------------+-----------------------------+-----------+----------+---------+----------+--------------+-------------
 id         | uuid                        |           | not null |         | plain    |              | 
 created_at | timestamp without time zone |           | not null |         | plain    |              | 
 updated_by | character varying           |           |          |         | extended |              | 
 updated_at | timestamp without time zone |           |          |         | plain    |              | 
 version    | integer                     |           | not null |         | plain    |              | 
 role       | character varying(256)      |           | not null |         | extended |              | 
Indexes:
    "roles_pk" PRIMARY KEY, lsm (id HASH)
    "roles_uindex" UNIQUE, lsm (lower(role::text) HASH)
Referenced by:
    TABLE "roles_authorities" CONSTRAINT "roles_authorities_roles_id_fk" FOREIGN KEY (role_id) REFERENCES roles(id)
    
Unique column: role (Roles with same role will be considered duplicate and cannot be allowed; case sensitive)

Sample record:
 9a7e5c14-4b96-11ed-bdc3-0242ac120002 | 2022-09-21 10:34:57.497467 |            |            |       1 | ROLE_DEVELOPERS

REST supported operations:

post          - Creates new role record, URI: <base context>/roles
list all      - Retrieves all roles, URI: <base-context>/roles/
list          - List specific role, URI: <base-context>/roles/{id}
update        - Updates specific role record, URI: <base-context>/roles/{id}
update status - Updates specific role record status, URI: <base-context>/roles/{id}
```

### Authorities
```go
Roles:
  Represents all the available authorities for users. It will be generated at the beginning of the API start-up using query script. One super user will be granted with the SUPER_ROLE authority which enables all endpoints.
  
Database table name: authorities

Database Table Structure:
                                             Table "rauth.authorities"
   Column   |            Type             | Collation | Nullable | Default | Storage  | Stats target | Description 
------------+-----------------------------+-----------+----------+---------+----------+--------------+-------------
 id         | uuid                        |           | not null |         | plain    |              | 
 created_at | timestamp without time zone |           | not null |         | plain    |              | 
 updated_by | character varying           |           |          |         | extended |              | 
 updated_at | timestamp without time zone |           |          |         | plain    |              | 
 version    | integer                     |           | not null |         | plain    |              | 
 authority  | character varying(256)      |           | not null |         | extended |              | 
Indexes:
    "authorities_pk" PRIMARY KEY, lsm (id HASH)
    "authorities_uindex" UNIQUE, lsm (lower(authority::text) HASH)
Referenced by:
    TABLE "roles_authorities" CONSTRAINT "roles_authorities_authorities_id_fk" FOREIGN KEY (authority_id) REFERENCES authorities(id)
    TABLE "users_authorities" CONSTRAINT "users_authorities_authorities_id_fk" FOREIGN KEY (authority_id) REFERENCES authorities(id)
    
Unique column: authority (Authorities with same authority will be considered duplicate and cannot be allowed; case sensitive)

Sample record:
87ad3c48-4db9-11ed-bdc3-0242ac120002 | 2022-09-21 10:34:57.497467 |            |            |       1 | UPDATE_CODEC

REST supported operations:

post          - Creates new authority record, URI: <base context>/authorities
list all      - Retrieves all authorities, URI: <base-context>/authorities/
list          - List specific authority, URI: <base-context>/authorities/{id}
update        - Updates specific authority record, URI: <base-context>/authorities/{id}
update status - Updates specific authority record status, URI: <base-context>/authorities/{id}
```

