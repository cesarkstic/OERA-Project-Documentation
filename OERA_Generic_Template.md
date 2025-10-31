# Generic OERA Template for Any Entity

## Quick Start Guide

Follow these steps to implement OERA for any new table:

1. Replace `{Entity}` with your entity name (e.g., Customer, Order, Product)
2. Replace `{entity}` with lowercase version (e.g., customer, order, product)
3. Replace field definitions with your actual table fields
4. Adjust validation and business rules
5. Copy and customize!

---

## Template Structure

```
/Business
    /DataDefinitions
        {Entity}Dataset.i
    /DataAccess
        I{Entity}DAO.cls
        {Entity}DAO.cls
    /Entity
        I{Entity}BE.cls
        {Entity}BE.cls
    /Task
        {Entity}Task.cls (optional for complex operations)
/Presentation
    {Entity}Window.w (or .p)
/Test
    /Mock
        Mock{Entity}DAO.cls
    /Unit
        {Entity}BETest.cls
```

---

## Step 1: Dataset Definition Include File

**File:** `/Business/DataDefinitions/{Entity}Dataset.i`

```openedge
/* {Entity}Dataset.i - Dataset definition for {Entity}
   
   Usage:
   {{Entity}Dataset.i}                               - For Presentation layer
   {{Entity}Dataset.i &REFERENCE-ONLY=REFERENCE-ONLY} - For DAO/BE/Task layers
*/

&IF "{&REFERENCE-ONLY}" = "" &THEN
    &SCOPED-DEFINE REFERENCE-ONLY
&ENDIF

/* Define temp-table structure matching your database table */
DEFINE TEMP-TABLE tt{Entity} NO-UNDO {&REFERENCE-ONLY}
    FIELD {Entity}ID        AS INTEGER      /* Primary key */
    FIELD Field1            AS CHARACTER    /* Replace with your fields */
    FIELD Field2            AS DATE
    FIELD Field3            AS DECIMAL
    FIELD Status            AS CHARACTER
    FIELD CreatedBy         AS CHARACTER
    FIELD CreatedDate       AS DATETIME
    FIELD ModifiedBy        AS CHARACTER
    FIELD ModifiedDate      AS DATETIME
    /* Add your additional fields here */
    INDEX idxPrimary IS PRIMARY UNIQUE {Entity}ID
    INDEX idxField1 Field1
    INDEX idxStatus Status.
    /* Add your additional indexes here */

DEFINE DATASET ds{Entity} {&REFERENCE-ONLY} FOR tt{Entity}.
```

---

## Step 2: DAO Interface

**File:** `/Business/DataAccess/I{Entity}DAO.cls`

```openedge
/* I{Entity}DAO.cls - Data Access Object Interface for {Entity} */

INTERFACE Business.DataAccess.I{Entity}DAO:
    
    /* Fetch single record by ID */
    METHOD PUBLIC VOID Fetch{Entity}(
        INPUT  i{Entity}ID AS INTEGER,
        OUTPUT DATASET ds{Entity} BY-REFERENCE).
    
    /* Fetch multiple records by criteria */
    METHOD PUBLIC VOID Fetch{Entity}ByStatus(
        INPUT  cStatus AS CHARACTER,
        OUTPUT DATASET ds{Entity} BY-REFERENCE).
    
    /* Add additional fetch methods as needed */
    /* Example: Fetch{Entity}ByDateRange, Fetch{Entity}ByCustomer, etc. */
    
    /* Save (Create/Update) */
    METHOD PUBLIC VOID Save{Entity}(
        INPUT-OUTPUT DATASET ds{Entity} BY-REFERENCE).
    
    /* Delete */
    METHOD PUBLIC VOID Delete{Entity}(
        INPUT i{Entity}ID AS INTEGER).
        
END INTERFACE.
```

---

## Step 3: DAO Implementation

**File:** `/Business/DataAccess/{Entity}DAO.cls`

