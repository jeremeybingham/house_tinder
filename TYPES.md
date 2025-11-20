### Rust Implementation Issues

The Rust app currently uses **loosely typed responses** for listings:

```rust
// In search.rs
pub struct ListingSearchResponse {
    pub listings: Vec<serde_json::Value>,  // ⚠️ Untyped!
    // ...
}

// In listing.rs
pub async fn get_listing(...) -> Result<serde_json::Value, RepliersError> {
    // ⚠️ Returns raw JSON
}
```

The comments acknowledge this:
> "Currently uses `serde_json::Value` for flexibility, as listing structures can vary based on MLS board and available data."

### TypeScript Types Available

The TypeScript files provide **comprehensive, strongly-typed definitions** for:

1. **Complete Listing structure** (`listings.ts`):
   - 500+ lines of detailed type definitions
   - Nested structures for Address, Details, Condominium, Room, Bathroom, etc.
   - Proper optional field handling
   - Enums for constrained values (Status, Class, Type, etc.)

2. **Type safety helpers** (`index.ts`):
   - Enums with const assertions (ClassValues, StatusValues, etc.)
   - Common types (DateFormat, YesNo, etc.)
   - Aggregates and Statistics types

## Recommendations

### 1. **Create Strongly-Typed Listing Structs** (High Priority)

Translate the TypeScript `Listing` interface into Rust structs:

