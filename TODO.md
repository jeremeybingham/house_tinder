# TODO: Python/FastAPI Implementation for Real Estate Listing Types

## 📊 Analysis Summary

### Data Structure Overview

After analyzing `report.json` (3,564 lines), `toronto_listings.json` (58,160 lines), and the TypeScript definitions, here's what we're working with:

#### 1. **Core Data Models** (from TypeScript → Python)

- **Listing**: Main entity with 30+ top-level fields and deeply nested objects
- **Address**: Street address components (14 fields)
- **Details**: Property details (50+ fields including bathrooms dict)
- **Condominium**: Condo-specific data (13 fields + nested fees object)
- **Room**: Room information (7 fields)
- **Bathroom**: Bathroom details (3 fields)
- **Agent**: Agent information with brokerage details (13 fields)
- **Timestamp**: Various date tracking fields (13 fields)
- **OpenHouse**: Open house scheduling (6 fields)

#### 2. **Enumerations** (Strongly Typed Values)

- **Status**: `["A", "U"]` (Active, Unavailable)
- **Class**: `["condo", "residential", "commercial"]`
- **Type**: `["sale", "lease"]`
- **YesNo**: `["Y", "N"]`
- **LastStatus**: `["Sus", "Exp", "Sld", "Ter", "Dft", "Lsd", "Sc", "Sce", "Lc", "Pc", "Ext", "New"]`
- **StreetDirection**: `["n", "e", "w", "s", ""]`
- **SortBy**: 25+ sorting options

#### 3. **API Request/Response Types**

- **SearchRequest**: 80+ optional query parameters for filtering listings
- **SearchResponse**: Paginated results with listings array + statistics
- **ListingRequest/Response**: Single listing retrieval
- **SimilarRequest/Response**: Similar property search
- **LocationsRequest/Response**: Geographic hierarchy (Board → Area → City → Neighborhood)

#### 4. **JSON Data Characteristics**

**report.json**:
- Aggregated analytics data
- Top cities with counts/avg prices
- Property type distribution
- Status distribution
- Price range statistics
- Sample listing objects

**toronto_listings.json**:
- Array of 42,749 listing objects
- Complete listing data with all nested structures
- Real-world data with many optional fields as `null`
- Demonstrates the TypeScript definitions in practice

---

## 🎯 Implementation Plan

### Phase 1: Project Structure & Models

#### 1.1 Create Project Directory Structure

```
house_tinder_api/
├── app/
│   ├── __init__.py
│   ├── main.py                    # FastAPI app initialization
│   ├── config.py                  # Configuration & environment vars
│   ├── dependencies.py            # Dependency injection
│   │
│   ├── models/                    # Pydantic models (matching TypeScript)
│   │   ├── __init__.py
│   │   ├── base.py                # Base model classes
│   │   ├── enums.py               # All enumerations
│   │   ├── listing.py             # Listing model
│   │   ├── address.py             # Address model
│   │   ├── details.py             # Details model
│   │   ├── condominium.py         # Condominium model
│   │   ├── agent.py               # Agent model
│   │   ├── search.py              # SearchRequest/Response models
│   │   └── locations.py           # Location hierarchy models
│   │
│   ├── schemas/                   # Database schemas (SQLAlchemy)
│   │   ├── __init__.py
│   │   ├── listing.py
│   │   ├── agent.py
│   │   ├── search.py
│   │   └── base.py
│   │
│   ├── api/                       # API routes
│   │   ├── __init__.py
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── listings.py        # Listing CRUD endpoints
│   │   │   ├── searches.py        # Search endpoints
│   │   │   ├── agents.py          # Agent endpoints
│   │   │   ├── locations.py       # Location endpoints
│   │   │   └── analytics.py       # Analytics endpoints
│   │
│   ├── crud/                      # Database operations
│   │   ├── __init__.py
│   │   ├── base.py                # Base CRUD class
│   │   ├── listing.py
│   │   ├── agent.py
│   │   └── search.py
│   │
│   ├── db/                        # Database configuration
│   │   ├── __init__.py
│   │   ├── session.py             # Database session management
│   │   └── init_db.py             # Database initialization
│   │
│   ├── services/                  # Business logic
│   │   ├── __init__.py
│   │   ├── listing_service.py     # Listing business logic
│   │   ├── search_service.py      # Search logic
│   │   ├── analytics_service.py   # Analytics computation
│   │   └── import_service.py      # JSON import logic
│   │
│   └── utils/                     # Utility functions
│       ├── __init__.py
│       ├── validators.py          # Custom validators
│       ├── converters.py          # Type converters
│       └── helpers.py             # Helper functions
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_models.py
│   ├── test_api/
│   │   ├── test_listings.py
│   │   ├── test_searches.py
│   │   └── test_analytics.py
│   └── test_services/
│
├── alembic/                       # Database migrations
│   └── versions/
│
├── scripts/
│   ├── import_listings.py         # Import toronto_listings.json
│   └── generate_report.py         # Generate report.json
│
├── requirements.txt
├── pyproject.toml
├── .env.example
└── README.md
```

---

### Phase 2: Pydantic Models (TypeScript → Python)

#### 2.1 Base Models & Enums (`app/models/enums.py`)

```python
from enum import Enum
from typing import Literal

class Status(str, Enum):
    """Listing status"""
    ACTIVE = "A"
    UNAVAILABLE = "U"

class Class(str, Enum):
    """Property class"""
    CONDO = "condo"
    RESIDENTIAL = "residential"
    COMMERCIAL = "commercial"

class ListingType(str, Enum):
    """Listing type (sale or lease)"""
    SALE = "sale"
    LEASE = "lease"

class YesNo(str, Enum):
    """Yes/No enumeration"""
    YES = "Y"
    NO = "N"

class LastStatus(str, Enum):
    """Last status of listing"""
    SUSPENDED = "Sus"
    EXPIRED = "Exp"
    SOLD = "Sld"
    TERMINATED = "Ter"
    DRAFT = "Dft"
    LEASED = "Lsd"
    SOLD_CONDITIONAL = "Sc"
    SALE_CONDITIONAL_EXPIRED = "Sce"
    LEASE_CONDITIONAL = "Lc"
    PRICE_CHANGE = "Pc"
    EXTENDED = "Ext"
    NEW = "New"

class StreetDirection(str, Enum):
    """Street direction"""
    NORTH = "n"
    EAST = "e"
    WEST = "w"
    SOUTH = "s"
    NONE = ""

class SortBy(str, Enum):
    """Sort options for search results"""
    CREATED_DESC = "createdOnDesc"
    UPDATED_DESC = "updatedOnDesc"
    CREATED_ASC = "createdOnAsc"
    DISTANCE_ASC = "distanceAsc"
    DISTANCE_DESC = "distanceDesc"
    SOLD_DATE_ASC = "soldDateAsc"
    SOLD_DATE_DESC = "soldDateDesc"
    SOLD_PRICE_ASC = "soldPriceAsc"
    SOLD_PRICE_DESC = "soldPriceDesc"
    LIST_PRICE_ASC = "listPriceAsc"
    LIST_PRICE_DESC = "listPriceDesc"
    # ... add all 25+ options

# Type aliases
DateFormat = str  # Format: YYYY-MM-DD
Operator = Literal["AND", "OR"]
```

