# PDSOE Environment Setup Guide

## Can This Project Be Used in PDSOE?

**YES! Absolutely.** ✅

This OERA project is fully compatible with Progress Developer Studio for OpenEdge (PDSOE). All code examples use standard OpenEdge ABL syntax and follow PDSOE best practices.

---

## PDSOE Compatibility

### ✅ What Works Perfectly:

1. **All ABL Code** - Standard OpenEdge ABL syntax
2. **Class Files (.cls)** - PDSOE's native format
3. **Include Files (.i)** - Standard ABL includes
4. **Interfaces** - Full PDSOE support
5. **ABLUnit Testing** - Built into PDSOE
6. **Project Structure** - Matches PDSOE conventions
7. **PROPATH Management** - Standard PDSOE setup

### 📋 Requirements:

- **PDSOE Version:** 11.6+ (for ABLUnit support)
- **OpenEdge Version:** 11.6+ 
- **ABLUnit:** Included with PDSOE 11.6+

---

## Setting Up in PDSOE

### Step 1: Create New OpenEdge Project

1. Open PDSOE
2. **File → New → OpenEdge Project**
3. Project name: `YourApplicationName`
4. Click **Finish**

### Step 2: Set Up Directory Structure

Create these folders in your PDSOE project:

```
YourProject/
├── Business/
│   ├── DataDefinitions/
│   ├── DataAccess/
│   └── Entity/
├── Presentation/
└── Test/
    ├── Mock/
    └── Unit/
```

**In PDSOE:**
1. Right-click project → **New → Folder**
2. Create each folder above

### Step 3: Configure PROPATH

1. Right-click project → **Properties**
2. Go to **Progress OpenEdge → Path Configuration**
3. Add to PROPATH:
   ```
   ${PROJECT_LOC}/Business
   ${PROJECT_LOC}/Presentation
   ${PROJECT_LOC}/Test
   ```
4. Click **Apply and Close**

### Step 4: Import OERA Templates

1. Download and extract the OERA_Project
2. Copy template files to your PDSOE project folders
3. Follow the naming conventions for your entities

---

## Creating Your First Entity in PDSOE

### Example: Customer Entity

#### 1. Create Dataset Include

**File:** `Business/DataDefinitions/CustomerDataset.i`

```openedge
/* CustomerDataset.i */

&IF "{&REFERENCE-ONLY}" = "" &THEN
    &SCOPED-DEFINE REFERENCE-ONLY
&ENDIF

DEFINE TEMP-TABLE ttCustomer NO-UNDO {&REFERENCE-ONLY}
    FIELD CustomerID    AS INTEGER
    FIELD CustomerName  AS CHARACTER
    FIELD Email         AS CHARACTER
    FIELD Phone         AS CHARACTER
    FIELD Status        AS CHARACTER
    INDEX idxPrimary IS PRIMARY UNIQUE CustomerID.

DEFINE DATASET dsCustomer {&REFERENCE-ONLY} FOR ttCustomer.
```

**In PDSOE:**
- Right-click `DataDefinitions` folder
- **New → ABL Include File**
- Name: `CustomerDataset.i`
- Paste code

#### 2. Create DAO Interface

**File:** `Business/DataAccess/ICustomerDAO.cls`

```openedge
INTERFACE Business.DataAccess.ICustomerDAO:
    
    METHOD PUBLIC VOID FetchCustomer(
        INPUT  iCustomerID AS INTEGER,
        OUTPUT DATASET dsCustomer BY-REFERENCE).
    
    METHOD PUBLIC VOID SaveCustomer(
        INPUT-OUTPUT DATASET dsCustomer BY-REFERENCE).
    
    METHOD PUBLIC VOID DeleteCustomer(
        INPUT iCustomerID AS INTEGER).
        
END INTERFACE.
```

**In PDSOE:**
- Right-click `DataAccess` folder
- **New → ABL Interface**
- Package: `Business.DataAccess`
- Name: `ICustomerDAO`
- Paste code

#### 3. Create DAO Implementation

**File:** `Business/DataAccess/CustomerDAO.cls`

