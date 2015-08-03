# conman

Luminus database connection management and SQL query generation library

The library provides pooled connections using the [clj-dbcp](https://github.com/kumarshantanu/clj-dbcp) library.

The queries are generated using [Yesql](https://github.com/krisajenkins/yesql/tree/devel) and wrapped with
connection aware functions.

## Usage

Conman relies on a dynamic variable to manage the connection. The dynamic variable allows the connection to be
rebound contextually for transactions. When working with multiple databases, a separate namespace is required
for each database connection.

### Defining Queiries

SQL statements should be populated in files that are accessible on the resource path.

The file format is: `(<name tag> [docstring comments]
<the query>)*`, see the official [Yesql docs](https://github.com/krisajenkins/yesql/tree/devel) for further
exmaples. For example, we could create a file `resources/sql/queries.sql` with
the following content:

``` sql
-- name: create-user!
-- creates a new user record
INSERT INTO users
(id, first_name, last_name, email, pass)
VALUES (:id, :first_name, :last_name, :email, :pass)

-- name: get-user
-- retrieve a user given the id.
SELECT * FROM users
WHERE id = :id

-- name: get-all-users
-- retrieve all users.
SELECT * FROM users
```

The queries are bound to the connection using the `bind-connection` macro. The function
accepts the connection atom followed one or more strings represting SQL query files.

```clojure
(use 'conman.core)

(defonce ^:dynamic conn (atom nil))

(conman/bind-connection conn "sql/queries.sql")
```

When the `bind-connection` macro then `create-user!` and `get-user` function will be
generated in the namespace where the macro was run.

These functions can be called three way:

```clojure
;; when called with no argument then the Yesql generated function
;; will be called with an empty paramter map and the connection specified in `conn`
(get-all-users)

;; when a parameter map is passed as an argument the map and the connection specified in `conn`
;; will be passed to the Yesql generated function
(create-user! {:id "foo" :first_name "Bob" :last_name "Bobberton" :email nil :pass nil})

;; finally, a parameter map and an explicit connection can be
;; passed to the function, in this case the specified connection is used
(get-user {:id "foo"} some-other-conn)

```

Next, the `connect!` function should be called to initialize the database connection.
The function accepts an atom to store the connection followed a map with the database
specification.

```clojure
(def pool-spec
  {:adapter    :postgresql
   :jdbc-url   "jdbc:postgresql://localhost/myapp?user=user&password=pass"
   :init-size  1
   :min-idle   1
   :max-idle   4
   :max-active 32})

(connect! conn pool-spec)
```

The connection can be terminated by running the `disconnect!` funciton:

```clojure
(disconnect! conn)
```

When using a dynamic connection atom it's possible to use the `with-transaction`
macro to rebind it to the transaction connection. The SQL query functions
generated by the `bind-queries` macro will automatically use the transaction
connection in that case:

```clojure
(with-transaction [t-conn conn]
  (jdbc/db-set-rollback-only! t-conn)
  (create-user!
    {:id         "foo"
     :first_name "Sam"
     :last_name  "Smith"
     :email      "sam.smith@example.com"})
  (get-user {:id "foo"}))
```

## License

Copyright © 2015 Dmitri Sotnikov and Carousel Apps Ltd.

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
