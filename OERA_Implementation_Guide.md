# OpenEdge OERA Implementation Guide

## Architecture Overview

OERA (OpenEdge Reference Architecture) organizes your application into distinct layers:

1. **Presentation Layer** - UI components (screens, web services)
2. **Business Logic Layer** - Business entities and business tasks
3. **Data Access Layer** - Data access objects (DAOs)

## Core OERA Principles

### 1. Separation of Concerns
- Each layer has a single, well-defined responsibility
- Presentation handles UI logic only
- Business logic contains validation, calculations, and business rules
- Data access handles CRUD operations and queries

### 2. Dataset Communication
- Use TEMP-TABLES and DATASETS for inter-layer communication
- Pass datasets BY-REFERENCE in parameters
- Define datasets as REFERENCE-ONLY to avoid unnecessary data copies
- This ensures performance and maintains data integrity across layers

### 3. Object-Oriented Design
- Use classes for encapsulation and reusability
- Apply SOLID principles throughout

## SOLID Principles in OpenEdge

### S - Single Responsibility Principle
Each class should have one reason to change. Example: Invoice BE handles invoice business logic only.

### O - Open/Closed Principle
Classes should be open for extension but closed for modification. Use inheritance and interfaces.

### L - Liskov Substitution Principle
Derived classes must be substitutable for their base classes.

### I - Interface Segregation Principle
Many specific interfaces are better than one general-purpose interface.

### D - Dependency Inversion Principle
Depend on abstractions, not concrete implementations. Use interfaces for DAOs and BEs.

---

## Invoice Example - Complete Implementation

### Step 1: Define Dataset Structure

```openedge
/* InvoiceDataset.i - Include file for dataset definition
   
   Usage:
   {InvoiceDataset.i}                    - Without REFERENCE-ONLY (for Presentation layer)
   {InvoiceDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}  - With REFERENCE-ONLY (for DAO/BE layers)
*/

&IF "{&REFERENCE-ONLY}" = "" &THEN
    &SCOPED-DEFINE REFERENCE-ONLY
&ENDIF

DEFINE TEMP-TABLE ttInvoice NO-UNDO {&REFERENCE-ONLY}
    FIELD InvoiceID     AS INTEGER
    FIELD CustomerID    AS INTEGER
    FIELD InvoiceDate   AS DATE
    FIELD TotalAmount   AS DECIMAL
    FIELD Status        AS CHARACTER
    FIELD Description   AS CHARACTER
    FIELD CreatedBy     AS CHARACTER
    FIELD CreatedDate   AS DATETIME
    FIELD ModifiedBy    AS CHARACTER
    FIELD ModifiedDate  AS DATETIME
    INDEX idxPrimary IS PRIMARY UNIQUE InvoiceID
    INDEX idxCustomer CustomerID
    INDEX idxDate InvoiceDate
    INDEX idxStatus Status.

DEFINE DATASET dsInvoice {&REFERENCE-ONLY} FOR ttInvoice.
```

### Step 2: Data Access Layer (DAO)

**DAO Interface**

```openedge
/* IInvoiceDAO.cls - Interface for Invoice DAO */

INTERFACE Business.DataAccess.IInvoiceDAO:
    
    METHOD PUBLIC VOID FetchInvoice(
        INPUT  iInvoiceID AS INTEGER,
        OUTPUT DATASET dsInvoice BY-REFERENCE).
    
    METHOD PUBLIC VOID FetchInvoicesByCustomer(
        INPUT  iCustomerID AS INTEGER,
        OUTPUT DATASET dsInvoice BY-REFERENCE).
    
    METHOD PUBLIC VOID FetchInvoicesByDateRange(
        INPUT  dtStartDate AS DATE,
        INPUT  dtEndDate AS DATE,
        OUTPUT DATASET dsInvoice BY-REFERENCE).
    
    METHOD PUBLIC VOID SaveInvoice(
        INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE).
    
    METHOD PUBLIC VOID DeleteInvoice(
        INPUT iInvoiceID AS INTEGER).
        
END INTERFACE.
```

**DAO Implementation**

