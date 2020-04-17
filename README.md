## Instructions 
If you are using a SQL database of some sort, you can any of the following:
- Copy the code for the provided test function and paste it in your own test file

__-----OR-----__

- You can either clone the repo, or simply just download a zip file and transfer the `store_test.go`
- RENAME the file based on the dialect of SQL you're using:
  - `mysqlstore_test.go` if you're using MySQL 
  - `postgressqlstore_test.go` if you're using PostgresSQL 
- Implement the rest of the test functions

If you are using MongoDB, the test function WILL NOT WORK for you, but you can still use the file to write out your test functions. If you do so, name the file `mongodbstore_test.go`

## Example/Tutorial
We're now at the point where we want to write our own unit tests to test functionality of our web servers and databases. However, we don't want to keep making calls to our __actual__ database because it can be expensive to do so and we don't want to mess with the actual data. Refer to <a href=https://youtu.be/tu9tWF4N-9E>William's Mocking Rationale video</a> for a novel example of mocking in JavaScript and a deeper explanation on why we do this.

By now you should've written unit tests in Golang, especially for Redis in A3. __If you have not written Golang unit tests yet__, please read Dr. Stearns reading on <a href=https://drstearns.github.io/tutorials/testing/>Automated Testing in Go</a>.

We will be mocking a SQL database using the <a href=https://github.com/DATA-DOG/go-sqlmock>SQLMock library</a>. You will also be given one of the test function `TestGetByID()` so you can test your `GetByID()` function and use this as a basis to write your other test functions! This README will contain a detailed explanation of what the code is doing. 
  
In essence, we will be giving our mock database basic information and set up __expectations__ to meet when testing our implementation. There are different functions to set up expectations. In this example, you will primarily deal with queries and will see `ExpectQuery().WithArgs()`, `WillReturnRows()` and `WillReturnError()`. __You will need to do some documentation reading__ when implementating insertions, updates, and deletions, but they syntax is similar.
  
### Create Header Function and High Level Overview of Function
First, create the function header, following the proper convention. Because we're testing `GetByID()`, the proper name for the function would be `TestGetByID()`. The parameter is also required. The `testing` library should be imported for you if you're using VS Code and the Go extensions. I'll also include the algorithm in pseudocode so you have a better high level overview of what is happening as tests functions can get pretty lengthy.
  
```
// TestGetByID is a test function for the SQLStore's GetByID
func TestGetByID(t *testing.T) {

  /* 
    Algorithm for this test:
    
    create a slice of all cases to test
    for all the cases in the slice {
      initialize a mock DB
      create and add an EXPECTED ROW to the mock db
      if the case expects an error {
        set up an EXPECTED QUERY and ensure that it returns an ERROR for an unsuccessful query
        test our GetByID() function to see that it returns an error 
      } else {
        set up an EXPECTED QUERY and ensure that it returns the EXPECTED ROW for a successful query
        test our GetByID() function to see that it returns the correct use  
      }
      make sure we met all expectations of the mock database
    }
  */
  
}
``` 

### Create Test Cases
We want to know what cases will be successful and which cases will not be. In this example, we will test a successful ID query, an unsuccessful ID query, and a query with a large ID value. In each case, we will need the __name__ of the case (for easy identification), the __user__ we will insert and expect to receive from the mock database, the __ID__ we will be using for the query, and whether or not we __expect an error__.
```
cases := []struct {
  name         string
  expectedUser *User
  idToGet      int64
  expectError  bool
}{
  {
    "User Found",
    &User{
      1,
      "test@test.com",
      []byte("passhash123"),
      "username",
      "firstname",
      "lastname",
      "photourl",
    },
    1,
    false,
  },
  {
    "User Not Found",
    &User{},
    2,
    true,
  },
  {
    "User With Large ID Found",
    &User{
      1234567890,
      "test@test.com",
      []byte("passhash123"),
      "username",
      "firstname",
      "lastname",
      "photourl",
    },
    1234567890,
    false,
  },
}
```

### Initializing the Mock DB
For each case, we will initialize a mock database using `sqlmock.New()`. This is similar to opening an actual database connection, but used for testing purposes only. Queries, insertion, updates, and deletion will work just like an actual database. 
```
db, mock, err := sqlmock.New()
if err != nil {
  t.Fatalf("There was a problem opening a database connection: [%v]", err)
}
defer db.Close()

// TODO: update based on the name of your type struct
mySQLStore := &SQLStore{db} 
```