```openedge
/* {Entity}DAO.cls - Data Access Object Implementation for {Entity} */

USING Business.DataAccess.I{Entity}DAO.
USING Progress.Lang.*.

CLASS Business.DataAccess.{Entity}DAO IMPLEMENTS I{Entity}DAO:
    
    {{Entity}Dataset.i &REFERENCE-ONLY=REFERENCE-ONLY}
    
    /* Fetch single record by ID */
    METHOD PUBLIC VOID Fetch{Entity}(
        INPUT  i{Entity}ID AS INTEGER,
        OUTPUT DATASET ds{Entity} BY-REFERENCE):
        
        DEFINE BUFFER b{Entity} FOR {Entity}.  /* Your database table name */
        
        EMPTY TEMP-TABLE tt{Entity}.
        
        FOR EACH b{Entity} NO-LOCK
            WHERE b{Entity}.{Entity}ID = i{Entity}ID:
            
            CREATE tt{Entity}.
            BUFFER-COPY b{Entity} TO tt{Entity}.
        END.
        
    END METHOD.
    
    /* Fetch by status */
    METHOD PUBLIC VOID Fetch{Entity}ByStatus(
        INPUT  cStatus AS CHARACTER,
        OUTPUT DATASET ds{Entity} BY-REFERENCE):
        
        DEFINE BUFFER b{Entity} FOR {Entity}.
        
        EMPTY TEMP-TABLE tt{Entity}.
        
        FOR EACH b{Entity} NO-LOCK
            WHERE b{Entity}.Status = cStatus:
            
            CREATE tt{Entity}.
            BUFFER-COPY b{Entity} TO tt{Entity}.
        END.
        
    END METHOD.
    
    /* Save (Create/Update) */
    METHOD PUBLIC VOID Save{Entity}(
        INPUT-OUTPUT DATASET ds{Entity} BY-REFERENCE):
        
        DEFINE BUFFER b{Entity} FOR {Entity}.
        DEFINE BUFFER btt{Entity} FOR tt{Entity}.
        
        DO TRANSACTION:
            
            FOR EACH btt{Entity}:
                
                FIND FIRST b{Entity} EXCLUSIVE-LOCK
                    WHERE b{Entity}.{Entity}ID = btt{Entity}.{Entity}ID
                    NO-ERROR.
                
                IF NOT AVAILABLE b{Entity} THEN DO:
                    /* New record */
                    CREATE b{Entity}.
                    ASSIGN b{Entity}.{Entity}ID = NEXT-VALUE({Entity}Seq).
                    ASSIGN btt{Entity}.{Entity}ID = b{Entity}.{Entity}ID.
                END.
                
                /* Copy data from temp-table to database */
                BUFFER-COPY btt{Entity} EXCEPT {Entity}ID TO b{Entity}.
            END.
            
        END. /* TRANSACTION */
        
    END METHOD.
    
    /* Delete */
    METHOD PUBLIC VOID Delete{Entity}(INPUT i{Entity}ID AS INTEGER):
        
        DEFINE BUFFER b{Entity} FOR {Entity}.
        
        DO TRANSACTION:
            
            FIND FIRST b{Entity} EXCLUSIVE-LOCK
                WHERE b{Entity}.{Entity}ID = i{Entity}ID
                NO-ERROR.
            
            IF AVAILABLE b{Entity} THEN
                DELETE b{Entity}.
                
        END. /* TRANSACTION */
        
    END METHOD.
    
END CLASS.
```

---

## Step 4: Business Entity Interface

**File:** `/Business/Entity/I{Entity}BE.cls`

```openedge
/* I{Entity}BE.cls - Business Entity Interface for {Entity} */

INTERFACE Business.Entity.I{Entity}BE:
    
    /* Get single record */
    METHOD PUBLIC VOID Get{Entity}(
        INPUT  i{Entity}ID AS INTEGER,
        OUTPUT DATASET ds{Entity} BY-REFERENCE).
    
    /* Get by criteria */
    METHOD PUBLIC VOID Get{Entity}ByStatus(
        INPUT  cStatus AS CHARACTER,
        OUTPUT DATASET ds{Entity} BY-REFERENCE).
    
    /* Save with validation */
    METHOD PUBLIC LOGICAL Save{Entity}(
        INPUT-OUTPUT DATASET ds{Entity} BY-REFERENCE,
        OUTPUT cErrorMessage AS CHARACTER).
    
    /* Delete with business rules */
    METHOD PUBLIC LOGICAL Delete{Entity}(
        INPUT  i{Entity}ID AS INTEGER,
        OUTPUT cErrorMessage AS CHARACTER).
        
END INTERFACE.
```

