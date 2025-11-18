---
title: "Build REST APIs Faster with YAML-Driven Code Generation in Spring Boot"
date: 2025-11-18
tags: ["spring-boot", "code-generation", "postgresql", "rest-api", "yaml", "java", "liquibase", "jpa"]
authors: ["Hao"]
---

# Build REST APIs Faster with YAML-Driven Code Generation in Spring Boot

What if you could define your data models in simple YAML files and automatically generate complete, production-ready CRUD APIs with validation, pagination, and database migrations? I built a code generation template for Spring Boot that does exactly that.

## The Problem: Repetitive Boilerplate Code

If you've built REST APIs with Spring Boot, you know the drill. For every new entity, you write:

- A JPA entity class with annotations
- A repository interface extending JpaRepository
- A service class with CRUD operations
- A REST controller with endpoints
- Liquibase or Flyway migration scripts
- DTOs and validation logic

This process is tedious, error-prone, and time-consuming. What if we could **define our models once and generate all of this automatically**?

## The Solution: YAML-Driven Code Generation

I created a Spring Boot template that generates complete CRUD functionality from YAML model definitions. Here's how simple it is:

### Step 1: Define Your Model in YAML

```yaml
# src/main/resources/models/Product.yaml
name: Product
tableName: products
description: Product catalog entity

fields:
  - name: title
    type: string
    required: true
    indexed: true
    length: 255

  - name: price
    type: decimal
    precision: 10
    scale: 2
    required: true
    validation:
      - type: min
        value: 0

  - name: sku
    type: string
    required: true
    unique: true
    length: 50

  - name: active
    type: boolean
    defaultValue: true
```

### Step 2: Generate Code

```bash
mvn generate-sources
```

That's it! The system automatically generates:

- **JPA Entity** - Complete with UUID primary key, validation annotations, and audit fields
- **Repository Interface** - Extends JpaRepository with custom query methods
- **Service Class** - All CRUD operations with proper transaction management
- **REST Controller** - RESTful endpoints with OpenAPI documentation
- **Liquibase Changelog** - Database migration script

### Step 3: Use Your API

The generated REST API is immediately available with full CRUD support:

```bash
# Create a product
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{"title":"Laptop","price":999.99,"sku":"LAP-001"}'

# Get all products with pagination
curl "http://localhost:8080/api/products?page=0&size=20&sortBy=title"

# Get by ID
curl http://localhost:8080/api/products/{uuid}

# Update
curl -X PUT http://localhost:8080/api/products/{uuid} \
  -d '{"title":"Updated Laptop","price":899.99}'

# Delete
curl -X DELETE http://localhost:8080/api/products/{uuid}
```

**Built-in API Documentation:** Visit `http://localhost:8080/swagger-ui.html` to access auto-generated OpenAPI documentation with an interactive API explorer.

## Key Features

### UUID Primary Keys

All entities use UUIDs instead of auto-increment IDs for better scalability, distributed systems support, and security (no predictable IDs).

### Built-in Validation

Define validation rules in YAML and get Jakarta Bean Validation annotations automatically:

```yaml
validation:
  - type: notBlank
  - type: size
    min: 3
    max: 100
  - type: email
  - type: min
    value: 0
```

### Relationship Mapping

Define relationships in YAML and get proper JPA annotations:

```yaml
relationships:
  - name: customer
    type: manyToOne
    targetEntity: User
    joinColumn: customer_id
  - name: items
    type: oneToMany
    targetEntity: OrderItem
    mappedBy: order
    cascade: [PERSIST, MERGE, REMOVE]
```

### Pagination & Sorting

All list endpoints support pagination and sorting out of the box using Spring Data's `Pageable` interface.

### Automatic Audit Fields

Every entity gets `createdAt` and `updatedAt` timestamps automatically (can be disabled with `auditable: false`).

## Rich Type System

The template supports a comprehensive set of data types that map to both Java and PostgreSQL:

| YAML Type  | Java Type     | PostgreSQL         |
|-----------|---------------|-------------------|
| string    | String        | VARCHAR(n)        |
| text      | String        | TEXT              |
| decimal   | BigDecimal    | DECIMAL(p,s)      |
| datetime  | LocalDateTime | TIMESTAMP         |
| uuid      | UUID          | UUID              |
| json      | String        | JSONB             |
| enum      | Enum          | VARCHAR(50)       |

## Real-World Example: E-Commerce Models

Let's look at a practical example with relationships. Here's how you'd define an Order entity with a relationship to a Customer:

```yaml
# Order.yaml
name: Order
tableName: orders
description: Customer orders

fields:
  - name: orderNumber
    type: string
    required: true
    unique: true
    length: 50

  - name: totalAmount
    type: decimal
    precision: 12
    scale: 2
    required: true
    validation:
      - type: min
        value: 0

  - name: status
    type: enum
    enumValues: [PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED]
    required: true
    defaultValue: PENDING
    indexed: true

relationships:
  - name: customer
    type: manyToOne
    targetEntity: User
    joinColumn: customer_id
    nullable: false
    fetch: LAZY

  - name: items
    type: oneToMany
    targetEntity: OrderItem
    mappedBy: order
    cascade: [PERSIST, MERGE, REMOVE]
```

This generates a complete, production-ready implementation with:

- Proper foreign key constraints
- Lazy loading for performance
- Cascade operations for order items
- Enum validation
- Decimal precision for money

## How It Works: Architecture

The code generation system is built on three key components:

### 1. Schema Parser

Uses Jackson YAML to parse model definitions into Java objects (`ModelDefinition`, `FieldDefinition`, etc.).

### 2. Template Engine

FreeMarker templates transform model definitions into Java code. Templates are stored in `src/main/resources/templates/codegen/` and can be customized.

### 3. Maven Integration

The generator runs during the `generate-sources` phase, before compilation. Generated code goes to `target/generated-sources/models/` (gitignored).

**Important:** Generated code should NOT be edited manually. Any changes will be overwritten on the next generation. Instead, extend or compose the generated classes.

## Project Structure

```
springboot_example/
├── src/main/
│   ├── java/com/haoxu/springbootexample/
│   │   ├── codegen/              # Code generation engine
│   │   │   ├── ModelGenerator.java
│   │   │   └── schema/           # YAML schema classes
│   │   └── ... (your app code)
│   └── resources/
│       ├── models/               # 📝 YAML definitions (COMMIT)
│       │   ├── User.yaml
│       │   └── Product.yaml
│       └── templates/codegen/    # FreeMarker templates (COMMIT)
│           ├── entity.ftl
│           ├── repository.ftl
│           ├── service.ftl
│           └── controller.ftl
└── target/
    └── generated-sources/models/ # ⚡ Generated code (GITIGNORED)
```

## Best Practices

1. **Version Control:** Commit YAML model files and templates, but NOT generated code. Add `target/generated-sources/` to `.gitignore`.

2. **Naming Conventions:** Use PascalCase for entity names, camelCase for fields, and snake_case for table/column names.

3. **Indexes:** Add indexes to frequently queried fields (foreign keys, search fields, unique fields).

4. **Validation:** Define validation rules in YAML for consistent data integrity across layers.

5. **Regeneration:** Always run `mvn generate-sources` after changing YAML files.

6. **Extension:** For custom logic, create wrapper services or additional controllers rather than modifying generated code.

## Technology Stack

- **Spring Boot 3.5.4** - Application framework
- **PostgreSQL** - Relational database
- **Liquibase** - Database migration management
- **FreeMarker 2.3.32** - Template engine for code generation
- **Jackson YAML** - YAML parsing
- **SpringDoc OpenAPI** - Automatic API documentation
- **Jakarta Validation** - Bean validation framework

## Benefits of This Approach

### Speed

Define a model in 20 lines of YAML instead of writing 200+ lines of boilerplate Java code. What used to take 30 minutes now takes 2 minutes.

### Consistency

All entities follow the same patterns. No more inconsistencies between different developers or different parts of the codebase.

### Maintainability

Need to add a field? Just update the YAML and regenerate. The change propagates through entities, repositories, services, controllers, and database migrations.

### Type Safety

Generated code is fully type-safe with compile-time checking. No runtime surprises.

### Documentation

YAML files serve as living documentation. New team members can understand the data model by reading YAML files instead of scattered Java classes.

## Limitations and Considerations

While this approach is powerful, it's not suitable for everything:

- **Complex Business Logic:** Generated code handles standard CRUD. For complex domain logic, you'll need custom services.
- **Advanced Queries:** Custom repository methods need to be added separately.
- **Custom Validation:** Cross-field validation requires custom validators.
- **Learning Curve:** Team members need to learn the YAML schema and generation workflow.

## Getting Started

Ready to try it? Here's how to get started:

1. Clone the template repository
2. Define your first model in `src/main/resources/models/`
3. Run `mvn generate-sources`
4. Start the application with `mvn spring-boot:run`
5. Open `http://localhost:8080/swagger-ui.html`

**Pro Tip:** Start with a simple entity (like User or Product) to understand the generation process before moving to complex models with relationships.

## Maven Commands

| Command | Description |
|---------|-------------|
| `mvn generate-sources` | Generate code only (fast) |
| `mvn clean generate-sources` | Clean and regenerate all code |
| `mvn compile` | Generate + compile (automatic) |
| `mvn package` | Full build with generation |
| `mvn spring-boot:run` | Run app (generates code first) |

## What Gets Generated

For each YAML model definition, the system generates:

```
Generated Output for Product.yaml:
├── Product.java                  # JPA Entity with validations
├── ProductRepository.java        # Spring Data JPA repository
├── ProductService.java           # Service with CRUD operations
├── ProductController.java        # REST controller with OpenAPI docs
└── changelog-products.xml        # Liquibase migration
```

### Generated Entity Example

```java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @NotNull
    @Size(max = 255)
    @Column(nullable = false, length = 255)
    private String title;

    @NotNull
    @DecimalMin("0")
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @NotNull
    @Column(nullable = false, unique = true, length = 50)
    private String sku;

    @Column(nullable = false)
    private Boolean active = true;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    // Getters, setters, equals, hashCode...
}
```

### Generated Controller Example

```java
@RestController
@RequestMapping("/api/products")
@Tag(name = "Product", description = "Product API")
public class ProductController {

    private final ProductService productService;

    @GetMapping
    public Page<Product> getAll(Pageable pageable) {
        return productService.findAll(pageable);
    }

    @GetMapping("/{id}")
    public ResponseEntity<Product> getById(@PathVariable UUID id) {
        return productService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public Product create(@Valid @RequestBody Product product) {
        return productService.save(product);
    }

    @PutMapping("/{id}")
    public ResponseEntity<Product> update(
        @PathVariable UUID id,
        @Valid @RequestBody Product product
    ) {
        if (!productService.existsById(id)) {
            return ResponseEntity.notFound().build();
        }
        product.setId(id);
        return ResponseEntity.ok(productService.save(product));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable UUID id) {
        if (!productService.existsById(id)) {
            return ResponseEntity.notFound().build();
        }
        productService.deleteById(id);
        return ResponseEntity.noContent().build();
    }
}
```

## Development Workflow

### 1. Define or Update Model

```bash
vi src/main/resources/models/MyEntity.yaml
```

### 2. Generate Code

```bash
mvn generate-sources
```

Output:

```
=================================================
  Model Code Generator
=================================================

Found 2 model definition(s):
  - User.yaml
  - Product.yaml

--- Processing: User.yaml ---
  Parsing YAML...
  Validating model...
    ✓ Model: User
    ✓ Table: users
    ✓ Fields: 7
    ✓ Relationships: 0
  Generating files...
    ✓ Entity: User.java
    ✓ Repository: UserRepository.java
    ✓ Service: UserService.java
    ✓ Controller: UserController.java
    ✓ Liquibase: changelog-users.xml
✓ Generated code for: User

=================================================
  Successfully generated 2 model(s)
  Output: target/generated-sources/models
=================================================
```

### 3. Build and Run

```bash
mvn spring-boot:run
```

### 4. Test API

Visit http://localhost:8080/swagger-ui.html

## Customization

### Modify Templates

Edit FreeMarker templates in `src/main/resources/templates/codegen/`:

- `entity.ftl` - JPA Entity template
- `repository.ftl` - Repository interface template
- `service.ftl` - Service class template
- `controller.ftl` - REST controller template
- `liquibase-changelog.ftl` - Database migration template

### Extend Generated Code

Generated code should NOT be edited manually. To add custom logic:

- **Create wrapper services:** Extend or compose generated services
- **Add custom controllers:** Create additional REST endpoints
- **Use custom repositories:** Add custom query methods in separate interfaces

## Conclusion

Code generation isn't a silver bullet, but for standard CRUD operations, it eliminates hours of repetitive work. This YAML-driven approach to Spring Boot development lets you:

- Ship features faster
- Maintain consistency across your codebase
- Reduce boilerplate and human error
- Focus on business logic instead of plumbing code

The generated code is production-ready with proper validation, error handling, pagination, and API documentation. It's not a prototype or a demo—it's what you'd write by hand, just generated automatically.

Whether you're building a microservice, a monolith, or exploring domain-driven design, this template provides a solid foundation for rapid API development while maintaining code quality and consistency.

## References

- [Spring Boot Documentation](https://spring.io/projects/spring-boot)
- [Spring Data JPA](https://spring.io/projects/spring-data-jpa)
- [Liquibase Best Practices](https://www.liquibase.org/get-started/best-practices)
- [FreeMarker Template Engine](https://freemarker.apache.org/)
- [Jakarta Bean Validation](https://beanvalidation.org/)
- [SpringDoc OpenAPI](https://springdoc.org/)

## Key Takeaways

1. **YAML is powerful** - Simple, readable, and perfect for model definitions
2. **Code generation saves time** - 20 lines of YAML vs 200+ lines of Java
3. **Consistency matters** - All entities follow the same patterns
4. **Documentation is code** - YAML files serve as living documentation
5. **Don't edit generated code** - Extend and compose instead

In future posts, I'll explore advanced topics like implementing custom query methods, building multi-tenant applications with this template, and scaling the code generation system for large enterprise applications.