### Creating and Adding an Expected Row to the Mock Database
Now, let's add a row to the mock database with the given expected user of each case. `NewRows()` takes in a string slice that contains the __column names__ of a given table. From there, you can use `AddRow()` to add the values of the expected user. The number of values passed into `AddRow()` MUST MATCH the number of columns passed into `NewRows()` or else you will get a runtime error. We can save this in a `row` variable that we will use later.
```
// Create an expected row to the mock DB
row := mock.NewRows([]string{
  "ID",
  "Email",
  "PassHash",
  "UserName",
  "FirstName",
  "LastName",
  "PhotoURL"},
).AddRow(
  c.expectedUser.ID,
  c.expectedUser.Email,
  c.expectedUser.PassHash,
  c.expectedUser.UserName,
  c.expectedUser.FirstName,
  c.expectedUser.LastName,
  c.expectedUser.PhotoURL,
)
```

### Using Your Query of Choice
Your Store functions `GetByID()`, `GetByEmail()`, and `GetByUserName()` should all use the standard `SELECT FROM WHERE` query with the given condition. The given query is what worked for me, and is probably the safest bet.

__NOTE__: Your query for the mock database must match the query you use in your function implementations (I believe mismatches in casing and whitespace is okay). I ran into runtime errors if I used `SELECT id,email,passhash...` in my function but used `SELECT *...` in my test function for the mock database. If you had it the other way around, you might fail a test case.

If you do opt to use `SELECT *` to retrieve all the columns, you might need to utilize the `regexp.QuoteMeta()` function to translate certain regular expression metacharacters to a literal string. 
```
// TODO: update to match the query used in your Store implementation
query := "select id,email,passHash,username,firstName,lastName,photoUrl from Users where id=?"
```

### Unsuccessful Query Check
If the case is supposed to be unsuccessful:
- Set up the mock database to expect a query that will return an error of some sort
- Run `GetByID()` and make sure there is an error.

`ExpectQuery()` takes in a string that will act as the query that the mock database is expecting to receive. `WithArgs()` takes the values to replace in the query. `WillReturnError()` will take in an error of some sort; this query will then expect an error.
```
// Set up expected query that will expect an error
mock.ExpectQuery(query).WithArgs(c.idToGet).WillReturnError(ErrUserNotFound)

// Test GetByID()
user, err := mainSQLStore.GetByID(c.idToGet)
if user != nil || err == nil {
  t.Errorf("Expected error [%v] but got [%v] instead", ErrUserNotFound, err)
}
```

### Successful Query Check
If the case is supposed to be succesful:
- Set up the mock database to expect a query that will return the expected user
- Run `GetByID()` and make sure it returns the correct user
- Make sure there are no errors 

`ExpectQuery()` takes in a string that will act as the query that the mock database is expecting to receive. `WithArgs()` takes the values to replace in the query. `WillReturnRows()` will take in a set or rows from the mock database. We will pass in our `row` variable that we created as that contains the row with the expected values.
```
// Set up an expected query with the expected row from the mock DB
mock.ExpectQuery(query).WithArgs(c.idToGet).WillReturnRows(row)

// Test GetByID()
user, err := mainSQLStore.GetByID(c.idToGet)
if err != nil {
  t.Errorf("Unexpected error on successful test [%s]: %v", c.name, err)
}
if !reflect.DeepEqual(user, c.expectedUser) {
  t.Errorf("Error, invalid match in test [%s]", c.name)
}
```

### Ensure that we Met All Expectations
We set up two main expectaions: 
- `ExpectQuery(query).WithArgs(c.idToGet).WillReturnRows(row)` for successful queries
- `ExpectQuery(query).WithArgs(c.idToGet).WillReturnError(ErrUserNotFound)` for unsuccessful queries

If any of these are unmet (either by skipping over them or not matching them), then we technically can't pass our test as there are still expectations to be met.
```
if err := mock.ExpectationsWereMet(); err != nil {
  t.Errorf("There were unfulfilled expectations: %s", err)
}
```

You can refer to the actual `store_test.go` file for the complete code!