---

## Step 5: Business Entity Implementation

**File:** `/Business/Entity/{Entity}BE.cls`

```openedge
/* {Entity}BE.cls - Business Entity Implementation for {Entity} */

USING Business.Entity.I{Entity}BE.
USING Business.DataAccess.I{Entity}DAO.
USING Business.DataAccess.{Entity}DAO.
USING Progress.Lang.*.

CLASS Business.Entity.{Entity}BE IMPLEMENTS I{Entity}BE:
    
    {{Entity}Dataset.i &REFERENCE-ONLY=REFERENCE-ONLY}
    
    DEFINE PRIVATE VARIABLE oDAO AS I{Entity}DAO NO-UNDO.
    
    /* Constructor - creates DAO */
    CONSTRUCTOR PUBLIC {Entity}BE():
        oDAO = NEW {Entity}DAO().
    END CONSTRUCTOR.
    
    /* Constructor with DAO injection (for testing) */
    CONSTRUCTOR PUBLIC {Entity}BE(INPUT poDAO AS I{Entity}DAO):
        oDAO = poDAO.
    END CONSTRUCTOR.
    
    /* Get single record */
    METHOD PUBLIC VOID Get{Entity}(
        INPUT  i{Entity}ID AS INTEGER,
        OUTPUT DATASET ds{Entity} BY-REFERENCE):
        
        oDAO:Fetch{Entity}(i{Entity}ID, OUTPUT DATASET ds{Entity} BY-REFERENCE).
        
    END METHOD.
    
    /* Get by status */
    METHOD PUBLIC VOID Get{Entity}ByStatus(
        INPUT  cStatus AS CHARACTER,
        OUTPUT DATASET ds{Entity} BY-REFERENCE):
        
        IF cStatus = "" OR cStatus = ? THEN
            RETURN.
            
        oDAO:Fetch{Entity}ByStatus(cStatus, OUTPUT DATASET ds{Entity} BY-REFERENCE).
        
    END METHOD.
    
    /* Save with validation */
    METHOD PUBLIC LOGICAL Save{Entity}(
        INPUT-OUTPUT DATASET ds{Entity} BY-REFERENCE,
        OUTPUT cErrorMessage AS CHARACTER):
        
        DEFINE VARIABLE lSuccess AS LOGICAL NO-UNDO INITIAL TRUE.
        
        /* Validate before save */
        lSuccess = Validate{Entity}(INPUT-OUTPUT DATASET ds{Entity} BY-REFERENCE,
                                    OUTPUT cErrorMessage).
        
        IF NOT lSuccess THEN
            RETURN FALSE.
        
        /* Set audit fields */
        SetAuditFields(INPUT-OUTPUT DATASET ds{Entity} BY-REFERENCE).
        
        /* Save through DAO */
        oDAO:Save{Entity}(INPUT-OUTPUT DATASET ds{Entity} BY-REFERENCE).
        
        RETURN TRUE.
        
        CATCH eError AS Progress.Lang.Error:
            cErrorMessage = eError:GetMessage(1).
            RETURN FALSE.
        END CATCH.
        
    END METHOD.
    
    /* Delete with business rules */
    METHOD PUBLIC LOGICAL Delete{Entity}(
        INPUT  i{Entity}ID AS INTEGER,
        OUTPUT cErrorMessage AS CHARACTER):
        
        DEFINE BUFFER btt{Entity} FOR tt{Entity}.
        
        /* Fetch record to check business rules */
        oDAO:Fetch{Entity}(i{Entity}ID, OUTPUT DATASET ds{Entity} BY-REFERENCE).
        
        FIND FIRST btt{Entity} NO-LOCK NO-ERROR.
        
        IF NOT AVAILABLE btt{Entity} THEN DO:
            cErrorMessage = "{Entity} not found.".
            RETURN FALSE.
        END.
        
        /* Business rule examples - customize for your needs */
        IF btt{Entity}.Status = "POSTED" THEN DO:
            cErrorMessage = "Cannot delete posted {entity}.".
            RETURN FALSE.
        END.
        
        /* Add your business rules here */
        
        /* Delete through DAO */
        oDAO:Delete{Entity}(i{Entity}ID).
        
        RETURN TRUE.
        
        CATCH eError AS Progress.Lang.Error:
            cErrorMessage = eError:GetMessage(1).
            RETURN FALSE.
        END CATCH.
        
    END METHOD.
    
    /* Private validation method - customize for your entity */
    METHOD PRIVATE LOGICAL Validate{Entity}(
        INPUT-OUTPUT DATASET ds{Entity} BY-REFERENCE,
        OUTPUT cErrorMessage AS CHARACTER):
        
        DEFINE BUFFER btt{Entity} FOR tt{Entity}.
        
        FOR EACH btt{Entity}:
            
            /* Required field validation */
            IF btt{Entity}.Field1 = "" OR btt{Entity}.Field1 = ? THEN DO:
                cErrorMessage = "Field1 is required.".
                RETURN FALSE.
            END.
            
            /* Date validation */
            IF btt{Entity}.Field2 = ? THEN DO:
                cErrorMessage = "Field2 (date) is required.".
                RETURN FALSE.
            END.
            
            IF btt{Entity}.Field2 > TODAY THEN DO:
                cErrorMessage = "Date cannot be in the future.".
                RETURN FALSE.
            END.
            
            /* Status validation */
            IF LOOKUP(btt{Entity}.Status, "DRAFT,ACTIVE,INACTIVE,CLOSED") = 0 THEN DO:
                cErrorMessage = "Invalid status value.".
                RETURN FALSE.
            END.
            
            /* Numeric validation */
            IF btt{Entity}.Field3 < 0 THEN DO:
                cErrorMessage = "Field3 cannot be negative.".
                RETURN FALSE.
            END.
            
            /* Add your custom validation rules here */
            
        END.
        
        RETURN TRUE.
        
    END METHOD.
    
    /* Private method to set audit fields */
    METHOD PRIVATE VOID SetAuditFields(
        INPUT-OUTPUT DATASET ds{Entity} BY-REFERENCE):
        
        DEFINE BUFFER btt{Entity} FOR tt{Entity}.
        DEFINE VARIABLE cCurrentUser AS CHARACTER NO-UNDO.
        DEFINE VARIABLE dtNow AS DATETIME NO-UNDO.
        
        ASSIGN
            cCurrentUser = USERID("DICTDB")
            dtNow = NOW.
        
        FOR EACH btt{Entity}:
            
            IF btt{Entity}.{Entity}ID = 0 OR btt{Entity}.{Entity}ID = ? THEN DO:
                /* New record */
                ASSIGN
                    btt{Entity}.CreatedBy = cCurrentUser
                    btt{Entity}.CreatedDate = dtNow.
            END.
            
            /* Always update modified fields */
            ASSIGN
                btt{Entity}.ModifiedBy = cCurrentUser
                btt{Entity}.ModifiedDate = dtNow.
        END.
        
    END METHOD.
    
END CLASS.
```

