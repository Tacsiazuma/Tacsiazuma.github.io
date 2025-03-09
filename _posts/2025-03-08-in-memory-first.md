---
layout: post
title:  "In-memory first"
author: kpapp
image: assets/images/in-memory.jpg
featured: true
hidden: false
categories:
    - Testing
    - Agile
    - Architecture
tags:
    - system design
    - database
    - in-memory first
---

> TLDR: Database-first design often leads to tightly coupling application logic with the database schema, making it hard to adapt as requirements evolve. An in-memory-first approach, where data is modeled in memory before choosing a database, offers more flexibility and allows for better scalability. By deferring database decisions, you can test with in-memory repositories and then implement the appropriate database (SQL, NoSQL, etc.) based on real-world usage patterns. This approach enhances adaptability, reduces long-term maintenance costs, and improves testing efficiency.

What I've noticed during my career is that people love databases. It fascinates us how some bright minds overcame all the quirks of the OS and hardware and made sure that our data will be safe and consistent. I'm also amazed by how simple concepts like LSM trees and SSTables power modern databases. 

Ask a developer about their favorite database, and you're bound to get a passionate response. Whether it's PostgreSQL, MongoDB, or SQL Server, many developers pick a database before even defining the problem they need to solve.

There are even favorite databases for some languages/frameworks:

- .NET folks love SQL Server
- PHP developers usually fall for MySQL

This love for databases shows up in system design interviews as well. The moment a problem is presented, candidates instinctively start modeling entities and relations—sometimes before even understanding the use cases.

Then they will start drawing boxes: 

```md
application -> database X
```

And just like that, a fundamental architectural decision has been made—often based on incomplete information and educated guesses rather than real-world usage patterns.

Don't get me wrong, it is not a problem at all. System design interviews are all like this and it is perfectly fine given the time constraint. The problem lies in when we reuse the same practice in a real world scenario.

> In the below examples I will assume a greenfield project.

### Database first design

We all know this practice, it is when we first design the database schema then we move towards the code. Essentially moving outwards from the database to the user interface. 
Database-first design can work when building a simple CRUD application with well-defined, unchanging requirements. But in real-world projects, requirements evolve, business priorities shift, and data access patterns emerge over time—often in ways we didn’t anticipate.

> I don't know about you but this was rarely the case for me.

What can possibly go wrong here? 

Let's say you have created the database structure upfront. Migrations, classes are there and then you start patching it together with the UI while implementing the different use cases.

With a database-first approach, it's natural to tightly couple application logic to the database schema. However, this can create problems when the schema needs to evolve. Since the application logic is closely tied to the database schema, even minor changes to the data model force widespread refactoring in the code.

It’s like designing the foundation of a house before deciding what kind of building you need.

But what happens when you realize your initial database design doesn’t fit your evolving requirements?

### In-memory first

Uncle Bob says: "The database is a detail". 

This approach will follow above principle along with the repository pattern.
The idea behind it is that everything stored in the database - and even their relations - can be eventually represented in objects/structs in the memory.

When we design in-memory first, we model data in the most natural way possible. Key-value pairs? Use maps. Relationships? Use object references. This mirrors what ORMs do behind the scenes. But by delaying the choice of database, we retain the ability to optimize for performance and scalability later—rather than being forced into costly refactors. MongoDB? It is just a set of objects each being an aggregate root.

Why does this work? Because databases themselves are software—they also store and manipulate data in memory before persisting it.

So how can we utilize this knowledge? 

What if we don't create the ERD diagram but focus on the use cases instead? Using an in-memory repository we will use only in-memory data structures to store our entities. With this approach we leave the decision of choosing a database to a later point when we see most of the access patterns and the structure of our data along with some of its properties.

Here are some real-world cases where this approach saved me—especially when business needs evolved in ways we couldn’t anticipate during the design phase:

- When we planned to use Amazon DynamoDB but figured out that the data stored in each value is bigger than its limit.
- When we also planned to use DynamoDB but we ran out of secondary indexes and having to scan tables are not an option.
- Instead of using some encrypted search mechanism in the database itself we moved that to MemCached where it can efficiently queried without the need of encryption and the limitations these algorithm have.


The in-memory repository can be used in unit-tests as a fake. That repository will be tested as well. When we figured out which database will fit our needs and the structure for it then we can create an implementation for it too and run the same (in this case "integration") test suite against it to make sure both works the same way.

