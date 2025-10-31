# OERA Project Documentation

## Overview

This project contains comprehensive documentation and templates for implementing OERA (OpenEdge Reference Architecture) in legacy OpenEdge applications. The goal is to modernize existing applications by following OERA principles, object-oriented design, and SOLID principles.

## Project Contents

### 1. OERA_Implementation_Guide.md
**Purpose:** Complete implementation guide with detailed Invoice example  
**Use When:** Learning OERA concepts and understanding the full architecture  

**Contents:**
- OERA architecture overview and principles
- SOLID principles explanation for OpenEdge
- Complete Invoice entity implementation (all layers)
- Dataset include file with REFERENCE-ONLY parameter strategy
- Interface usage and dependency injection
- Unit testing introduction
- Best practices and common pitfalls
- Quick reference for parameter patterns

### 2. OERA_Generic_Template.md
**Purpose:** Copy-paste template for implementing any entity  
**Use When:** Creating new functionality for any table in your application  

**Contents:**
- Generic template with `{Entity}` placeholders
- Complete code for all layers (Dataset, DAO Interface/Class, BE Interface/Class, Presentation)
- Mock DAO and unit test templates
- Find-and-replace guide
- Step-by-step checklist
- Common validation patterns
- Customization examples

### 3. UNIT_TESTING_GUIDE.md
**Purpose:** Complete guide to unit testing with ABLUnit framework  
**Use When:** Setting up unit tests for your OERA implementation  

**Contents:**
- ABLUnit framework basics
- Creating mock DAO objects
- Writing test classes
- Testing validation logic
- Testing business rules
- Running tests
- Best practices
- Complete working examples

## Quick Start

### For Learning OERA:
1. Read `OERA_Implementation_Guide.md` thoroughly
2. Study the Invoice example to understand layer interaction
3. Review the SOLID principles section
4. Understand the REFERENCE-ONLY strategy

### For Implementing New Functionality:
1. Open `OERA_Generic_Template.md`
2. Identify which entity/table you're working on
3. Copy each template section (Dataset, DAO, BE, Presentation)
4. Find-and-replace `{Entity}` with your entity name
5. Customize fields, validation, and business rules
6. Follow the checklist to ensure completeness

## Core OERA Principles

### 1. Layer Separation
- **Presentation Layer:** UI only, no business logic or database access
- **Business Entity Layer:** Business logic, validation, calculations
- **Data Access Layer:** Database CRUD operations only

### 2. Dataset Communication
- Define datasets once with `{&REFERENCE-ONLY}` parameter
- Use `BY-REFERENCE` for all dataset parameters
- Presentation layer includes dataset WITHOUT REFERENCE-ONLY
- Business and Data layers include dataset WITH REFERENCE-ONLY

### 3. Object-Oriented Design
- Use interfaces for DAO layer (minimum requirement)
- Optionally use interfaces for BE layer (for testing/flexibility)
- Apply dependency injection through constructors
- Follow SOLID principles

## File Organization

Recommended directory structure:

```
/YourApplication
    /Business
        /DataDefinitions
            InvoiceDataset.i
            CustomerDataset.i
            OrderDataset.i
            ...
        /DataAccess
            IInvoiceDAO.cls
            InvoiceDAO.cls
            ICustomerDAO.cls
            CustomerDAO.cls
            ...
        /Entity
            IInvoiceBE.cls
            InvoiceBE.cls
            ICustomerBE.cls
            CustomerBE.cls
            ...
        /Task
            InvoicePostingTask.cls
            OrderProcessingTask.cls
            ...
    /Presentation
        InvoiceWindow.w
        CustomerWindow.w
        OrderWindow.w
        ...
    /Test
        /Mock
            MockInvoiceDAO.cls
            MockCustomerDAO.cls
            ...
        /Unit
            InvoiceBETest.cls
            CustomerBETest.cls
            ...
```

## Why Use Interfaces?

Interfaces provide critical benefits for your OERA implementation:

### 1. **Unit Testing**
- Test business logic without database
- Create mock implementations for testing
- Run tests in milliseconds instead of seconds
- Test validation and business rules in isolation

### 2. **Flexibility**
- Multiple implementations (local DB, AppServer, REST API)
- Easy to swap implementations
- Support different data sources

### 3. **Dependency Inversion (SOLID)**
- Depend on abstractions, not concrete classes
- Loose coupling between layers
- Easier to maintain and extend