---

## Step 6: Presentation Layer

**File:** `/Presentation/{Entity}Window.w`

```openedge
/* {Entity}Window.w - Presentation layer for {Entity} */

/* Include WITHOUT REFERENCE-ONLY parameter for presentation layer */
{{Entity}Dataset.i}

DEFINE VARIABLE o{Entity}BE AS Business.Entity.I{Entity}BE NO-UNDO.
DEFINE VARIABLE cErrorMessage AS CHARACTER NO-UNDO.
DEFINE VARIABLE lSuccess AS LOGICAL NO-UNDO.
DEFINE VARIABLE iCurrent{Entity}ID AS INTEGER NO-UNDO.

/* Initialize Business Entity */
o{Entity}BE = NEW Business.Entity.{Entity}BE().

/* Load record */
PROCEDURE Load{Entity}:
    DEFINE INPUT PARAMETER pi{Entity}ID AS INTEGER NO-UNDO.
    
    ASSIGN iCurrent{Entity}ID = pi{Entity}ID.
    
    o{Entity}BE:Get{Entity}(pi{Entity}ID, OUTPUT DATASET ds{Entity} BY-REFERENCE).
    
    /* Bind dataset to UI controls */
    /* Example: */
    /* BROWSE brw{Entity}:QUERY:ADD-NEW-FIELD("Field1", "CHARACTER").
       BROWSE brw{Entity}:ADD-BUFFER-FIELD(...).
       BROWSE brw{Entity}:QUERY:OPEN-QUERY(). */
    
END PROCEDURE.

/* Save record */
PROCEDURE Save{Entity}:
    
    /* Get data from UI into dataset */
    /* Your UI-to-dataset code here */
    
    lSuccess = o{Entity}BE:Save{Entity}(
        INPUT-OUTPUT DATASET ds{Entity} BY-REFERENCE,
        OUTPUT cErrorMessage).
    
    IF lSuccess THEN
        MESSAGE "{Entity} saved successfully." VIEW-AS ALERT-BOX INFORMATION.
    ELSE
        MESSAGE cErrorMessage VIEW-AS ALERT-BOX ERROR.
        
END PROCEDURE.

/* Delete record */
PROCEDURE Delete{Entity}:
    
    IF iCurrent{Entity}ID = 0 OR iCurrent{Entity}ID = ? THEN DO:
        MESSAGE "No {entity} selected." VIEW-AS ALERT-BOX WARNING.
        RETURN.
    END.
    
    MESSAGE "Are you sure you want to delete this {entity}?" 
        VIEW-AS ALERT-BOX QUESTION BUTTONS YES-NO 
        UPDATE lConfirm AS LOGICAL.
    
    IF NOT lConfirm THEN
        RETURN.
    
    lSuccess = o{Entity}BE:Delete{Entity}(iCurrent{Entity}ID, OUTPUT cErrorMessage).
    
    IF lSuccess THEN DO:
        MESSAGE "{Entity} deleted successfully." VIEW-AS ALERT-BOX INFORMATION.
        /* Refresh your list/browse */
    END.
    ELSE
        MESSAGE cErrorMessage VIEW-AS ALERT-BOX ERROR.
        
END PROCEDURE.

/* Search/List */
PROCEDURE List{Entity}ByStatus:
    DEFINE INPUT PARAMETER pcStatus AS CHARACTER NO-UNDO.
    
    o{Entity}BE:Get{Entity}ByStatus(pcStatus, OUTPUT DATASET ds{Entity} BY-REFERENCE).
    
    /* Bind to browse/grid */
    
END PROCEDURE.
```