#### 2.2 Address Model (`app/models/address.py`)

```python
from typing import Optional
from pydantic import BaseModel, Field

class Address(BaseModel):
    """Property address information"""

    address_key: Optional[str] = Field(None, alias="addressKey")
    area: Optional[str] = None
    city: Optional[str] = None
    country: Optional[str] = None
    district: Optional[str] = None
    major_intersection: Optional[str] = Field(None, alias="majorIntersection")
    neighborhood: Optional[str] = None
    street_direction: Optional[str] = Field(None, alias="streetDirection")
    street_direction_prefix: Optional[str] = Field(None, alias="streetDirectionPrefix")
    street_name: Optional[str] = Field(None, alias="streetName")
    street_number: Optional[str] = Field(None, alias="streetNumber")
    street_suffix: Optional[str] = Field(None, alias="streetSuffix")
    unit_number: Optional[str] = Field(None, alias="unitNumber")
    zip: Optional[str] = None
    state: Optional[str] = None
    community_code: Optional[str] = Field(None, alias="communityCode")

    class Config:
        populate_by_name = True  # Allow both snake_case and camelCase
        json_schema_extra = {
            "example": {
                "streetNumber": "16328",
                "streetDirectionPrefix": "W",
                "streetName": "166th",
                "streetSuffix": "Place",
                "city": "Olathe",
                "state": "KS",
                "zip": "66062",
                "neighborhood": "Stonebridge Trails"
            }
        }
```

#### 2.3 Details Model (`app/models/details.py`)

```python
from typing import Optional, Dict
from pydantic import BaseModel, Field
from .enums import YesNo

class Bathroom(BaseModel):
    """Bathroom details"""
    pieces: Optional[str] = None
    level: Optional[str] = None
    count: Optional[str] = None

class Details(BaseModel):
    """Property details"""

    air_conditioning: Optional[str] = Field(None, alias="airConditioning")
    basement1: Optional[str] = None
    basement2: Optional[str] = None
    central_vac: Optional[str] = Field(None, alias="centralVac")
    den: Optional[YesNo] = None
    description: Optional[str] = None
    elevator: Optional[str] = None
    exterior_construction1: Optional[str] = Field(None, alias="exteriorConstruction1")
    exterior_construction2: Optional[str] = Field(None, alias="exteriorConstruction2")
    extras: Optional[str] = None
    furnished: Optional[str] = None
    garage: Optional[str] = None
    heating: Optional[str] = None
    num_bathrooms: Optional[str] = Field(None, alias="numBathrooms")
    num_bathrooms_plus: Optional[str] = Field(None, alias="numBathroomsPlus")
    num_bathrooms_half: Optional[str] = Field(None, alias="numBathroomsHalf")
    num_bedrooms: Optional[str] = Field(None, alias="numBedrooms")
    num_bedrooms_plus: Optional[str] = Field(None, alias="numBedroomsPlus")
    num_fireplaces: Optional[str] = Field(None, alias="numFireplaces")
    num_garage_spaces: Optional[str] = Field(None, alias="numGarageSpaces")
    num_parking_spaces: Optional[str] = Field(None, alias="numParkingSpaces")
    property_type: Optional[str] = Field(None, alias="propertyType")
    sqft: Optional[str] = None
    style: Optional[str] = None
    swimming_pool: Optional[str] = Field(None, alias="swimmingPool")
    virtual_tour_url: Optional[str] = Field(None, alias="virtualTourUrl")
    year_built: Optional[str] = Field(None, alias="yearBuilt")

    # Additional fields
    bathrooms: Optional[Dict[str, Bathroom]] = None
    zoning: Optional[str] = None
    driveway: Optional[str] = None
    water_source: Optional[str] = Field(None, alias="waterSource")
    sewer: Optional[str] = None

    # Allow extra fields for extensibility
    class Config:
        populate_by_name = True
        extra = "allow"
```

#### 2.4 Condominium Model (`app/models/condominium.py`)

```python
from typing import Optional, List
from pydantic import BaseModel, Field
from .enums import YesNo

class CondominiumFees(BaseModel):
    """Condominium fee details"""
    cable_incl: Optional[YesNo] = Field(None, alias="cableIncl")
    heat_incl: Optional[YesNo] = Field(None, alias="heatIncl")
    hydro_incl: Optional[YesNo] = Field(None, alias="hydroIncl")
    maintenance: Optional[str] = None
    parking_incl: Optional[YesNo] = Field(None, alias="parkingIncl")
    taxes_incl: Optional[YesNo] = Field(None, alias="taxesIncl")
    water_incl: Optional[YesNo] = Field(None, alias="waterIncl")

    class Config:
        populate_by_name = True

class Condominium(BaseModel):
    """Condominium-specific information"""

    amenities: Optional[List[str]] = Field(default_factory=list, alias="ammenities")
    building_insurance: Optional[str] = Field(None, alias="buildingInsurance")
    condo_corp: Optional[str] = Field(None, alias="condoCorp")
    condo_corp_num: Optional[str] = Field(None, alias="condoCorpNum")
    exposure: Optional[str] = None
    locker: Optional[str] = None
    locker_number: Optional[str] = Field(None, alias="lockerNumber")
    locker_level: Optional[str] = Field(None, alias="lockerLevel")
    parking_type: Optional[str] = Field(None, alias="parkingType")
    pets: Optional[str] = None
    property_mgr: Optional[str] = Field(None, alias="propertyMgr")
    stories: Optional[str] = None
    fees: Optional[CondominiumFees] = None
    ensuite_laundry: Optional[YesNo] = Field(None, alias="ensuiteLaundry")

    class Config:
        populate_by_name = True
```

