# NoSQL Database Design — LAB SHEET 5
## Questions and Solutions

## 1. Install and configure Riak KV on Linux. Create a `students` bucket and perform CRUD operations.

Start Riak KV using Docker (if the image is available in your environment):

```bash
docker pull basho/riak-kv:2.2.3
docker run -d --name riak -p 8098:8098 -p 8087:8087 basho/riak-kv:2.2.3
curl http://localhost:8098/ping
```

**Insert:**
```bash
curl -X PUT http://localhost:8098/buckets/students/keys/101 \
  -H "Content-Type: application/json" \
  -d '{"id":101,"name":"Aman","age":21,"course":"BCA"}'
```

**Retrieve:**
```bash
curl http://localhost:8098/buckets/students/keys/101
```

**Update:**
```bash
curl -X PUT http://localhost:8098/buckets/students/keys/101 \
  -H "Content-Type: application/json" \
  -d '{"id":101,"name":"Aman","age":22,"course":"MCA"}'
```

**Delete:**
```bash
curl -X DELETE http://localhost:8098/buckets/students/keys/101
```

## 2. Create a `user_sessions` bucket and store session information.

```bash
curl -X PUT http://localhost:8098/buckets/user_sessions/keys/session101 \
  -H "Content-Type: application/json" \
  -d '{"session_id":"session101","user_id":"user101","login_time":"2026-10-15T09:00:00","status":"active"}'

curl http://localhost:8098/buckets/user_sessions/keys/session101
```

Update session status:
```bash
curl -X PUT http://localhost:8098/buckets/user_sessions/keys/session101 \
  -H "Content-Type: application/json" \
  -d '{"session_id":"session101","user_id":"user101","login_time":"2026-10-15T09:00:00","status":"logged_out"}'
```

## 3. Store user profiles and preferences.

```bash
curl -X PUT http://localhost:8098/buckets/user_profiles/keys/user101 \
  -H "Content-Type: application/json" \
  -d '{"name":"Rahul","email":"rahul@example.com","city":"Delhi","preferences":{"language":"English","theme":"dark","notifications":true}}'

curl http://localhost:8098/buckets/user_profiles/keys/user101
```

Update:
```bash
curl -X PUT http://localhost:8098/buckets/user_profiles/keys/user101 \
  -H "Content-Type: application/json" \
  -d '{"name":"Rahul Sharma","email":"rahul@example.com","city":"Roorkee","preferences":{"language":"Hindi","theme":"light","notifications":false}}'
```

## 4. Implement a shopping cart using Riak KV.

Store the cart as one JSON document per user:

```bash
curl -X PUT http://localhost:8098/buckets/shopping_carts/keys/user101 \
  -H "Content-Type: application/json" \
  -d '{"user_id":"user101","products":[{"product_id":"P101","quantity":2,"price":500},{"product_id":"P102","quantity":1,"price":300}],"total":1300}'

curl http://localhost:8098/buckets/shopping_carts/keys/user101
```

Update quantity:
```bash
curl -X PUT http://localhost:8098/buckets/shopping_carts/keys/user101 \
  -H "Content-Type: application/json" \
  -d '{"user_id":"user101","products":[{"product_id":"P101","quantity":3,"price":500},{"product_id":"P102","quantity":1,"price":300}],"total":1800}'
```

Remove product P102 by replacing the cart:
```bash
curl -X PUT http://localhost:8098/buckets/shopping_carts/keys/user101 \
  -H "Content-Type: application/json" \
  -d '{"user_id":"user101","products":[{"product_id":"P101","quantity":3,"price":500}],"total":1500}'
```

Delete the whole cart:
```bash
curl -X DELETE http://localhost:8098/buckets/shopping_carts/keys/user101
```

## 5. Store strings, JSON documents, and lists in Riak.

**String:**
```bash
curl -X PUT http://localhost:8098/buckets/data/keys/message \
  -H "Content-Type: text/plain" --data-binary 'Hello Riak'
curl http://localhost:8098/buckets/data/keys/message
```

**JSON document:**
```bash
curl -X PUT http://localhost:8098/buckets/data/keys/student \
  -H "Content-Type: application/json" \
  -d '{"id":1,"name":"Priya","age":20}'
curl http://localhost:8098/buckets/data/keys/student
```

**List represented as a JSON array:**
```bash
curl -X PUT http://localhost:8098/buckets/data/keys/subjects \
  -H "Content-Type: application/json" \
  -d '["Python","Java","DBMS"]'
curl http://localhost:8098/buckets/data/keys/subjects
```