---

## Step 7: Unit Testing (Optional)

**File:** `/Test/Mock/Mock{Entity}DAO.cls`

```openedge
/* Mock{Entity}DAO.cls - Mock for unit testing */

USING Business.DataAccess.I{Entity}DAO.

CLASS Test.Mock.Mock{Entity}DAO IMPLEMENTS I{Entity}DAO:
    
    {{Entity}Dataset.i &REFERENCE-ONLY=REFERENCE-ONLY}
    
    DEFINE PUBLIC PROPERTY SaveWasCalled AS LOGICAL NO-UNDO GET. SET.
    
    /* Implement methods with mock data - no database! */
    METHOD PUBLIC VOID Fetch{Entity}(...):
        CREATE tt{Entity}.
        /* Return mock data */
    END METHOD.
    
    METHOD PUBLIC VOID Save{Entity}(...):
        SaveWasCalled = TRUE.
    END METHOD.
    
    /* ... */
END CLASS.
```

**File:** `/Test/Unit/{Entity}BETest.cls`

```openedge
/* {Entity}BETest.cls - ABLUnit tests */

USING OpenEdge.Core.Assert.
USING Business.Entity.{Entity}BE.
USING Test.Mock.Mock{Entity}DAO.

CLASS Test.Unit.{Entity}BETest:
    
    DEFINE PRIVATE VARIABLE oMockDAO AS Mock{Entity}DAO NO-UNDO.
    DEFINE PRIVATE VARIABLE oBE AS {Entity}BE NO-UNDO.
    
    @Before.
    METHOD PUBLIC VOID setUp():
        oMockDAO = NEW Mock{Entity}DAO().
        oBE = NEW {Entity}BE(oMockDAO).  /* Inject mock */
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testSave_ValidData_ShouldSucceed():
        /* Write your test */
        Assert:IsTrue(condition).
    END METHOD.
    
END CLASS.
```