```openedge
/* InvoiceDAO.cls - Data Access Object Implementation */

USING Business.DataAccess.IInvoiceDAO.
USING Progress.Lang.*.

CLASS Business.DataAccess.InvoiceDAO IMPLEMENTS IInvoiceDAO:
    
    {InvoiceDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}
    
    /* Fetch single invoice */
    METHOD PUBLIC VOID FetchInvoice(
        INPUT  iInvoiceID AS INTEGER,
        OUTPUT DATASET dsInvoice BY-REFERENCE):
        
        DEFINE BUFFER bInvoice FOR Invoice.
        
        EMPTY TEMP-TABLE ttInvoice.
        
        /* Fetch invoice */
        FOR EACH bInvoice NO-LOCK
            WHERE bInvoice.InvoiceID = iInvoiceID:
            
            CREATE ttInvoice.
            BUFFER-COPY bInvoice TO ttInvoice.
        END.
        
    END METHOD.
    
    /* Fetch invoices by customer */
    METHOD PUBLIC VOID FetchInvoicesByCustomer(
        INPUT  iCustomerID AS INTEGER,
        OUTPUT DATASET dsInvoice BY-REFERENCE):
        
        DEFINE BUFFER bInvoice FOR Invoice.
        
        EMPTY TEMP-TABLE ttInvoice.
        
        FOR EACH bInvoice NO-LOCK
            WHERE bInvoice.CustomerID = iCustomerID:
            
            CREATE ttInvoice.
            BUFFER-COPY bInvoice TO ttInvoice.
        END.
        
    END METHOD.
    
    /* Fetch invoices by date range */
    METHOD PUBLIC VOID FetchInvoicesByDateRange(
        INPUT  dtStartDate AS DATE,
        INPUT  dtEndDate AS DATE,
        OUTPUT DATASET dsInvoice BY-REFERENCE):
        
        DEFINE BUFFER bInvoice FOR Invoice.
        
        EMPTY TEMP-TABLE ttInvoice.
        
        FOR EACH bInvoice NO-LOCK
            WHERE bInvoice.InvoiceDate >= dtStartDate
            AND bInvoice.InvoiceDate <= dtEndDate:
            
            CREATE ttInvoice.
            BUFFER-COPY bInvoice TO ttInvoice.
        END.
        
    END METHOD.
    
    /* Save invoice (Create/Update) */
    METHOD PUBLIC VOID SaveInvoice(
        INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE):
        
        DEFINE BUFFER bInvoice FOR Invoice.
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        DO TRANSACTION:
            
            /* Process invoice */
            FOR EACH bttInvoice:
                
                FIND FIRST bInvoice EXCLUSIVE-LOCK
                    WHERE bInvoice.InvoiceID = bttInvoice.InvoiceID
                    NO-ERROR.
                
                IF NOT AVAILABLE bInvoice THEN DO:
                    /* New invoice */
                    CREATE bInvoice.
                    ASSIGN bInvoice.InvoiceID = NEXT-VALUE(InvoiceSeq).
                    ASSIGN bttInvoice.InvoiceID = bInvoice.InvoiceID.
                END.
                
                /* Copy data from temp-table to database */
                BUFFER-COPY bttInvoice EXCEPT InvoiceID TO bInvoice.
            END.
            
        END. /* TRANSACTION */
        
    END METHOD.
    
    /* Delete invoice */
    METHOD PUBLIC VOID DeleteInvoice(INPUT iInvoiceID AS INTEGER):
        
        DEFINE BUFFER bInvoice FOR Invoice.
        
        DO TRANSACTION:
            
            /* Delete invoice */
            FIND FIRST bInvoice EXCLUSIVE-LOCK
                WHERE bInvoice.InvoiceID = iInvoiceID
                NO-ERROR.
            
            IF AVAILABLE bInvoice THEN
                DELETE bInvoice.
                
        END. /* TRANSACTION */
        
    END METHOD.
    
END CLASS.
```

### Step 3: Business Entity Layer (BE)

**Business Entity Interface**