### 4. **Design by Contract**
- Clear contract definition
- Self-documenting code
- Method signatures enforced by compiler

**See [UNIT_TESTING_GUIDE.md](UNIT_TESTING_GUIDE.md) for complete testing examples.**

---

## Dataset Include File Strategy

### Single Include File with Parameter

```openedge
/* EntityDataset.i */
&IF "{&REFERENCE-ONLY}" = "" &THEN
    &SCOPED-DEFINE REFERENCE-ONLY
&ENDIF

DEFINE TEMP-TABLE ttEntity NO-UNDO {&REFERENCE-ONLY}
    FIELD EntityID AS INTEGER
    ...
DEFINE DATASET dsEntity {&REFERENCE-ONLY} FOR ttEntity.
```

### Usage by Layer

**Presentation Layer (creates actual dataset):**
```openedge
{EntityDataset.i}
```

**Business/Data Access Layers (reference only):**
```openedge
{EntityDataset.i &REFERENCE-ONLY=REFERENCE-ONLY}
```

## When to Use Interfaces

### Recommended Approach for Legacy Applications:

**Minimum (Pragmatic):**
- DAO Interface: YES (for testing and AppServer flexibility)
- BE Interface: OPTIONAL (add later if needed)

**Full OERA (Best Practice):**
- DAO Interface: YES
- BE Interface: YES

### Benefits of Interfaces:
1. Unit testing with mock objects
2. Multiple implementations (different data sources)
3. AppServer deployment flexibility
4. Dependency injection and loose coupling
5. Design by contract

## Common Patterns

### Required Field Validation
```openedge
IF bttEntity.FieldName = "" OR bttEntity.FieldName = ? THEN DO:
    cErrorMessage = "FieldName is required.".
    RETURN FALSE.
END.
```

### Status Validation
```openedge
IF LOOKUP(bttEntity.Status, "ACTIVE,INACTIVE,PENDING") = 0 THEN DO:
    cErrorMessage = "Invalid status.".
    RETURN FALSE.
END.
```

### Date Range Validation
```openedge
IF bttEntity.StartDate > bttEntity.EndDate THEN DO:
    cErrorMessage = "Start date cannot be after end date.".
    RETURN FALSE.
END.
```

### Audit Fields
```openedge
ASSIGN
    cCurrentUser = USERID("DICTDB")
    dtNow = NOW.

IF bttEntity.EntityID = 0 OR bttEntity.EntityID = ? THEN
    ASSIGN bttEntity.CreatedBy = cCurrentUser
           bttEntity.CreatedDate = dtNow.

ASSIGN bttEntity.ModifiedBy = cCurrentUser
       bttEntity.ModifiedDate = dtNow.
```

## Implementation Checklist

For each new entity, complete these steps:

### 1. Dataset Definition
- [ ] Create `EntityDataset.i` with `{&REFERENCE-ONLY}` parameter
- [ ] Define temp-table with all fields matching database
- [ ] Add appropriate indexes
- [ ] Define dataset with REFERENCE-ONLY parameter

### 2. Data Access Layer
- [ ] Create `IEntityDAO.cls` interface
- [ ] Include dataset with REFERENCE-ONLY parameter
- [ ] Define Fetch methods (OUTPUT DATASET BY-REFERENCE)
- [ ] Define Save method (INPUT-OUTPUT DATASET BY-REFERENCE)
- [ ] Define Delete method
- [ ] Create `EntityDAO.cls` implementation
- [ ] Implement all interface methods
- [ ] Use buffers for database access
- [ ] Wrap saves in transactions

### 3. Business Entity Layer
- [ ] Create `IEntityBE.cls` interface
- [ ] Include dataset with REFERENCE-ONLY parameter
- [ ] Define Get methods
- [ ] Define Save with validation
- [ ] Define Delete with business rules
- [ ] Create `EntityBE.cls` implementation
- [ ] Inject DAO through constructor
- [ ] Add constructor overload for testing
- [ ] Implement validation logic
- [ ] Implement business rules
- [ ] Add audit field management

### 4. Presentation Layer
- [ ] Include dataset WITHOUT REFERENCE-ONLY parameter
- [ ] Create UI components
- [ ] Instantiate BE using interface type
- [ ] Call BE methods with BY-REFERENCE
- [ ] Handle errors from BE layer
- [ ] Never access DAO or database directly