**For complete testing guide:** See [UNIT_TESTING_GUIDE.md](UNIT_TESTING_GUIDE.md)

---

## Quick Copy-Paste Checklist

When creating a new entity, follow this checklist:

### 1. Create Dataset Definition
- [ ] Copy `{Entity}Dataset.i` template
- [ ] Replace `{Entity}` with your entity name
- [ ] Update field definitions to match your table
- [ ] Add appropriate indexes
- [ ] Save to `/Business/DataDefinitions/`

### 2. Create DAO Interface
- [ ] Copy `I{Entity}DAO.cls` template
- [ ] Replace `{Entity}` with your entity name
- [ ] Add/remove methods as needed for your queries
- [ ] Save to `/Business/DataAccess/`

### 3. Create DAO Implementation
- [ ] Copy `{Entity}DAO.cls` template
- [ ] Replace `{Entity}` with your entity name
- [ ] Update buffer names to match your database table
- [ ] Implement all methods from interface
- [ ] Add any custom query methods
- [ ] Save to `/Business/DataAccess/`

### 4. Create BE Interface
- [ ] Copy `I{Entity}BE.cls` template
- [ ] Replace `{Entity}` with your entity name
- [ ] Add/remove methods as needed
- [ ] Save to `/Business/Entity/`

### 5. Create BE Implementation
- [ ] Copy `{Entity}BE.cls` template
- [ ] Replace `{Entity}` with your entity name
- [ ] Customize validation rules
- [ ] Customize business rules for delete
- [ ] Add any calculation methods needed
- [ ] Save to `/Business/Entity/`

### 6. Create Presentation Layer
- [ ] Copy `{Entity}Window.w` template
- [ ] Replace `{Entity}` with your entity name
- [ ] Implement UI-to-dataset binding
- [ ] Add any additional UI procedures
- [ ] Save to `/Presentation/`

### 7. Create Unit Tests (Optional)
- [ ] Copy `Mock{Entity}DAO.cls` template
- [ ] Copy `{Entity}BETest.cls` template
- [ ] Replace `{Entity}` with your entity name
- [ ] Write test methods for validation and business rules
- [ ] Run tests to verify
- [ ] Save to `/Test/Mock/` and `/Test/Unit/`

---

## Find and Replace Guide

Use your editor's find/replace feature with these patterns:

| Find | Replace With | Example |
|------|--------------|---------|
| `{Entity}` | YourEntityName | Customer |
| `{entity}` | yourentityname | customer |
| `Field1`, `Field2`, `Field3` | Your actual field names | CustomerName, OrderDate, Amount |

---

## Common Customizations

### Add Additional Query Methods

In **DAO Interface**:
```openedge
METHOD PUBLIC VOID Fetch{Entity}ByDateRange(
    INPUT dtStartDate AS DATE,
    INPUT dtEndDate AS DATE,
    OUTPUT DATASET ds{Entity} BY-REFERENCE).
```

In **DAO Implementation**:
```openedge
METHOD PUBLIC VOID Fetch{Entity}ByDateRange(
    INPUT dtStartDate AS DATE,
    INPUT dtEndDate AS DATE,
    OUTPUT DATASET ds{Entity} BY-REFERENCE):
    
    DEFINE BUFFER b{Entity} FOR {Entity}.
    
    EMPTY TEMP-TABLE tt{Entity}.
    
    FOR EACH b{Entity} NO-LOCK
        WHERE b{Entity}.YourDateField >= dtStartDate
        AND b{Entity}.YourDateField <= dtEndDate:
        
        CREATE tt{Entity}.
        BUFFER-COPY b{Entity} TO tt{Entity}.
    END.
    
END METHOD.
```

### Add Calculated Fields

In **BE Implementation**:
```openedge
METHOD PUBLIC VOID CalculateTotal(
    INPUT-OUTPUT DATASET ds{Entity} BY-REFERENCE):
    
    DEFINE BUFFER btt{Entity} FOR tt{Entity}.
    
    FOR EACH btt{Entity}:
        ASSIGN btt{Entity}.TotalField = 
            btt{Entity}.Field1 * btt{Entity}.Field2.
    END.
    
END METHOD.
```