```openedge
/* IInvoiceBE.cls - Interface for Invoice Business Entity */

INTERFACE Business.Entity.IInvoiceBE:
    
    METHOD PUBLIC VOID GetInvoice(
        INPUT  iInvoiceID AS INTEGER,
        OUTPUT DATASET dsInvoice BY-REFERENCE).
    
    METHOD PUBLIC VOID GetInvoicesByCustomer(
        INPUT  iCustomerID AS INTEGER,
        OUTPUT DATASET dsInvoice BY-REFERENCE).
    
    METHOD PUBLIC VOID GetInvoicesByDateRange(
        INPUT  dtStartDate AS DATE,
        INPUT  dtEndDate AS DATE,
        OUTPUT DATASET dsInvoice BY-REFERENCE).
    
    METHOD PUBLIC LOGICAL SaveInvoice(
        INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
        OUTPUT cErrorMessage AS CHARACTER).
    
    METHOD PUBLIC LOGICAL DeleteInvoice(
        INPUT  iInvoiceID AS INTEGER,
        OUTPUT cErrorMessage AS CHARACTER).
    
    METHOD PUBLIC DECIMAL CalculateInvoiceTotal(
        INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE).
        
END INTERFACE.
```

**Business Entity Implementation**

```openedge
/* InvoiceBE.cls - Business Entity Implementation */

USING Business.Entity.IInvoiceBE.
USING Business.DataAccess.IInvoiceDAO.
USING Business.DataAccess.InvoiceDAO.
USING Progress.Lang.*.

CLASS Business.Entity.InvoiceBE IMPLEMENTS IInvoiceBE:
    
    {InvoiceDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}
    
    DEFINE PRIVATE VARIABLE oDAO AS IInvoiceDAO NO-UNDO.
    
    /* Constructor - Dependency Injection */
    CONSTRUCTOR PUBLIC InvoiceBE():
        oDAO = NEW InvoiceDAO().
    END CONSTRUCTOR.
    
    /* Constructor with DAO injection for testing */
    CONSTRUCTOR PUBLIC InvoiceBE(INPUT poDAO AS IInvoiceDAO):
        oDAO = poDAO.
    END CONSTRUCTOR.
    
    /* Get single invoice */
    METHOD PUBLIC VOID GetInvoice(
        INPUT  iInvoiceID AS INTEGER,
        OUTPUT DATASET dsInvoice BY-REFERENCE):
        
        oDAO:FetchInvoice(iInvoiceID, OUTPUT DATASET dsInvoice BY-REFERENCE).
        
    END METHOD.
    
    /* Get invoices by customer */
    METHOD PUBLIC VOID GetInvoicesByCustomer(
        INPUT  iCustomerID AS INTEGER,
        OUTPUT DATASET dsInvoice BY-REFERENCE):
        
        IF iCustomerID <= 0 THEN
            RETURN.
            
        oDAO:FetchInvoicesByCustomer(iCustomerID, OUTPUT DATASET dsInvoice BY-REFERENCE).
        
    END METHOD.
    
    /* Get invoices by date range */
    METHOD PUBLIC VOID GetInvoicesByDateRange(
        INPUT  dtStartDate AS DATE,
        INPUT  dtEndDate AS DATE,
        OUTPUT DATASET dsInvoice BY-REFERENCE):
        
        IF dtStartDate = ? OR dtEndDate = ? THEN
            RETURN.
            
        IF dtStartDate > dtEndDate THEN
            RETURN.
            
        oDAO:FetchInvoicesByDateRange(dtStartDate, dtEndDate, 
            OUTPUT DATASET dsInvoice BY-REFERENCE).
        
    END METHOD.
    
    /* Save invoice with business validation */
    METHOD PUBLIC LOGICAL SaveInvoice(
        INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
        OUTPUT cErrorMessage AS CHARACTER):
        
        DEFINE VARIABLE lSuccess AS LOGICAL NO-UNDO INITIAL TRUE.
        
        /* Validate before save */
        lSuccess = ValidateInvoice(INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
                                   OUTPUT cErrorMessage).
        
        IF NOT lSuccess THEN
            RETURN FALSE.
        
        /* Calculate totals */
        CalculateInvoiceTotal(INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE).
        
        /* Set audit fields */
        SetAuditFields(INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE).
        
        /* Save through DAO */
        oDAO:SaveInvoice(INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE).
        
        RETURN TRUE.
        
        CATCH eError AS Progress.Lang.Error:
            cErrorMessage = eError:GetMessage(1).
            RETURN FALSE.
        END CATCH.
        
    END METHOD.
    
    /* Delete invoice with business rules */
    METHOD PUBLIC LOGICAL DeleteInvoice(
        INPUT  iInvoiceID AS INTEGER,
        OUTPUT cErrorMessage AS CHARACTER):
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        /* Fetch invoice to check business rules */
        oDAO:FetchInvoice(iInvoiceID, OUTPUT DATASET dsInvoice BY-REFERENCE).
        
        FIND FIRST bttInvoice NO-LOCK NO-ERROR.
        
        IF NOT AVAILABLE bttInvoice THEN DO:
            cErrorMessage = "Invoice not found.".
            RETURN FALSE.
        END.
        
        /* Business rule: Can't delete posted invoices */
        IF bttInvoice.Status = "POSTED" THEN DO:
            cErrorMessage = "Cannot delete posted invoices.".
            RETURN FALSE.
        END.
        
        /* Business rule: Can't delete paid invoices */
        IF bttInvoice.Status = "PAID" THEN DO:
            cErrorMessage = "Cannot delete paid invoices.".
            RETURN FALSE.
        END.
        
        /* Delete through DAO */
        oDAO:DeleteInvoice(iInvoiceID).
        
        RETURN TRUE.
        
        CATCH eError AS Progress.Lang.Error:
            cErrorMessage = eError:GetMessage(1).
            RETURN FALSE.
        END CATCH.
        
    END METHOD.
    
    /* Calculate or validate invoice total */
    METHOD PUBLIC DECIMAL CalculateInvoiceTotal(
        INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE):
        
        DEFINE VARIABLE dTotal AS DECIMAL NO-UNDO.
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        FOR EACH bttInvoice:
            dTotal = bttInvoice.TotalAmount.
        END.
        
        RETURN dTotal.
        
    END METHOD.
    
    /* Private validation method */
    METHOD PRIVATE LOGICAL ValidateInvoice(
        INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
        OUTPUT cErrorMessage AS CHARACTER):
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        FOR EACH bttInvoice:
            
            /* Validate customer */
            IF bttInvoice.CustomerID <= 0 THEN DO:
                cErrorMessage = "Invalid Customer ID.".
                RETURN FALSE.
            END.
            
            /* Validate date */
            IF bttInvoice.InvoiceDate = ? THEN DO:
                cErrorMessage = "Invoice date is required.".
                RETURN FALSE.
            END.
            
            /* Validate date not in future */
            IF bttInvoice.InvoiceDate > TODAY THEN DO:
                cErrorMessage = "Invoice date cannot be in the future.".
                RETURN FALSE.
            END.
            
            /* Validate status */
            IF LOOKUP(bttInvoice.Status, "DRAFT,POSTED,PAID,VOID") = 0 THEN DO:
                cErrorMessage = "Invalid invoice status.".
                RETURN FALSE.
            END.
            
            /* Validate total amount */
            IF bttInvoice.TotalAmount < 0 THEN DO:
                cErrorMessage = "Total amount cannot be negative.".
                RETURN FALSE.
            END.
        END.
        
        RETURN TRUE.
        
    END METHOD.
    
    /* Private method to set audit fields */
    METHOD PRIVATE VOID SetAuditFields(
        INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE):
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        DEFINE VARIABLE cCurrentUser AS CHARACTER NO-UNDO.
        DEFINE VARIABLE dtNow AS DATETIME NO-UNDO.
        
        ASSIGN
            cCurrentUser = USERID("DICTDB")
            dtNow = NOW.
        
        FOR EACH bttInvoice:
            
            IF bttInvoice.InvoiceID = 0 THEN DO:
                /* New record */
                ASSIGN
                    bttInvoice.CreatedBy = cCurrentUser
                    bttInvoice.CreatedDate = dtNow.
            END.
            
            /* Always update modified fields */
            ASSIGN
                bttInvoice.ModifiedBy = cCurrentUser
                bttInvoice.ModifiedDate = dtNow.
        END.
        
    END METHOD.
    
END CLASS.
```