#### 2.5 Agent Model (`app/models/agent.py`)

```python
from typing import Optional, List, Dict
from pydantic import BaseModel, Field, EmailStr

class AgentPhoto(BaseModel):
    """Agent photo URLs"""
    small: Optional[str] = None
    large: Optional[str] = None
    updated_on: Optional[str] = Field(None, alias="updatedOn")

    class Config:
        populate_by_name = True

class BrokerageAddress(BaseModel):
    """Brokerage address"""
    address1: Optional[str] = None
    address2: Optional[str] = None
    city: Optional[str] = None
    state: Optional[str] = None
    postal: Optional[str] = None
    country: Optional[str] = None

class Brokerage(BaseModel):
    """Brokerage information"""
    name: str
    address: Optional[BrokerageAddress] = None

class Agent(BaseModel):
    """Real estate agent information"""

    agent_id: Optional[str] = Field(None, alias="agentId")
    board_agent_id: Optional[str] = Field(None, alias="boardAgentId")
    office_id: Optional[str] = Field(None, alias="officeId")
    board_office_id: Optional[str] = Field(None, alias="boardOfficeId")
    updated_on: Optional[str] = Field(None, alias="updatedOn")
    name: Optional[str] = None
    board: Optional[str] = None
    position: Optional[str] = None
    email: Optional[str] = None  # Can be "REDACTED"
    phones: Optional[List[str]] = Field(default_factory=list)
    social: Optional[List[str]] = Field(default_factory=list)
    website: Optional[str] = None
    photo: Optional[AgentPhoto] = None
    brokerage: Optional[Brokerage] = None

    class Config:
        populate_by_name = True
```

#### 2.6 Main Listing Model (`app/models/listing.py`)

```python
from typing import Optional, List, Dict, Any
from pydantic import BaseModel, Field
from .enums import Status, Class, ListingType, LastStatus
from .address import Address
from .details import Details
from .condominium import Condominium
from .agent import Agent

class Map(BaseModel):
    """Geographic coordinates"""
    latitude: str
    longitude: str
    point: str

class Room(BaseModel):
    """Room information"""
    description: Optional[str] = None
    features: Optional[str] = None
    features2: Optional[str] = None
    features3: Optional[str] = None
    length: Optional[str] = None
    width: Optional[str] = None
    level: Optional[str] = None

class Lot(BaseModel):
    """Lot information"""
    acres: Optional[str] = None
    depth: Optional[str] = None
    irregular: Optional[str] = None
    legal_description: Optional[str] = Field(None, alias="legalDescription")
    measurement: Optional[str] = None
    width: Optional[str] = None
    size: Optional[str] = None

    class Config:
        populate_by_name = True

class Taxes(BaseModel):
    """Tax information"""
    annual_amount: Optional[str] = Field(None, alias="annualAmount")
    assessment_year: Optional[str] = Field(None, alias="assessmentYear")

    class Config:
        populate_by_name = True

class Timestamp(BaseModel):
    """Timestamp tracking"""
    idx_updated: Optional[str] = Field(None, alias="idxUpdated")
    listing_updated: Optional[str] = Field(None, alias="listingUpdated")
    photos_updated: Optional[str] = Field(None, alias="photosUpdated")
    list_date: Optional[str] = Field(None, alias="listDate")
    sold_date: Optional[str] = Field(None, alias="soldDate")
    repliers_updated_on: Optional[str] = Field(None, alias="repliersUpdatedOn")

    class Config:
        populate_by_name = True

class Listing(BaseModel):
    """
    Complete listing structure.
    All fields are optional as per TypeScript definition.
    """

    # Core identifiers
    mls_number: Optional[str] = Field(None, alias="mlsNumber")
    resource: Optional[str] = None
    board_id: Optional[int] = Field(None, alias="boardId")

    # Status & classification
    status: Optional[Status] = None
    class_: Optional[str] = Field(None, alias="class")  # Can be "CondoProperty" etc
    type: Optional[ListingType] = None
    last_status: Optional[LastStatus] = Field(None, alias="lastStatus")

    # Pricing
    list_price: Optional[str] = Field(None, alias="listPrice")
    sold_price: Optional[str] = Field(None, alias="soldPrice")
    original_price: Optional[str] = Field(None, alias="originalPrice")

    # Dates
    list_date: Optional[str] = Field(None, alias="listDate")
    sold_date: Optional[str] = Field(None, alias="soldDate")
    updated_on: Optional[str] = Field(None, alias="updatedOn")
    days_on_market: Optional[str] = Field(None, alias="daysOnMarket")

    # Location
    address: Optional[Address] = None
    map: Optional[Map] = None

    # Details
    details: Optional[Details] = None
    condominium: Optional[Condominium] = None
    lot: Optional[Lot] = None
    taxes: Optional[Taxes] = None

    # Media
    images: Optional[List[str]] = Field(default_factory=list)
    photo_count: Optional[int] = Field(None, alias="photoCount")

    # Relationships
    agents: Optional[List[Agent]] = Field(default_factory=list)
    rooms: Optional[Dict[str, Room]] = None

    # Additional
    assignment: Optional[str] = None
    occupancy: Optional[str] = None
    timestamps: Optional[Timestamp] = None

    # Extensibility - capture unknown fields
    class Config:
        populate_by_name = True
        extra = "allow"  # Allow additional fields
        json_schema_extra = {
            "example": {
                "mlsNumber": "HMS1234567",
                "status": "A",
                "class": "CondoProperty",
                "type": "sale",
                "listPrice": "599000",
                "address": {
                    "city": "Toronto",
                    "state": "ON"
                }
            }
        }
```

#### 2.7 Search Models (`app/models/search.py`)

