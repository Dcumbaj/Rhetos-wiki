Table of contents:

1. [Essential info](#1-essential-info)
2. [Simple row permission rules](#2-simple-row-permission-rules)
3. [Combining multiple rules](#3-combining-multiple-rules)
4. [Inheriting row permissions](#4-inheriting-row-permissions)
5. [Client code - Reading data with row permissions](#5-client-code---reading-data-with-row-permissions)
6. [Server code - Manually verifying row permissions](#6-server-code---manually-verifying-row-permissions)

## 1. Essential info

Row permissions are intended for implementing business requirements that can be formed in rules of the following format: *Some subset of employees may access (read or write) some subset of records.*
Row permissions are **not intended** for limiting access to specific **actions**, or filtering out some records in one part of the application while allowing the user to access those records in another part of the application.

Row permissions are based on filters that are applied when reading and writing entity's records:

* The user will be **denied read or write command** if the involved records do not pass the given row permissions filter.
* A client may **explicitly apply** the row permissions filter (`Common.RowPermissionsReadItems`) when reading data from the Rhetos server, in order to avoid any access-denied errors.
* Row permissions are **ignored** inside a server-side functions (inside an **Action** or a **FilterBy**, e.g.). To explicitly verify user's permissions in a sever action, see "Manually verifying row permissions" below.

## 2. Simple row permission rules

### Example requirements

* Each *employee* belongs to a *division*.
* An employee may read and write (insert/update/delete) the *documents* from his division.

### Solution

    Module DemoRowPermissions1
    {
        Entity Division
        {
            ShortString Name;
        }
        
        Entity Employee
        {
            ShortString UserName;
            Reference Division;
        }
        
        Entity Document
        {
            ShortString Title;
            DateTime Created { CreationTime; }
            Reference Division;
            
            RowPermissions
            {
                Allow WithinDivision 'context =>
                    {
                        Guid myDivisionId = context.Repository.DemoRowPermissions1.Employee.Query()
                            .Where(e => e.UserName == context.UserInfo.UserName)
                            .Select(e => e.Division.ID)
                            .SingleOrDefault();
                        return item => item.Division.ID == myDivisionId;
                    }';
            }
        }
    }

*This solution is implemented in Rhetos unit tests, see `RowPermissionsDemo.rhe` and `RowPermissionsDemo.cs` in the Rhetos source repository.*
 
### Explanation

`RowPermissions` is the root concept for row permission rules on a data structure.
It allows combining multiple rules and inheriting rules from one data structure to another.

`Allow` (both read and write), `Deny` (both read and write), `AllowRead`, `DenyRead`, `AllowWrite` and `DenyWrite` concepts inside `RowPermissions` represent the rules that allow or deny current user access to selected records.

Each rule has the following parameters:

1. **Rule name**
2. **Filter expression function**
    * This is a function that returns the filter expression: the resulting expression for each record (`item`) returns whether the rule applies to the record or not.  
    * The function typically first collects the data that will be used to filter the records, such as a list of business permission that are applied to the current user.
    * The parameter `context` can be used to retrieve the current user's name (`context.UserInfo.UserName`) or access data repositories (`context.Repository.SomeModule.SomeEntity.Query()`, replacing SomeModule and SomeEntity with the actual names).
    * The resulting expression will be translated to an SQL query (WHERE part), therefore it should be made as simple as possible, in order to allow efficient ORM translation.
    * This function is formatted as a lambda expression of type: `Func<Common.ExecutionContext, Expression<Func<TEntity, bool>>>`.


## 3. Combining multiple rules

### Example requirements

* Each *employee* belongs to a *division*.
* An employee may read and write (insert/update/delete) the *documents* from his division.
* Each division belongs to a *region*.
* An employee may be a *region supervisor*. A region supervisor may read (but not write) all the documents from all the divisions inside that region.
* The documents from *previous years* are read-only.

### Solution

    Module DemoRowPermissions2
    {
        Entity Region
        {
            ShortString Name;
        }
        
        Entity Division
        {
            ShortString Name;
            Reference Region;
        }
        
        Entity Employee
        {
            ShortString UserName;
            Reference Division;
        }
        
        Entity RegionSupervisor
        {
            Reference Employee;
            Reference Region;
        }
        
        Entity Document
        {
            ShortString Title;
            DateTime Created { CreationTime; }
            Reference Division;
            
            RowPermissions
            {
                Allow WithinDivision 'context =>
                    {
                        Guid myDivisionId = context.Repository.DemoRowPermissions2.Employee.Query()
                            .Where(e => e.UserName == context.UserInfo.UserName)
                            .Select(e => e.Division.ID)
                            .SingleOrDefault();
                        return item => item.Division.ID == myDivisionId;
                    }';
                
                AllowRead SupervisedRegions 'context =>
                    {
                        List<Guid> myRegionIds = context.Repository
                          .DemoRowPermissions2.RegionSupervisor.Query()
                            .Where(rs => rs.Employee.UserName == context.UserInfo.UserName)
                            .Select(rs => rs.Region.ID)
                            .ToList();
                        
                        if (myRegionIds.Count == 0)
                            return item => false; // Minor optimization.
                        
                        return item => myRegionIds.Contains(item.Division.Region.ID);
                    }';
                
                DenyWrite PreviousYears 'context =>
                    {
                        return item => item.Created < new DateTime(DateTime.Today.Year, 1, 1);
                    }';
            }
        }
    }

*This solution is implemented in Rhetos unit tests, see `RowPermissionsDemo.rhe` and `RowPermissionsDemo.cs` in the Rhetos source repository.*

### Explanation

The rules are combined into a single filter (check the generated SQL query in the SQL Profiler), based on the following principles:

* When the `RowPermissions` concept is added to an entity, everything is denied by default.
* Multiple `Allow*` rules are combined as a union of allowed data.
* `Deny*` rules have priority over `Allow*` rules.

Note that the `SupervisedRegions` rule implements an additional **optimization** that will simplify the final row permissions filter. The following filter expressions will be optimized when checking the row permissions:

* If the resulting filter expression is `item => false` (no items selected), the rule will be ignore when generating the final SQL query.
* Similarly, if an `Allow*` rule returns filter expression `item => true` (all items are selected), the other `Allow*` rules will be ignored, and the final SQL query will not contain any rule-specific filtering.


## 4. Inheriting row permissions

### Example requirements

* Row permissions for a `Document` should also be applied (*inherited*) to all of the document's **extensions**, including the **Browse** data structures, and to all of the document's **detail** entities.
* In addition to the inherited row permissions, a `DocumentApproval` record should be read-only for everyone except the employee that approved it.

### Solution 1

Add this script to the previous example's solution.

    Module DemoRowPermissions2
    {
        AutoInheritRowPermissions;
        
        Browse DocumentBrowse DemoRowPermissions2.Document
        {
            Take 'Title';
            Take 'Division.Name';
        }
        
        Entity DocumentComment
        {
            Reference Document { Detail; }
            ShortString Comment;
        }
        
        Entity DocumentApproval  
        {
            Extends DemoRowPermissions2.Document;
            Reference ApprovedBy DemoRowPermissions2.Employee;
            ShortString Note;
            
            RowPermissions
            {
                // This rule is joined with the inherited rules from DemoRowPermissions2.Document.
                DenyWrite ApprovedByCurrentUser 'context =>
                    {
                        var myEmployeeId = context.Repository.DemoRowPermissions2.Employee.Query()
                            .Where(e => e.UserName == context.UserInfo.UserName)
                            .Select(e => e.ID)
                            .SingleOrDefault();
                        return item => item.ApprovedBy.ID != myEmployeeId;
                    }';
            }
        }
    }

*This solution is implemented in Rhetos unit tests, see `RowPermissionsDemo.rhe` and `RowPermissionsDemo.cs` in the Rhetos source repository.*

### Explanation 1

By using `AutoInheritRowPermissions`, the row permissions from `Document` entity are copied to the following related entities:

* extensions of the entity (`DocumentApproval`, see **Extends**).
* browse data structures (`DocumentBrowse`, **Browse** is also an extension).
* details of the entity (`DocumentComment`, see **Detail** reference). 

### Solution 2 

Instead of using `AutoInheritRowPermissions` (see the previous solution), row permissions inheritance may be explicitly set for selected entities.  

    Module DemoRowPermissions2
    {
        // NOT USING AutoInheritRowPermissions;
        
        Browse DocumentBrowse DemoRowPermissions2.Document
        {
            Take 'Title';
            Take 'Division.Name';
            
            RowPermissions { InheritFromBase; }
        }
        
        Entity DocumentComment
        {
            Reference Document { Detail; }
            ShortString Comment;
            
            RowPermissions { InheritFrom DemoRowPermissions2.DocumentComment.Document; }
        }
        
        Entity DocumentApproval  
        {
            Extends DemoRowPermissions2.Document;
            Reference ApprovedBy DemoRowPermissions2.Employee;
            ShortString Note;
            
            RowPermissions
            {
                InheritFromBase;
                
                // This rule is joined with the inherited rules from DemoRowPermissions2.Document.
                DenyWrite ApprovedByCurrentUser 'context =>
                    {
                        var myEmployeeId = context.Repository.DemoRowPermissions2.Employee.Query()
                            .Where(e => e.UserName == context.UserInfo.UserName)
                            .Select(e => e.ID)
                            .SingleOrDefault();
                        return item => item.ApprovedBy.ID != myEmployeeId;
                    }';
            }
        }
    }

### Explanation 2

* **InheritFromBase** can be used on a **Browse** data structures and on entities with **Extends** concept.
* **InheritFrom** concept's parameter is the full name of `DocumentComment` entity's reference property (`Reference Document`) that references the "parent" entity with row permissions that will be inherited.


## 5. Client code - Reading data with row permissions

*The following examples use the test data from Rhetos unit tests. To prepare the data, open the `CommonConceptsTest.sln` solution in the Rhetos source and run the tests in the `RowPermissionsDemo` class.*

### Reading all documents (access denied)

Use REST API to read all documents, including those that a current user should no access: 

    http://localhost/Rhetos/rest/DemoRowPermissions1/Document/

Sever response should contain an error:

    "UserMessage": "Insufficient permissions to access some or all of the data requested."

### Reading the user's documents

Add the `Common.RowPermissionsReadItems` filter to read only documents that a user is allowed to read:

    http://localhost/Rhetos/rest/DemoRowPermissions1/Document/?filters=[{"Filter":"Common.RowPermissionsReadItems"}]

Response example:

    {
        "Records": [
            {
                "Created": "/Date(1421326381787+0100)/",
                "DivisionID": "0734c084-7427-4b13-9101-8e8e200131de",
                "ID": "8353d88d-ee2e-479c-a3cb-f574ee0ea8c1",
                "Title": "doc1"
            }
        ]
    }


## 6. Server code - Manually verifying row permissions

Row permissions are automatically checked for client's read and write requests. Row permissions are ignored inside a server-side functions that use ServerDom repositories the read and write data (inside an **Action** or a report, for example). To explicitly verify the current user's permissions inside the server code, use one of the following methods:

1. Use `IProcessingEngine` to execute server commands to read/write data. The processing engine will check the current user's permissions, including row permission.
2. Directly use row permissions filters (`Common.RowPermissionsReadItems` and `Common.RowPermissionsWriteItems`) on an entity with row permissions, to verify the data before writing it to the database or sending it to the user.