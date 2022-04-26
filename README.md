# Lab 3: Develop Solutions Using Amazon DynamoDB

## Task 1: Create a DynamoDB Table using the CLI
- Step 11: Created a DynamoDB Table
```
$ aws dynamodb create-table ^
  --table-name Notes ^
  --attribute-definitions AttributeName=UserId,AttributeType=S AttributeName=NoteId,AttributeType=N ^
  --key-schema AttributeName=UserId,KeyType=HASH AttributeName=NoteId,KeyType=RANGE ^
  --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5 

TableDescription:
  AttributeDefinitions:
  - AttributeName: NoteId
    AttributeType: N
  - AttributeName: UserId
    AttributeType: S
  CreationDateTime: '2022-04-26T15:06:06.045000+00:00'
  ItemCount: 0
  KeySchema:
  - AttributeName: UserId
    KeyType: HASH
  - AttributeName: NoteId
    KeyType: RANGE
  ProvisionedThroughput:
    NumberOfDecreasesToday: 0
    ReadCapacityUnits: 5
    WriteCapacityUnits: 5
  TableArn: arn:aws:dynamodb:eu-west-2:760155606124:table/Notes
  TableId: f4da8eb0-1b39-4d9a-ba9c-880a6a09f305
  TableName: Notes
  TableSizeBytes: 0
  TableStatus: CREATING
```

- Step 12: Waited until the table exists
```
$ aws dynamodb wait table-exists --table-name Notes
(no output)
```

- Step 13: Confirmed the table was created:
```
$ aws dynamodb describe-table --table-name Notes | findstr TableStatus
  TableStatus: ACTIVE
```

## Task 2: Loading data into the table
Editing notesLoadData.java

- Step 16: Created a new `AmazonDynamoDB` instance to connect to the DB
```java
        //Create DynamoDB client
        AmazonDynamoDB client = AmazonDynamoDBClientBuilder.standard()
                .build();

        //Use the DynamoDB document API wrapper
        DynamoDB dynamoDB = new DynamoDB(client);
```

- Step 17: Iterated over the entries in notes.json and added each one to the DB
```java
table.putItem(
               new Item()
               .withPrimaryKey("UserId", userId, "NoteId", noteId)
               .withString("Note", note)
                );
```
- Step 21: Executed the class to insert the data:
```
$ mvn -q exec:java -Dexec.mainClass="dev.labs.dynamodb.notesLoadData"
 Loading "Notes" table with data from file "notes.json"

PutItem succeeded: testuser 1 hello
PutItem succeeded: testuser 2 this is my first note
PutItem succeeded: newbie 1 Free swag code: 1234
PutItem succeeded: newbie 2 I love DynamoDB
PutItem succeeded: student 1 DynamoDB is NoSQL
PutItem succeeded: student 2 A DynamoDB table is schemaless
PutItem succeeded: student 3 PartiQL is a SQL compatible language for DynamoDB
PutItem succeeded: student 5 Maximum size of an item is ____ KB ?
PutItem succeeded: student 4 I love DynamoDB
```

## Task 3: Querying data
Editing notesQuery.java
- Step 25: Created a `QuerySpec` instance to query the note for a given student:
```java
        QuerySpec spec = new QuerySpec()
                .withProjectionExpression("NoteId, Note")
                .withKeyConditionExpression("UserId = :v_Id")
                .withValueMap(new ValueMap()
                        .withString(":v_Id", userId));
```
- Step 27: Executed the query to read from the table:
```java
 ItemCollection<QueryOutcome> items = table.query(spec);
```
- Step 31: Executed the class to execute the query:
```
$ mvn -q exec:java -Dexec.mainClass="dev.labs.dynamodb.notesQuery"
 Query all notes belong to a user "student" and displaying "NoteId" and "Note" attributes only:
{
  "NoteId" : 1,
  "Note" : "DynamoDB is NoSQL"
}
{
  "NoteId" : 2,
  "Note" : "A DynamoDB table is schemaless"
}
{
  "NoteId" : 3,
  "Note" : "PartiQL is a SQL compatible language for DynamoDB"
}
{
  "NoteId" : 4,
  "Note" : "I love DynamoDB"
}
{
  "NoteId" : 5,
  "Note" : "Maximum size of an item is ____ KB ?"
}

```

