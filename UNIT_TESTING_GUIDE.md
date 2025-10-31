# OERA Unit Testing Guide with ABLUnit

## Overview

This guide shows you how to write unit tests for your OERA implementation using OpenEdge ABLUnit testing framework. Unit testing allows you to test business logic in isolation without requiring a database connection.

## Prerequisites

- OpenEdge 11.6+ (ABLUnit framework included)
- OERA implementation with interfaces (DAO and BE interfaces)
- Basic understanding of unit testing concepts

## Table of Contents

1. [ABLUnit Framework Basics](#ablunit-framework-basics)
2. [Setting Up Test Structure](#setting-up-test-structure)
3. [Creating Mock Objects](#creating-mock-objects)
4. [Testing DAO Layer](#testing-dao-layer)
5. [Testing Business Entity Layer](#testing-business-entity-layer)
6. [Testing Business Tasks](#testing-business-tasks)
7. [Running Tests](#running-tests)
8. [Best Practices](#best-practices)

---

## ABLUnit Framework Basics

### What is ABLUnit?

ABLUnit is OpenEdge's built-in testing framework, similar to JUnit for Java or NUnit for .NET. It allows you to:
- Write automated tests
- Run tests without manual intervention
- Get clear pass/fail results
- Integrate with CI/CD pipelines

### Key ABLUnit Annotations

```openedge
@Test.              /* Marks a method as a test */
@Before.            /* Runs before each test */
@After.             /* Runs after each test */
@BeforeClass.       /* Runs once before all tests */
@AfterClass.        /* Runs once after all tests */
@Ignore.            /* Skip this test */
```

### Basic Test Structure

```openedge
USING Progress.Lang.*.
USING OpenEdge.Core.Assert.

CLASS MyTest:
    
    @Before.
    METHOD PUBLIC VOID setUp():
        /* Initialize test data before each test */
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testSomething():
        /* Your test code */
        Assert:IsTrue(condition).
    END METHOD.
    
    @After.
    METHOD PUBLIC VOID tearDown():
        /* Clean up after each test */
    END METHOD.
    
END CLASS.
```

---

## Setting Up Test Structure

### Directory Organization

```
/YourApplication
    /Business
        /DataDefinitions
        /DataAccess
        /Entity
        /Task
    /Presentation
    /Test
        /Mock                   ← Mock implementations
            MockInvoiceDAO.cls
            MockCustomerDAO.cls
        /Unit                   ← Unit tests
            InvoiceDAOTest.cls
            InvoiceBETest.cls
            InvoiceTaskTest.cls
        /Integration            ← Integration tests (optional)
            InvoiceIntegrationTest.cls
        TestRunner.p            ← Test runner program
```

---

## Creating Mock Objects

### Mock DAO Pattern

Mock objects simulate real objects for testing. Here's a complete mock DAO:

```openedge
/* MockInvoiceDAO.cls - Mock implementation of IInvoiceDAO */

USING Business.DataAccess.IInvoiceDAO.
USING Progress.Lang.*.

CLASS Test.Mock.MockInvoiceDAO IMPLEMENTS IInvoiceDAO:
    
    {InvoiceDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}
    
    /* Properties to track method calls */
    DEFINE PUBLIC PROPERTY FetchInvoiceCallCount AS INTEGER NO-UNDO 
        GET. SET.
    
    DEFINE PUBLIC PROPERTY SaveInvoiceCallCount AS INTEGER NO-UNDO 
        GET. SET.
    
    DEFINE PUBLIC PROPERTY DeleteInvoiceCallCount AS INTEGER NO-UNDO 
        GET. SET.
    
    DEFINE PUBLIC PROPERTY LastInvoiceIDFetched AS INTEGER NO-UNDO 
        GET. SET.
    
    DEFINE PUBLIC PROPERTY LastInvoiceIDDeleted AS INTEGER NO-UNDO 
        GET. SET.
    
    DEFINE PUBLIC PROPERTY ShouldThrowError AS LOGICAL NO-UNDO 
        GET. SET.
    
    DEFINE PUBLIC PROPERTY ErrorMessageToThrow AS CHARACTER NO-UNDO 
        GET. SET.
    
    /* Constructor */
    CONSTRUCTOR PUBLIC MockInvoiceDAO():
        Reset().
    END CONSTRUCTOR.
    
    /* Reset all tracking properties */
    METHOD PUBLIC VOID Reset():
        ASSIGN
            FetchInvoiceCallCount = 0
            SaveInvoiceCallCount = 0
            DeleteInvoiceCallCount = 0
            LastInvoiceIDFetched = 0
            LastInvoiceIDDeleted = 0
            ShouldThrowError = FALSE
            ErrorMessageToThrow = "".
    END METHOD.
    
    /* Mock implementation of FetchInvoice */
    METHOD PUBLIC VOID FetchInvoice(
        INPUT  iInvoiceID AS INTEGER,
        OUTPUT DATASET dsInvoice BY-REFERENCE):
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        FetchInvoiceCallCount = FetchInvoiceCallCount + 1.
        LastInvoiceIDFetched = iInvoiceID.
        
        IF ShouldThrowError THEN
            UNDO, THROW NEW AppError(ErrorMessageToThrow).
        
        EMPTY TEMP-TABLE ttInvoice.
        
        /* Return mock data */
        CREATE bttInvoice.
        ASSIGN
            bttInvoice.InvoiceID = iInvoiceID
            bttInvoice.CustomerID = 100
            bttInvoice.InvoiceDate = TODAY
            bttInvoice.TotalAmount = 1000.00
            bttInvoice.Status = "DRAFT"
            bttInvoice.Description = "Mock Invoice"
            bttInvoice.CreatedBy = "TEST"
            bttInvoice.CreatedDate = NOW.
        
    END METHOD.
    
    /* Mock implementation of FetchInvoicesByCustomer */
    METHOD PUBLIC VOID FetchInvoicesByCustomer(
        INPUT  iCustomerID AS INTEGER,
        OUTPUT DATASET dsInvoice BY-REFERENCE):
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        DEFINE VARIABLE i AS INTEGER NO-UNDO.
        
        IF ShouldThrowError THEN
            UNDO, THROW NEW AppError(ErrorMessageToThrow).
        
        EMPTY TEMP-TABLE ttInvoice.
        
        /* Return multiple mock invoices */
        DO i = 1 TO 3:
            CREATE bttInvoice.
            ASSIGN
                bttInvoice.InvoiceID = i
                bttInvoice.CustomerID = iCustomerID
                bttInvoice.InvoiceDate = TODAY - i
                bttInvoice.TotalAmount = 100.00 * i
                bttInvoice.Status = "DRAFT"
                bttInvoice.CreatedBy = "TEST"
                bttInvoice.CreatedDate = NOW.
        END.
        
    END METHOD.
    
    /* Mock implementation of FetchInvoicesByDateRange */
    METHOD PUBLIC VOID FetchInvoicesByDateRange(
        INPUT  dtStartDate AS DATE,
        INPUT  dtEndDate AS DATE,
        OUTPUT DATASET dsInvoice BY-REFERENCE):
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        IF ShouldThrowError THEN
            UNDO, THROW NEW AppError(ErrorMessageToThrow).
        
        EMPTY TEMP-TABLE ttInvoice.
        
        /* Return mock data */
        CREATE bttInvoice.
        ASSIGN
            bttInvoice.InvoiceID = 1
            bttInvoice.CustomerID = 100
            bttInvoice.InvoiceDate = dtStartDate
            bttInvoice.TotalAmount = 500.00
            bttInvoice.Status = "DRAFT"
            bttInvoice.CreatedBy = "TEST"
            bttInvoice.CreatedDate = NOW.
        
    END METHOD.
    
    /* Mock implementation of SaveInvoice */
    METHOD PUBLIC VOID SaveInvoice(
        INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE):
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        SaveInvoiceCallCount = SaveInvoiceCallCount + 1.
        
        IF ShouldThrowError THEN
            UNDO, THROW NEW AppError(ErrorMessageToThrow).
        
        /* Simulate assigning ID to new invoices */
        FOR EACH bttInvoice WHERE bttInvoice.InvoiceID = 0 OR bttInvoice.InvoiceID = ?:
            ASSIGN bttInvoice.InvoiceID = 999.  /* Mock ID */
        END.
        
    END METHOD.
    
    /* Mock implementation of DeleteInvoice */
    METHOD PUBLIC VOID DeleteInvoice(INPUT iInvoiceID AS INTEGER):
        
        DeleteInvoiceCallCount = DeleteInvoiceCallCount + 1.
        LastInvoiceIDDeleted = iInvoiceID.
        
        IF ShouldThrowError THEN
            UNDO, THROW NEW AppError(ErrorMessageToThrow).
        
    END METHOD.
    
END CLASS.
```

---

## Testing Business Entity Layer

### Complete BE Test Example

```openedge
/* InvoiceBETest.cls - Unit tests for InvoiceBE */

USING Progress.Lang.*.
USING OpenEdge.Core.Assert.
USING Business.Entity.InvoiceBE.
USING Test.Mock.MockInvoiceDAO.

CLASS Test.Unit.InvoiceBETest:
    
    {InvoiceDataset.i}
    
    DEFINE PRIVATE VARIABLE oMockDAO AS MockInvoiceDAO NO-UNDO.
    DEFINE PRIVATE VARIABLE oBE AS InvoiceBE NO-UNDO.
    DEFINE PRIVATE VARIABLE cErrorMessage AS CHARACTER NO-UNDO.
    DEFINE PRIVATE VARIABLE lResult AS LOGICAL NO-UNDO.
    
    /* Run before each test */
    @Before.
    METHOD PUBLIC VOID setUp():
        oMockDAO = NEW MockInvoiceDAO().
        oBE = NEW InvoiceBE(oMockDAO).  /* Inject mock */
    END METHOD.
    
    /* Run after each test */
    @After.
    METHOD PUBLIC VOID tearDown():
        DELETE OBJECT oMockDAO NO-ERROR.
        DELETE OBJECT oBE NO-ERROR.
        EMPTY TEMP-TABLE ttInvoice.
    END METHOD.
    
    /* ========================================
       VALIDATION TESTS
       ======================================== */
    
    @Test.
    METHOD PUBLIC VOID testSaveInvoice_MissingDate_ShouldFail():
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        /* Arrange - Create invoice with missing date */
        CREATE bttInvoice.
        ASSIGN
            bttInvoice.InvoiceID = 0
            bttInvoice.CustomerID = 123
            bttInvoice.InvoiceDate = ?          /* Invalid! */
            bttInvoice.Status = "DRAFT"
            bttInvoice.TotalAmount = 100.00.
        
        /* Act - Try to save */
        lResult = oBE:SaveInvoice(
            INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
            OUTPUT cErrorMessage).
        
        /* Assert - Should fail validation */
        Assert:IsFalse(lResult, "Save should fail with missing date").
        Assert:Equals("Invoice date is required.", cErrorMessage).
        
        /* Verify DAO was NOT called (validation stopped early) */
        Assert:Equals(0, oMockDAO:SaveInvoiceCallCount, 
            "DAO should not be called when validation fails").
        
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testSaveInvoice_FutureDate_ShouldFail():
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        /* Arrange */
        CREATE bttInvoice.
        ASSIGN
            bttInvoice.InvoiceID = 0
            bttInvoice.CustomerID = 123
            bttInvoice.InvoiceDate = TODAY + 30  /* Future date - invalid! */
            bttInvoice.Status = "DRAFT"
            bttInvoice.TotalAmount = 100.00.
        
        /* Act */
        lResult = oBE:SaveInvoice(
            INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
            OUTPUT cErrorMessage).
        
        /* Assert */
        Assert:IsFalse(lResult).
        Assert:Equals("Invoice date cannot be in the future.", cErrorMessage).
        Assert:Equals(0, oMockDAO:SaveInvoiceCallCount).
        
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testSaveInvoice_InvalidCustomerID_ShouldFail():
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        /* Arrange */
        CREATE bttInvoice.
        ASSIGN
            bttInvoice.InvoiceID = 0
            bttInvoice.CustomerID = -1          /* Invalid! */
            bttInvoice.InvoiceDate = TODAY
            bttInvoice.Status = "DRAFT"
            bttInvoice.TotalAmount = 100.00.
        
        /* Act */
        lResult = oBE:SaveInvoice(
            INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
            OUTPUT cErrorMessage).
        
        /* Assert */
        Assert:IsFalse(lResult).
        Assert:Equals("Invalid Customer ID.", cErrorMessage).
        
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testSaveInvoice_InvalidStatus_ShouldFail():
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        /* Arrange */
        CREATE bttInvoice.
        ASSIGN
            bttInvoice.InvoiceID = 0
            bttInvoice.CustomerID = 123
            bttInvoice.InvoiceDate = TODAY
            bttInvoice.Status = "INVALID_STATUS"  /* Invalid! */
            bttInvoice.TotalAmount = 100.00.
        
        /* Act */
        lResult = oBE:SaveInvoice(
            INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
            OUTPUT cErrorMessage).
        
        /* Assert */
        Assert:IsFalse(lResult).
        Assert:Equals("Invalid invoice status.", cErrorMessage).
        
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testSaveInvoice_NegativeAmount_ShouldFail():
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        /* Arrange */
        CREATE bttInvoice.
        ASSIGN
            bttInvoice.InvoiceID = 0
            bttInvoice.CustomerID = 123
            bttInvoice.InvoiceDate = TODAY
            bttInvoice.Status = "DRAFT"
            bttInvoice.TotalAmount = -100.00.    /* Invalid! */
        
        /* Act */
        lResult = oBE:SaveInvoice(
            INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
            OUTPUT cErrorMessage).
        
        /* Assert */
        Assert:IsFalse(lResult).
        Assert:Equals("Total amount cannot be negative.", cErrorMessage).
        
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testSaveInvoice_ValidData_ShouldSucceed():
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        /* Arrange - Valid invoice */
        CREATE bttInvoice.
        ASSIGN
            bttInvoice.InvoiceID = 0
            bttInvoice.CustomerID = 123
            bttInvoice.InvoiceDate = TODAY
            bttInvoice.Status = "DRAFT"
            bttInvoice.TotalAmount = 100.00
            bttInvoice.Description = "Test Invoice".
        
        /* Act */
        lResult = oBE:SaveInvoice(
            INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
            OUTPUT cErrorMessage).
        
        /* Assert */
        Assert:IsTrue(lResult, "Valid invoice should save successfully").
        Assert:Equals("", cErrorMessage, "No error message expected").
        
        /* Verify DAO was called */
        Assert:Equals(1, oMockDAO:SaveInvoiceCallCount, 
            "DAO Save should be called once").
        
        /* Verify audit fields were set */
        FIND FIRST bttInvoice NO-LOCK.
        Assert:NotNull(bttInvoice.CreatedBy, "CreatedBy should be set").
        Assert:NotNull(bttInvoice.CreatedDate, "CreatedDate should be set").
        Assert:NotNull(bttInvoice.ModifiedBy, "ModifiedBy should be set").
        Assert:NotNull(bttInvoice.ModifiedDate, "ModifiedDate should be set").
        
    END METHOD.
    
    /* ========================================
       DELETE BUSINESS RULE TESTS
       ======================================== */
    
    @Test.
    METHOD PUBLIC VOID testDeleteInvoice_PostedInvoice_ShouldFail():
        
        /* Arrange - Mock will return POSTED invoice */
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        /* Get the posted invoice from mock */
        oBE:GetInvoice(1, OUTPUT DATASET dsInvoice BY-REFERENCE).
        
        /* Change status to POSTED */
        FIND FIRST bttInvoice EXCLUSIVE-LOCK.
        ASSIGN bttInvoice.Status = "POSTED".
        
        /* Mock will return this data when DeleteInvoice fetches */
        
        /* Act */
        lResult = oBE:DeleteInvoice(1, OUTPUT cErrorMessage).
        
        /* Assert */
        Assert:IsFalse(lResult).
        Assert:Equals("Cannot delete posted invoices.", cErrorMessage).
        
        /* Verify DAO Delete was NOT called */
        Assert:Equals(0, oMockDAO:DeleteInvoiceCallCount).
        
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testDeleteInvoice_PaidInvoice_ShouldFail():
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        /* Arrange */
        oBE:GetInvoice(1, OUTPUT DATASET dsInvoice BY-REFERENCE).
        FIND FIRST bttInvoice EXCLUSIVE-LOCK.
        ASSIGN bttInvoice.Status = "PAID".
        
        /* Act */
        lResult = oBE:DeleteInvoice(1, OUTPUT cErrorMessage).
        
        /* Assert */
        Assert:IsFalse(lResult).
        Assert:Equals("Cannot delete paid invoices.", cErrorMessage).
        Assert:Equals(0, oMockDAO:DeleteInvoiceCallCount).
        
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testDeleteInvoice_DraftInvoice_ShouldSucceed():
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        /* Arrange */
        oBE:GetInvoice(1, OUTPUT DATASET dsInvoice BY-REFERENCE).
        FIND FIRST bttInvoice NO-LOCK.
        Assert:Equals("DRAFT", bttInvoice.Status, "Mock should return DRAFT").
        
        /* Act */
        lResult = oBE:DeleteInvoice(1, OUTPUT cErrorMessage).
        
        /* Assert */
        Assert:IsTrue(lResult, "DRAFT invoice should be deletable").
        Assert:Equals("", cErrorMessage).
        
        /* Verify DAO Delete was called */
        Assert:Equals(1, oMockDAO:DeleteInvoiceCallCount).
        Assert:Equals(1, oMockDAO:LastInvoiceIDDeleted).
        
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testDeleteInvoice_NonExistentInvoice_ShouldFail():
        
        /* Arrange - Mock returns empty dataset */
        oMockDAO:Reset().
        
        /* Act - Try to delete non-existent invoice */
        lResult = oBE:DeleteInvoice(99999, OUTPUT cErrorMessage).
        
        /* Assert */
        Assert:IsFalse(lResult).
        Assert:Equals("Invoice not found.", cErrorMessage).
        
    END METHOD.
    
    /* ========================================
       GET METHOD TESTS
       ======================================== */
    
    @Test.
    METHOD PUBLIC VOID testGetInvoice_ValidID_ReturnsData():
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        /* Act */
        oBE:GetInvoice(123, OUTPUT DATASET dsInvoice BY-REFERENCE).
        
        /* Assert */
        Assert:Equals(1, oMockDAO:FetchInvoiceCallCount).
        Assert:Equals(123, oMockDAO:LastInvoiceIDFetched).
        
        FIND FIRST bttInvoice NO-LOCK NO-ERROR.
        Assert:IsTrue(AVAILABLE bttInvoice, "Invoice should be returned").
        Assert:Equals(123, bttInvoice.InvoiceID).
        
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testGetInvoicesByCustomer_ValidID_ReturnsMultiple():
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        DEFINE VARIABLE iCount AS INTEGER NO-UNDO.
        
        /* Act */
        oBE:GetInvoicesByCustomer(100, OUTPUT DATASET dsInvoice BY-REFERENCE).
        
        /* Assert */
        FOR EACH bttInvoice NO-LOCK:
            iCount = iCount + 1.
        END.
        
        Assert:Equals(3, iCount, "Mock should return 3 invoices").
        
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testGetInvoicesByCustomer_InvalidID_ReturnsNothing():
        
        /* Act */
        oBE:GetInvoicesByCustomer(-1, OUTPUT DATASET dsInvoice BY-REFERENCE).
        
        /* Assert - BE should not call DAO for invalid input */
        /* Implementation dependent - adjust based on your BE logic */
        
    END METHOD.
    
    /* ========================================
       ERROR HANDLING TESTS
       ======================================== */
    
    @Test.
    METHOD PUBLIC VOID testSaveInvoice_DAOError_ReturnsError():
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        /* Arrange - Configure mock to throw error */
        ASSIGN
            oMockDAO:ShouldThrowError = TRUE
            oMockDAO:ErrorMessageToThrow = "Database connection lost".
        
        CREATE bttInvoice.
        ASSIGN
            bttInvoice.InvoiceID = 0
            bttInvoice.CustomerID = 123
            bttInvoice.InvoiceDate = TODAY
            bttInvoice.Status = "DRAFT"
            bttInvoice.TotalAmount = 100.00.
        
        /* Act */
        lResult = oBE:SaveInvoice(
            INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
            OUTPUT cErrorMessage).
        
        /* Assert */
        Assert:IsFalse(lResult).
        Assert:Equals("Database connection lost", cErrorMessage).
        
    END METHOD.
    
    /* ========================================
       AUDIT FIELDS TESTS
       ======================================== */
    
    @Test.
    METHOD PUBLIC VOID testSaveInvoice_NewRecord_SetsCreatedFields():
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        
        /* Arrange */
        CREATE bttInvoice.
        ASSIGN
            bttInvoice.InvoiceID = 0  /* New record */
            bttInvoice.CustomerID = 123
            bttInvoice.InvoiceDate = TODAY
            bttInvoice.Status = "DRAFT"
            bttInvoice.TotalAmount = 100.00.
        
        /* Act */
        lResult = oBE:SaveInvoice(
            INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
            OUTPUT cErrorMessage).
        
        /* Assert */
        Assert:IsTrue(lResult).
        
        FIND FIRST bttInvoice NO-LOCK.
        Assert:NotNull(bttInvoice.CreatedBy).
        Assert:NotNull(bttInvoice.CreatedDate).
        Assert:NotEquals("", bttInvoice.CreatedBy).
        
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testSaveInvoice_ExistingRecord_UpdatesModifiedFields():
        
        DEFINE BUFFER bttInvoice FOR ttInvoice.
        DEFINE VARIABLE dtCreatedDate AS DATETIME NO-UNDO.
        
        /* Arrange - Existing record */
        CREATE bttInvoice.
        ASSIGN
            bttInvoice.InvoiceID = 123  /* Existing record */
            bttInvoice.CustomerID = 123
            bttInvoice.InvoiceDate = TODAY
            bttInvoice.Status = "DRAFT"
            bttInvoice.TotalAmount = 100.00
            bttInvoice.CreatedBy = "ORIGINAL"
            bttInvoice.CreatedDate = NOW
            dtCreatedDate = bttInvoice.CreatedDate.
        
        /* Act */
        lResult = oBE:SaveInvoice(
            INPUT-OUTPUT DATASET dsInvoice BY-REFERENCE,
            OUTPUT cErrorMessage).
        
        /* Assert */
        Assert:IsTrue(lResult).
        
        FIND FIRST bttInvoice NO-LOCK.
        /* CreatedBy should not change */
        Assert:Equals("ORIGINAL", bttInvoice.CreatedBy).
        /* ModifiedBy should be set */
        Assert:NotNull(bttInvoice.ModifiedBy).
        Assert:NotNull(bttInvoice.ModifiedDate).
        
    END METHOD.
    
END CLASS.
```

---

## Testing Business Tasks

### Business Task Test Example

```openedge
/* InvoicePostingTaskTest.cls */

USING Progress.Lang.*.
USING OpenEdge.Core.Assert.
USING Business.Task.InvoicePostingTask.
USING Test.Mock.MockInvoiceDAO.

CLASS Test.Unit.InvoicePostingTaskTest:
    
    {InvoiceDataset.i}
    
    DEFINE PRIVATE VARIABLE oTask AS InvoicePostingTask NO-UNDO.
    DEFINE PRIVATE VARIABLE cErrorMessage AS CHARACTER NO-UNDO.
    DEFINE PRIVATE VARIABLE lResult AS LOGICAL NO-UNDO.
    
    @Before.
    METHOD PUBLIC VOID setUp():
        oTask = NEW InvoicePostingTask().
    END METHOD.
    
    @After.
    METHOD PUBLIC VOID tearDown():
        DELETE OBJECT oTask NO-ERROR.
        EMPTY TEMP-TABLE ttInvoice.
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testPostInvoice_DraftStatus_ShouldSucceed():
        
        /* Arrange - Task internally uses BE which uses mock DAO */
        /* For this test, we'd need to inject dependencies through the task */
        
        /* Act */
        lResult = oTask:PostInvoice(1, OUTPUT cErrorMessage).
        
        /* Assert */
        Assert:IsTrue(lResult).
        Assert:Equals("", cErrorMessage).
        
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testPostInvoice_AlreadyPosted_ShouldFail():
        
        /* This test would require the mock to return a POSTED invoice */
        /* Implementation depends on your dependency injection approach */
        
    END METHOD.
    
END CLASS.
```

---

## Running Tests

### Test Runner Program

```openedge
/* TestRunner.p - Runs all unit tests */

USING OpenEdge.Core.*.
USING OpenEdge.ABLUnit.*.

DEFINE VARIABLE oRunner AS ABLUnitTestRunner NO-UNDO.
DEFINE VARIABLE oResults AS IABLUnitTestResult NO-UNDO.

/* Create test runner */
oRunner = NEW ABLUnitTestRunner().

/* Add test classes */
oRunner:AddTest("Test.Unit.InvoiceBETest").
oRunner:AddTest("Test.Unit.InvoicePostingTaskTest").
/* Add more test classes here */

/* Run tests */
oResults = oRunner:Run().

/* Display results */
MESSAGE 
    "Tests Run:    " oResults:TestsRun SKIP
    "Tests Passed: " oResults:TestsPassed SKIP
    "Tests Failed: " oResults:TestsFailed SKIP
    "Tests Ignored:" oResults:TestsIgnored
    VIEW-AS ALERT-BOX INFORMATION.

/* Exit with error code if tests failed */
IF oResults:TestsFailed > 0 THEN
    QUIT.
```

### Running from Command Line

```bash
# Run all tests
pro -b TestRunner.p

# Run specific test class
pro -b -p "Test.Unit.InvoiceBETest"
```

### Integration with Ant/Build Script

```xml
<!-- build.xml -->
<target name="test">
    <PCTRun procedure="Test/TestRunner.p" failonerror="true">
        <propath>
            <pathelement location="${src.dir}"/>
            <pathelement location="${test.dir}"/>
        </propath>
    </PCTRun>
</target>
```

---

## Best Practices

### 1. Test Naming Conventions

```openedge
/* Pattern: test[MethodName]_[Scenario]_[ExpectedResult] */

testSaveInvoice_ValidData_ShouldSucceed
testSaveInvoice_MissingDate_ShouldFail
testDeleteInvoice_PostedInvoice_ShouldFail
```

### 2. Arrange-Act-Assert Pattern

```openedge
@Test.
METHOD PUBLIC VOID testExample():
    
    /* ARRANGE - Set up test data */
    CREATE ttInvoice.
    ASSIGN ttInvoice.InvoiceDate = ?.
    
    /* ACT - Execute the method being tested */
    lResult = oBE:SaveInvoice(...).
    
    /* ASSERT - Verify the results */
    Assert:IsFalse(lResult).
    Assert:Equals("Expected error", cErrorMessage).
    
END METHOD.
```

### 3. Test Independence

```openedge
/* Each test should be independent */

@Before.
METHOD PUBLIC VOID setUp():
    /* Reset state before EACH test */
    oMockDAO = NEW MockInvoiceDAO().
    oBE = NEW InvoiceBE(oMockDAO).
    EMPTY TEMP-TABLE ttInvoice.
END METHOD.
```

### 4. Test One Thing

```openedge
/* BAD - Tests multiple things */
@Test.
METHOD PUBLIC VOID testInvoice():
    /* Tests validation AND save AND delete - too much! */
END METHOD.

/* GOOD - Focused tests */
@Test.
METHOD PUBLIC VOID testSaveInvoice_ValidData_ShouldSucceed():
    /* Tests only save with valid data */
END METHOD.

@Test.
METHOD PUBLIC VOID testSaveInvoice_InvalidData_ShouldFail():
    /* Tests only save with invalid data */
END METHOD.
```

### 5. Use Descriptive Assertions

```openedge
/* BAD */
Assert:IsTrue(lResult).

/* GOOD */
Assert:IsTrue(lResult, "Valid invoice should save successfully").
```

### 6. Test Edge Cases

```openedge
/* Don't just test the happy path */

@Test. METHOD PUBLIC VOID testWithNullValue(): END METHOD.
@Test. METHOD PUBLIC VOID testWithEmptyString(): END METHOD.
@Test. METHOD PUBLIC VOID testWithMaxValue(): END METHOD.
@Test. METHOD PUBLIC VOID testWithNegativeValue(): END METHOD.
@Test. METHOD PUBLIC VOID testWithZero(): END METHOD.
```

### 7. Mock External Dependencies

```openedge
/* Always mock external dependencies */

/* ✓ Mock DAO - no database */
/* ✓ Mock web services - no network */
/* ✓ Mock file system - no disk I/O */
/* ✓ Mock email service - no SMTP */
```

### 8. Keep Tests Fast

```openedge
/* Unit tests should run in milliseconds */

/* ✗ Avoid database access */
/* ✗ Avoid file I/O */
/* ✗ Avoid network calls */
/* ✗ Avoid sleep/wait statements */
/* ✓ Use mocks for everything external */
```

---

## Common Assert Methods

```openedge
/* Equality */
Assert:Equals(expected, actual).
Assert:Equals(expected, actual, "message").
Assert:NotEquals(notExpected, actual).

/* Boolean */
Assert:IsTrue(condition).
Assert:IsTrue(condition, "message").
Assert:IsFalse(condition).

/* Null checks */
Assert:IsNull(object).
Assert:NotNull(object).

/* Type checks */
Assert:IsType(object, "ClassName").

/* Exceptions */
Assert:Throws(statement, "ExpectedErrorType").
```

---

## Example Test Output

```
Running tests...

Test.Unit.InvoiceBETest
  ✓ testSaveInvoice_MissingDate_ShouldFail (2ms)
  ✓ testSaveInvoice_FutureDate_ShouldFail (1ms)
  ✓ testSaveInvoice_InvalidCustomerID_ShouldFail (1ms)
  ✓ testSaveInvoice_InvalidStatus_ShouldFail (1ms)
  ✓ testSaveInvoice_ValidData_ShouldSucceed (2ms)
  ✓ testDeleteInvoice_PostedInvoice_ShouldFail (1ms)
  ✓ testDeleteInvoice_DraftInvoice_ShouldSucceed (1ms)
  ✓ testGetInvoice_ValidID_ReturnsData (1ms)

Tests Run: 8
Tests Passed: 8
Tests Failed: 0
Tests Ignored: 0

Total Time: 10ms
```

---

## Summary

**With ABLUnit and Mock Objects, you can:**

✅ Test business logic without database  
✅ Run hundreds of tests in seconds  
✅ Catch bugs early in development  
✅ Refactor with confidence  
✅ Document expected behavior  
✅ Integrate with CI/CD pipelines  
✅ Improve code quality  

**Key Takeaways:**

1. Use interfaces for DAO and BE layers
2. Create mock implementations for testing
3. Inject mocks through constructors
4. Write focused, independent tests
5. Follow Arrange-Act-Assert pattern
6. Test edge cases and error scenarios
7. Keep tests fast (no database, no I/O)
8. Run tests frequently during development

---

**Next Steps:**

1. Implement interfaces for your DAO and BE layers
2. Create mock DAO implementations
3. Write your first test class
4. Run tests and verify they pass
5. Add tests as you develop new features
6. Integrate tests into your build process

Happy Testing! 🎉