```python
from typing import Optional, List, Tuple
from pydantic import BaseModel, Field
from .enums import Status, Class, ListingType, SortBy, Operator, YesNo
from .listing import Listing

class SearchRequest(BaseModel):
    """
    Search request matching TypeScript SearchRequest.
    80+ optional parameters for filtering listings.
    """

    # Basic filters
    city: Optional[List[str]] = None
    state: Optional[str] = None
    zip: Optional[str] = None
    neighborhood: Optional[List[str]] = None
    area: Optional[str] = None

    # Classification
    class_: Optional[List[Class]] = Field(None, alias="class")
    status: Optional[List[Status]] = None
    type: Optional[List[ListingType]] = None
    last_status: Optional[List[str]] = Field(None, alias="lastStatus")

    # Price range
    min_price: Optional[float] = Field(None, alias="minPrice")
    max_price: Optional[float] = Field(None, alias="maxPrice")
    min_sold_price: Optional[float] = Field(None, alias="minSoldPrice")
    max_sold_price: Optional[float] = Field(None, alias="maxSoldPrice")

    # Bedrooms/Bathrooms
    min_bedrooms: Optional[int] = Field(None, alias="minBedrooms")
    max_bedrooms: Optional[int] = Field(None, alias="maxBedrooms")
    min_bedrooms_plus: Optional[int] = Field(None, alias="minBedroomsPlus")
    max_bedrooms_plus: Optional[int] = Field(None, alias="maxBedroomsPlus")
    min_baths: Optional[int] = Field(None, alias="minBaths")
    max_baths: Optional[int] = Field(None, alias="maxBaths")

    # Square footage
    min_sqft: Optional[int] = Field(None, alias="minSqft")
    max_sqft: Optional[int] = Field(None, alias="maxSqft")

    # Features
    property_type: Optional[List[str]] = Field(None, alias="propertyType")
    style: Optional[List[str]] = None
    garage: Optional[List[str]] = None
    basement: Optional[List[str]] = None
    swimming_pool: Optional[List[str]] = Field(None, alias="swimmingPool")
    heating: Optional[List[str]] = None

    # Dates
    min_list_date: Optional[str] = Field(None, alias="minListDate")
    max_list_date: Optional[str] = Field(None, alias="maxListDate")
    min_sold_date: Optional[str] = Field(None, alias="minSoldDate")
    max_sold_date: Optional[str] = Field(None, alias="maxSoldDate")

    # Geographic
    lat: Optional[str] = None
    long: Optional[str] = None
    radius: Optional[float] = None
    map: Optional[List[List[Tuple[float, float]]]] = None

    # Pagination & sorting
    page_num: Optional[int] = Field(1, alias="pageNum")
    results_per_page: Optional[int] = Field(25, alias="resultsPerPage")
    sort_by: Optional[SortBy] = Field(None, alias="sortBy")

    # Filters
    has_images: Optional[bool] = Field(None, alias="hasImages")
    has_agents: Optional[bool] = Field(None, alias="hasAgents")

    # Advanced
    aggregates: Optional[List[str]] = None
    fields: Optional[str] = None  # Comma-separated field names
    operator: Optional[Operator] = Field("AND", alias="operator")

    class Config:
        populate_by_name = True

class SearchStatistics(BaseModel):
    """Search result statistics"""
    available: Optional[Dict] = None
    new: Optional[Dict] = None
    closed: Optional[Dict] = None
    sold_price: Optional[Dict] = Field(None, alias="soldPrice")
    list_price: Optional[Dict] = Field(None, alias="listPrice")
    days_on_market: Optional[Dict] = Field(None, alias="daysOnMarket")

    class Config:
        populate_by_name = True

class SearchResponse(BaseModel):
    """Search response with pagination"""
    page: int
    num_pages: int = Field(..., alias="numPages")
    page_size: int = Field(..., alias="pageSize")
    count: int
    listings: List[Listing]
    statistics: Optional[SearchStatistics] = None

    class Config:
        populate_by_name = True
```

---

### Phase 3: FastAPI CRUD Endpoints

#### 3.1 Listings Router (`app/api/v1/listings.py`)

```python
from fastapi import APIRouter, Depends, HTTPException, Query, Path
from sqlalchemy.orm import Session
from typing import List, Optional

from app.models.listing import Listing
from app.crud import listing as crud_listing
from app.db.session import get_db

router = APIRouter(prefix="/listings", tags=["listings"])

@router.post("/", response_model=Listing, status_code=201)
async def create_listing(
    listing: Listing,
    db: Session = Depends(get_db)
):
    """
    Create a new listing.

    - **listing**: Full listing object with all fields
    """
    return crud_listing.create(db, obj_in=listing)

@router.get("/{mls_number}", response_model=Listing)
async def get_listing(
    mls_number: str = Path(..., description="MLS number"),
    board_id: Optional[int] = Query(None, description="Board ID"),
    fields: Optional[str] = Query(None, description="Comma-separated fields to return"),
    db: Session = Depends(get_db)
):
    """
    Get a specific listing by MLS number.

    - **mls_number**: The MLS number
    - **board_id**: Optional board ID for disambiguation
    - **fields**: Optional field filtering (e.g., "mlsNumber,listPrice,address")
    """
    listing = crud_listing.get_by_mls_number(
        db, mls_number=mls_number, board_id=board_id
    )
    if not listing:
        raise HTTPException(status_code=404, detail="Listing not found")

    # TODO: Implement field filtering if fields parameter provided
    return listing

@router.put("/{mls_number}", response_model=Listing)
async def update_listing(
    mls_number: str = Path(..., description="MLS number"),
    listing_update: Listing = ...,
    db: Session = Depends(get_db)
):
    """
    Update an existing listing.

    - **mls_number**: The MLS number to update
    - **listing_update**: Updated listing data (partial updates supported)
    """
    existing = crud_listing.get_by_mls_number(db, mls_number=mls_number)
    if not existing:
        raise HTTPException(status_code=404, detail="Listing not found")

    return crud_listing.update(db, db_obj=existing, obj_in=listing_update)

@router.delete("/{mls_number}", status_code=204)
async def delete_listing(
    mls_number: str = Path(..., description="MLS number"),
    board_id: Optional[int] = Query(None),
    db: Session = Depends(get_db)
):
    """
    Delete a listing.

    - **mls_number**: The MLS number to delete
    - **board_id**: Optional board ID
    """
    listing = crud_listing.get_by_mls_number(
        db, mls_number=mls_number, board_id=board_id
    )
    if not listing:
        raise HTTPException(status_code=404, detail="Listing not found")

    crud_listing.remove(db, id=listing.id)
    return None

@router.get("/", response_model=List[Listing])
async def list_listings(
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    status: Optional[str] = Query(None),
    class_: Optional[str] = Query(None, alias="class"),
    city: Optional[str] = Query(None),
    db: Session = Depends(get_db)
):
    """
    List listings with basic filtering.

    For advanced search, use /search endpoint.
    """
    filters = {}
    if status:
        filters["status"] = status
    if class_:
        filters["class"] = class_
    if city:
        filters["city"] = city

    return crud_listing.get_multi(db, skip=skip, limit=limit, filters=filters)
```

#### 3.2 Search Router (`app/api/v1/searches.py`)