## Task 4: Scanning the table with a paginator
editing notesScan.java
- Step 35: Created a `ScanSpec` instance to search for notes containing a given String:
```java
ScanSpec scanSpec = new ScanSpec()
                .withFilterExpression("contains (Note, :v_txt)")
                .withValueMap(new ValueMap().withString(":v_txt", searchText))
                .withProjectionExpression("UserId, NoteId, Note");
```
- Step 43: Executed the class to scan the table
```
$ mvn -q exec:java -Dexec.mainClass="dev.labs.dynamodb.notesScan"
 Scan table to list items with search text "SQL" as part of the note:

SCANNING TABLE...

Page: 1

Page: 2

Page: 3
{
  "NoteId" : 1,
  "UserId" : "student",
  "Note" : "DynamoDB is NoSQL"
}

Page: 4

Page: 5
{
  "NoteId" : 3,
  "UserId" : "student",
  "Note" : "PartiQL is a SQL compatible language for DynamoDB"
}

Page: 6

Page: 7

Page: 8

Page: 9

Page: 10
```

## Task 5: Update an item in the table
editing notesUpdate.java
- Step 47: Created an `UpdateItemSpec` instance to update an item and return all new/changed items
```java
UpdateItemSpec updateItemSpec = new UpdateItemSpec()
.withPrimaryKey("UserId", userId, "NoteId", noteId)
.withUpdateExpression("set #inc = :val1")
.withNameMap(new NameMap()
.with("#inc", "Is_Incomplete"))
.withValueMap(new ValueMap()
.withString(":val1", "Yes"))
.withReturnValues(ReturnValue.ALL_NEW);
```
- Step 50: Executed the class to perform the update:
```
$ mvn -q exec:java -Dexec.mainClass="dev.labs.dynamodb.notesUpdate"
UPDATE#1: Printing item after adding the new attribute "Is_Incomplete" :
{
  "NoteId" : 5,
  "Is_Incomplete" : "Yes",
  "UserId" : "student",
  "Note" : "Maximum size of an item is ____ KB ?"
}
```
- Step 57: Caught an exception because the condition did not match:
```java
catch (ConditionalCheckFailedException e) {
            System.out.println("\nUPDATE#2 - REPEAT: Printing item after the conditional update for the item - \"" + userId + "\" and \"" + noteId + "\"  - FAILURE:");
            System.out.println("UpdateItem failed on item due to unmatching condition!");
            System.err.println(e.getMessage());
        }
```
- Step 59: Performed the update twice to force the exception being thrown:
```java
        //Allow update to the Notes item only if the note is incomplete - SUCCESS
        updateExistingAttributeConditionally(table, qUserId, qNoteId, newNote);

        //Allow update to the Notes item only if the note is incomplete - FAILURE
        updateExistingAttributeConditionally(table, qUserId, qNoteId, newNote);
```
- Step 62: Executed the class to see what happens:
```
$ mvn -q exec:java -Dexec.mainClass="dev.labs.dynamodb.notesUpdate"
UPDATE#1: Printing item after adding the new attribute "Is_Incomplete" :
{
  "NoteId" : 5,
  "Is_Incomplete" : "Yes",
  "UserId" : "student",
  "Note" : "Maximum size of an item is ____ KB ?"
}

UPDATE#2: Printing item after the conditional update for the item - "student" and "5"  - SUCCESS:
{
  "Is_Incomplete" : "No",
  "Note" : "Maximum item size in DynamoDB is ___ KB"
}

UPDATE#2 - REPEAT: Printing item after the conditional update for the item - "student" and "5"  - FAILURE:
UpdateItem failed on item due to unmatching condition!
The conditional request failed (Service: AmazonDynamoDBv2; Status Code: 400; Error Code: ConditionalCheckFailedException; Request ID: 9MRJP0GNBU0REDKGS8JBO7U48JVV4KQNSO5AEMVJF66Q9ASUAAJG; Proxy: null)
```

## Task 6: Using the DynamoDBMapper for CRUD operations
editing notesCRUDmapper.java
- 

To read up:
- DynamoDb