- **Bucket:** groups related records.
- **Key:** identifies one record within a bucket.
- **Value:** the stored content, such as a string or JSON document.

## 6. Demonstrate consistency using read/write quorum values.

For a multi-node Riak cluster and a bucket configured with `n_val = 3`:

```bash
curl -X PUT 'http://localhost:8098/buckets/consistency/keys/key1?w=2&dw=2' \
  -H "Content-Type: application/json" \
  -d '{"message":"Consistent data"}'

curl 'http://localhost:8098/buckets/consistency/keys/key1?r=2'
curl 'http://localhost:8098/buckets/consistency/keys/key1?r=all'
```

`w=2` requests two write acknowledgements, `dw=2` requests two durable-write acknowledgements, and `r=2` requests two replica responses. Quorums balance consistency and availability but do not provide serializable multi-key transactions.

## 7. Demonstrate scaling in Riak KV.

Submit 1,000 records to each of four buckets:

```bash
for bucket in students products orders customers; do
  for i in $(seq 1 1000); do
    curl -sS -o /dev/null -X PUT \
      "http://localhost:8098/buckets/$bucket/keys/record$i" \
      -H "Content-Type: application/json" \
      -d "{\"id\":$i,\"bucket\":\"$bucket\"}"
  done
done
echo "Submitted 4000 records."
```

Riak distributes data across a partitioned consistent-hash ring and replicates data. Actual horizontal scaling requires a multi-node cluster; one Docker node only demonstrates CRUD.

## 8. Implement a transaction-like shopping scenario.

Save as `shopping_transaction.py`:

```python
import requests

BASE_URL = "http://localhost:8098/buckets/shopping_carts/keys"
HEADERS = {"Content-Type": "application/json"}


def get_cart(user_id):
    response = requests.get(f"{BASE_URL}/{user_id}")
    response.raise_for_status()
    return response.json()


def save_cart(user_id, cart):
    response = requests.put(
        f"{BASE_URL}/{user_id}", headers=HEADERS, json=cart
    )
    response.raise_for_status()


def add_item(user_id, product_id, price, quantity):
    cart = get_cart(user_id)
    for item in cart["products"]:
        if item["product_id"] == product_id:
            item["quantity"] += quantity
            break
    else:
        cart["products"].append({
            "product_id": product_id,
            "price": price,
            "quantity": quantity
        })

    cart["total"] = sum(
        item["price"] * item["quantity"] for item in cart["products"]
    )
    save_cart(user_id, cart)


if __name__ == "__main__":
    user_id = "user101"
    save_cart(user_id, {"user_id": user_id, "products": [], "total": 0})
    add_item(user_id, "P101", 500, 1)
    add_item(user_id, "P101", 500, 2)
    print(get_cart(user_id))
```

Install and run:
```bash
pip install requests
python shopping_transaction.py
```

Riak does not provide a multi-key ACID transaction here. Concurrent updates can overwrite one another; use conflict handling and conditional updates where appropriate.

## 9. Implement query-by-data for user records.

Save as `query_users.py`:

```python
import requests

BASE_URL = "http://localhost:8098/buckets/users/keys"
HEADERS = {"Content-Type": "application/json"}


def save_user(user_id, user):
    response = requests.put(
        f"{BASE_URL}/{user_id}", json=user, headers=HEADERS
    )
    response.raise_for_status()


def get_user(user_id):
    response = requests.get(f"{BASE_URL}/{user_id}")
    response.raise_for_status()
    return response.json()


users = {
    "U101": {"name": "Aman", "city": "Delhi", "age": 21,
             "preferences": ["Python", "Music"]},
    "U102": {"name": "Priya", "city": "Mumbai", "age": 23,
             "preferences": ["Java", "Reading"]},
    "U103": {"name": "Rahul", "city": "Delhi", "age": 25,
             "preferences": ["Python", "Sports"]}
}

for user_id, user in users.items():
    save_user(user_id, user)

records = {user_id: get_user(user_id) for user_id in users}

print("Users from Delhi:")
for user_id, user in records.items():
    if user["city"] == "Delhi":
        print(user_id, user)

print("\\nUsers older than 22:")
for user_id, user in records.items():
    if user["age"] > 22:
        print(user_id, user)

print("\\nUsers interested in Python:")
for user_id, user in records.items():
    if "Python" in user["preferences"]:
        print(user_id, user)
```