```python
from fastapi import APIRouter, Depends, Body
from sqlalchemy.orm import Session

from app.models.search import SearchRequest, SearchResponse
from app.services.search_service import SearchService
from app.db.session import get_db

router = APIRouter(prefix="/search", tags=["search"])

@router.post("/", response_model=SearchResponse)
async def search_listings(
    search_params: SearchRequest = Body(...),
    db: Session = Depends(get_db)
):
    """
    Advanced search for listings.

    Supports 80+ query parameters matching the TypeScript SearchRequest interface:
    - Geographic filters (city, state, neighborhood, radius, map polygons)
    - Price ranges (min/max price, sold price)
    - Property features (bedrooms, bathrooms, sqft, garage, pool, etc.)
    - Date filters (list date, sold date, updated date)
    - Classification (status, class, type, property type)
    - Sorting and pagination
    - Field selection
    - Aggregations and statistics

    Example request body:
    ```json
    {
        "city": ["Toronto"],
        "minPrice": 500000,
        "maxPrice": 1000000,
        "minBedrooms": 2,
        "propertyType": ["Condo"],
        "status": ["A"],
        "hasImages": true,
        "sortBy": "listPriceAsc",
        "pageNum": 1,
        "resultsPerPage": 25
    }
    ```
    """
    search_service = SearchService(db)
    return search_service.search(search_params)

@router.post("/similar/{mls_number}", response_model=SearchResponse)
async def find_similar_listings(
    mls_number: str,
    radius: int = 5,
    list_price_range: int = 50000,
    db: Session = Depends(get_db)
):
    """
    Find similar listings based on a reference listing.

    - **mls_number**: Reference listing MLS number
    - **radius**: Search radius in km (default: 5)
    - **list_price_range**: Price range +/- (default: 50000)
    """
    search_service = SearchService(db)
    return search_service.find_similar(
        mls_number=mls_number,
        radius=radius,
        list_price_range=list_price_range
    )
```

#### 3.3 Analytics Router (`app/api/v1/analytics.py`)

```python
from fastapi import APIRouter, Depends, Query
from sqlalchemy.orm import Session
from typing import List, Optional

from app.services.analytics_service import AnalyticsService
from app.db.session import get_db

router = APIRouter(prefix="/analytics", tags=["analytics"])

@router.get("/report")
async def generate_report(
    cities: Optional[List[str]] = Query(None),
    status: Optional[List[str]] = Query(None),
    db: Session = Depends(get_db)
):
    """
    Generate analytics report similar to report.json.

    Returns:
    - Total properties count
    - Top cities with counts and average prices
    - Property type distribution
    - Status distribution
    - Price range statistics
    - Sample listings
    """
    analytics_service = AnalyticsService(db)
    return analytics_service.generate_report(cities=cities, status=status)

@router.get("/statistics")
async def get_statistics(
    city: Optional[str] = Query(None),
    property_type: Optional[str] = Query(None),
    db: Session = Depends(get_db)
):
    """
    Get statistical aggregations.

    Returns averages, medians, min/max for:
    - List prices
    - Sold prices
    - Days on market
    - Square footage
    """
    analytics_service = AnalyticsService(db)
    return analytics_service.calculate_statistics(
        city=city,
        property_type=property_type
    )

@router.get("/price-distribution")
async def get_price_distribution(
    city: Optional[str] = Query(None),
    bins: int = Query(6, ge=3, le=20),
    db: Session = Depends(get_db)
):
    """
    Get price distribution histogram.

    - **city**: Filter by city
    - **bins**: Number of price bins (default: 6)
    """
    analytics_service = AnalyticsService(db)
    return analytics_service.get_price_distribution(city=city, bins=bins)
```

---

### Phase 4: CRUD Operations

#### 4.1 Base CRUD Class (`app/crud/base.py`)

```python
from typing import Any, Dict, Generic, List, Optional, Type, TypeVar, Union
from fastapi.encoders import jsonable_encoder
from pydantic import BaseModel
from sqlalchemy.orm import Session

from app.db.base_class import Base

ModelType = TypeVar("ModelType", bound=Base)
CreateSchemaType = TypeVar("CreateSchemaType", bound=BaseModel)
UpdateSchemaType = TypeVar("UpdateSchemaType", bound=BaseModel)

class CRUDBase(Generic[ModelType, CreateSchemaType, UpdateSchemaType]):
    """
    Base class for CRUD operations.
    Provides generic create, read, update, delete methods.
    """

    def __init__(self, model: Type[ModelType]):
        self.model = model

    def get(self, db: Session, id: Any) -> Optional[ModelType]:
        """Get a single record by ID"""
        return db.query(self.model).filter(self.model.id == id).first()

    def get_multi(
        self,
        db: Session,
        *,
        skip: int = 0,
        limit: int = 100,
        filters: Optional[Dict[str, Any]] = None
    ) -> List[ModelType]:
        """Get multiple records with optional filters"""
        query = db.query(self.model)

        if filters:
            for key, value in filters.items():
                if hasattr(self.model, key):
                    query = query.filter(getattr(self.model, key) == value)

        return query.offset(skip).limit(limit).all()

    def create(self, db: Session, *, obj_in: CreateSchemaType) -> ModelType:
        """Create a new record"""
        obj_in_data = jsonable_encoder(obj_in)
        db_obj = self.model(**obj_in_data)
        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj

    def update(
        self,
        db: Session,
        *,
        db_obj: ModelType,
        obj_in: Union[UpdateSchemaType, Dict[str, Any]]
    ) -> ModelType:
        """Update an existing record"""
        obj_data = jsonable_encoder(db_obj)

        if isinstance(obj_in, dict):
            update_data = obj_in
        else:
            update_data = obj_in.dict(exclude_unset=True)

        for field in obj_data:
            if field in update_data:
                setattr(db_obj, field, update_data[field])

        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj

    def remove(self, db: Session, *, id: int) -> ModelType:
        """Delete a record"""
        obj = db.query(self.model).get(id)
        db.delete(obj)
        db.commit()
        return obj
```

#### 4.2 Listing CRUD (`app/crud/listing.py`)