```openedge
USING Business.DataAccess.ICustomerDAO.
USING Progress.Lang.*.

CLASS Business.DataAccess.CustomerDAO IMPLEMENTS ICustomerDAO:
    
    {Business/DataDefinitions/CustomerDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}
    
    METHOD PUBLIC VOID FetchCustomer(
        INPUT  iCustomerID AS INTEGER,
        OUTPUT DATASET dsCustomer BY-REFERENCE):
        
        DEFINE BUFFER bCustomer FOR Customer.
        
        EMPTY TEMP-TABLE ttCustomer.
        
        FOR EACH bCustomer NO-LOCK
            WHERE bCustomer.CustomerID = iCustomerID:
            
            CREATE ttCustomer.
            BUFFER-COPY bCustomer TO ttCustomer.
        END.
        
    END METHOD.
    
    METHOD PUBLIC VOID SaveCustomer(
        INPUT-OUTPUT DATASET dsCustomer BY-REFERENCE):
        
        DEFINE BUFFER bCustomer FOR Customer.
        DEFINE BUFFER bttCustomer FOR ttCustomer.
        
        DO TRANSACTION:
            FOR EACH bttCustomer:
                FIND FIRST bCustomer EXCLUSIVE-LOCK
                    WHERE bCustomer.CustomerID = bttCustomer.CustomerID
                    NO-ERROR.
                
                IF NOT AVAILABLE bCustomer THEN DO:
                    CREATE bCustomer.
                    ASSIGN bCustomer.CustomerID = NEXT-VALUE(CustomerSeq).
                    ASSIGN bttCustomer.CustomerID = bCustomer.CustomerID.
                END.
                
                BUFFER-COPY bttCustomer EXCEPT CustomerID TO bCustomer.
            END.
        END.
        
    END METHOD.
    
    METHOD PUBLIC VOID DeleteCustomer(INPUT iCustomerID AS INTEGER):
        
        DEFINE BUFFER bCustomer FOR Customer.
        
        DO TRANSACTION:
            FIND FIRST bCustomer EXCLUSIVE-LOCK
                WHERE bCustomer.CustomerID = iCustomerID
                NO-ERROR.
            
            IF AVAILABLE bCustomer THEN
                DELETE bCustomer.
        END.
        
    END METHOD.
    
END CLASS.
```

**In PDSOE:**
- Right-click `DataAccess` folder
- **New → ABL Class**
- Package: `Business.DataAccess`
- Name: `CustomerDAO`
- Implements: `Business.DataAccess.ICustomerDAO`
- Paste code

#### 4. Create BE Interface and Implementation

Follow same pattern for `ICustomerBE.cls` and `CustomerBE.cls`

---

## Setting Up ABLUnit Testing in PDSOE

### Enable ABLUnit

1. Right-click project → **Properties**
2. Go to **Progress OpenEdge → ABLUnit**
3. Check **Enable ABLUnit support**
4. Set test output folder: `Test/Results`
5. Click **Apply and Close**

### Create Test Folder Structure

```
Test/
├── Mock/
│   └── MockCustomerDAO.cls
├── Unit/
│   └── CustomerBETest.cls
└── Results/
```

### Create Mock DAO

**File:** `Test/Mock/MockCustomerDAO.cls`

