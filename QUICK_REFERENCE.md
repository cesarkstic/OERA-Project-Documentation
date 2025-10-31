# OERA Quick Reference Cheat Sheet

## Layer Responsibilities

| Layer | Responsibility | What it Does | What it NEVER Does |
|-------|----------------|--------------|-------------------|
| **Presentation** | UI Logic | Display data, user input, call BE methods | Business logic, database access, validation |
| **Business Entity** | Business Logic | Validation, business rules, calculations | UI logic, direct database access |
| **Data Access** | Database CRUD | Fetch, save, delete from database | Business logic, validation, UI |

---

## Dataset Include Strategy

### Include File Template
```openedge
/* EntityDataset.i */
&IF "{&REFERENCE-ONLY}" = "" &THEN
    &SCOPED-DEFINE REFERENCE-ONLY
&ENDIF

DEFINE TEMP-TABLE ttEntity NO-UNDO {&REFERENCE-ONLY}
    FIELD EntityID AS INTEGER
    FIELD Field1 AS CHARACTER
    ...
    INDEX idxPrimary IS PRIMARY UNIQUE EntityID.

DEFINE DATASET dsEntity {&REFERENCE-ONLY} FOR ttEntity.
```

### Usage by Layer
```openedge
/* Presentation Layer */
{EntityDataset.i}

/* Business/Data Access Layers */
{EntityDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}
```

---

## Method Signatures

### DAO Layer
```openedge
/* Fetch - OUTPUT only */
METHOD PUBLIC VOID FetchEntity(
    INPUT  iEntityID AS INTEGER,
    OUTPUT DATASET dsEntity BY-REFERENCE).

/* Save - INPUT-OUTPUT for round-trip */
METHOD PUBLIC VOID SaveEntity(
    INPUT-OUTPUT DATASET dsEntity BY-REFERENCE).

/* Delete - INPUT only */
METHOD PUBLIC VOID DeleteEntity(
    INPUT iEntityID AS INTEGER).
```

### Business Entity Layer
```openedge
/* Get - OUTPUT only */
METHOD PUBLIC VOID GetEntity(
    INPUT  iEntityID AS INTEGER,
    OUTPUT DATASET dsEntity BY-REFERENCE).

/* Save with validation - INPUT-OUTPUT + error message */
METHOD PUBLIC LOGICAL SaveEntity(
    INPUT-OUTPUT DATASET dsEntity BY-REFERENCE,
    OUTPUT cErrorMessage AS CHARACTER).

/* Delete with business rules - INPUT + error message */
METHOD PUBLIC LOGICAL DeleteEntity(
    INPUT  iEntityID AS INTEGER,
    OUTPUT cErrorMessage AS CHARACTER).
```

---

## Common Code Snippets

### DAO Fetch Pattern
```openedge
DEFINE BUFFER bEntity FOR Entity.

EMPTY TEMP-TABLE ttEntity.

FOR EACH bEntity NO-LOCK WHERE bEntity.EntityID = iEntityID:
    CREATE ttEntity.
    BUFFER-COPY bEntity TO ttEntity.
END.
```

### DAO Save Pattern
```openedge
DEFINE BUFFER bEntity FOR Entity.
DEFINE BUFFER bttEntity FOR ttEntity.

DO TRANSACTION:
    FOR EACH bttEntity:
        FIND FIRST bEntity EXCLUSIVE-LOCK
            WHERE bEntity.EntityID = bttEntity.EntityID
            NO-ERROR.
        
        IF NOT AVAILABLE bEntity THEN DO:
            CREATE bEntity.
            ASSIGN bEntity.EntityID = NEXT-VALUE(EntitySeq).
            ASSIGN bttEntity.EntityID = bEntity.EntityID.
        END.
        
        BUFFER-COPY bttEntity EXCEPT EntityID TO bEntity.
    END.
END.
```