```rust
// src/models/listing.rs (NEW FILE)

use serde::{Deserialize, Serialize};

/// Property class type
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum Class {
    Condo,
    Residential,
    Commercial,
}

/// Listing status
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum Status {
    #[serde(rename = "A")]
    Active,
    #[serde(rename = "U")]
    Unavailable,
}

/// Listing type (sale or lease)
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum Type {
    Sale,
    Lease,
}

/// Last status of listing
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum LastStatus {
    Sus, Exp, Sld, Ter, Dft, Lsd, Sc, Sce, Lc, Pc, Ext, New,
}

/// Yes/No enumeration
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum YesNo {
    Y,
    N,
}

/// Property address information
#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct Address {
    pub area: Option<String>,
    pub city: Option<String>,
    pub country: Option<String>,
    pub district: Option<String>,
    #[serde(rename = "majorIntersection")]
    pub major_intersection: Option<String>,
    pub neighborhood: Option<String>,
    #[serde(rename = "streetDirection")]
    pub street_direction: Option<String>,
    #[serde(rename = "streetName")]
    pub street_name: Option<String>,
    #[serde(rename = "streetNumber")]
    pub street_number: Option<String>,
    #[serde(rename = "streetSuffix")]
    pub street_suffix: Option<String>,
    #[serde(rename = "unitNumber")]
    pub unit_number: Option<String>,
    pub zip: Option<String>,
    pub state: Option<String>,
    #[serde(rename = "communityCode")]
    pub community_code: Option<String>,
    #[serde(rename = "streetDirectionPrefix")]
    pub street_direction_prefix: Option<String>,
}

/// Geographic coordinates
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Map {
    pub latitude: String,
    pub longitude: String,
    pub point: String,
}

/// Property details
#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct Details {
    #[serde(rename = "airConditioning")]
    pub air_conditioning: Option<String>,
    pub basement1: Option<String>,
    pub basement2: Option<String>,
    #[serde(rename = "centralVac")]
    pub central_vac: Option<String>,
    pub den: Option<YesNo>,
    pub description: Option<String>,
    pub elevator: Option<String>,
    #[serde(rename = "exteriorConstruction1")]
    pub exterior_construction1: Option<String>,
    #[serde(rename = "exteriorConstruction2")]
    pub exterior_construction2: Option<String>,
    pub extras: Option<String>,
    pub garage: Option<String>,
    pub heating: Option<String>,
    #[serde(rename = "numBathrooms")]
    pub num_bathrooms: Option<String>,
    #[serde(rename = "numBathroomsPlus")]
    pub num_bathrooms_plus: Option<String>,
    #[serde(rename = "numBedrooms")]
    pub num_bedrooms: Option<String>,
    #[serde(rename = "numBedroomsPlus")]
    pub num_bedrooms_plus: Option<String>,
    #[serde(rename = "propertyType")]
    pub property_type: Option<String>,
    pub sqft: Option<String>,
    pub style: Option<String>,
    #[serde(rename = "swimmingPool")]
    pub swimming_pool: Option<String>,
    #[serde(rename = "yearBuilt")]
    pub year_built: Option<String>,
    // ... add remaining fields from TypeScript
}

/// Condominium-specific information
#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct Condominium {
    pub ammenities: Option<Vec<String>>,
    #[serde(rename = "buildingInsurance")]
    pub building_insurance: Option<String>,
    #[serde(rename = "condoCorp")]
    pub condo_corp: Option<String>,
    pub exposure: Option<String>,
    pub locker: Option<String>,
    #[serde(rename = "parkingType")]
    pub parking_type: Option<String>,
    pub pets: Option<String>,
    pub stories: Option<String>,
    pub fees: Option<CondominiumFees>,
    // ... add remaining fields
}

#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct CondominiumFees {
    #[serde(rename = "cableIncl")]
    pub cable_incl: Option<YesNo>,
    #[serde(rename = "heatIncl")]
    pub heat_incl: Option<YesNo>,
    #[serde(rename = "hydroIncl")]
    pub hydro_incl: Option<YesNo>,
    pub maintenance: Option<String>,
    // ... more fee fields
}

/// Room information
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Room {
    pub description: Option<String>,
    pub features: Option<String>,
    pub features2: Option<String>,
    pub features3: Option<String>,
    pub length: Option<String>,
    pub width: Option<String>,
    pub level: Option<String>,
}

/// Complete listing structure
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Listing {
    #[serde(rename = "mlsNumber")]
    pub mls_number: Option<String>,
    pub resource: Option<String>,
    pub status: Option<Status>,
    pub class: Option<Class>,
    #[serde(rename = "type")]
    pub listing_type: Option<Type>,
    #[serde(rename = "listPrice")]
    pub list_price: Option<String>,
    #[serde(rename = "listDate")]
    pub list_date: Option<String>,
    #[serde(rename = "lastStatus")]
    pub last_status: Option<LastStatus>,
    #[serde(rename = "soldPrice")]
    pub sold_price: Option<String>,
    #[serde(rename = "soldDate")]
    pub sold_date: Option<String>,
    pub address: Option<Address>,
    pub map: Option<Map>,
    pub details: Option<Details>,
    pub condominium: Option<Condominium>,
    pub images: Option<Vec<String>>,
    #[serde(rename = "photoCount")]
    pub photo_count: Option<i32>,
    #[serde(rename = "daysOnMarket")]
    pub days_on_market: Option<String>,
    pub rooms: Option<std::collections::HashMap<String, Room>>,
    // ... add remaining top-level fields
    
    // Use serde(flatten) for extensibility
    #[serde(flatten)]
    pub extra: std::collections::HashMap<String, serde_json::Value>,
}
```

### 2. **Update Response Types** (High Priority)

Replace `serde_json::Value` with proper types:

```rust
// src/models/search.rs - UPDATE

use super::listing::Listing;

#[derive(Debug, Clone, Deserialize)]
pub struct ListingSearchResponse {
    // Change from Vec<serde_json::Value> to Vec<Listing>
    pub listings: Vec<Listing>,
    pub page: u32,
    #[serde(rename = "numPages")]
    pub num_pages: u32,
    #[serde(rename = "pageSize")]
    pub page_size: u32,
    pub count: u32,
}
```

### 3. **Hybrid Approach for Flexibility** (Medium Priority)

Keep flexibility for unknown fields while gaining type safety:

```rust
/// Listing with strong typing for known fields
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Listing {
    // Strongly typed required/common fields
    #[serde(rename = "mlsNumber")]
    pub mls_number: Option<String>,
    pub status: Option<Status>,
    pub class: Option<Class>,
    pub address: Option<Address>,
    // ...
    
    /// Catch-all for MLS-specific or unknown fields
    #[serde(flatten)]
    pub extensions: std::collections::HashMap<String, serde_json::Value>,
}

impl Listing {
    /// Type-safe accessor with fallback to extensions
    pub fn get_field<T: serde::de::DeserializeOwned>(&self, key: &str) -> Option<T> {
        self.extensions
            .get(key)
            .and_then(|v| serde_json::from_value(v.clone()).ok())
    }
}
```

### 4. **Add Builder Pattern for Search** (Low Priority)

The TypeScript types show extensive search parameters - improve the Rust API:

```rust
// Already partially implemented, but expand it:

impl ListingSearchRequest {
    pub fn builder() -> ListingSearchRequestBuilder {
        ListingSearchRequestBuilder::default()
    }
}

pub struct ListingSearchRequestBuilder {
    // ... existing fields
    
    // Add more from TypeScript SearchRequest
    aggregates: Option<Vec<String>>,
    amenities: Option<Vec<String>>,
    cluster: Option<bool>,
    has_images: Option<bool>,
    // ... etc
}

impl ListingSearchRequestBuilder {
    // Fluent API methods
    pub fn with_aggregates(mut self, agg: Vec<String>) -> Self {
        self.aggregates = Some(agg);
        self
    }
    
    pub fn with_images_only(mut self) -> Self {
        self.has_images = Some(true);
        self
    }
    
    // ... more builder methods
}
```

### 5. **Add Type Validation** (Medium Priority)

Use the TypeScript const arrays as validation sources:

```rust
// src/models/validation.rs (NEW FILE)

pub const AGGREGATE_VALUES: &[&str] = &[
    "class",
    "status",
    "lastStatus",
    "type",
    "listPrice",
    "address.area",
    "address.city",
    // ... from TypeScript AggregatesValues
];

pub const SORT_BY_VALUES: &[&str] = &[
    "createdOnDesc",
    "updatedOnDesc",
    "distanceAsc",
    // ... from TypeScript SortByValues
];

impl ListingSearchRequest {
    pub fn validate(&self) -> Result<(), ValidationError> {
        // Validate sortBy if present
        if let Some(ref sort) = self.sort_by {
            if !SORT_BY_VALUES.contains(&sort.as_str()) {
                return Err(ValidationError::InvalidSortBy(sort.clone()));
            }
        }
        Ok(())
    }
}
```

## Implementation Priority

### Phase 1 (Immediate - High Value)
1. Create `src/models/listing.rs` with core types (Status, Class, Type, Address, Listing)
2. Update `ListingSearchResponse` to use `Vec<Listing>`
3. Update `get_listing()` to return `Result<Listing, _>`

### Phase 2 (Near-term)
1. Add remaining nested types (Details, Condominium, Room, etc.)
2. Add validation helpers
3. Update documentation with type examples

### Phase 3 (Future Enhancement)
1. Expand SearchRequest with all TypeScript parameters
2. Add compile-time validation where possible
3. Create type-safe aggregation builders

## Benefits

1. **Type Safety**: Catch errors at compile time instead of runtime
2. **Better IDE Support**: Autocomplete for listing fields
3. **Documentation**: Self-documenting code with types
4. **Maintainability**: Easier to track API changes
5. **Performance**: No repeated JSON parsing for common fields
6. **Flexibility**: `flatten` attribute preserves unknown fields

## Migration Path

This can be done incrementally:
1. Add new typed structs alongside existing `serde_json::Value` usage
2. Gradually migrate functions to use typed versions
3. Keep `Value` as fallback for truly dynamic cases
4. Document which MLS boards might have additional fields

The TypeScript definitions provide an excellent blueprint for a more robust, type-safe Rust implementation while maintaining the flexibility needed for varying MLS data sources. This approach ensures that all API interactions are handled with clarity and reliability, making it easier to maintain and evolve the Repliers API client in the future.