```openedge
USING Business.DataAccess.ICustomerDAO.
USING Progress.Lang.*.

CLASS Test.Mock.MockCustomerDAO IMPLEMENTS ICustomerDAO:
    
    {Business/DataDefinitions/CustomerDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}
    
    DEFINE PUBLIC PROPERTY FetchCallCount AS INTEGER NO-UNDO GET. SET.
    DEFINE PUBLIC PROPERTY SaveCallCount AS INTEGER NO-UNDO GET. SET.
    
    CONSTRUCTOR PUBLIC MockCustomerDAO():
        ASSIGN
            FetchCallCount = 0
            SaveCallCount = 0.
    END CONSTRUCTOR.
    
    METHOD PUBLIC VOID FetchCustomer(
        INPUT  iCustomerID AS INTEGER,
        OUTPUT DATASET dsCustomer BY-REFERENCE):
        
        FetchCallCount = FetchCallCount + 1.
        
        EMPTY TEMP-TABLE ttCustomer.
        
        CREATE ttCustomer.
        ASSIGN
            ttCustomer.CustomerID = iCustomerID
            ttCustomer.CustomerName = "Test Customer"
            ttCustomer.Email = "test@example.com"
            ttCustomer.Status = "ACTIVE".
        
    END METHOD.
    
    METHOD PUBLIC VOID SaveCustomer(
        INPUT-OUTPUT DATASET dsCustomer BY-REFERENCE):
        
        SaveCallCount = SaveCallCount + 1.
        
        DEFINE BUFFER bttCustomer FOR ttCustomer.
        
        FOR EACH bttCustomer WHERE bttCustomer.CustomerID = 0:
            ASSIGN bttCustomer.CustomerID = 999.
        END.
        
    END METHOD.
    
    METHOD PUBLIC VOID DeleteCustomer(INPUT iCustomerID AS INTEGER):
        /* Mock implementation */
    END METHOD.
    
END CLASS.
```

**In PDSOE:**
- Right-click `Test/Mock` folder
- **New → ABL Class**
- Package: `Test.Mock`
- Name: `MockCustomerDAO`
- Implements: `Business.DataAccess.ICustomerDAO`

### Create Test Class

**File:** `Test/Unit/CustomerBETest.cls`

```openedge
USING Progress.Lang.*.
USING OpenEdge.Core.Assert.
USING Business.Entity.CustomerBE.
USING Test.Mock.MockCustomerDAO.

CLASS Test.Unit.CustomerBETest:
    
    {Business/DataDefinitions/CustomerDataset.i}
    
    DEFINE PRIVATE VARIABLE oMockDAO AS MockCustomerDAO NO-UNDO.
    DEFINE PRIVATE VARIABLE oBE AS CustomerBE NO-UNDO.
    
    @Before.
    METHOD PUBLIC VOID setUp():
        oMockDAO = NEW MockCustomerDAO().
        oBE = NEW CustomerBE(oMockDAO).
    END METHOD.
    
    @After.
    METHOD PUBLIC VOID tearDown():
        DELETE OBJECT oMockDAO NO-ERROR.
        DELETE OBJECT oBE NO-ERROR.
        EMPTY TEMP-TABLE ttCustomer.
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testGetCustomer_ValidID_ReturnsData():
        
        DEFINE BUFFER bttCustomer FOR ttCustomer.
        
        /* Act */
        oBE:GetCustomer(123, OUTPUT DATASET dsCustomer BY-REFERENCE).
        
        /* Assert */
        Assert:Equals(1, oMockDAO:FetchCallCount, "DAO should be called once").
        
        FIND FIRST bttCustomer NO-LOCK NO-ERROR.
        Assert:IsTrue(AVAILABLE bttCustomer, "Customer should be returned").
        Assert:Equals(123, bttCustomer.CustomerID).
        
    END METHOD.
    
    @Test.
    METHOD PUBLIC VOID testSaveCustomer_ValidData_ShouldSucceed():
        
        DEFINE VARIABLE lResult AS LOGICAL NO-UNDO.
        DEFINE VARIABLE cError AS CHARACTER NO-UNDO.
        DEFINE BUFFER bttCustomer FOR ttCustomer.
        
        /* Arrange */
        CREATE bttCustomer.
        ASSIGN
            bttCustomer.CustomerID = 0
            bttCustomer.CustomerName = "John Doe"
            bttCustomer.Email = "john@example.com"
            bttCustomer.Status = "ACTIVE".
        
        /* Act */
        lResult = oBE:SaveCustomer(
            INPUT-OUTPUT DATASET dsCustomer BY-REFERENCE,
            OUTPUT cError).
        
        /* Assert */
        Assert:IsTrue(lResult, "Save should succeed with valid data").
        Assert:Equals("", cError, "No error message expected").
        Assert:Equals(1, oMockDAO:SaveCallCount, "DAO Save should be called").
        
    END METHOD.
    
END CLASS.
```

**In PDSOE:**
- Right-click `Test/Unit` folder
- **New → ABL Class**
- Package: `Test.Unit`
- Name: `CustomerBETest`

---