### 5. Unit Testing (Optional but Recommended)
- [ ] Create mock DAO implementation
- [ ] Create test class with ABLUnit
- [ ] Write tests for validation rules
- [ ] Write tests for business rules
- [ ] Run tests and verify all pass
- [ ] See [UNIT_TESTING_GUIDE.md](UNIT_TESTING_GUIDE.md)

## Key Rules to Remember

1. **REFERENCE-ONLY is for definitions, not data**
   - Prevents multiple copies of dataset schema
   - Use with BY-REFERENCE for true pass-by-reference

2. **Presentation owns the dataset**
   - Does NOT use REFERENCE-ONLY
   - Creates the actual dataset instance
   - Passes to business/data layers BY-REFERENCE

3. **Business and Data layers reference the dataset**
   - DO use REFERENCE-ONLY
   - Work with datasets passed from caller
   - Never create their own dataset instances

4. **Always use BY-REFERENCE for dataset parameters**
   - Avoids copying data
   - Enables all layers to work with same dataset
   - Critical for performance

5. **Layer responsibilities**
   - Presentation: UI only
   - Business Entity: Validation, business rules, calculations
   - Data Access: CRUD operations only

6. **Use interfaces for flexibility**
   - Minimum: DAO layer
   - Recommended: DAO and BE layers
   - Enables testing, multiple implementations

## Common Pitfalls to Avoid

1. ❌ **DON'T** access database directly from presentation layer
2. ❌ **DON'T** put business logic in DAOs
3. ❌ **DON'T** put UI logic in Business Entities
4. ❌ **DON'T** pass datasets BY-VALUE (performance issue)
5. ❌ **DON'T** forget to validate in the BE layer
6. ❌ **DON'T** create "god classes" that do everything
7. ❌ **DON'T** skip error handling
8. ❌ **DON'T** use REFERENCE-ONLY in presentation layer

## Benefits of This Approach

1. **Maintainability**
   - Clear separation of concerns
   - Easy to locate and fix issues
   - Changes isolated to specific layers

2. **Testability**
   - Mock objects for unit testing
   - Test layers independently
   - Validate business logic without database

3. **Reusability**
   - Business logic used by multiple UIs
   - Same BE for web, desktop, mobile
   - DAO reused across business entities

4. **Performance**
   - BY-REFERENCE avoids data copying
   - REFERENCE-ONLY avoids schema duplication
   - Efficient memory usage

5. **Scalability**
   - Layers can be deployed separately
   - AppServer for business/data layers
   - Multiple presentation options

6. **Team Development**
   - Multiple developers on different layers
   - Clear contracts between layers
   - Parallel development possible

## FAQ

### Q: Do I need both DAO and BE interfaces?
**A:** Minimum requirement is DAO interface. BE interface is optional but recommended for complex business logic and testing.

### Q: What's the difference between REFERENCE-ONLY and BY-REFERENCE?
**A:** 
- REFERENCE-ONLY: Prevents creating dataset definition/schema copy
- BY-REFERENCE: Prevents creating dataset data copy
- Use both together for optimal performance

### Q: When should I create a Business Task class?
**A:** When you have complex operations that:
- Involve multiple entities
- Require workflow orchestration
- Need transaction coordination across multiple BEs
- Have complex business processes (e.g., order processing, invoice posting)

### Q: Can I skip the interface for simple entities?
**A:** For very simple CRUD with no testing needs, you can skip BE interface. However, always create DAO interface for database flexibility.

### Q: How do I handle parent-child relationships?
**A:** 
- Define both temp-tables in same include file
- Use DATA-RELATION in dataset definition
- DAO fetches both parent and children
- BE validates both levels

### Q: What about multi-tenant applications?
**A:** 
- Use interfaces for maximum flexibility
- Different BE implementations per tenant
- Inject appropriate implementation at runtime

## Version History

- **v1.0** - Initial documentation with Invoice example and generic template
- Created: 2025-01-27

## Support and Feedback

For questions or suggestions about this documentation:
1. Review the examples in the Implementation Guide
2. Check the Generic Template for copy-paste code
3. Refer to this README for quick answers

## Next Steps

1. ✅ Read the Implementation Guide thoroughly
2. ✅ Study the Invoice example
3. ✅ Use the Generic Template for your first entity
4. ✅ Follow the checklist for consistency
5. ✅ Build and test each layer independently
6. ✅ Maintain this pattern for all new functionality

---

**Remember:** Consistency is key. Follow these patterns for every new entity to maintain a clean, maintainable, and scalable codebase.