```python
from typing import List, Optional
from sqlalchemy.orm import Session

from app.crud.base import CRUDBase
from app.models.listing import Listing
from app.schemas.listing import ListingDB

class CRUDListing(CRUDBase[ListingDB, Listing, Listing]):
    """CRUD operations for Listing"""

    def get_by_mls_number(
        self,
        db: Session,
        *,
        mls_number: str,
        board_id: Optional[int] = None
    ) -> Optional[ListingDB]:
        """Get listing by MLS number and optional board ID"""
        query = db.query(ListingDB).filter(ListingDB.mls_number == mls_number)

        if board_id:
            query = query.filter(ListingDB.board_id == board_id)

        return query.first()

    def get_by_city(
        self,
        db: Session,
        *,
        city: str,
        skip: int = 0,
        limit: int = 100
    ) -> List[ListingDB]:
        """Get listings by city"""
        return (
            db.query(ListingDB)
            .filter(ListingDB.address["city"].astext == city)
            .offset(skip)
            .limit(limit)
            .all()
        )

    def get_active_listings(
        self,
        db: Session,
        skip: int = 0,
        limit: int = 100
    ) -> List[ListingDB]:
        """Get all active listings"""
        return (
            db.query(ListingDB)
            .filter(ListingDB.status == "A")
            .offset(skip)
            .limit(limit)
            .all()
        )

    def bulk_import(self, db: Session, listings: List[Listing]) -> int:
        """
        Bulk import listings from JSON.
        Returns count of imported listings.
        """
        count = 0
        for listing_data in listings:
            # Check if listing exists
            existing = self.get_by_mls_number(
                db,
                mls_number=listing_data.mls_number,
                board_id=listing_data.board_id
            )

            if existing:
                # Update existing
                self.update(db, db_obj=existing, obj_in=listing_data)
            else:
                # Create new
                self.create(db, obj_in=listing_data)

            count += 1

        return count

listing = CRUDListing(ListingDB)
```

---

### Phase 5: Services Layer

#### 5.1 Search Service (`app/services/search_service.py`)

```python
from typing import List
from sqlalchemy.orm import Session
from sqlalchemy import and_, or_, func

from app.models.search import SearchRequest, SearchResponse, SearchStatistics
from app.models.listing import Listing
from app.schemas.listing import ListingDB

class SearchService:
    """Business logic for listing searches"""

    def __init__(self, db: Session):
        self.db = db

    def search(self, params: SearchRequest) -> SearchResponse:
        """
        Execute advanced search with all parameters.
        Matches TypeScript SearchRequest interface.
        """
        query = self.db.query(ListingDB)

        # Apply filters
        query = self._apply_filters(query, params)

        # Count total results
        total_count = query.count()

        # Apply sorting
        query = self._apply_sorting(query, params.sort_by)

        # Apply pagination
        skip = (params.page_num - 1) * params.results_per_page
        listings = query.offset(skip).limit(params.results_per_page).all()

        # Calculate statistics if requested
        statistics = None
        if params.aggregates or params.statistics:
            statistics = self._calculate_statistics(query, params)

        return SearchResponse(
            page=params.page_num,
            numPages=(total_count + params.results_per_page - 1) // params.results_per_page,
            pageSize=params.results_per_page,
            count=total_count,
            listings=[self._to_listing_model(db_listing) for db_listing in listings],
            statistics=statistics
        )

    def _apply_filters(self, query, params: SearchRequest):
        """Apply all search filters"""
        filters = []

        # Geographic filters
        if params.city:
            filters.append(ListingDB.address["city"].astext.in_(params.city))
        if params.state:
            filters.append(ListingDB.address["state"].astext == params.state)
        if params.neighborhood:
            filters.append(ListingDB.address["neighborhood"].astext.in_(params.neighborhood))

        # Status filters
        if params.status:
            filters.append(ListingDB.status.in_(params.status))
        if params.class_:
            filters.append(ListingDB.class_.in_(params.class_))
        if params.type:
            filters.append(ListingDB.type.in_(params.type))

        # Price filters
        if params.min_price:
            filters.append(func.cast(ListingDB.list_price, Float) >= params.min_price)
        if params.max_price:
            filters.append(func.cast(ListingDB.list_price, Float) <= params.max_price)

        # Bedroom filters
        if params.min_bedrooms:
            filters.append(
                func.cast(ListingDB.details["numBedrooms"].astext, Integer) >= params.min_bedrooms
            )
        if params.max_bedrooms:
            filters.append(
                func.cast(ListingDB.details["numBedrooms"].astext, Integer) <= params.max_bedrooms
            )

        # Bathroom filters
        if params.min_baths:
            filters.append(
                func.cast(ListingDB.details["numBathrooms"].astext, Integer) >= params.min_baths
            )

        # Square footage
        if params.min_sqft:
            filters.append(
                func.cast(ListingDB.details["sqft"].astext, Integer) >= params.min_sqft
            )
        if params.max_sqft:
            filters.append(
                func.cast(ListingDB.details["sqft"].astext, Integer) <= params.max_sqft
            )

        # Feature filters
        if params.property_type:
            filters.append(
                ListingDB.details["propertyType"].astext.in_(params.property_type)
            )

        # Image filter
        if params.has_images:
            filters.append(ListingDB.photo_count > 0)

        # Date filters
        if params.min_list_date:
            filters.append(ListingDB.list_date >= params.min_list_date)
        if params.max_list_date:
            filters.append(ListingDB.list_date <= params.max_list_date)

        # Combine filters based on operator
        if filters:
            if params.operator == "OR":
                query = query.filter(or_(*filters))
            else:
                query = query.filter(and_(*filters))

        return query

    def _apply_sorting(self, query, sort_by: str):
        """Apply sorting"""
        if not sort_by:
            return query.order_by(ListingDB.updated_on.desc())

        sort_mapping = {
            "listPriceAsc": ListingDB.list_price.asc(),
            "listPriceDesc": ListingDB.list_price.desc(),
            "updatedOnDesc": ListingDB.updated_on.desc(),
            "updatedOnAsc": ListingDB.updated_on.asc(),
            "soldDateDesc": ListingDB.sold_date.desc(),
            "soldDateAsc": ListingDB.sold_date.asc(),
            # Add more sort options
        }

        return query.order_by(sort_mapping.get(sort_by, ListingDB.updated_on.desc()))

    def _calculate_statistics(self, query, params: SearchRequest) -> SearchStatistics:
        """Calculate search statistics"""
        # TODO: Implement aggregations
        return SearchStatistics()

    def _to_listing_model(self, db_listing: ListingDB) -> Listing:
        """Convert DB model to Pydantic model"""
        return Listing.from_orm(db_listing)

    def find_similar(
        self,
        mls_number: str,
        radius: int,
        list_price_range: int
    ) -> SearchResponse:
        """Find similar listings"""
        # Get reference listing
        ref_listing = self.db.query(ListingDB).filter(
            ListingDB.mls_number == mls_number
        ).first()

        if not ref_listing:
            return SearchResponse(
                page=1, numPages=0, pageSize=0, count=0, listings=[]
            )

        # Build search params based on reference listing
        search_params = SearchRequest(
            city=[ref_listing.address.get("city")] if ref_listing.address else None,
            minPrice=float(ref_listing.list_price) - list_price_range,
            maxPrice=float(ref_listing.list_price) + list_price_range,
            propertyType=[ref_listing.details.get("propertyType")] if ref_listing.details else None,
            status=["A"],
            # Add more similarity criteria
        )

        return self.search(search_params)
```