### Add Complex Business Rules

In **BE Implementation** Validate method:
```openedge
/* Check for duplicates */
DEFINE BUFFER bCheck{Entity} FOR {Entity}.

FIND FIRST bCheck{Entity} NO-LOCK
    WHERE bCheck{Entity}.UniqueField = btt{Entity}.UniqueField
    AND bCheck{Entity}.{Entity}ID <> btt{Entity}.{Entity}ID
    NO-ERROR.

IF AVAILABLE bCheck{Entity} THEN DO:
    cErrorMessage = "Duplicate value found for UniqueField.".
    RETURN FALSE.
END.
```

---

## Example: Creating a Customer Entity

Here's a complete example showing the replacements:

**CustomerDataset.i:**
```openedge
{CustomerDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}

DEFINE TEMP-TABLE ttCustomer NO-UNDO {&REFERENCE-ONLY}
    FIELD CustomerID    AS INTEGER
    FIELD CustomerName  AS CHARACTER
    FIELD Email         AS CHARACTER
    FIELD Phone         AS CHARACTER
    FIELD Status        AS CHARACTER
    FIELD CreatedBy     AS CHARACTER
    FIELD CreatedDate   AS DATETIME
    FIELD ModifiedBy    AS CHARACTER
    FIELD ModifiedDate  AS DATETIME
    INDEX idxPrimary IS PRIMARY UNIQUE CustomerID
    INDEX idxName CustomerName
    INDEX idxEmail Email.

DEFINE DATASET dsCustomer {&REFERENCE-ONLY} FOR ttCustomer.
```

**Files to create:**
- ICustomerDAO.cls - DAO interface
- CustomerDAO.cls - DAO implementation  
- ICustomerBE.cls - BE interface
- CustomerBE.cls - BE implementation
- CustomerWindow.w - Presentation
- MockCustomerDAO.cls - Mock for testing
- CustomerBETest.cls - Unit tests

Follow the same pattern, replacing {Entity} with Customer

---

## Tips for Success

1. **Start Simple** - Begin with basic CRUD, add complexity as needed
2. **Test as You Go** - Test each layer independently
3. **Consistent Naming** - Stick to the naming conventions
4. **Document Changes** - Note any deviations from the template
5. **Reuse Validation** - Create common validation utilities for repeated patterns
6. **Version Control** - Commit after each working layer

---

## Common Patterns for Validation

### Required Field Check
```openedge
IF btt{Entity}.FieldName = "" OR btt{Entity}.FieldName = ? THEN DO:
    cErrorMessage = "FieldName is required.".
    RETURN FALSE.
END.
```

### Range Check
```openedge
IF btt{Entity}.Amount < 0 OR btt{Entity}.Amount > 9999999 THEN DO:
    cErrorMessage = "Amount must be between 0 and 9,999,999.".
    RETURN FALSE.
END.
```

### Format Check (Email example)
```openedge
IF INDEX(btt{Entity}.Email, "@") = 0 THEN DO:
    cErrorMessage = "Invalid email format.".
    RETURN FALSE.
END.
```

### Status/Enum Check
```openedge
IF LOOKUP(btt{Entity}.Status, "ACTIVE,INACTIVE,PENDING") = 0 THEN DO:
    cErrorMessage = "Invalid status. Must be ACTIVE, INACTIVE, or PENDING.".
    RETURN FALSE.
END.
```

### Date Range Check
```openedge
IF btt{Entity}.StartDate > btt{Entity}.EndDate THEN DO:
    cErrorMessage = "Start date cannot be after end date.".
    RETURN FALSE.
END.
```

---

## What to Customize for Each Entity

| Component | What to Change |
|-----------|----------------|
| Dataset | Field names, data types, indexes |
| DAO | Buffer names (must match DB table), query criteria |
| BE Validation | Business rules specific to the entity |
| BE Delete Rules | Conditions when deletion is not allowed |
| Presentation | UI controls, layout, user interactions |

---

This template provides everything you need to implement OERA for any table in your legacy application. Just copy, customize, and go!