Riak KV does not provide general SQL-style querying over arbitrary JSON attributes by default; this example retrieves known keys and filters in application code.

## 10. Implement set operations using Riak KV.

Save as `set_operations.py`:

```python
import requests

URL = "http://localhost:8098/buckets/user_interests/keys/user101"
HEADERS = {"Content-Type": "application/json"}


def save_interests(interests):
    response = requests.put(
        URL, json=sorted(set(interests)), headers=HEADERS
    )
    response.raise_for_status()


def get_interests():
    response = requests.get(URL)
    response.raise_for_status()
    return set(response.json())


save_interests(["Python", "Music", "Sports"])

interests = get_interests()
interests.add("Reading")
save_interests(interests)
print("After adding:", get_interests())

interests = get_interests()
interests.discard("Music")
save_interests(interests)
print("After removing:", get_interests())

interests = get_interests()
print("Python exists:", "Python" in interests)

user1 = get_interests()
user2 = {"Python", "Gaming", "Reading"}
print("Common interests:", user1.intersection(user2))
```

Install and run:
```bash
pip install requests
python set_operations.py
```

## 11. Install Neo4j and create Student, Course, and Department nodes.

Start Neo4j:
```bash
docker run -d --name neo4j -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/password123 neo4j:5
```

Open `http://localhost:7474`, log in as `neo4j` with password `password123`, and run:

```cypher
CREATE (:Student {id: 1, name: 'Aman', age: 21});
CREATE (:Student {id: 2, name: 'Priya', age: 22});
CREATE (:Course {id: 101, name: 'Computer Science'});
CREATE (:Course {id: 102, name: 'Database Systems'});
CREATE (:Department {id: 201, name: 'Computer Science Department'});

MATCH (s:Student {id: 1}), (c:Course {id: 101})
CREATE (s)-[:ENROLLED_IN]->(c);

MATCH (s:Student {id: 2}), (c:Course {id: 102})
CREATE (s)-[:ENROLLED_IN]->(c);

MATCH (c:Course {id: 101}), (d:Department {id: 201})
CREATE (c)-[:BELONGS_TO]->(d);

MATCH (c:Course {id: 102}), (d:Department {id: 201})
CREATE (c)-[:BELONGS_TO]->(d);

MATCH (n)-[r]->(m)
RETURN n, r, m;
```

## 12. Create a social-network graph in Neo4j.

```cypher
CREATE
  (a:Person {name: 'Aman'}),
  (b:Person {name: 'Priya'}),
  (c:Person {name: 'Rahul'}),
  (d:Company {name: 'ABC Technologies'});

MATCH (a:Person {name: 'Aman'}), (b:Person {name: 'Priya'})
CREATE (a)-[:FRIENDS_WITH]->(b);

MATCH (c:Person {name: 'Rahul'}), (a:Person {name: 'Aman'})
CREATE (c)-[:FOLLOWS]->(a);

MATCH (a:Person {name: 'Aman'}), (d:Company {name: 'ABC Technologies'})
CREATE (a)-[:WORKS_AT]->(d);

// Find friends
MATCH (p:Person {name: 'Aman'})-[:FRIENDS_WITH]-(friend:Person)
RETURN DISTINCT friend.name AS Friend;

// Find followers
MATCH (follower:Person)-[:FOLLOWS]->(p:Person {name: 'Aman'})
RETURN follower.name AS Follower;
```

## 13. Design an e-commerce graph in Neo4j.

```cypher
CREATE
  (c:Customer {id: 1, name: 'Aman'}),
  (p1:Product {id: 101, name: 'Laptop', price: 50000}),
  (p2:Product {id: 102, name: 'Mouse', price: 500}),
  (cat:Category {name: 'Electronics'}),
  (o:Order {id: 1001, date: '2026-10-15'});

MATCH (c:Customer {id: 1}), (o:Order {id: 1001})
CREATE (c)-[:PLACED]->(o);

MATCH (o:Order {id: 1001}), (p:Product {id: 101})
CREATE (o)-[:CONTAINS {quantity: 1}]->(p);

MATCH (o:Order {id: 1001}), (p:Product {id: 102})
CREATE (o)-[:CONTAINS {quantity: 2}]->(p);

MATCH (p:Product {id: 101}), (cat:Category {name: 'Electronics'})
CREATE (p)-[:BELONGS_TO]->(cat);

MATCH (p:Product {id: 102}), (cat:Category {name: 'Electronics'})
CREATE (p)-[:BELONGS_TO]->(cat);

// Products purchased by customer
MATCH (c:Customer {name: 'Aman'})-[:PLACED]->(:Order)-[r:CONTAINS]->(p:Product)
RETURN p.name AS Product, p.price AS Price, r.quantity AS Quantity;

// Calculate order total
MATCH (:Customer {id: 1})-[:PLACED]->(:Order)-[r:CONTAINS]->(p:Product)
RETURN sum(p.price * r.quantity) AS Total;
```