#### 5.2 Import Service (`app/services/import_service.py`)

```python
import json
from typing import List
from sqlalchemy.orm import Session

from app.models.listing import Listing
from app.crud.listing import listing as crud_listing

class ImportService:
    """Service for importing JSON data"""

    @staticmethod
    def import_listings_from_json(
        db: Session,
        json_file_path: str
    ) -> dict:
        """
        Import listings from JSON file (e.g., toronto_listings.json).
        Returns import statistics.
        """
        with open(json_file_path, 'r') as f:
            data = json.load(f)

        # Handle both array format and object with listings key
        if isinstance(data, dict) and 'listings' in data:
            listings_data = data['listings']
        else:
            listings_data = data

        # Convert to Pydantic models
        listings = []
        errors = []

        for idx, item in enumerate(listings_data):
            try:
                listing = Listing(**item)
                listings.append(listing)
            except Exception as e:
                errors.append({
                    'index': idx,
                    'mls_number': item.get('mlsNumber', 'unknown'),
                    'error': str(e)
                })

        # Bulk import
        imported_count = crud_listing.bulk_import(db, listings)

        return {
            'total_processed': len(listings_data),
            'imported': imported_count,
            'errors': len(errors),
            'error_details': errors[:10]  # First 10 errors
        }
```

---

### Phase 6: Database Setup

#### 6.1 Database Configuration (`app/db/session.py`)

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

from app.config import settings