### Step 4: Business Task Layer (Optional - for complex operations)

```openedge
/* InvoicePostingTask.cls - Business Task for complex operations */

USING Business.Entity.IInvoiceBE.
USING Business.Entity.InvoiceBE.
USING Progress.Lang.*.

CLASS Business.Task.InvoicePostingTask:
    
    {InvoiceDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}
    
    DEFINE PRIVATE VARIABLE oInvoiceBE AS IInvoiceBE NO-UNDO.
    
    CONSTRUCTOR PUBLIC InvoicePostingTask():
        oInvoiceBE = NEW InvoiceBE().
    END CONSTRUCTOR.
    
    /* Post invoice - complex business operation */
    METHOD PUBLIC LOGICAL PostInvoice(
        INPUT  iInvoiceID AS INTEGER,
        OUTPUT cErrorMessage AS CHARACTER):
        
        DEFINE VARIABLE lSuccess AS LOGICAL NO-UNDO.
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        /* Get invoice */
        oInvoiceBE:GetInvoice(iInvoiceID, OUTPUT DATASET dsInvoice BY-REFERENCE).
        
        FIND FIRST bttInvoice EXCLUSIVE-LOCK NO-ERROR.
        
        IF NOT AVAILABLE bttInvoice THEN DO:
            cErrorMessage = "Invoice not found.".
            RETURN FALSE.
        END.
        
        /* Business rule: Can only post draft invoices */
        IF bttInvoice.Status <> "DRAFT" THEN DO:
            cErrorMessage = "Only draft invoices can be posted.".
            RETURN FALSE.
        END.
        
        /* Validate invoice total */
        IF bttInvoice.TotalAmount <= 0 THEN DO:
            cErrorMessage = "Invoice total must be greater than zero.".
            RETURN FALSE.
        END.
        
        /* Change status */
        ASSIGN bttInvoice.Status = "POSTED".
        
        /* Save */
        lSuccess = oInvoiceBE:SaveInvoice(
            INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
            OUTPUT cErrorMessage).
        
        IF lSuccess THEN DO:
            /* Additional posting tasks */
            /* - Create accounting entries */
            /* - Update inventory */
            /* - Send notifications */
        END.
        
        RETURN lSuccess.
        
    END METHOD.
    
END CLASS.
```

