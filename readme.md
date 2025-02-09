MysqliDb -- Simple MySQLi wrapper and object mapper with prepared statements
<hr>
### Table of Contents
**[Initialization](#initialization)**  
**[Objects mapping](#objects-mapping)**  
**[Insert Query](#insert-query)**  
**[Update Query](#update-query)**  
**[Select Query](#select-query)**  
**[Delete Query](#delete-query)**  
**[Running raw SQL queries](#running-raw-sql-queries)**  
**[Query Keywords](#query-keywords)**  
**[Raw Query](#raw-query-method)**  
**[Where Conditions](#where-method)**  
**[Order Conditions](#ordering-method)**  
**[Group Conditions](#grouping-method)**  
**[Properties Sharing](#properties-sharing)**  
**[Joining Tables](#join-method)**  
**[Subqueries](#subqueries)**  
**[EXISTS / NOT EXISTS condition](#exists--not-exists-condition)**  
**[Has method](#has-method)**  
**[Helper Functions](#helper-commands)**  
**[Transaction Helpers](#transaction-helpers)**  

### Installation
To utilize this class, first import MysqliDb.php into your project, and require it.

```php
require_once ('MysqliDb.php');
```

### Installation with composer
It is also possible to install library via composer
```
composer require joshcam/mysqli-database-class:dev-master
```

### Initialization
Simple initialization with utf8 charset set by default:
```php
$DBLIB = new MysqliDb ('host', 'username', 'password', 'databaseName');
```

Advanced initialization:
```php
$DBLIB = new MysqliDb (Array (
                'host' => 'host',
                'username' => 'username', 
                'password' => 'password',
                'db'=> 'databaseName',
                'port' => 3306,
                'prefix' => 'my_',
                'charset' => 'utf8'));
```
table prefix, port and database charset params are optional.
If no charset should be set charset, set it to null

Also it is possible to reuse already connected mysqli object:
```php
$mysqli = new mysqli ('host', 'username', 'password', 'databaseName');
$DBLIB = new MysqliDb ($mysqli);
```

If no table prefix were set during object creation its possible to set it later with a separate call:
```php
$DBLIB->setPrefix ('my_');
```

If you need to get already created mysqliDb object from another class or function use
```php
    function init () {
        // db staying private here
        $DBLIB = new MysqliDb ('host', 'username', 'password', 'databaseName');
    }
    ...
    function myfunc () {
        // obtain db object created in init  ()
        $DBLIB = MysqliDb::getInstance();
        ...
    }
```

### Objects mapping
dbObject.php is an object mapping library built on top of mysqliDb to provide model representation functionality.
See <a href='dbObject.md'>dbObject manual for more information</a>

### Insert Query
Simple example
```php
$data = Array ("login" => "admin",
               "firstName" => "John",
               "lastName" => 'Doe'
);
$id = $DBLIB->insert ('users', $data);
if($id)
    echo 'user was created. Id=' . $id;
```

Insert with functions use
```php
$data = Array (
	'login' => 'admin',
    'active' => true,
	'firstName' => 'John',
	'lastName' => 'Doe',
	'password' => $DBLIB->func('SHA1(?)',Array ("secretpassword+salt")),
	// password = SHA1('secretpassword+salt')
	'createdAt' => $DBLIB->now(),
	// createdAt = NOW()
	'expires' => $DBLIB->now('+1Y')
	// expires = NOW() + interval 1 year
	// Supported intervals [s]econd, [m]inute, [h]hour, [d]day, [M]onth, [Y]ear
);

$id = $DBLIB->insert ('users', $data);
if ($id)
    echo 'user was created. Id=' . $id;
else
    echo 'insert failed: ' . $DBLIB->getLastError();
```

Insert with on duplicate key update
```php
$data = Array ("login" => "admin",
               "firstName" => "John",
               "lastName" => 'Doe',
               "createdAt" => $DBLIB->now(),
               "updatedAt" => $DBLIB->now(),
);
$updateColumns = Array ("updateAt");
$lastInsertId = "id";
$DBLIB->onDuplicate($updateColumns, $lastInsertId);
$id = $DBLIB->insert ('users', $data);
```

### Replace Query
<a href='https://dev.mysql.com/doc/refman/5.0/en/replace.html'>Replace()</a> method implements same API as insert();

### Update Query
```php
$data = Array (
	'firstName' => 'Bobby',
	'lastName' => 'Tables',
	'editCount' => $DBLIB->inc(2),
	// editCount = editCount + 2;
	'active' => $DBLIB->not()
	// active = !active;
);
$DBLIB->where ('id', 1);
if ($DBLIB->update ('users', $data))
    echo $DBLIB->count . ' records were updated';
else
    echo 'update failed: ' . $DBLIB->getLastError();
```

### Select Query
After any select/get function calls amount or returned rows is stored in $count variable
```php
$users = $DBLIB->get('users'); //contains an Array of all users 
$users = $DBLIB->get('users', 10); //contains an Array 10 users
```

or select with custom columns set. Functions also could be used

```php
$cols = Array ("id", "name", "email");
$users = $DBLIB->get ("users", null, $cols);
if ($DBLIB->count > 0)
    foreach ($users as $user) { 
        print_r ($user);
    }
```

or select just one row

```php
$DBLIB->where ("id", 1);
$user = $DBLIB->getOne ("users");
echo $user['id'];

$stats = $DBLIB->getOne ("users", "sum(id), count(*) as cnt");
echo "total ".$stats['cnt']. "users found";
```

or select one column value or function result

```php
$count = $DBLIB->getValue ("users", "count(*)");
echo "{$count} users found";
```

select one column value or function result from multiple rows:
``php
$logins = $DBLIB->getValue ("users", "login", null);
// select login from users
$logins = $DBLIB->getValue ("users", "login", 5);
// select login from users limit 5
foreach ($logins as $login)
    echo $login;
```


### Defining a return type
MysqliDb can return result in 3 different formats: Array of Array, Array of Objects and a Json string. To select a return type use ArrayBuilder(), ObjectBuilder() and JsonBuilder() methods. Note that ArrayBuilder() is a default return type
```php
// Array return type
$= $DBLIB->getOne("users");
echo $u['login'];
// Object return type
$u = $DBLIB->ObjectBuilder()->getOne("users");
echo $u->login;
// Json return type
$json = $DBLIB->JsonBuilder()->getOne("users");
```

### Running raw SQL queries
```php
$users = $DBLIB->rawQuery('SELECT * from users where id >= ?', Array (10));
foreach ($users as $user) {
    print_r ($user);
}
```
To avoid long if checks there are couple helper functions to work with raw query select results:

Get 1 row of results:
```php
$user = $DBLIB->rawQueryOne ('select * from users where id=?', Array(10));
echo $user['login'];
// Object return type
$user = $DBLIB->ObjectBuilder()->rawQueryOne ('select * from users where id=?', Array(10));
echo $user->login;
```
Get 1 column value as a string:
```php
$password = $DBLIB->rawQueryValue ('select password from users where id=? limit 1', Array(10));
echo "Password is {$password}";
NOTE: for a rawQueryValue() to return string instead of an array 'limit 1' should be added to the end of the query.
```
Get 1 column value from multiple rows:
```php
$logins = $DBLIB->rawQueryValue ('select login from users limit 10');
foreach ($logins as $login)
    echo $login;
```

More advanced examples:
```php
$params = Array(1, 'admin');
$users = $DBLIB->rawQuery("SELECT id, firstName, lastName FROM users WHERE id = ? AND login = ?", $params);
print_r($users); // contains Array of returned rows

// will handle any SQL query
$params = Array(10, 1, 10, 11, 2, 10);
$q = "(
    SELECT a FROM t1
        WHERE a = ? AND B = ?
        ORDER BY a LIMIT ?
) UNION (
    SELECT a FROM t2 
        WHERE a = ? AND B = ?
        ORDER BY a LIMIT ?
)";
$resutls = $DBLIB->rawQuery ($q, $params);
print_r ($results); // contains Array of returned rows
```

### Where Method
This method allows you to specify where parameters of the query.

WARNING: In order to use column to column comparisons only raw where conditions should be used as column name or functions cant be passed as a bind variable.

Regular == operator with variables:
```php
$DBLIB->where ('id', 1);
$DBLIB->where ('login', 'admin');
$results = $DBLIB->get ('users');
// Gives: SELECT * FROM users WHERE id=1 AND login='admin';
```

Regular == operator with column to column comparison:
```php
// WRONG
$DBLIB->where ('lastLogin', 'createdAt');
// CORRECT
$DBLIB->where ('lastLogin = createdAt');
$results = $DBLIB->get ('users');
// Gives: SELECT * FROM users WHERE lastLogin = createdAt;
```

```php
$DBLIB->where ('id', 50, ">=");
// or $DBLIB->where ('id', Array ('>=' => 50));
$results = $DBLIB->get ('users');
// Gives: SELECT * FROM users WHERE id >= 50;
```

BETWEEN / NOT BETWEEN:
```php
$DBLIB->where('id', Array (4, 20), 'BETWEEN');
// or $DBLIB->where ('id', Array ('BETWEEN' => Array(4, 20)));

$results = $DBLIB->get('users');
// Gives: SELECT * FROM users WHERE id BETWEEN 4 AND 20
```

IN / NOT IN:
```php
$DBLIB->where('id', Array(1, 5, 27, -1, 'd'), 'IN');
// or $DBLIB->where('id', Array( 'IN' => Array(1, 5, 27, -1, 'd') ) );

$results = $DBLIB->get('users');
// Gives: SELECT * FROM users WHERE id IN (1, 5, 27, -1, 'd');
```

OR CASE
```php
$DBLIB->where ('firstName', 'John');
$DBLIB->orWhere ('firstName', 'Peter');
$results = $DBLIB->get ('users');
// Gives: SELECT * FROM users WHERE firstName='John' OR firstName='peter'
```

NULL comparison:
```php
$DBLIB->where ("lastName", NULL, '<=>');
$results = $DBLIB->get("users");
// Gives: SELECT * FROM users where lastName <=> NULL
```

Also you can use raw where conditions:
```php
$DBLIB->where ("id != companyId");
$DBLIB->where ("DATE(createdAt) = DATE(lastLogin)");
$results = $DBLIB->get("users");
```

Or raw condition with variables:
```php
$DBLIB->where ("(id = ? or id = ?)", Array(6,2));
$DBLIB->where ("login","mike")
$res = $DBLIB->get ("users");
// Gives: SELECT * FROM users WHERE (id = 6 or id = 2) and login='mike';
```


Find the total number of rows matched. Simple pagination example:
```php
$offset = 10;
$count = 15;
$users = $DBLIB->withTotalCount()->get('users', Array ($offset, $count));
echo "Showing {$count} from {$DBLIB->totalCount}";
```

### Query Keywords
To add LOW PRIORITY | DELAYED | HIGH PRIORITY | IGNORE and the rest of the mysql keywords to INSERT (), REPLACE (), GET (), UPDATE (), DELETE() method or FOR UPDATE | LOCK IN SHARE MODE into SELECT ():
```php
$DBLIB->setQueryOption ('LOW_PRIORITY')->insert ($table, $param);
// GIVES: INSERT LOW_PRIORITY INTO table ...
```
```php
$DBLIB->setQueryOption ('FOR UPDATE')->get ('users');
// GIVES: SELECT * FROM USERS FOR UPDATE;
```

Also you can use an array of keywords:
```php
$DBLIB->setQueryOption (Array('LOW_PRIORITY', 'IGNORE'))->insert ($table,$param);
// GIVES: INSERT LOW_PRIORITY IGNORE INTO table ...
```

Same way keywords could be used in SELECT queries as well:
```php
$DBLIB->setQueryOption ('SQL_NO_CACHE');
$DBLIB->get("users");
// GIVES: SELECT SQL_NO_CACHE * FROM USERS;
```

Optionally you can use method chaining to call where multiple times without referencing your object over an over:

```php
$results = $DBLIB
	->where('id', 1)
	->where('login', 'admin')
	->get('users');
```

### Delete Query
```php
$DBLIB->where('id', 1);
if($DBLIB->delete('users')) echo 'successfully deleted';
```


### Ordering method
```php
$DBLIB->orderBy("id","asc");
$DBLIB->orderBy("login","Desc");
$DBLIB->orderBy("RAND ()");
$results = $DBLIB->get('users');
// Gives: SELECT * FROM users ORDER BY id ASC,login DESC, RAND ();
```

Order by values example:
```php
$DBLIB->orderBy('userGroup', 'ASC', array('superuser', 'admin', 'users'));
$DBLIB->get('users');
// Gives: SELECT * FROM users ORDER BY FIELD (userGroup, 'superuser', 'admin', 'users') ASC;
```

If you are using setPrefix () functionality and need to use table names in orderBy() method make sure that table names are escaped with ``.

```php
$DBLIB->setPrefix ("t_");
$DBLIB->orderBy ("users.id","asc");
$results = $DBLIB->get ('users');
// WRONG: That will give: SELECT * FROM t_users ORDER BY users.id ASC;

$DBLIB->setPrefix ("t_");
$DBLIB->orderBy ("`users`.id", "asc");
$results = $DBLIB->get ('users');
// CORRECT: That will give: SELECT * FROM t_users ORDER BY t_users.id ASC;
```

### Grouping method
```php
$DBLIB->groupBy ("name");
$results = $DBLIB->get ('users');
// Gives: SELECT * FROM users GROUP BY name;
```

Join table products with table users with LEFT JOIN by tenantID
### JOIN method
```php
$DBLIB->join("users u", "p.tenantID=u.tenantID", "LEFT");
$DBLIB->where("u.id", 6);
$products = $DBLIB->get ("products p", null, "u.name, p.productName");
print_r ($products);
```

### Properties sharing
Its is also possible to copy properties

```php
$DBLIB->where ("agentId", 10);
$DBLIB->where ("active", true);

$customers = $DBLIB->copy ();
$res = $customers->get ("customers", Array (10, 10));
// SELECT * FROM customers where agentId = 10 and active = 1 limit 10, 10

$cnt = $DBLIB->getValue ("customers", "count(id)");
echo "total records found: " . $cnt;
// SELECT count(id) FROM users where agentId = 10 and active = 1
```

### Subqueries
Subquery init

Subquery init without an alias to use in inserts/updates/where Eg. (select * from users)
```php
$sq = $DBLIB->subQuery();
$sq->get ("users");
```
 
A subquery with an alias specified to use in JOINs . Eg. (select * from users) sq
```php
$sq = $DBLIB->subQuery("sq");
$sq->get ("users");
```

Subquery in selects:
```php
$ids = $DBLIB->subQuery ();
$ids->where ("qty", 2, ">");
$ids->get ("products", null, "userId");

$DBLIB->where ("id", $ids, 'in');
$res = $DBLIB->get ("users");
// Gives SELECT * FROM users WHERE id IN (SELECT userId FROM products WHERE qty > 2)
```

Subquery in inserts:
```php
$userIdQ = $DBLIB->subQuery ();
$userIdQ->where ("id", 6);
$userIdQ->getOne ("users", "name"),

$data = Array (
    "productName" => "test product",
    "userId" => $userIdQ,
    "lastUpdated" => $DBLIB->now()
);
$id = $DBLIB->insert ("products", $data);
// Gives INSERT INTO PRODUCTS (productName, userId, lastUpdated) values ("test product", (SELECT name FROM users WHERE id = 6), NOW());
```

Subquery in joins:
```php
$usersQ = $DBLIB->subQuery ("u");
$usersQ->where ("active", 1);
$usersQ->get ("users");

$DBLIB->join($usersQ, "p.userId=u.id", "LEFT");
$products = $DBLIB->get ("products p", null, "u.login, p.productName");
print_r ($products);
// SELECT u.login, p.productName FROM products p LEFT JOIN (SELECT * FROM t_users WHERE active = 1) u on p.userId=u.id;
```

###EXISTS / NOT EXISTS condition
```php
$sub = $DBLIB->subQuery();
    $sub->where("company", 'testCompany');
    $sub->get ("users", null, 'userId');
$DBLIB->where (null, $sub, 'exists');
$products = $DBLIB->get ("products");
// Gives SELECT * FROM products WHERE EXISTS (select userId from users where company='testCompany')
```

### Has method
A convenient function that returns TRUE if exists at least an element that satisfy the where condition specified calling the "where" method before this one.
```php
$DBLIB->where("user", $user);
$DBLIB->where("password", md5($password));
if($DBLIB->has("users")) {
    return "You are logged";
} else {
    return "Wrong user/password";
}
``` 
### Helper commands
Reconnect in case mysql connection died:
```php
if (!$DBLIB->ping())
    $DBLIB->connect()
```

Get last executed SQL query:
Please note that function returns SQL query only for debugging purposes as its execution most likely will fail due missing quotes around char variables.
```php
    $DBLIB->get('users');
    echo "Last executed query was ". $DBLIB->getLastQuery();
```

Check if table exists:
```php
    if ($DBLIB->tableExists ('users'))
        echo "hooray";
```
### Transaction helpers
Please keep in mind that transactions are working on innoDB tables.
Rollback transaction if insert fails:
```php
$DBLIB->startTransaction();
...
if (!$DBLIB->insert ('myTable', $insertData)) {
    //Error while saving, cancel new record
    $DBLIB->rollback();
} else {
    //OK
    $DBLIB->commit();
}
```

### Query exectution time benchmarking
To track query execution time setTrace() function should be called.
```php
$DBLIB->setTrace (true);
// As a second parameter it is possible to define prefix of the path which should be striped from filename
// $DBLIB->setTrace (true, $_SERVER['SERVER_ROOT']);
$DBLIB->get("users");
$DBLIB->get("test");
print_r ($DBLIB->trace);
```

```
    [0] => Array
        (
            [0] => SELECT  * FROM t_users ORDER BY `id` ASC
            [1] => 0.0010669231414795
            [2] => MysqliDb->get() >>  file "/avb/work/PHP-MySQLi-Database-Class/tests.php" line #151
        )

    [1] => Array
        (
            [0] => SELECT  * FROM t_test
            [1] => 0.00069189071655273
            [2] => MysqliDb->get() >>  file "/avb/work/PHP-MySQLi-Database-Class/tests.php" line #152
        )

```