> Important. By in-memory database I am not talking about H2, Entity Framework with in-memory engine or SQLite. If you start using them from the start then you possibly tie yourself to a relational approach which might be required but there is a chance it is not. Not to mention the false sense of security they gave although these might behave differently than your actual database.

### But how does it looks like in practice?

Let's assume you are creating this new app using sociable unit tests which means you don't just test a single class but a class and a graph of its dependencies and only use tests doubles at I/O operations, like the database. This test double will be our in-memory repository.

The example is pretty simple, lets imagine we have orders with descriptions. We are not complicating it with rich domain objects and DDD because we should focus. Simple domain object with 2 fields:

```java
// Order.java
public class Order {
    private Long id;
    private String description;

    // Constructors
    public Order() { }

    public Order(Long id, String description) {
        this.id = id;
        this.description = description;
    }

    // Getters and setters
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }
}
```

As we plan different implementations we can have an interface early on but this can be postponed to later as we can extract interfaces in modern IDEs pretty easily:

```java
// OrderRepository.java
import java.util.List;
import java.util.Optional;

public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(Long id);
    List<Order> findAll();
    void deleteById(Long id);
}
```

Let's imagine that we are developing a simple use case, which validates, saves the order, notifies the user and a third party system. 

We create a test for this use case and eventually we will reach a point where we need our repository. 
We will use the in-memory implementation (for our tests and initially for production code as well) and test it indirectly through the use case first.

```java
// InMemoryOrderRepository.java
import org.springframework.stereotype.Repository;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

@Repository
public class InMemoryOrderRepository implements OrderRepository {

    private final Map<Long, Order> orderStorage = new ConcurrentHashMap<>();
    private final AtomicLong idGenerator = new AtomicLong();

    @Override
    public Order save(Order order) {
        if (order.getId() == null) {
            order.setId(idGenerator.incrementAndGet());
        }
        orderStorage.put(order.getId(), order);
        return order;
    }

    @Override
    public Optional<Order> findById(Long id) {
        return Optional.ofNullable(orderStorage.get(id));
    }

    @Override
    public List<Order> findAll() {
        return new ArrayList<>(orderStorage.values());
    }

    @Override
    public void deleteById(Long id) {
        orderStorage.remove(id);
    }
}

```
Later when we think that e.g. SQL is the way we can create a different implementation of the same interface using `JdbcTemplate` here.

```java
// JdbcOrderRepository.java
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.support.GeneratedKeyHolder;
import org.springframework.jdbc.support.KeyHolder;
import org.springframework.stereotype.Repository;

import java.sql.PreparedStatement;
import java.sql.Statement;
import java.util.List;
import java.util.Optional;

@Repository
public class JdbcOrderRepository implements OrderRepository {

    private final JdbcTemplate jdbcTemplate;

    public JdbcOrderRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public Order save(Order order) {
        if (order.getId() == null) {
            // Insert new order and retrieve generated key
            KeyHolder keyHolder = new GeneratedKeyHolder();
            jdbcTemplate.update(connection -> {
                PreparedStatement ps = connection.prepareStatement(
                        "INSERT INTO orders (description) VALUES (?)",
                        Statement.RETURN_GENERATED_KEYS
                );
                ps.setString(1, order.getDescription());
                return ps;
            }, keyHolder);
            order.setId(keyHolder.getKey().longValue());
        } else {
            // Update existing order
            jdbcTemplate.update("UPDATE orders SET description = ? WHERE id = ?",
                    order.getDescription(), order.getId());
        }
        return order;
    }

    @Override
    public Optional<Order> findById(Long id) {
        List<Order> orders = jdbcTemplate.query(
            "SELECT id, description FROM orders WHERE id = ?",
            new Object[]{id},
            (rs, rowNum) -> new Order(rs.getLong("id"), rs.getString("description"))
        );
        return orders.stream().findFirst();
    }

    @Override
    public List<Order> findAll() {
        return jdbcTemplate.query(
            "SELECT id, description FROM orders",
            (rs, rowNum) -> new Order(rs.getLong("id"), rs.getString("description"))
        );
    }

    @Override
    public void deleteById(Long id) {
        jdbcTemplate.update("DELETE FROM orders WHERE id = ?", id);
    }
}

```
But how do we make sure that the SQL implementation works the same way?
We will have a shared test suite against the two implementations:

```java
import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Optional;

public abstract class AbstractOrderRepositoryTest {

    protected OrderRepository repository;

    // Concrete tests must provide a repository instance.
    protected abstract OrderRepository createRepository();

    @BeforeEach
    public void setUp() {
        repository = createRepository();
    }

    @Test
    public void findByIdShouldReturnSavedElement() {
        Order order = new Order(null, "Test Order");
        Order savedOrder = repository.save(order);
        assertNotNull(savedOrder.getId(), "Saved order should have an ID");
        
        Optional<Order> found = repository.findById(savedOrder.getId());
        assertTrue(found.isPresent(), "Order should be found by ID");
        assertEquals("Test Order", found.get().getDescription(), "Order description should match");
    }

    @Test
    public void findAllShouldReturnAllElements() {
        repository.save(new Order(null, "Order 1"));
        repository.save(new Order(null, "Order 2"));
        List<Order> orders = repository.findAll();
        assertEquals(2, orders.size(), "Should return 2 orders");
    }

    @Test
    public void updateShouldUpdateDescription() {
        Order order = repository.save(new Order(null, "Initial"));
        order.setDescription("Updated");
        repository.save(order);
        
        Optional<Order> updated = repository.findById(order.getId());
        assertTrue(updated.isPresent(), "Updated order should be found");
        assertEquals("Updated", updated.get().getDescription(), "Description should be updated");
    }

    @Test
    public void deleteShouldRemoveFromRepository() {
        Order order = repository.save(new Order(null, "Delete me"));
        repository.deleteById(order.getId());
        
        Optional<Order> deleted = repository.findById(order.getId());
        assertFalse(deleted.isPresent(), "Deleted order should not be found");
    }
}
```
This can be run against both repositories individually to prove that both works the same way:

```java
public class InMemoryOrderRepositoryTest extends AbstractOrderRepositoryTest {

    @Override
    protected OrderRepository createRepository() {
        return new InMemoryOrderRepository();
    }
}
```

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.datasource.DriverManagerDataSource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
public class JdbcOrderRepositoryTest  extends AbstractOrderRepositoryTest {

    private JdbcTemplate jdbcTemplate;

    @Container
    public static PostgreSQLContainer<?> postgresContainer = new PostgreSQLContainer<>("postgres:latest")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @BeforeEach
    public void setupDatabase() {
        // Create a DataSource based on the container's connection details.
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName(postgresContainer.getDriverClassName());
        dataSource.setUrl(postgresContainer.getJdbcUrl());
        dataSource.setUsername(postgresContainer.getUsername());
        dataSource.setPassword(postgresContainer.getPassword());
        jdbcTemplate = new JdbcTemplate(dataSource);

        // Setup the schema for the test (drop if exists, then create table).
        jdbcTemplate.execute("DROP TABLE IF EXISTS orders");
        jdbcTemplate.execute("CREATE TABLE orders (id SERIAL PRIMARY KEY, description VARCHAR(255) NOT NULL)");

        repository = new JdbcOrderRepository(jdbcTemplate);
    }

    @Override
    protected OrderRepository createRepository() {
        return new JdbcOrderRepository(jdbcTemplate);
    }
}
```
If this test suite is green then we can be sure that both implementation works the same way and we are safe to replace the in-memory one with the jdbc one in production, but our unit tests for the different use cases still use the in-memory one because it is fast.

But from a design standpoint, what’s even more important is the insight we gain about our data model. We see that these orders can be stored as key/value pairs in memory. Depending on our consistency and scalability requirements, SQL might not be the best choice, and we could move towards key-value solutions instead. However, if we need to list all orders (as seen in `findAll`), SQL might still be the better fit. 

### Conclusion

Designing systems with a database-first approach can work well for CRUD applications or when requirements are well-defined upfront. However, in many real-world scenarios, business needs evolve, and rigidly coupling your application logic to a database schema can create long-term maintenance challenges.

By adopting an in-memory-first approach, we defer database decisions until we better understand access patterns, scalability needs, and the structure of our data. This flexibility allows us to iterate quickly, adapt to changing requirements, and choose the most suitable storage solution—whether it's a relational database, a key-value store, or something else entirely.

Additionally, this approach enhances testability. Using an in-memory repository in unit tests ensures fast and reliable testing while maintaining a shared test suite that guarantees consistency across different database implementations.

Ultimately, treating the database as a detail rather than a starting point helps better system design, improved maintainability, and greater adaptability to business needs. Instead of letting the database dictate how our software is structured, we let the software define how data should be stored.

The next time you start a greenfield project, try focusing on the use cases first. Design your data structures in-memory before committing to a database choice. You might be surprised at how much flexibility and maintainability you gain.
