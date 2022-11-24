# R-Data High Level overview
## Architecture Flow Diagram
### Update createdBy,updatedBy fields Flow
```mermaid
  sequenceDiagram
  autonumber
  participant RDataController
  participant RDataServiceImpl
  participant R_Auth
  R_Auth->>RDataController: Validate granted authorities
  alt Validate success
      R_Auth->>RDataController: Provide Principal which contains the username info of the login user
      RDataController->>RDataServiceImpl: Update createdBy,updatedBy fields with the Principal.getName() 
  else Validate failed
      R_Auth->>RDataController: Exception thrown InvalidGrantException
      end
```
```mermaid
  sequenceDiagram
  autonumber
  participant TemplatesAttributeUpdateRequest
  participant TemplatesAttribute
  participant TemplatesAttributeController
  TemplatesAttributeUpdateRequest->>TemplatesAttribute: Compare with each others' attribute id
  alt UUID exist in request but not in existed attributes
      TemplatesAttributeUpdateRequest->>TemplatesAttributeController: save new attributes to db
  else UUID exist in existed attributes but not in request
      TemplatesAttribute->>TemplatesAttributeController: soft delete the missing attributes
  else UUID exist in both request and existed attributes
      alt Active true
          TemplatesAttributeUpdateRequest->>TemplatesAttribute: update the existed attributes
      else Active false
          TemplatesAttributeUpdateRequest->>TemplatesAttribute: activate the deactivated attributes back
          end
      TemplatesAttribute->>TemplatesAttributeController: save the updated attributes to db
      end
```