engine = create_engine(
    settings.DATABASE_URL,
    pool_pre_ping=True,
    json_serializer=lambda obj: json.dumps(obj, ensure_ascii=False)
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()

def get_db():
    """Dependency for getting database session"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

#### 6.2 Database Schema (`app/schemas/listing.py`)

```python
from sqlalchemy import Column, Integer, String, DateTime, JSON, Text
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.sql import func

from app.db.base_class import Base

class ListingDB(Base):
    """
    SQLAlchemy model for Listing.
    Uses JSONB for nested structures to maintain flexibility.
    """
    __tablename__ = "listings"

    # Primary key
    id = Column(Integer, primary_key=True, index=True)

    # Core identifiers (indexed for performance)
    mls_number = Column(String(50), index=True, nullable=False)
    board_id = Column(Integer, index=True)
    resource = Column(String(100))

    # Status & classification (indexed)
    status = Column(String(10), index=True)
    class_ = Column(String(50), index=True)
    type = Column(String(20), index=True)
    last_status = Column(String(10))

    # Pricing (stored as strings to match API, indexed)
    list_price = Column(String(20), index=True)
    sold_price = Column(String(20))
    original_price = Column(String(20))

    # Dates (indexed)
    list_date = Column(DateTime, index=True)
    sold_date = Column(DateTime)
    updated_on = Column(DateTime, index=True, server_default=func.now())

    # Metrics
    days_on_market = Column(String(10))
    photo_count = Column(Integer, default=0)

    # Nested structures (JSONB for PostgreSQL, JSON for others)
    address = Column(JSONB)  # or JSON for MySQL/SQLite
    map = Column(JSONB)
    details = Column(JSONB)
    condominium = Column(JSONB)
    lot = Column(JSONB)
    taxes = Column(JSONB)
    timestamps = Column(JSONB)

    # Arrays
    images = Column(JSONB)  # List of image URLs
    agents = Column(JSONB)  # List of agent objects
    rooms = Column(JSONB)  # Dictionary of room objects

    # Other fields
    assignment = Column(String(100))
    occupancy = Column(String(100))

    # Extensibility - catch-all for unknown fields
    extra_data = Column(JSONB)

    # Timestamps
    created_at = Column(DateTime, server_default=func.now())
    updated_at = Column(DateTime, server_default=func.now(), onupdate=func.now())

    def __repr__(self):
        return f"<Listing {self.mls_number}>"
```

---

### Phase 7: Scripts & Utilities

#### 7.1 Import Script (`scripts/import_listings.py`)

```python
#!/usr/bin/env python3
"""
Import listings from toronto_listings.json into database.
"""

import sys
from pathlib import Path

# Add parent directory to path
sys.path.insert(0, str(Path(__file__).parent.parent))

from sqlalchemy.orm import Session
from app.db.session import SessionLocal, engine
from app.db.base_class import Base
from app.services.import_service import ImportService

def main():
    # Create tables
    Base.metadata.create_all(bind=engine)

    # Import listings
    db = SessionLocal()
    try:
        result = ImportService.import_listings_from_json(
            db,
            json_file_path="../toronto_listings.json"
        )

        print(f"Import complete!")
        print(f"Total processed: {result['total_processed']}")
        print(f"Successfully imported: {result['imported']}")
        print(f"Errors: {result['errors']}")

        if result['error_details']:
            print("\nFirst few errors:")
            for error in result['error_details']:
                print(f"  - Index {error['index']} (MLS: {error['mls_number']}): {error['error']}")

    finally:
        db.close()

if __name__ == "__main__":
    main()
```

#### 7.2 Analytics Script (`scripts/generate_report.py`)

```python
#!/usr/bin/env python3
"""
Generate analytics report similar to report.json.
"""

import sys
from pathlib import Path
import json

sys.path.insert(0, str(Path(__file__).parent.parent))

from app.db.session import SessionLocal
from app.services.analytics_service import AnalyticsService

def main():
    db = SessionLocal()
    try:
        service = AnalyticsService(db)
        report = service.generate_report()

        # Save to file
        with open("generated_report.json", "w") as f:
            json.dump(report, f, indent=2)

        print("Report generated: generated_report.json")

    finally:
        db.close()

if __name__ == "__main__":
    main()
```

---

### Phase 8: Testing

#### 8.1 Test Configuration (`tests/conftest.py`)

```python
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from app.main import app
from app.db.session import get_db
from app.db.base_class import Base

# Test database
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False}
)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

@pytest.fixture
def db():
    Base.metadata.create_all(bind=engine)
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()
        Base.metadata.drop_all(bind=engine)

@pytest.fixture
def client(db):
    def override_get_db():
        try:
            yield db
        finally:
            pass

    app.dependency_overrides[get_db] = override_get_db
    return TestClient(app)
```

#### 8.2 API Tests (`tests/test_api/test_listings.py`)

```python
def test_create_listing(client):
    """Test creating a new listing"""
    listing_data = {
        "mlsNumber": "TEST123",
        "status": "A",
        "class": "residential",
        "type": "sale",
        "listPrice": "599000",
        "address": {
            "city": "Toronto",
            "state": "ON"
        }
    }

    response = client.post("/api/v1/listings/", json=listing_data)
    assert response.status_code == 201
    data = response.json()
    assert data["mlsNumber"] == "TEST123"
    assert data["status"] == "A"

def test_get_listing(client):
    """Test retrieving a listing"""
    # Create listing first
    client.post("/api/v1/listings/", json={"mlsNumber": "TEST123"})

    # Get listing
    response = client.get("/api/v1/listings/TEST123")
    assert response.status_code == 200
    data = response.json()
    assert data["mlsNumber"] == "TEST123"

def test_search_listings(client):
    """Test search functionality"""
    search_params = {
        "city": ["Toronto"],
        "minPrice": 500000,
        "maxPrice": 1000000,
        "status": ["A"]
    }

    response = client.post("/api/v1/search/", json=search_params)
    assert response.status_code == 200
    data = response.json()
    assert "listings" in data
    assert "count" in data
    assert "page" in data
```

---

## 📋 Implementation Checklist

### Setup (Week 1)
- [ ] Create project directory structure
- [ ] Set up virtual environment (`python -m venv venv`)
- [ ] Install dependencies (`fastapi`, `sqlalchemy`, `pydantic`, `uvicorn`, `alembic`, `psycopg2`)
- [ ] Configure environment variables (`.env` file)
- [ ] Set up database (PostgreSQL recommended for JSONB support)

### Models (Week 1-2)
- [ ] Create all enumerations (`enums.py`)
- [ ] Create Address model
- [ ] Create Details model with Bathroom nested type
- [ ] Create Condominium model with Fees nested type
- [ ] Create Agent model with Photo, Brokerage nested types
- [ ] Create Room, Lot, Taxes, Timestamp models
- [ ] Create main Listing model with all relationships
- [ ] Create SearchRequest/Response models
- [ ] Add validation and examples to all models

### Database (Week 2)
- [ ] Create SQLAlchemy base configuration
- [ ] Create Listing database schema with JSONB columns
- [ ] Create indexes on frequently queried fields
- [ ] Set up Alembic for migrations
- [ ] Create initial migration

### CRUD & Services (Week 2-3)
- [ ] Implement base CRUD class
- [ ] Implement Listing CRUD operations
- [ ] Implement SearchService with all filters
- [ ] Implement AnalyticsService for report generation
- [ ] Implement ImportService for JSON imports
- [ ] Add bulk operations support

### API Routes (Week 3)
- [ ] Create listings router (CRUD endpoints)
- [ ] Create search router (advanced search)
- [ ] Create analytics router (reports & statistics)
- [ ] Create locations router (geographic hierarchy)
- [ ] Add OpenAPI documentation examples
- [ ] Add request/response validation

### Import & Migration (Week 3)
- [ ] Create import script for `toronto_listings.json`
- [ ] Test import with sample data
- [ ] Create analytics generation script
- [ ] Validate imported data against TypeScript types

### Testing (Week 4)
- [ ] Set up test database configuration
- [ ] Write model validation tests
- [ ] Write CRUD operation tests
- [ ] Write API endpoint tests
- [ ] Write search functionality tests
- [ ] Write analytics tests
- [ ] Achieve >80% code coverage

### Documentation & Deployment (Week 4)
- [ ] Write comprehensive README
- [ ] Document API endpoints (Swagger/OpenAPI)
- [ ] Create deployment guide
- [ ] Set up Docker containerization
- [ ] Configure production database
- [ ] Deploy to production environment

---

## 🎁 Bonus Features

### Advanced Search
- [ ] Implement full-text search on descriptions
- [ ] Add geographic polygon search (map parameter)
- [ ] Implement radius-based search with Haversine formula
- [ ] Add fuzzy matching for street names

### Performance Optimization
- [ ] Add Redis caching for frequent searches
- [ ] Implement database query optimization
- [ ] Add pagination cursor support
- [ ] Create database views for complex queries

### Real-time Features
- [ ] WebSocket support for live listing updates
- [ ] Saved search notifications
- [ ] Price change alerts

### Integration
- [ ] Create TypeScript SDK generator from Pydantic models
- [ ] Add GraphQL API layer
- [ ] Implement webhook system for listing changes

---

## 📚 Dependencies

```txt
# requirements.txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
pydantic-settings==2.1.0
sqlalchemy==2.0.23
alembic==1.12.1
psycopg2-binary==2.9.9  # PostgreSQL
python-dotenv==1.0.0
python-multipart==0.0.6
email-validator==2.1.0

# Testing
pytest==7.4.3
pytest-cov==4.1.0
httpx==0.25.2

# Development
black==23.11.0
ruff==0.1.6
mypy==1.7.1
```

---

## 🎯 Success Criteria

1. **Type Safety**: All TypeScript types accurately translated to Pydantic models
2. **Data Integrity**: Successfully import all 42,749 listings from `toronto_listings.json`
3. **API Completeness**: All CRUD operations functional for all entity types
4. **Search Functionality**: Advanced search supporting 80+ parameters
5. **Performance**: Search queries return in <200ms for simple filters
6. **Test Coverage**: >80% code coverage with passing tests
7. **Documentation**: Complete API documentation with examples
8. **Modularity**: Code can be easily integrated into other projects

---

## 📖 Next Steps

1. Review this plan and adjust based on requirements
2. Set up development environment
3. Start with Phase 1: Project structure and base models
4. Implement incrementally, testing each phase
5. Import sample data and validate
6. Deploy to staging environment for testing
7. Production deployment

---

**Notes**:
- Use PostgreSQL for production (JSONB support is critical)
- SQLite works for development/testing
- Consider using Docker for consistent environments
- All JSON fields maintain flexibility while providing type safety
- The implementation preserves the TypeScript API contract