### Step 5: Presentation Layer

```openedge
/* InvoiceWindow.w - Window or procedure */

/* Include WITHOUT REFERENCE-ONLY parameter for presentation layer */
{InvoiceDataset.i}

DEFINE VARIABLE oInvoiceBE AS Business.Entity.IInvoiceBE NO-UNDO.
DEFINE VARIABLE cErrorMessage AS CHARACTER NO-UNDO.
DEFINE VARIABLE lSuccess AS LOGICAL NO-UNDO.

/* Initialize */
oInvoiceBE = NEW Business.Entity.InvoiceBE().

/* Load invoice */
PROCEDURE LoadInvoice:
    DEFINE INPUT PARAMETER iInvoiceID AS INTEGER NO-UNDO.
    
    oInvoiceBE:GetInvoice(iInvoiceID, OUTPUT DATASET dsInvoice BY-REFERENCE).
    
    /* Bind dataset to UI controls */
    /* ... UI binding code ... */
    
END PROCEDURE.

/* Save invoice */
PROCEDURE SaveInvoice:
    
    /* Get data from UI into dataset */
    /* ... UI to dataset code ... */
    
    lSuccess = oInvoiceBE:SaveInvoice(
        INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
        OUTPUT cErrorMessage).
    
    IF lSuccess THEN
        MESSAGE "Invoice saved successfully." VIEW-AS ALERT-BOX INFORMATION.
    ELSE
        MESSAGE cErrorMessage VIEW-AS ALERT-BOX ERROR.
        
END PROCEDURE.
```

---

## Dataset Include File Strategy

### Single Include File with Parameter

We use a **single include file** with an **include parameter** to control the REFERENCE-ONLY keyword:

```openedge
/* InvoiceDataset.i */
&IF "{&REFERENCE-ONLY}" = "" &THEN
    &SCOPED-DEFINE REFERENCE-ONLY
&ENDIF

DEFINE TEMP-TABLE ttInvoice NO-UNDO {&REFERENCE-ONLY}
    ...
DEFINE DATASET dsInvoice {&REFERENCE-ONLY} FOR ttInvoice.
```

### How to Use:

**In DAO and BE Layers (with REFERENCE-ONLY):**
```openedge
{InvoiceDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}
```
This passes "REFERENCE-ONLY" as the value, so the definition becomes:
```openedge
DEFINE TEMP-TABLE ttInvoice NO-UNDO REFERENCE-ONLY ...
DEFINE DATASET dsInvoice REFERENCE-ONLY FOR ttInvoice.
```