### BE Validation Pattern
```openedge
METHOD PRIVATE LOGICAL ValidateEntity(
    INPUT-OUTPUT DATASET dsEntity BY-REFERENCE,
    OUTPUT cErrorMessage AS CHARACTER):
    
    DEFINE BUFFER bttEntity FOR ttEntity.
    
    FOR EACH bttEntity:
        IF bttEntity.RequiredField = ? THEN DO:
            cErrorMessage = "Required field is missing.".
            RETURN FALSE.
        END.
        
        IF LOOKUP(bttEntity.Status, "VALID,LIST,HERE") = 0 THEN DO:
            cErrorMessage = "Invalid status value.".
            RETURN FALSE.
        END.
    END.
    
    RETURN TRUE.
END METHOD.
```

### BE Audit Fields Pattern
```openedge
METHOD PRIVATE VOID SetAuditFields(
    INPUT-OUTPUT DATASET dsEntity BY-REFERENCE):
    
    DEFINE BUFFER bttEntity FOR ttEntity.
    DEFINE VARIABLE cUser AS CHARACTER NO-UNDO.
    DEFINE VARIABLE dtNow AS DATETIME NO-UNDO.
    
    ASSIGN cUser = USERID("DICTDB")
           dtNow = NOW.
    
    FOR EACH bttEntity:
        IF bttEntity.EntityID = 0 OR bttEntity.EntityID = ? THEN
            ASSIGN bttEntity.CreatedBy = cUser
                   bttEntity.CreatedDate = dtNow.
        
        ASSIGN bttEntity.ModifiedBy = cUser
               bttEntity.ModifiedDate = dtNow.
    END.
END METHOD.
```

### Presentation Call Pattern
```openedge
{EntityDataset.i}  /* No REFERENCE-ONLY */

DEFINE VARIABLE oEntityBE AS IEntityBE NO-UNDO.
DEFINE VARIABLE cError AS CHARACTER NO-UNDO.
DEFINE VARIABLE lSuccess AS LOGICAL NO-UNDO.

oEntityBE = NEW EntityBE().

/* Load */
oEntityBE:GetEntity(123, OUTPUT DATASET dsEntity BY-REFERENCE).

/* Save */
lSuccess = oEntityBE:SaveEntity(
    INPUT-OUTPUT DATASET dsEntity BY-REFERENCE,
    OUTPUT cError).

IF NOT lSuccess THEN
    MESSAGE cError VIEW-AS ALERT-BOX ERROR.
```

---

## Validation Patterns

### Required Field
```openedge
IF FieldName = "" OR FieldName = ? THEN DO:
    cErrorMessage = "FieldName is required.".
    RETURN FALSE.
END.
```

### Numeric Range
```openedge
IF Amount < 0 OR Amount > 999999 THEN DO:
    cErrorMessage = "Amount must be between 0 and 999,999.".
    RETURN FALSE.
END.
```

### Status/Enum
```openedge
IF LOOKUP(Status, "ACTIVE,INACTIVE,PENDING") = 0 THEN DO:
    cErrorMessage = "Invalid status.".
    RETURN FALSE.
END.
```

### Date Validation
```openedge
IF TransDate = ? THEN DO:
    cErrorMessage = "Date is required.".
    RETURN FALSE.
END.

IF TransDate > TODAY THEN DO:
    cErrorMessage = "Date cannot be in future.".
    RETURN FALSE.
END.
```

### Date Range
```openedge
IF StartDate > EndDate THEN DO:
    cErrorMessage = "Start date must be before end date.".
    RETURN FALSE.
END.
```

### Email Format
```openedge
IF INDEX(Email, "@") = 0 THEN DO:
    cErrorMessage = "Invalid email format.".
    RETURN FALSE.
END.
```