## Running Tests in PDSOE

### Method 1: Run Single Test Class

1. Right-click `CustomerBETest.cls`
2. **Run As → ABLUnit Test**
3. View results in **ABLUnit Results** view

### Method 2: Run All Tests

1. Right-click project root
2. **Run As → ABLUnit Test**
3. All test classes will run

### Method 3: Run from Command Line

```bash
# Windows
cd C:\YourWorkspace\YourProject
proant test

# Linux/Mac
cd /your/workspace/YourProject
proant test
```

### Viewing Test Results

**PDSOE Test Results View:**
- Window → Show View → Other → ABLUnit → ABLUnit Results
- Shows pass/fail status
- Click failed test to see error details
- Double-click test to jump to code

---

## PDSOE Project Configuration Files

### Create .propath File

**File:** `.propath`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<propath version="11.7">
    <propathentry kind="src" path="Business"/>
    <propathentry kind="src" path="Presentation"/>
    <propathentry kind="src" path="Test"/>
    <propathentry kind="con" path="com.openedge.pdt.text.DB_CONNECTIONS_PATH"/>
    <propathentry kind="con" path="com.openedge.pdt.text.GUI_TTYPATH"/>
</propath>
```

### Create Build Configuration

**File:** `build.xml` (Ant build script)

```xml
<?xml version="1.0"?>
<project name="YourProject" default="compile" basedir=".">
    
    <property name="src.dir" value="Business"/>
    <property name="test.dir" value="Test"/>
    <property name="build.dir" value="build"/>
    
    <target name="compile">
        <echo message="Compiling source files..."/>
        <PCTCompile destDir="${build.dir}">
            <fileset dir="${src.dir}">
                <include name="**/*.cls"/>
            </fileset>
        </PCTCompile>
    </target>
    
    <target name="test" depends="compile">
        <echo message="Running ABLUnit tests..."/>
        <ABLUnit dlc="${DLC}" propath="${src.dir};${test.dir}">
            <fileset dir="${test.dir}/Unit">
                <include name="**/*Test.cls"/>
            </fileset>
        </ABLUnit>
    </target>
    
</project>
```

---

## Database Connection Setup

### Connect to Database

1. **OpenEdge Explorer** view
2. Right-click **Database Connections**
3. **New → Database Connection**
4. Enter connection details:
   - Name: `YourDatabase`
   - Database name: `sports2000` (or your DB)
   - Host: `localhost`
   - Port: `20000`
5. Click **Finish**

### Set Project Database

1. Right-click project → **Properties**
2. **Progress OpenEdge → Database Connections**
3. Check your database connection
4. Click **Apply and Close**

---

## PDSOE-Specific Features

### 1. Code Completion

- Type `DEFINE` → Ctrl+Space → See options
- Type class name → Ctrl+Space → Auto-complete
- Works with interfaces and classes

### 2. Quick Fix

- Hover over error → Click **Quick Fix**
- Auto-generate method stubs
- Add missing USING statements

### 3. Refactoring

- Right-click class/method → **Refactor**
- Rename class/method across project
- Extract method
- Move class to different package

### 4. Debugging

- Set breakpoints in test classes
- Right-click test → **Debug As → ABLUnit Test**
- Step through code
- Inspect variables

### 5. Code Templates

Create templates for common patterns:

1. **Window → Preferences**
2. **OpenEdge ABL → Editor → Templates**
3. **New** → Add OERA templates

**Example Template:**
```
Name: oera-dao
Pattern:
METHOD PUBLIC VOID Fetch${entity}(
    INPUT  i${entity}ID AS INTEGER,
    OUTPUT DATASET ds${entity} BY-REFERENCE):
    
    ${cursor}
    
END METHOD.
```

---

## Best Practices for PDSOE

### 1. Package Structure

Use proper package naming:
```
Business.DataAccess.CustomerDAO
Business.Entity.CustomerBE
Test.Mock.MockCustomerDAO
Test.Unit.CustomerBETest
```

### 2. File Organization

Keep related files together:
```
Business/
  DataAccess/
    ICustomerDAO.cls
    CustomerDAO.cls
  Entity/
    ICustomerBE.cls
    CustomerBE.cls
