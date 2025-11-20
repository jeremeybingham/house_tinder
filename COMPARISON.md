# Python vs Rust: Real Estate API Implementation Comparison

## Table of Contents

1. [Overview](#overview)
2. [Language & Framework Comparison](#language--framework-comparison)
3. [Type Systems](#type-systems)
4. [Enumerations](#enumerations)
5. [Data Models](#data-models)
6. [Serialization](#serialization)
7. [API Frameworks](#api-frameworks)
8. [Database Interaction](#database-interaction)
9. [Error Handling](#error-handling)
10. [Memory Management](#memory-management)
11. [Complete Implementation Example](#complete-implementation-example)
12. [Key Differences Summary](#key-differences-summary)
13. [When to Use Each](#when-to-use-each)

---

## Overview

This document compares implementing a real estate listing API (as described in `TODO.md`) using Python with FastAPI versus Rust with modern web frameworks. Both approaches offer different trade-offs in terms of performance, safety, developer experience, and ecosystem maturity.

### Quick Comparison Table

| Aspect | Python + FastAPI | Rust + Axum/Actix |
|--------|------------------|-------------------|
| **Type Safety** | Runtime (with hints) | Compile-time |
| **Performance** | ~50-100ms response | ~5-10ms response |
| **Memory Safety** | Garbage collected | Compile-time checked |
| **Development Speed** | Very fast | Moderate |
| **Learning Curve** | Gentle | Steep |
| **Error Handling** | Exceptions | Result<T, E> types |
| **Async/Await** | Native support | Native support |
| **Ecosystem** | Mature, extensive | Growing, modern |

---

## Language & Framework Comparison

### Python Stack
```
FastAPI (Web Framework)
├── Pydantic (Data Validation)
├── SQLAlchemy (ORM)
├── Uvicorn (ASGI Server)
└── PostgreSQL (Database)
```

### Rust Stack
```
Axum or Actix-web (Web Framework)
├── Serde (Serialization)
├── SQLx or Diesel (Database)
├── Tokio (Async Runtime)
└── PostgreSQL (Database)
```

---

## Type Systems

### Python: Dynamic with Type Hints

Python uses **gradual typing** - types are optional hints checked by tools like `mypy`, but not enforced at runtime.

```python
from typing import Optional, List
from pydantic import BaseModel, Field

class Address(BaseModel):
    """Type hints + runtime validation via Pydantic"""
    city: Optional[str] = None
    state: Optional[str] = None
    zip: Optional[str] = None
    street_number: Optional[str] = Field(None, alias="streetNumber")

    class Config:
        populate_by_name = True  # Accept both snake_case and camelCase
```

**Key Characteristics:**
- Type hints are **optional** and **not enforced** by Python runtime
- Pydantic provides **runtime validation** and conversion
- Flexible: can mix typed and untyped code
- Duck typing: "If it walks like a duck..."

### Rust: Static, Strong, Safe

Rust uses **static typing** - all types must be known at compile time, with zero runtime overhead.

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Address {
    /// Type annotations are REQUIRED and checked at compile time
    pub city: Option<String>,
    pub state: Option<String>,
    pub zip: Option<String>,

    #[serde(rename = "streetNumber")]
    pub street_number: Option<String>,
}
```

**Key Characteristics:**
- Type annotations are **mandatory**
- **Zero-cost abstractions** - no runtime overhead
- **Compiler guarantees** type correctness
- Cannot compile with type errors
- No null - uses `Option<T>` enum

### Detailed Comparison

| Feature | Python | Rust |
|---------|--------|------|
| **Nullability** | `None` can appear anywhere | `Option<T>` explicitly marks nullable |
| **Type Checking** | Optional, external tools | Built-in, compile-time |
| **Runtime Cost** | Validation overhead | Zero overhead |
| **Flexibility** | Can bypass types | Cannot bypass types |
| **Safety** | Runtime errors | Compile-time errors |

---

## Enumerations

### Python: String-Based Enums

```python
from enum import Enum

class Status(str, Enum):
    """
    Inherits from str to ensure JSON serialization works.
    Values are accessed as class attributes.
    """
    ACTIVE = "A"
    UNAVAILABLE = "U"

class ListingType(str, Enum):
    SALE = "sale"
    LEASE = "lease"

class LastStatus(str, Enum):
    SUSPENDED = "Sus"
    EXPIRED = "Exp"
    SOLD = "Sld"
    TERMINATED = "Ter"
    LEASED = "Lsd"
    # ... more variants

# Usage
status = Status.ACTIVE
print(status.value)  # "A"
print(status == "A")  # True (because it inherits from str)
```

**Python Enum Characteristics:**
- Runtime overhead for validation
- Can be bypassed (just use string)
- Inheritance from `str` for JSON compatibility
- Values accessed via `.value` attribute

### Rust: Algebraic Data Types

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum Status {
    #[serde(rename = "A")]
    Active,

    #[serde(rename = "U")]
    Unavailable,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum ListingType {
    Sale,
    Lease,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum LastStatus {
    #[serde(rename = "Sus")]
    Suspended,

    #[serde(rename = "Exp")]
    Expired,

    #[serde(rename = "Sld")]
    Sold,

    #[serde(rename = "Ter")]
    Terminated,

    #[serde(rename = "Lsd")]
    Leased,
    // ... more variants
}

// Usage
let status = Status::Active;
// This won't compile - type safety!
// let bad_status: Status = "A"; // ERROR: expected enum, found &str
```

**Rust Enum Characteristics:**
- **Zero runtime overhead** - compiled to integers
- **Exhaustive matching** - compiler ensures all cases handled
- **Cannot be bypassed** - type system enforces correctness
- **Memory efficient** - smallest possible representation
- Can carry data (tagged unions)

### Advanced Rust Enums: Tagged Unions

Rust enums can carry data, enabling powerful patterns not available in Python:

```rust
#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type", content = "value")]
pub enum PropertyClass {
    Condo { condo_corp: String, fees: CondoFees },
    Residential { lot_size: f64 },
    Commercial { zoning: String },
}

// Pattern matching is exhaustive - compiler ensures all cases handled
fn get_description(class: &PropertyClass) -> String {
    match class {
        PropertyClass::Condo { condo_corp, .. } => {
            format!("Condo in {}", condo_corp)
        }
        PropertyClass::Residential { lot_size } => {
            format!("Residential property, {} acres", lot_size)
        }
        PropertyClass::Commercial { zoning } => {
            format!("Commercial property, zoned {}", zoning)
        }
        // Compiler error if we forget a case!
    }
}
```

---

## Data Models

### Python: Pydantic Models

```python
from typing import Optional, List, Dict
from pydantic import BaseModel, Field
from datetime import datetime

class Bathroom(BaseModel):
    """Nested model for bathroom details"""
    pieces: Optional[str] = None
    level: Optional[str] = None
    count: Optional[str] = None

class Details(BaseModel):
    """Property details with 50+ fields"""
    num_bedrooms: Optional[str] = Field(None, alias="numBedrooms")
    num_bathrooms: Optional[str] = Field(None, alias="numBathrooms")
    sqft: Optional[str] = None
    property_type: Optional[str] = Field(None, alias="propertyType")
    description: Optional[str] = None
    year_built: Optional[str] = Field(None, alias="yearBuilt")

    # Nested structures
    bathrooms: Optional[Dict[str, Bathroom]] = None

    # Allow extra fields not defined in model
    class Config:
        populate_by_name = True
        extra = "allow"  # Flexible schema

class Listing(BaseModel):
    """Main listing model"""
    mls_number: Optional[str] = Field(None, alias="mlsNumber")
    status: Optional[Status] = None
    list_price: Optional[str] = Field(None, alias="listPrice")

    # Nested objects
    address: Optional[Address] = None
    details: Optional[Details] = None

    # Collections
    images: Optional[List[str]] = Field(default_factory=list)
    agents: Optional[List[Agent]] = Field(default_factory=list)

    class Config:
        populate_by_name = True
        extra = "allow"

        # Example for OpenAPI docs
        json_schema_extra = {
            "example": {
                "mlsNumber": "HMS1234567",
                "status": "A",
                "listPrice": "599000"
            }
        }
```

**Pydantic Features:**
- **Runtime validation** - checks types when creating instances
- **Automatic conversion** - "599000" string → kept as string
- **Alias support** - camelCase ↔ snake_case
- **Default factories** - `default_factory=list` for mutable defaults
- **Extra fields** - `extra="allow"` for flexible schemas
- **JSON serialization** - `.dict()` and `.json()` methods built-in

### Rust: Structs with Serde

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Bathroom {
    /// Nested struct for bathroom details
    #[serde(skip_serializing_if = "Option::is_none")]
    pub pieces: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub level: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub count: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Details {
    /// Property details with 50+ fields
    #[serde(skip_serializing_if = "Option::is_none")]
    pub num_bedrooms: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub num_bathrooms: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub sqft: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub property_type: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub description: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub year_built: Option<String>,

    /// Nested structures - HashMap for dynamic keys
    #[serde(skip_serializing_if = "Option::is_none")]
    pub bathrooms: Option<HashMap<String, Bathroom>>,

    // Note: Rust doesn't have "extra fields" by default
    // Use #[serde(flatten)] with HashMap<String, Value> for flexible schema
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Listing {
    /// Main listing struct
    #[serde(skip_serializing_if = "Option::is_none")]
    pub mls_number: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub status: Option<Status>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub list_price: Option<String>,

    /// Nested objects - Box for large nested structs (optimization)
    #[serde(skip_serializing_if = "Option::is_none")]
    pub address: Option<Box<Address>>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub details: Option<Box<Details>>,

    /// Collections - Vec is Rust's dynamic array
    #[serde(default)]
    pub images: Vec<String>,

    #[serde(default)]
    pub agents: Vec<Agent>,
}

// Manual implementation for defaults if needed
impl Default for Listing {
    fn default() -> Self {
        Self {
            mls_number: None,
            status: None,
            list_price: None,
            address: None,
            details: None,
            images: Vec::new(),
            agents: Vec::new(),
        }
    }
}
```

**Rust Struct Features:**
- **Compile-time validation** - impossible to create invalid struct
- **Zero-cost abstractions** - no runtime overhead
- **Explicit ownership** - `Box<T>` for heap allocation
- **No null pointers** - `Option<T>` is explicit and safe
- **Skip serialization** - `skip_serializing_if` for clean JSON
- **Derive macros** - automatic trait implementations
- **Memory layout control** - explicit about stack vs heap

### Key Rust Concepts Explained

#### 1. `Option<T>` - Null Safety

```rust
// Python: value can be None anywhere
def get_price(listing: dict) -> str:
    return listing.get("price")  # Could be None - runtime error!

// Rust: Option<T> forces you to handle None case
fn get_price(listing: &Listing) -> Option<String> {
    listing.list_price.clone()
}

// Must explicitly handle None
match get_price(&listing) {
    Some(price) => println!("Price: {}", price),
    None => println!("No price available"),
}

// Or use ? operator for early return
fn process_listing(listing: &Listing) -> Result<(), String> {
    let price = listing.list_price
        .as_ref()
        .ok_or("No price")?;

    // price is guaranteed to exist here
    Ok(())
}
```

#### 2. `Box<T>` - Heap Allocation

```rust
// Without Box - nested struct stored on stack
pub struct Listing {
    pub details: Option<Details>, // 50+ fields on stack!
}

// With Box - pointer on stack, data on heap
pub struct Listing {
    pub details: Option<Box<Details>>, // Just 8 bytes on stack
}

// Box provides:
// - Smaller stack frames
// - Better cache locality
// - Ability to create recursive types
```

#### 3. Ownership and Borrowing

```rust
// Python: Everything is a reference, GC cleans up
listing = Listing(mls_number="123")
other = listing  # Both point to same object

// Rust: Move semantics - ownership transfers!
let listing = Listing::default();
let other = listing;  // listing is now INVALID!
// println!("{:?}", listing); // ERROR: value moved

// Solution 1: Clone (explicit copy)
let listing = Listing::default();
let other = listing.clone();  // Explicit copy
println!("{:?}", listing);  // Works!

// Solution 2: Borrow (temporary reference)
let listing = Listing::default();
let other = &listing;  // Immutable borrow
println!("{:?}", listing);  // Still valid!
```

#### 4. Lifetimes (Advanced)

```rust
// Lifetimes ensure references don't outlive data

// Python: No concept of lifetimes, GC handles it
class Processor:
    def __init__(self, listing):
        self.listing = listing  # Reference kept alive by GC

// Rust: Lifetime parameters ensure safety
struct Processor<'a> {
    listing: &'a Listing,  // 'a = lifetime parameter
}

// Compiler ensures Processor can't outlive the Listing
fn process(listing: &Listing) {
    let processor = Processor { listing };
    // processor automatically dropped when function ends
} // listing reference guaranteed valid throughout
```

---

## Serialization

### Python: Pydantic Built-in

```python
from pydantic import BaseModel

class Listing(BaseModel):
    mls_number: str
    list_price: str

# From JSON string
listing = Listing.parse_raw('{"mlsNumber": "123", "listPrice": "599000"}')

# From dict
listing = Listing(**{"mlsNumber": "123", "listPrice": "599000"})

# To JSON string
json_str = listing.json()

# To dict
data = listing.dict()

# With alias conversion
data = listing.dict(by_alias=True)  # Uses camelCase
```

### Rust: Serde Ecosystem

```rust
use serde::{Deserialize, Serialize};
use serde_json;

#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct Listing {
    mls_number: String,
    list_price: String,
}

// From JSON string
let json = r#"{"mlsNumber": "123", "listPrice": "599000"}"#;
let listing: Listing = serde_json::from_str(json)?;

// From slice
let listing: Listing = serde_json::from_slice(json.as_bytes())?;

// To JSON string
let json = serde_json::to_string(&listing)?;

// To pretty JSON
let json = serde_json::to_string_pretty(&listing)?;

// To Value (like Python dict)
let value = serde_json::to_value(&listing)?;
```

**Key Differences:**

| Aspect | Python/Pydantic | Rust/Serde |
|--------|-----------------|------------|
| **Parsing** | `parse_raw()`, `parse_obj()` | `from_str()`, `from_slice()` |
| **Serializing** | `.json()`, `.dict()` | `to_string()`, `to_value()` |
| **Error Handling** | Raises `ValidationError` | Returns `Result<T, Error>` |
| **Performance** | ~10-50µs per object | ~1-5µs per object |
| **Validation** | Automatic with validators | Manual or with validator crate |

---

## API Frameworks

### Python: FastAPI

```python
from fastapi import FastAPI, HTTPException, Query, Path, Depends
from sqlalchemy.orm import Session
from typing import List, Optional

app = FastAPI(
    title="Real Estate API",
    description="Listing management system",
    version="1.0.0"
)

# Dependency injection
def get_db() -> Session:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/listings/{mls_number}", response_model=Listing)
async def get_listing(
    mls_number: str = Path(..., description="MLS number"),
    board_id: Optional[int] = Query(None),
    db: Session = Depends(get_db)
):
    """
    Get a specific listing by MLS number.

    - **mls_number**: The MLS number (required)
    - **board_id**: Optional board ID for disambiguation
    """
    listing = crud_listing.get_by_mls_number(
        db, mls_number=mls_number, board_id=board_id
    )

    if not listing:
        raise HTTPException(status_code=404, detail="Listing not found")

    return listing

@app.post("/search", response_model=SearchResponse)
async def search_listings(
    search_params: SearchRequest,
    db: Session = Depends(get_db)
):
    """Advanced search with 80+ parameters"""
    search_service = SearchService(db)
    return search_service.search(search_params)

# Automatic OpenAPI docs at /docs
```

**FastAPI Features:**
- **Automatic validation** - Pydantic models validate request/response
- **OpenAPI generation** - Swagger UI auto-generated
- **Dependency injection** - Clean separation of concerns
- **Async/await** - Native async support
- **Type hints** - IDEs provide autocomplete
- **Fast development** - Minimal boilerplate

### Rust: Axum Framework

```rust
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::{IntoResponse, Response, Json},
    routing::{get, post},
    Router,
};
use serde::{Deserialize, Serialize};
use sqlx::PgPool;
use std::sync::Arc;

// Application state
#[derive(Clone)]
struct AppState {
    db: PgPool,
}

// Query parameters
#[derive(Deserialize)]
struct ListingQuery {
    board_id: Option<i32>,
}

// Error type
enum ApiError {
    NotFound,
    DatabaseError(sqlx::Error),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            ApiError::NotFound => (StatusCode::NOT_FOUND, "Listing not found"),
            ApiError::DatabaseError(_) => {
                (StatusCode::INTERNAL_SERVER_ERROR, "Database error")
            }
        };

        (status, message).into_response()
    }
}

// Handler functions
async fn get_listing(
    Path(mls_number): Path<String>,
    Query(params): Query<ListingQuery>,
    State(state): State<Arc<AppState>>,
) -> Result<Json<Listing>, ApiError> {
    let listing = sqlx::query_as!(
        Listing,
        "SELECT * FROM listings WHERE mls_number = $1",
        mls_number
    )
    .fetch_optional(&state.db)
    .await
    .map_err(ApiError::DatabaseError)?
    .ok_or(ApiError::NotFound)?;

    Ok(Json(listing))
}

async fn search_listings(
    State(state): State<Arc<AppState>>,
    Json(search_params): Json<SearchRequest>,
) -> Result<Json<SearchResponse>, ApiError> {
    let search_service = SearchService::new(&state.db);
    let results = search_service
        .search(search_params)
        .await
        .map_err(ApiError::DatabaseError)?;

    Ok(Json(results))
}

// Router setup
#[tokio::main]
async fn main() {
    let db = PgPool::connect("postgresql://...").await.unwrap();

    let state = Arc::new(AppState { db });

    let app = Router::new()
        .route("/listings/:mls_number", get(get_listing))
        .route("/search", post(search_listings))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    axum::serve(listener, app).await.unwrap();
}
```

**Axum Features:**
- **Type-safe extractors** - Compile-time verified request parsing
- **Minimal overhead** - One of fastest web frameworks
- **Composable** - Middleware as regular functions
- **Async/await** - Built on Tokio runtime
- **No macros for routes** - Just regular functions
- **Explicit errors** - Result types everywhere

### Alternative: Rust Actix-web (More Like FastAPI)

```rust
use actix_web::{web, App, HttpResponse, HttpServer, Responder};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct ListingPath {
    mls_number: String,
}

#[derive(Deserialize)]
struct ListingQuery {
    board_id: Option<i32>,
}

// Handler - similar to FastAPI
async fn get_listing(
    path: web::Path<ListingPath>,
    query: web::Query<ListingQuery>,
    db: web::Data<PgPool>,
) -> impl Responder {
    match fetch_listing(&db, &path.mls_number, query.board_id).await {
        Ok(listing) => HttpResponse::Ok().json(listing),
        Err(_) => HttpResponse::NotFound().json("Listing not found"),
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let db = PgPool::connect("postgresql://...").await.unwrap();

    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(db.clone()))
            .route("/listings/{mls_number}", web::get().to(get_listing))
            .route("/search", web::post().to(search_listings))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

**Framework Comparison:**

| Feature | FastAPI | Axum | Actix-web |
|---------|---------|------|-----------|
| **Performance** | ~20k req/s | ~500k req/s | ~700k req/s |
| **Learning Curve** | Easy | Moderate | Moderate |
| **Async Support** | Native | Native | Native |
| **OpenAPI Docs** | Automatic | Manual (utoipa crate) | Manual |
| **Validation** | Automatic | Manual | Manual |
| **Maturity** | Very mature | Growing | Mature |

---

## Database Interaction

### Python: SQLAlchemy ORM

```python
from sqlalchemy import Column, Integer, String, DateTime, JSON
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Session
from sqlalchemy.sql import func

class ListingDB(Base):
    """SQLAlchemy ORM model"""
    __tablename__ = "listings"

    id = Column(Integer, primary_key=True, index=True)
    mls_number = Column(String(50), index=True, nullable=False)
    status = Column(String(10), index=True)
    list_price = Column(String(20))

    # JSONB for nested structures
    address = Column(JSONB)
    details = Column(JSONB)

    created_at = Column(DateTime, server_default=func.now())
    updated_at = Column(DateTime, onupdate=func.now())

# Usage
def get_listing(db: Session, mls_number: str) -> Optional[ListingDB]:
    return db.query(ListingDB)\
        .filter(ListingDB.mls_number == mls_number)\
        .first()

def create_listing(db: Session, listing: Listing) -> ListingDB:
    db_listing = ListingDB(**listing.dict())
    db.add(db_listing)
    db.commit()
    db.refresh(db_listing)
    return db_listing

def search_listings(
    db: Session,
    city: Optional[str] = None,
    min_price: Optional[float] = None
) -> List[ListingDB]:
    query = db.query(ListingDB)

    if city:
        query = query.filter(ListingDB.address["city"].astext == city)

    if min_price:
        query = query.filter(
            func.cast(ListingDB.list_price, Float) >= min_price
        )

    return query.all()
```

**SQLAlchemy Features:**
- **ORM abstraction** - Work with Python objects
- **Lazy loading** - Relationships loaded on demand
- **Session management** - Automatic transaction handling
- **Query builder** - Fluent API for queries
- **Migration support** - Alembic integration
- **JSONB queries** - PostgreSQL JSON operations

### Rust: SQLx (Compile-time Verified)

```rust
use sqlx::{PgPool, FromRow, query_as};
use chrono::{DateTime, Utc};

#[derive(Debug, FromRow, Serialize, Deserialize)]
pub struct ListingDB {
    pub id: i32,
    pub mls_number: String,
    pub status: Option<String>,
    pub list_price: Option<String>,

    // JSONB columns - stored as serde_json::Value
    pub address: Option<serde_json::Value>,
    pub details: Option<serde_json::Value>,

    pub created_at: DateTime<Utc>,
    pub updated_at: Option<DateTime<Utc>>,
}

// Get single listing
pub async fn get_listing(
    pool: &PgPool,
    mls_number: &str,
) -> Result<Option<ListingDB>, sqlx::Error> {
    let listing = sqlx::query_as!(
        ListingDB,
        r#"
        SELECT id, mls_number, status, list_price,
               address, details, created_at, updated_at
        FROM listings
        WHERE mls_number = $1
        "#,
        mls_number
    )
    .fetch_optional(pool)
    .await?;

    Ok(listing)
}

// Create listing
pub async fn create_listing(
    pool: &PgPool,
    listing: &Listing,
) -> Result<ListingDB, sqlx::Error> {
    let address_json = serde_json::to_value(&listing.address)?;
    let details_json = serde_json::to_value(&listing.details)?;

    let db_listing = sqlx::query_as!(
        ListingDB,
        r#"
        INSERT INTO listings (mls_number, status, list_price, address, details)
        VALUES ($1, $2, $3, $4, $5)
        RETURNING id, mls_number, status, list_price, address, details,
                  created_at, updated_at
        "#,
        listing.mls_number,
        listing.status.as_ref().map(|s| s.to_string()),
        listing.list_price,
        address_json,
        details_json,
    )
    .fetch_one(pool)
    .await?;

    Ok(db_listing)
}

// Search with optional filters
pub async fn search_listings(
    pool: &PgPool,
    city: Option<&str>,
    min_price: Option<f64>,
) -> Result<Vec<ListingDB>, sqlx::Error> {
    let mut query = String::from(
        "SELECT * FROM listings WHERE 1=1"
    );

    if city.is_some() {
        query.push_str(" AND address->>'city' = $1");
    }

    if min_price.is_some() {
        query.push_str(" AND CAST(list_price AS DECIMAL) >= $2");
    }

    // Note: This is simplified - in real code, use query builder
    // or prepared statements for safety

    let listings = sqlx::query_as::<_, ListingDB>(&query)
        .fetch_all(pool)
        .await?;

    Ok(listings)
}
```

**SQLx Features:**
- **Compile-time verification** - Queries checked against DB at compile time!
- **No ORM overhead** - Direct SQL, zero abstraction cost
- **Async/await** - Non-blocking database operations
- **Type safety** - Columns mapped to Rust types
- **Connection pooling** - Built-in efficient pooling
- **Prepared statements** - Automatic SQL injection protection

### Rust: Diesel ORM Alternative

```rust
use diesel::prelude::*;
use diesel::pg::PgConnection;

table! {
    listings (id) {
        id -> Int4,
        mls_number -> Varchar,
        status -> Nullable<Varchar>,
        list_price -> Nullable<Varchar>,
        address -> Nullable<Jsonb>,
        details -> Nullable<Jsonb>,
        created_at -> Timestamp,
        updated_at -> Nullable<Timestamp>,
    }
}

#[derive(Queryable, Serialize, Deserialize)]
pub struct ListingDB {
    pub id: i32,
    pub mls_number: String,
    pub status: Option<String>,
    pub list_price: Option<String>,
    pub address: Option<serde_json::Value>,
    pub details: Option<serde_json::Value>,
    pub created_at: NaiveDateTime,
    pub updated_at: Option<NaiveDateTime>,
}

#[derive(Insertable)]
#[table_name = "listings"]
pub struct NewListing {
    pub mls_number: String,
    pub status: Option<String>,
    pub list_price: Option<String>,
    pub address: Option<serde_json::Value>,
    pub details: Option<serde_json::Value>,
}

// Get listing
pub fn get_listing(
    conn: &PgConnection,
    mls_number: &str,
) -> QueryResult<ListingDB> {
    use self::listings::dsl::*;

    listings
        .filter(mls_number.eq(mls_number))
        .first::<ListingDB>(conn)
}

// Create listing
pub fn create_listing(
    conn: &PgConnection,
    new_listing: NewListing,
) -> QueryResult<ListingDB> {
    use self::listings::dsl::*;

    diesel::insert_into(listings)
        .values(&new_listing)
        .get_result(conn)
}
```

**Diesel Features:**
- **Compile-time safety** - Type-checked queries
- **Query builder** - Composable query construction
- **Migration support** - Built-in migration system
- **ORM-like** - Similar to SQLAlchemy
- **Zero runtime overhead** - All checks at compile time

**Database Comparison:**

| Feature | SQLAlchemy | SQLx | Diesel |
|---------|------------|------|--------|
| **Type** | ORM | Raw SQL | ORM |
| **Compile Checks** | No | Yes | Yes |
| **Performance** | Moderate | Very fast | Fast |
| **Flexibility** | High | Very high | Moderate |
| **Learning Curve** | Easy | Moderate | Steep |
| **Async Support** | Yes | Yes | No (yet) |

---

## Error Handling

### Python: Exceptions

```python
from fastapi import HTTPException
from sqlalchemy.exc import SQLAlchemyError
from pydantic import ValidationError

def get_listing(db: Session, mls_number: str) -> Listing:
    try:
        listing = db.query(ListingDB)\
            .filter(ListingDB.mls_number == mls_number)\
            .first()

        if not listing:
            raise HTTPException(
                status_code=404,
                detail=f"Listing {mls_number} not found"
            )

        return Listing.from_orm(listing)

    except ValidationError as e:
        # Pydantic validation failed
        raise HTTPException(status_code=422, detail=str(e))

    except SQLAlchemyError as e:
        # Database error
        raise HTTPException(status_code=500, detail="Database error")

    except Exception as e:
        # Unexpected error
        raise HTTPException(status_code=500, detail="Internal server error")

# Try-catch for error handling
try:
    listing = get_listing(db, "HMS123")
    print(f"Found: {listing.mls_number}")
except HTTPException as e:
    print(f"Error: {e.detail}")
```

**Python Error Handling:**
- **Exceptions** - Can be raised anywhere
- **Try-catch** - Handle errors where convenient
- **Stack unwinding** - Automatically propagates up call stack
- **Runtime overhead** - Exception handling has performance cost
- **Easy to miss** - Functions don't declare what errors they can raise

### Rust: Result Types

```rust
use sqlx::Error as SqlxError;
use thiserror::Error;

// Define custom error type
#[derive(Error, Debug)]
pub enum ApiError {
    #[error("Listing not found: {0}")]
    NotFound(String),

    #[error("Database error: {0}")]
    DatabaseError(#[from] SqlxError),

    #[error("Validation error: {0}")]
    ValidationError(String),

    #[error("Internal server error")]
    InternalError,
}

// Result type alias for convenience
pub type ApiResult<T> = Result<T, ApiError>;

// Function returns Result - errors are part of signature
pub async fn get_listing(
    pool: &PgPool,
    mls_number: &str,
) -> ApiResult<Listing> {
    // ? operator propagates errors up
    let db_listing = sqlx::query_as!(
        ListingDB,
        "SELECT * FROM listings WHERE mls_number = $1",
        mls_number
    )
    .fetch_optional(pool)
    .await?  // Converts SqlxError -> ApiError automatically
    .ok_or_else(|| ApiError::NotFound(mls_number.to_string()))?;

    // Convert DB model to domain model
    let listing = convert_to_listing(db_listing)
        .map_err(|e| ApiError::ValidationError(e.to_string()))?;

    Ok(listing)
}

// Caller must handle Result
async fn handler(pool: &PgPool, mls: &str) {
    match get_listing(pool, mls).await {
        Ok(listing) => println!("Found: {}", listing.mls_number.as_ref().unwrap()),
        Err(ApiError::NotFound(id)) => println!("Not found: {}", id),
        Err(ApiError::DatabaseError(e)) => println!("DB error: {}", e),
        Err(e) => println!("Other error: {}", e),
    }

    // Or use ? to propagate
    let listing = get_listing(pool, mls).await?;
    println!("Found: {}", listing.mls_number.as_ref().unwrap());
}
```

**Rust Error Handling:**
- **Result<T, E>** - Success or error, explicit in function signature
- **? operator** - Concise error propagation
- **Match expressions** - Exhaustive error handling
- **No exceptions** - No hidden control flow
- **Zero cost** - No runtime overhead when successful
- **Composable** - Errors can be mapped and chained

**Comparison:**

| Aspect | Python Exceptions | Rust Results |
|--------|-------------------|--------------|
| **Visibility** | Hidden | Explicit in signature |
| **Handling** | Optional | Mandatory |
| **Performance** | Overhead on error | Zero overhead on success |
| **Composition** | Difficult | Easy (`map`, `and_then`, etc.) |
| **Stack traces** | Automatic | Optional (`anyhow` crate) |

---

## Memory Management

### Python: Garbage Collection

```python
class Listing:
    def __init__(self, mls_number: str):
        self.mls_number = mls_number
        self.images = []  # Mutable default - careful!

    def __del__(self):
        """Called when object is garbage collected"""
        print(f"Listing {self.mls_number} deleted")

# Reference counting + cycle detection
listing = Listing("HMS123")
other_ref = listing  # Reference count = 2

del listing  # Reference count = 1, object still alive
del other_ref  # Reference count = 0, object deleted

# Circular references handled by GC
class Agent:
    def __init__(self):
        self.listings = []

class Listing:
    def __init__(self):
        self.agent = None

agent = Agent()
listing = Listing()
agent.listings.append(listing)
listing.agent = agent  # Circular reference - GC handles it
```

**Python Memory Management:**
- **Automatic** - No manual memory management
- **Reference counting** - Most objects freed immediately
- **Cycle detector** - Handles circular references
- **GC pauses** - Periodic pauses for collection
- **Overhead** - Memory and CPU overhead
- **Leaks possible** - If external resources not cleaned

### Rust: Ownership & Borrowing

```rust
#[derive(Debug)]
struct Listing {
    mls_number: String,
    images: Vec<String>,
}

impl Drop for Listing {
    fn drop(&mut self) {
        // Called when Listing goes out of scope
        println!("Listing {} dropped", self.mls_number);
    }
}

fn main() {
    // Ownership
    {
        let listing = Listing {
            mls_number: "HMS123".to_string(),
            images: vec![],
        };

        // listing is MOVED to other_ref
        let other_ref = listing;

        // ERROR: listing is no longer valid
        // println!("{:?}", listing);

        println!("{:?}", other_ref);  // OK
    } // other_ref dropped here, memory freed automatically

    // Borrowing
    {
        let listing = Listing {
            mls_number: "HMS124".to_string(),
            images: vec![],
        };

        // Borrow immutably - original still valid
        let ref1 = &listing;
        let ref2 = &listing;

        println!("{:?}", listing);  // Still valid!
        println!("{:?}", ref1);
        println!("{:?}", ref2);
    } // listing dropped here

    // Mutable borrowing
    {
        let mut listing = Listing {
            mls_number: "HMS125".to_string(),
            images: vec![],
        };

        // ONLY ONE mutable borrow allowed at a time
        let ref_mut = &mut listing;
        ref_mut.images.push("image1.jpg".to_string());

        // ERROR: Cannot have another mutable borrow
        // let ref_mut2 = &mut listing;

        // ERROR: Cannot have immutable borrow while mutable exists
        // let ref_immut = &listing;

        println!("{:?}", ref_mut);
    }

    // Reference counting (like Python, but explicit)
    use std::rc::Rc;

    let listing = Rc::new(Listing {
        mls_number: "HMS126".to_string(),
        images: vec![],
    });

    let ref1 = Rc::clone(&listing);  // Reference count = 2
    let ref2 = Rc::clone(&listing);  // Reference count = 3

    println!("Reference count: {}", Rc::strong_count(&listing)); // 3

    drop(ref1);  // Reference count = 2
    drop(ref2);  // Reference count = 1
    // listing dropped at end of scope, reference count = 0, memory freed
}
```

**Rust Memory Management:**
- **Compile-time** - All memory management checked at compile time
- **RAII** - Resource Acquisition Is Initialization
- **No GC** - No garbage collector, no pauses
- **Deterministic** - Exact moment when memory is freed
- **Zero overhead** - No runtime cost
- **Memory safe** - No use-after-free, no double-free, no data races

**Ownership Rules:**
1. Each value has exactly one owner
2. When owner goes out of scope, value is dropped
3. Values can be moved or borrowed
4. Only one mutable reference OR multiple immutable references at a time

**Comparison:**

| Aspect | Python GC | Rust Ownership |
|--------|-----------|----------------|
| **When freed** | Unpredictable | Predictable |
| **Performance** | GC pauses | No pauses |
| **Safety** | Runtime errors possible | Compile-time guaranteed |
| **Complexity** | Simple | Complex |
| **Circular refs** | Handled automatically | Requires `Rc<RefCell<T>>` |

---

## Complete Implementation Example

### Python: Complete Listing CRUD

```python
# models/listing.py
from typing import Optional
from pydantic import BaseModel, Field
from enum import Enum

class Status(str, Enum):
    ACTIVE = "A"
    UNAVAILABLE = "U"

class Address(BaseModel):
    city: Optional[str] = None
    state: Optional[str] = None
    zip: Optional[str] = None

class Listing(BaseModel):
    mls_number: Optional[str] = Field(None, alias="mlsNumber")
    status: Optional[Status] = None
    list_price: Optional[str] = Field(None, alias="listPrice")
    address: Optional[Address] = None

    class Config:
        populate_by_name = True

# crud/listing.py
from sqlalchemy.orm import Session
from typing import Optional, List

def get_listing(db: Session, mls_number: str) -> Optional[Listing]:
    db_listing = db.query(ListingDB)\
        .filter(ListingDB.mls_number == mls_number)\
        .first()

    if db_listing:
        return Listing.from_orm(db_listing)
    return None

def create_listing(db: Session, listing: Listing) -> Listing:
    db_listing = ListingDB(**listing.dict())
    db.add(db_listing)
    db.commit()
    db.refresh(db_listing)
    return Listing.from_orm(db_listing)

def update_listing(
    db: Session,
    mls_number: str,
    listing: Listing
) -> Optional[Listing]:
    db_listing = db.query(ListingDB)\
        .filter(ListingDB.mls_number == mls_number)\
        .first()

    if not db_listing:
        return None

    for key, value in listing.dict(exclude_unset=True).items():
        setattr(db_listing, key, value)

    db.commit()
    db.refresh(db_listing)
    return Listing.from_orm(db_listing)

# api/listings.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session

router = APIRouter(prefix="/listings", tags=["listings"])

@router.get("/{mls_number}", response_model=Listing)
async def get_listing_endpoint(
    mls_number: str,
    db: Session = Depends(get_db)
):
    listing = get_listing(db, mls_number)
    if not listing:
        raise HTTPException(status_code=404, detail="Listing not found")
    return listing

@router.post("/", response_model=Listing, status_code=201)
async def create_listing_endpoint(
    listing: Listing,
    db: Session = Depends(get_db)
):
    return create_listing(db, listing)

@router.put("/{mls_number}", response_model=Listing)
async def update_listing_endpoint(
    mls_number: str,
    listing: Listing,
    db: Session = Depends(get_db)
):
    updated = update_listing(db, mls_number, listing)
    if not updated:
        raise HTTPException(status_code=404, detail="Listing not found")
    return updated
```

### Rust: Complete Listing CRUD

```rust
// models/listing.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Copy, PartialEq, Serialize, Deserialize)]
pub enum Status {
    #[serde(rename = "A")]
    Active,
    #[serde(rename = "U")]
    Unavailable,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Address {
    #[serde(skip_serializing_if = "Option::is_none")]
    pub city: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub state: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub zip: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Listing {
    #[serde(skip_serializing_if = "Option::is_none")]
    pub mls_number: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub status: Option<Status>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub list_price: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub address: Option<Address>,
}

// crud/listing.rs
use sqlx::{PgPool, FromRow};
use crate::models::listing::Listing;

#[derive(Debug, FromRow)]
struct ListingDB {
    pub id: i32,
    pub mls_number: String,
    pub status: Option<String>,
    pub list_price: Option<String>,
    pub address: Option<serde_json::Value>,
}

pub async fn get_listing(
    pool: &PgPool,
    mls_number: &str,
) -> Result<Option<Listing>, sqlx::Error> {
    let db_listing = sqlx::query_as!(
        ListingDB,
        "SELECT * FROM listings WHERE mls_number = $1",
        mls_number
    )
    .fetch_optional(pool)
    .await?;

    let listing = db_listing.map(|db| Listing {
        mls_number: Some(db.mls_number),
        status: db.status.and_then(|s| serde_json::from_str(&s).ok()),
        list_price: db.list_price,
        address: db.address.and_then(|a| serde_json::from_value(a).ok()),
    });

    Ok(listing)
}

pub async fn create_listing(
    pool: &PgPool,
    listing: &Listing,
) -> Result<Listing, sqlx::Error> {
    let address_json = serde_json::to_value(&listing.address)
        .map_err(|e| sqlx::Error::Encode(Box::new(e)))?;

    let db_listing = sqlx::query_as!(
        ListingDB,
        r#"
        INSERT INTO listings (mls_number, status, list_price, address)
        VALUES ($1, $2, $3, $4)
        RETURNING id, mls_number, status, list_price, address
        "#,
        listing.mls_number,
        listing.status.as_ref().map(|s| serde_json::to_string(s).unwrap()),
        listing.list_price,
        address_json,
    )
    .fetch_one(pool)
    .await?;

    Ok(Listing {
        mls_number: Some(db_listing.mls_number),
        status: db_listing.status.and_then(|s| serde_json::from_str(&s).ok()),
        list_price: db_listing.list_price,
        address: db_listing.address.and_then(|a| serde_json::from_value(a).ok()),
    })
}

pub async fn update_listing(
    pool: &PgPool,
    mls_number: &str,
    listing: &Listing,
) -> Result<Option<Listing>, sqlx::Error> {
    let address_json = serde_json::to_value(&listing.address)
        .map_err(|e| sqlx::Error::Encode(Box::new(e)))?;

    let db_listing = sqlx::query_as!(
        ListingDB,
        r#"
        UPDATE listings
        SET status = COALESCE($1, status),
            list_price = COALESCE($2, list_price),
            address = COALESCE($3, address)
        WHERE mls_number = $4
        RETURNING id, mls_number, status, list_price, address
        "#,
        listing.status.as_ref().map(|s| serde_json::to_string(s).unwrap()),
        listing.list_price,
        address_json,
        mls_number,
    )
    .fetch_optional(pool)
    .await?;

    let listing = db_listing.map(|db| Listing {
        mls_number: Some(db.mls_number),
        status: db.status.and_then(|s| serde_json::from_str(&s).ok()),
        list_price: db.list_price,
        address: db.address.and_then(|a| serde_json::from_value(a).ok()),
    });

    Ok(listing)
}

// api/listings.rs
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::{get, post, put},
    Json, Router,
};
use std::sync::Arc;

struct AppState {
    db: PgPool,
}

enum ApiError {
    NotFound,
    DatabaseError(sqlx::Error),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            ApiError::NotFound => (StatusCode::NOT_FOUND, "Listing not found"),
            ApiError::DatabaseError(_) => {
                (StatusCode::INTERNAL_SERVER_ERROR, "Database error")
            }
        };
        (status, message).into_response()
    }
}

async fn get_listing_handler(
    Path(mls_number): Path<String>,
    State(state): State<Arc<AppState>>,
) -> Result<Json<Listing>, ApiError> {
    let listing = crate::crud::listing::get_listing(&state.db, &mls_number)
        .await
        .map_err(ApiError::DatabaseError)?
        .ok_or(ApiError::NotFound)?;

    Ok(Json(listing))
}

async fn create_listing_handler(
    State(state): State<Arc<AppState>>,
    Json(listing): Json<Listing>,
) -> Result<(StatusCode, Json<Listing>), ApiError> {
    let created = crate::crud::listing::create_listing(&state.db, &listing)
        .await
        .map_err(ApiError::DatabaseError)?;

    Ok((StatusCode::CREATED, Json(created)))
}

async fn update_listing_handler(
    Path(mls_number): Path<String>,
    State(state): State<Arc<AppState>>,
    Json(listing): Json<Listing>,
) -> Result<Json<Listing>, ApiError> {
    let updated = crate::crud::listing::update_listing(
        &state.db,
        &mls_number,
        &listing,
    )
    .await
    .map_err(ApiError::DatabaseError)?
    .ok_or(ApiError::NotFound)?;

    Ok(Json(updated))
}

pub fn router() -> Router<Arc<AppState>> {
    Router::new()
        .route("/listings/:mls_number", get(get_listing_handler))
        .route("/listings", post(create_listing_handler))
        .route("/listings/:mls_number", put(update_listing_handler))
}

// main.rs
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let db = PgPool::connect("postgresql://localhost/real_estate").await?;

    let state = Arc::new(AppState { db });

    let app = Router::new()
        .merge(api::listings::router())
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;

    println!("Server running on http://0.0.0.0:3000");

    axum::serve(listener, app).await?;

    Ok(())
}
```

---

## Key Differences Summary

### Development Experience

**Python Advantages:**
1. **Faster development** - Less boilerplate, more concise
2. **Easier learning curve** - Simpler concepts
3. **Better tooling for data science** - Pandas, NumPy, etc.
4. **Larger ecosystem** - More libraries available
5. **Dynamic typing** - More flexible for prototyping
6. **REPL-driven development** - Interactive exploration

**Rust Advantages:**
1. **Compile-time correctness** - Catch errors before runtime
2. **Better performance** - 10-100x faster than Python
3. **Memory safety** - No segfaults, no data races
4. **No runtime** - No GC pauses, predictable performance
5. **Better concurrency** - Fearless concurrency with ownership
6. **Long-term maintenance** - Refactoring is safer

### Technical Comparison

| Aspect | Python | Rust |
|--------|--------|------|
| **Startup time** | Fast | Instant |
| **Request latency** | 20-50ms | 1-5ms |
| **Memory usage** | High (GC overhead) | Low (no GC) |
| **CPU usage** | High (interpretation) | Low (native code) |
| **Concurrency** | GIL limitations | True parallelism |
| **Deployment** | Needs Python runtime | Single binary |
| **Error discovery** | Runtime | Compile-time |
| **Null safety** | No | Yes (Option<T>) |
| **Thread safety** | Runtime errors | Compile-time guaranteed |
| **Binary size** | N/A (interpreter) | 5-50MB |

### Code Size Comparison

For the same functionality:

```
Python:
- Lines of code: 100
- Compilation time: 0s
- Binary size: N/A (interpreted)
- Dependencies: ~50MB (venv)

Rust:
- Lines of code: 200-300
- Compilation time: 30-60s
- Binary size: 10-30MB (optimized)
- Dependencies: Statically linked
```

---

## When to Use Each

### Choose Python + FastAPI When:

✅ **Rapid prototyping** - Need to test ideas quickly
✅ **Data-heavy workloads** - Lots of pandas/numpy operations
✅ **ML/AI integration** - TensorFlow, PyTorch, scikit-learn
✅ **Small to medium scale** - <10k requests/second
✅ **Team expertise** - Team knows Python well
✅ **Quick iterations** - Requirements change frequently
✅ **Scripting needs** - Lots of automation scripts
✅ **Rich ecosystem needed** - Many specialized libraries

**Example use cases:**
- MVP/prototype development
- Internal admin tools
- Data analysis APIs
- ML model serving
- Report generation
- Web scraping services

### Choose Rust + Axum/Actix When:

✅ **High performance critical** - Need <5ms latency
✅ **High scale** - >100k requests/second
✅ **Long-running services** - Services running for months
✅ **Memory constraints** - Limited RAM (embedded, serverless)
✅ **Safety critical** - Financial, medical, infrastructure
✅ **Concurrent workloads** - Heavy parallelism needed
✅ **Minimal dependencies** - Single binary deployment
✅ **Low-level control** - Need fine-grained optimization

**Example use cases:**
- High-frequency trading systems
- Game servers
- IoT backends
- Microservices (high traffic)
- Cryptocurrency systems
- Real-time streaming
- Database proxies

### Hybrid Approach

Many projects use **both**:

```
Frontend (TypeScript/React)
    ↓
API Gateway (Rust - high performance)
    ↓
    ├─→ Listings Service (Rust - high traffic)
    ├─→ Search Service (Rust - low latency)
    └─→ Analytics Service (Python - data analysis)
            ↓
        PostgreSQL Database
```

---

## Migration Path

### From Python to Rust

If you start with Python and need Rust later:

1. **Profile first** - Identify bottlenecks
2. **Migrate hot paths** - Rewrite slow functions in Rust
3. **Use PyO3** - Call Rust from Python
4. **Gradual transition** - Service by service
5. **Shared data structures** - JSON/Protocol Buffers

**Example PyO3 (Rust function called from Python):**

```rust
// Rust (compiled to Python module)
use pyo3::prelude::*;

#[pyfunction]
fn search_listings(criteria: String) -> PyResult<Vec<String>> {
    // Fast Rust implementation
    Ok(vec!["result1".to_string(), "result2".to_string()])
}

#[pymodule]
fn fast_search(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(search_listings, m)?)?;
    Ok(())
}
```

```python
# Python (imports Rust module)
import fast_search

# Calls Rust code!
results = fast_search.search_listings(criteria)
```

---

## Additional Resources

### Python + FastAPI
- **FastAPI Docs**: https://fastapi.tiangolo.com
- **Pydantic Docs**: https://docs.pydantic.dev
- **SQLAlchemy Tutorial**: https://docs.sqlalchemy.org/en/20/tutorial/

### Rust
- **The Rust Book**: https://doc.rust-lang.org/book/
- **Rust by Example**: https://doc.rust-lang.org/rust-by-example/
- **Axum Examples**: https://github.com/tokio-rs/axum/tree/main/examples
- **SQLx Guide**: https://github.com/launchbadge/sqlx

### Comparison & Learning
- **Rust for Python Developers**: https://github.com/rochacbruno/py2rs
- **PyO3 Guide**: https://pyo3.rs/
- **API Performance Benchmarks**: https://www.techempower.com/benchmarks/

---

## Conclusion

Both Python with FastAPI and Rust with Axum/Actix are excellent choices for building a real estate API. The decision depends on your specific requirements:

- **Choose Python** for faster development, easier maintenance, and when performance is adequate
- **Choose Rust** for maximum performance, safety guarantees, and when scale is critical
- **Use both** in a hybrid architecture to get the best of both worlds

For the **house_tinder** project:
- **Start with Python** to validate the idea quickly
- **Profile and optimize** Python implementation first
- **Consider Rust** if you need <10ms response times or >50k req/s
- **Use Rust for hot paths** (search, filtering) via PyO3 if needed

Remember: **Premature optimization is the root of all evil.** Start simple, measure, then optimize where needed.