**In Presentation Layer (without REFERENCE-ONLY):**
```openedge
{InvoiceDataset.i}
```
No parameter passed, so `{&REFERENCE-ONLY}` expands to empty string:
```openedge
DEFINE TEMP-TABLE ttInvoice NO-UNDO ...
DEFINE DATASET dsInvoice FOR ttInvoice.
```

### Benefits of This Approach:

1. **Single Source of Truth** - Only one include file to maintain
2. **Consistent Schema** - All layers use identical structure
3. **Clear Intent** - The include statement shows which layer you're in
4. **Easy to Template** - Copy/paste pattern for new entities
5. **No Duplication** - Avoids maintaining separate include files

### Layer-by-Layer Usage:

| Layer | Include Statement | Result |
|-------|------------------|--------|
| Presentation | `{InvoiceDataset.i}` | No REFERENCE-ONLY (owns data) |
| Business Entity | `{InvoiceDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}` | With REFERENCE-ONLY |
| Business Task | `{InvoiceDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}` | With REFERENCE-ONLY |
| Data Access | `{InvoiceDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}` | With REFERENCE-ONLY |

---

## Step 6: Unit Testing (Optional but Recommended)

### Why Interfaces Enable Testing

Interfaces allow you to create mock implementations for testing business logic without database access.

**Mock DAO Example:**

```openedge
/* MockInvoiceDAO.cls - For testing only */

USING Business.DataAccess.IInvoiceDAO.

CLASS Test.Mock.MockInvoiceDAO IMPLEMENTS IInvoiceDAO:
    
    {InvoiceDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}
    
    DEFINE PUBLIC PROPERTY SaveWasCalled AS LOGICAL NO-UNDO GET. SET.
    
    METHOD PUBLIC VOID FetchInvoice(
        INPUT  iInvoiceID AS INTEGER,
        OUTPUT DATASET dsInvoice BY-REFERENCE):
        
        /* Return mock data - no database! */
        CREATE ttInvoice.
        ASSIGN
            ttInvoice.InvoiceID = iInvoiceID
            ttInvoice.CustomerID = 100
            ttInvoice.InvoiceDate = TODAY
            ttInvoice.TotalAmount = 1000.00
            ttInvoice.Status = "DRAFT".
    END METHOD.
    
    METHOD PUBLIC VOID SaveInvoice(
        INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE):
        SaveWasCalled = TRUE.
        /* No database access! */
    END METHOD.
    
    /* Implement other methods similarly */
    
END CLASS.
```

**Test Class Example:**

```openedge
/* InvoiceBETest.cls - ABLUnit test */

USING Progress.Lang.*.
USING OpenEdge.Core.Assert.
USING Business.Entity.InvoiceBE.
USING Test.Mock.MockInvoiceDAO.

CLASS Test.Unit.InvoiceBETest:
    
    {InvoiceDataset.i}
    
    DEFINE PRIVATE VARIABLE oMockDAO AS MockInvoiceDAO NO-UNDO.
    DEFINE PRIVATE VARIABLE oBE AS InvoiceBE NO-UNDO.
    
    @Before.
    METHOD PUBLIC VOID setUp():
        oMockDAO = NEW MockInvoiceDAO().
        oBE = NEW InvoiceBE(oMockDAO).  /* Inject mock! */
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testSaveInvoice_MissingDate_ShouldFail():
        DEFINE VARIABLE lResult AS LOGICAL NO-UNDO.
        DEFINE VARIABLE cError AS CHARACTER NO-UNDO.
        
        /* Arrange */
        CREATE ttInvoice.
        ASSIGN
            ttInvoice.InvoiceDate = ?  /* Invalid */
            ttInvoice.CustomerID = 123
            ttInvoice.Status = "DRAFT".
        
        /* Act */
        lResult = oBE:SaveInvoice(
            INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
            OUTPUT cError).
        
        /* Assert */
        Assert:IsFalse(lResult).
        Assert:Equals("Invoice date is required.", cError).
        
        /* Verify DAO not called (validation failed early) */
        Assert:IsFalse(oMockDAO:SaveWasCalled).
    END METHOD.
    
END CLASS.
```

**For complete testing guide, see:** [UNIT_TESTING_GUIDE.md](UNIT_TESTING_GUIDE.md)

---

## Template Checklist for New Functionality

When implementing new functionality, follow this checklist:

### 1. Define Data Structure
- [ ] Create include file with TEMP-TABLEs using `{&REFERENCE-ONLY}` placeholder
- [ ] Define DATASET using `{&REFERENCE-ONLY}` placeholder
- [ ] Add preprocessor logic to handle empty parameter
- [ ] Define indexes and relationships
- [ ] Document all fields and usage

### 2. Create DAO Interface and Class
- [ ] Create `I{Entity}DAO.cls` interface
- [ ] Define Fetch methods (OUTPUT DATASET BY-REFERENCE)
- [ ] Define Save method (INPUT-OUTPUT DATASET BY-REFERENCE)
- [ ] Define Delete method
- [ ] Create `{Entity}DAO.cls` implementation
- [ ] Include dataset with `{DatasetName.i &REFERENCE-ONLY=REFERENCE-ONLY}`
- [ ] Implement all interface methods
- [ ] Use buffers for all database access
- [ ] Wrap saves in transactions
- [ ] Handle errors appropriately

### 3. Create BE Interface and Class
- [ ] Create `I{Entity}BE.cls` interface
- [ ] Define Get methods
- [ ] Define Save with validation (returns LOGICAL + error message)
- [ ] Define Delete with business rules (returns LOGICAL + error message)
- [ ] Create `{Entity}BE.cls` implementation
- [ ] Include dataset with `{DatasetName.i &REFERENCE-ONLY=REFERENCE-ONLY}`
- [ ] Implement all interface methods
- [ ] Inject DAO through constructor (Dependency Inversion)
- [ ] Add constructor overload for testing (accepts IDao parameter)
- [ ] Add private validation methods
- [ ] Add private audit field methods

### 4. Create Business Task (if needed)
- [ ] Include dataset with `{DatasetName.i &REFERENCE-ONLY=REFERENCE-ONLY}`
- [ ] Complex multi-entity operations
- [ ] Workflow management
- [ ] Orchestrate multiple BEs

### 5. Presentation Layer
- [ ] Include dataset with `{DatasetName.i}` (NO parameter - no REFERENCE-ONLY)
- [ ] Keep UI logic separate
- [ ] Call BE methods only
- [ ] Handle errors from BE layer
- [ ] Never access DAO directly

### 6. Testing Considerations
- [ ] Test DAO with mock data
- [ ] Test BE with mock DAO
- [ ] Test validation rules
- [ ] Test business rules
- [ ] Integration testing

---

## Key Benefits of This Approach

1. **Maintainability**: Clear separation makes changes easier
2. **Testability**: Interfaces allow mocking and unit testing
3. **Reusability**: Business logic can be used by multiple UIs
4. **Performance**: BY-REFERENCE and REFERENCE-ONLY minimize data copying
5. **Scalability**: Layers can be deployed separately (AppServer)
6. **Team Development**: Multiple developers can work on different layers

---

## Common Pitfalls to Avoid

1. **DON'T** access database directly from presentation layer
2. **DON'T** put business logic in DAOs
3. **DON'T** put UI logic in Business Entities
4. **DON'T** pass datasets BY-VALUE (performance issue)
5. **DON'T** forget to validate in the BE layer
6. **DON'T** create "god classes" that do everything
7. **DON'T** skip error handling

---

## Quick Reference: Parameter Patterns

```openedge
/* Fetching data - output only */
METHOD PUBLIC VOID FetchEntity(
    INPUT  iID AS INTEGER,
    OUTPUT DATASET dsEntity BY-REFERENCE).

/* Saving data - input-output for round-trip */
METHOD PUBLIC VOID SaveEntity(
    INPUT-OUTPUT DATASET dsEntity BY-REFERENCE).

/* Multiple datasets */
METHOD PUBLIC VOID GetRelatedData(
    INPUT  iID AS INTEGER,
    OUTPUT DATASET dsEntity BY-REFERENCE,
    OUTPUT DATASET dsRelated BY-REFERENCE).
```

---

## Next Steps

1. Review this invoice example thoroughly
2. Adapt the template for your specific entity
3. Follow the checklist for each new feature
4. Maintain consistency across all implementations
5. Document any deviations from the pattern

This template ensures your new functionality is:
- OERA compliant
- Object-oriented with SOLID principles
- Performant with proper dataset handling
- Maintainable and testable