### Duplicate Check
```openedge
DEFINE BUFFER bCheck FOR Entity.

FIND FIRST bCheck NO-LOCK
    WHERE bCheck.UniqueField = bttEntity.UniqueField
    AND bCheck.EntityID <> bttEntity.EntityID
    NO-ERROR.

IF AVAILABLE bCheck THEN DO:
    cErrorMessage = "Duplicate value found.".
    RETURN FALSE.
END.
```

---

## Constructor Patterns

### Standard Constructor
```openedge
CLASS EntityBE:
    DEFINE PRIVATE VARIABLE oDAO AS IEntityDAO NO-UNDO.
    
    /* Default constructor */
    CONSTRUCTOR PUBLIC EntityBE():
        oDAO = NEW EntityDAO().
    END CONSTRUCTOR.
    
    /* Test constructor - dependency injection */
    CONSTRUCTOR PUBLIC EntityBE(INPUT poDAO AS IEntityDAO):
        oDAO = poDAO.
    END CONSTRUCTOR.
END CLASS.
```

---

## Error Handling Pattern

### In BE Methods
```openedge
METHOD PUBLIC LOGICAL SaveEntity(...):
    
    /* Validation */
    IF NOT ValidateEntity(...) THEN
        RETURN FALSE.
    
    /* Business logic */
    SetAuditFields(...).
    
    /* Save */
    oDAO:SaveEntity(INPUT-OUTPUT DATASET dsEntity BY-REFERENCE).
    
    RETURN TRUE.
    
    CATCH eError AS Progress.Lang.Error:
        cErrorMessage = eError:GetMessage(1).
        RETURN FALSE.
    END CATCH.
    
END METHOD.
```

### In Presentation
```openedge
lSuccess = oEntityBE:SaveEntity(
    INPUT-OUTPUT DATASET dsEntity BY-REFERENCE,
    OUTPUT cError).

IF lSuccess THEN
    MESSAGE "Saved successfully." VIEW-AS ALERT-BOX INFORMATION.
ELSE
    MESSAGE cError VIEW-AS ALERT-BOX ERROR.
```

---

## Business Rules Pattern

### Delete Restrictions
```openedge
METHOD PUBLIC LOGICAL DeleteEntity(...):
    
    /* Fetch to check rules */
    oDAO:FetchEntity(iEntityID, OUTPUT DATASET dsEntity BY-REFERENCE).
    
    FIND FIRST bttEntity NO-LOCK NO-ERROR.
    
    IF NOT AVAILABLE bttEntity THEN DO:
        cErrorMessage = "Record not found.".
        RETURN FALSE.
    END.
    
    /* Business rules */
    IF bttEntity.Status = "POSTED" THEN DO:
        cErrorMessage = "Cannot delete posted records.".
        RETURN FALSE.
    END.
    
    IF bttEntity.HasChildren THEN DO:
        cErrorMessage = "Cannot delete record with dependencies.".
        RETURN FALSE.
    END.
    
    /* Delete */
    oDAO:DeleteEntity(iEntityID).
    RETURN TRUE.
    
END METHOD.
```

---

## Find and Replace Checklist

When using generic template:

| Find | Replace With | Example |
|------|--------------|---------|
| `{Entity}` | PascalCase name | Customer |
| `{entity}` | lowercase name | customer |
| `Field1`, `Field2` | Actual field names | CustomerName, Email |
| `Entity` (DB table) | Actual DB table | Customer |
| `EntitySeq` | Actual sequence | CustomerSeq |

---

## File Naming Conventions

| File Type | Naming Pattern | Example |
|-----------|---------------|---------|
| Dataset Include | `{Entity}Dataset.i` | CustomerDataset.i |
| DAO Interface | `I{Entity}DAO.cls` | ICustomerDAO.cls |
| DAO Class | `{Entity}DAO.cls` | CustomerDAO.cls |
| BE Interface | `I{Entity}BE.cls` | ICustomerBE.cls |
| BE Class | `{Entity}BE.cls` | CustomerBE.cls |
| Task Class | `{Entity}Task.cls` | CustomerTask.cls |
| Presentation | `{Entity}Window.w` | CustomerWindow.w |
| Mock DAO | `Mock{Entity}DAO.cls` | MockCustomerDAO.cls |
| Unit Test | `{Entity}BETest.cls` | CustomerBETest.cls |

