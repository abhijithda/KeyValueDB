# KeyValueDB

A simple Key Value DB with transaction support 


### My initial thoughts were to (in the 45 min window):

Use simple map[string]int for keyvalue store/db (without transaction consideration at that point).
With transaction support, I would use stack to keep track of databases. Accessing a key that was not set in current transaction means, going back in stack step-by-step for each entry of db in stack.
For delete key, I mentioned 3 approaches earlier - using separate map to maintain deleted keys info; save commands i.e., audit-trail for replay later during commit; and using specific value to indicate it's deleted.


### Optimizations I thought later:

To help understand better, I have reorganized the contents (so, it is currently not in the thought flow anymore!)...

High Level

KeyValStore:
    | Key |  Value | Deleted | UpdatesStack []{Values, Deleted, Transaction-ID} |

```go
    map[string] struct { Key string, Value int, Deleted bool, UpdatesStack []struct{ Values int, Deleted bool, tID string} } 
```
    
Transactions-Stack (tracking all transactions IDs and their Keys):
    {tID-N, *tKeys} | {tID-N-1, *tKeys} | ..... | { tID-2, *tKeys} | {tID-1, *tKeys} |

```go
    []struct{ tID string, tKeys map[string]struct{}}
```

Where, tKeys (Transaction-Keys) DB will track individual transaction updates:
    | Key |
    
```go
    map[string]struct{}
```

SET:
If Transactions-Stack is empty, it means update only Key and Value in KeyValStore.

Time Complexity: O(1)

GET:
If Transactions-Stack is empty, retrieve Value field.

Time Complexity: O(1)

UNSET:
If Transactions-Stack is empty, delete Key i.e., entire row (Or set Deleted field --- any cases where we need this?!).

Time Complexity: O(1)

BEGIN:
Create a new transaction ID, and push it on to Transactions-Stack

Time Complexity: O(1)

SET (after BEGIN):
If Key doesn't exist, then add a new row along with setting 'Deleted' field. and change UpdatesStack and Transaction-Keys as below.
Transaction-Keys will add Key if not already present.
If UpdatesStack in KeyValStore is empty, then push speficied value along with current transaction-id (by getting top transaction ID from Transactions-Stack).
If UpdatesStack in KeyValStore is not empty, then 
    check if top of the stack is from the same transaction, 
        if yes, change value present on top of stack and reset "Deleted" flag.
        if no, push a new entry - value and transaction ID.

Time Complexity: O(1)

GET (after BEGIN): 
Transactions-Stack is not empty, so, get Value from top of the UpdatesStack.
If top of the UpdatesStack has Deleted flag set, then return Null.

Time Complexity: O(1)

UNSET (after BEGIN): 
If Key not present in KeyValStore, then no-op.
If Key present, then
Transaction-Keys will add Key if not already present  (Not needed anymore: along with setting Deleted flag).
Transactions-Stack may not be empty, so, set Deleted to true if top of the UpdatesStack is of the same transaction ID.
If transaction ID of top value of UpdatesStack is not same, then push new entry by setting 'Deleted' field and the current transaction ID.

Time Complexity: O(1)

COMMIT/ROLLBACK without BEGIN:
I.e., Transactions-Stack is empty, so no-op.

COMMIT (after BEGIN):
Transactions-Stack is not empty, so, for each key in Transaction-Keys, perform the action on the main Key/Value, by popping and taking the value (or setting Delete field) from top entry from UpdatesStack when transaction ID matches.

At the end of COMMIT, the transaction ID from Transactions-Stack would be popped, Transactions-Keys list would deleted, And all the transactions/value on the UpdatesStack would also be removed.

Time Complexity: O(K) - where K is the number of keys set or unset.

ROLLBACK (after BEGIN):
Transactions-Stack is not empty, so, for each key in Transaction-Keys, pop latest entry from UpdatesStack if transaction ID matches.

Time Complexity: O(K) - where K is the number of keys set or unset.



//// BEGIN> OLD content ----------------------------------

Instead of a special value to indicate a deleted key, a new "deleted" field could be used. Using a separate map or a specific value may save space compared with new "deleted" field in each row. Any Trade-offs for any use cases?!)

type KeyValDB map[string]struct{   value int   deleted bool}

GET:With multiple BEGIN statements, instead of going through stack for DBs, and checking if key is set... I would keep a new column in DB/map to maintain stack of values at the key/row level.  With this I would need to always look at the top of the stack to get the value of a key (and don't have to go through multiple dbs to see if the key was set).
For COMMIT's and ROLLBACK's sake, I would still need to maintain keys set or deleted in the transaction. 

Everytime a "BEGIN" statement is executed, we need to create a transaction ID, and push it on to transactions stack.

When a SET is run, if transactions stack is empty, then changes should be done at the row/key level. 



//// END> OLD content ----------------------------------