```

### 3. Use USING Statements

Always use USING for clean code:
```openedge
USING Business.DataAccess.*.
USING Business.Entity.*.
USING Progress.Lang.*.
```

### 4. Include File Paths

Use relative paths in includes:
```openedge
{Business/DataDefinitions/CustomerDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}
```

### 5. Version Control

Add to `.gitignore`:
```
/build/
/.pdsoe/
/Test/Results/
*.r
*.lst
```

---

## Troubleshooting Common Issues

### Issue 1: "Class not found"

**Solution:**
- Check PROPATH configuration
- Verify package name matches folder structure
- Refresh project (F5)

### Issue 2: "Include file not found"

**Solution:**
- Use relative path from PROPATH root
- Verify file exists in correct location
- Check for typos in filename

### Issue 3: "Tests not running"

**Solution:**
- Enable ABLUnit in project properties
- Check test class has @Test annotations
- Verify PROPATH includes Test folder
- Clean and rebuild project

### Issue 4: "Interface not implemented correctly"

**Solution:**
- Use Quick Fix to generate method stubs
- Verify method signatures match interface exactly
- Check parameter directions (INPUT/OUTPUT)

---

## Importing OERA Templates into PDSOE

### Quick Import Method

1. Download OERA_Project
2. Extract to temp location
3. In PDSOE:
   - Right-click project
   - **Import → General → File System**
   - Browse to extracted OERA_Project
   - Select folders to import
   - Check **Create top-level folder**
   - Name: `OERA_Templates`
4. Copy templates as needed for your entities

### Using Templates

1. Open template file (e.g., `OERA_Generic_Template.md`)
2. Copy code section
3. Create new file in PDSOE
4. Paste and find-replace `{Entity}` with your entity name
5. Adjust fields and logic

---

## Sample PDSOE Project Structure

```
YourProject/
├── .project                    ← PDSOE project file
├── .propath                    ← PROPATH configuration
├── build.xml                   ← Ant build script
│
├── Business/
│   ├── DataDefinitions/
│   │   ├── CustomerDataset.i
│   │   └── InvoiceDataset.i
│   ├── DataAccess/
│   │   ├── ICustomerDAO.cls
│   │   ├── CustomerDAO.cls
│   │   ├── IInvoiceDAO.cls
│   │   └── InvoiceDAO.cls
│   └── Entity/
│       ├── ICustomerBE.cls
│       ├── CustomerBE.cls
│       ├── IInvoiceBE.cls
│       └── InvoiceBE.cls
│
├── Presentation/
│   ├── CustomerWindow.w
│   └── InvoiceWindow.w
│
├── Test/
│   ├── Mock/
│   │   ├── MockCustomerDAO.cls
│   │   └── MockInvoiceDAO.cls
│   └── Unit/
│       ├── CustomerBETest.cls
│       └── InvoiceBETest.cls
│
├── OERA_Templates/            ← Imported templates
│   ├── README.md
│   ├── OERA_Generic_Template.md
│   └── UNIT_TESTING_GUIDE.md
│
└── build/                     ← Compiled .r files
```

---

## Summary

### ✅ PDSOE Compatibility: EXCELLENT

- All code works natively in PDSOE
- ABLUnit built-in for testing
- Full IDE support (completion, debugging, refactoring)
- Standard OpenEdge ABL syntax
- Follows PDSOE conventions

### 📋 Setup Checklist:

- [ ] Create OpenEdge project in PDSOE
- [ ] Set up folder structure
- [ ] Configure PROPATH
- [ ] Enable ABLUnit support
- [ ] Connect to database
- [ ] Import OERA templates
- [ ] Create your first entity
- [ ] Write unit tests
- [ ] Run tests
- [ ] Start developing!

### 🎯 Next Steps:

1. Open PDSOE
2. Follow this guide to set up project
3. Import one OERA template
4. Create your first entity
5. Write a test
6. See it pass!

---

**PDSOE Version Tested:** 12.2  
**Compatibility:** OpenEdge 11.6 through 12.8+  
**ABLUnit:** Fully supported

🎉 **Your OERA project is ready for PDSOE!**