## 14. Demonstrate transactions, consistency, and availability in Neo4j.

These Browser commands demonstrate CREATE, SET, MATCH, and DELETE as individual statements:

```cypher
CREATE
  (c:Customer {id: 10, name: 'Riya'}),
  (p:Product {id: 20, name: 'Keyboard', price: 1200});

MATCH (p:Product {id: 20})
SET p.price = 1500
RETURN p;

MATCH (c:Customer {id: 10}), (p:Product {id: 20})
CREATE (c)-[:INTERESTED_IN]->(p);

MATCH (c:Customer {id: 10})-[r:INTERESTED_IN]->(p:Product {id: 20})
DELETE r;

MATCH (c:Customer {id: 10})
DETACH DELETE c;

MATCH (p:Product {id: 20})
DELETE p;
```

For a single atomic transaction containing multiple operations, use the Python driver:

```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver(
    "bolt://localhost:7687",
    auth=("neo4j", "password123")
)


def create_order(tx):
    tx.run("""
        CREATE (c:Customer {id: 50, name: 'Neha'})
        CREATE (p:Product {id: 60, name: 'Phone', price: 20000})
        CREATE (o:Order {id: 70, total: 20000})
        CREATE (c)-[:PLACED]->(o)
        CREATE (o)-[:CONTAINS {quantity: 1}]->(p)
    """).consume()


try:
    with driver.session() as session:
        session.execute_write(create_order)
finally:
    driver.close()
```

Neo4j transactions commit or roll back as a unit. High availability requires a properly configured cluster; a single-node setup is not highly available.

## 15. Develop a Neo4j graph database application using Python.

Install:
```bash
pip install neo4j
```

Save as `neo4j_crud.py`:

```python
from neo4j import GraphDatabase

URI = "bolt://localhost:7687"
AUTH = ("neo4j", "password123")
driver = GraphDatabase.driver(URI, auth=AUTH)


def create_student(tx, student_id, name, age):
    tx.run("""
        MERGE (s:Student {id: $id})
        SET s.name = $name, s.age = $age
    """, id=student_id, name=name, age=age).consume()


def read_students(tx):
    result = tx.run("""
        MATCH (s:Student)
        RETURN s.id AS id, s.name AS name, s.age AS age
        ORDER BY s.id
    """)
    return [record.data() for record in result]


def update_student(tx, student_id, name):
    tx.run("""
        MATCH (s:Student {id: $id})
        SET s.name = $name
    """, id=student_id, name=name).consume()


def delete_student(tx, student_id):
    tx.run("""
        MATCH (s:Student {id: $id})
        DETACH DELETE s
    """, id=student_id).consume()


def enroll_student(tx, student_id, course_name):
    tx.run("""
        MATCH (s:Student {id: $id})
        MERGE (c:Course {name: $course})
        MERGE (s)-[:ENROLLED_IN]->(c)
    """, id=student_id, course=course_name).consume()


try:
    with driver.session() as session:
        session.execute_write(create_student, 1, "Aman", 21)
        session.execute_write(create_student, 2, "Priya", 22)
        session.execute_write(enroll_student, 1, "Python")

        print("Students:")
        for student in session.execute_read(read_students):
            print(student)

        session.execute_write(update_student, 1, "Aman Sharma")

        result = session.run("""
            MATCH (s:Student)-[:ENROLLED_IN]->(c:Course)
            RETURN s.name AS student, c.name AS course
        """)
        print("Enrollments:")
        for record in result:
            print(record.data())

        session.execute_write(delete_student, 2)
finally:
    driver.close()
```

Run:
```bash
python neo4j_crud.py
```

Example relationship queries:
```cypher
MATCH (s:Student)-[:ENROLLED_IN]->(c:Course {name: 'Python'})
RETURN s.name;

MATCH (s:Student {name: 'Aman Sharma'})-[:ENROLLED_IN]->(c:Course)
RETURN c.name;
```

Neo4j supports clustering and replication when configured appropriately. Graph databases are useful for social networks, recommendation systems, fraud detection, and knowledge graphs.