---

## REFERENCE-ONLY Rules

| Layer | Include Syntax | Creates Dataset? | Purpose |
|-------|---------------|------------------|---------|
| Presentation | `{Dataset.i}` | ✅ YES | Owns the data |
| Business Entity | `{Dataset.i &REFERENCE-ONLY=REFERENCE-ONLY}` | ❌ NO | References only |
| Business Task | `{Dataset.i &REFERENCE-ONLY=REFERENCE-ONLY}` | ❌ NO | References only |
| Data Access | `{Dataset.i &REFERENCE-ONLY=REFERENCE-ONLY}` | ❌ NO | References only |

**Key Point:** REFERENCE-ONLY is about the DEFINITION, not the data!

---

## Common Mistakes to Avoid

| ❌ DON'T | ✅ DO |
|---------|------|
| Pass datasets BY-VALUE | Pass datasets BY-REFERENCE |
| Use REFERENCE-ONLY in Presentation | Omit REFERENCE-ONLY in Presentation |
| Put business logic in DAO | Keep business logic in BE |
| Access database from Presentation | Call BE methods only |
| Skip validation in BE | Always validate in BE |
| Forget audit fields | Set CreatedBy/ModifiedBy |
| Hard-code user ID | Use USERID("DICTDB") |
| Skip error handling | Use CATCH blocks |

---

## Unit Testing Pattern

### Mock DAO
```openedge
CLASS MockEntityDAO IMPLEMENTS IEntityDAO:
    DEFINE PUBLIC PROPERTY SaveWasCalled AS LOGICAL GET. SET.
    
    METHOD PUBLIC VOID FetchEntity(...):
        CREATE ttEntity.
        /* Return mock data */
    END METHOD.
    
    METHOD PUBLIC VOID SaveEntity(...):
        SaveWasCalled = TRUE.
    END METHOD.
END CLASS.
```

### Test Class
```openedge
USING OpenEdge.Core.Assert.

CLASS EntityBETest:
    DEFINE PRIVATE VARIABLE oMockDAO AS MockEntityDAO NO-UNDO.
    DEFINE PRIVATE VARIABLE oBE AS EntityBE NO-UNDO.
    
    @Before.
    METHOD PUBLIC VOID setUp():
        oMockDAO = NEW MockEntityDAO().
        oBE = NEW EntityBE(oMockDAO).  /* Inject mock */
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testValidation():
        /* Test code */
        Assert:IsFalse(lResult).
    END METHOD.
END CLASS.
```

**See [UNIT_TESTING_GUIDE.md](UNIT_TESTING_GUIDE.md) for complete examples**

---

## Quick Implementation Steps

1. ✅ Create dataset include with REFERENCE-ONLY parameter
2. ✅ Create DAO interface with Fetch/Save/Delete methods
3. ✅ Implement DAO class with database operations
4. ✅ Create BE interface with Get/Save/Delete methods
5. ✅ Implement BE class with validation and business rules
6. ✅ Add test constructor for dependency injection
7. ✅ Create presentation layer calling BE methods
8. ✅ (Optional) Create mock DAO and unit tests
9. ✅ Test each layer independently

---

## Key Principles

1. **Separation of Concerns** - Each layer has one job
2. **Dependency Inversion** - Depend on interfaces, not implementations
3. **Single Responsibility** - Each class does one thing well
4. **Open/Closed** - Open for extension, closed for modification
5. **Testability** - Use interfaces for unit testing
6. **Pass by Reference** - Always use BY-REFERENCE for datasets
7. **Reference Only** - Use in middle layers, not presentation

---

**Print this page and keep it handy for quick reference!**
