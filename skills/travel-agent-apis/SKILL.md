---
name: travel-agent-apis
description: APIs for building an AI travel agent - accommodation (Booking.com, vacation rentals), routes, places, and how to combine them
---

# Travel Agent APIs Skill

This skill covers the domain knowledge needed to build an AI travel agent, including accommodation booking (hotels AND vacation rentals), route planning, and place discovery.

## The Accommodation Problem

**Airbnb has NO public API** (shut down 2018). Your options:

| Source                | API Type                | What You Get                         | Vacation Rentals?     |
| --------------------- | ----------------------- | ------------------------------------ | --------------------- |
| **Booking.com**       | Affiliate (recommended) | Hotels + apartments + vacation homes | ✅ Yes                |
| **Travelpayouts**     | Affiliate aggregator    | Access to 90+ travel brands          | ✅ Yes (via partners) |
| **Google Places**     | Public                  | Basic hotel info only                | ❌ No                 |
| **Amadeus**           | Public (GDS)            | Major hotel chains                   | ❌ Limited            |
| **RapidAPI scrapers** | Unofficial              | Airbnb data (risky)                  | ⚠️ Unreliable         |

**Recommendation**: Use **Booking.com Demand API** as primary (has vacation rentals!) + Google Places for attractions/restaurants.

---

## Booking.com Demand API (Accommodations)

This is your PRIMARY accommodation source. Booking.com has:

- Hotels, hostels, B&Bs
- **Apartments and vacation rentals** (their "Homes & Apartments" category)
- Real-time availability and pricing
- You earn commission on bookings

### Getting Access

1. Register as affiliate: https://secure.booking.com/affiliate-program/register.html
2. Apply for API access through Partner Centre
3. Receive API credentials (username/password for Basic Auth)

### Base URL

```
https://distribution-xml.booking.com/2.10/json/
```

### Authentication

HTTP Basic Authentication with your affiliate credentials.

### Key Endpoints

#### 1. Search Hotels/Accommodations

```
GET /hotels?city_ids=-2140479&checkin=2026-03-15&checkout=2026-03-18&guest_qty=2
```

Parameters:

- `city_ids` - Booking.com city ID (get from /cities endpoint)
- `checkin` / `checkout` - Dates (YYYY-MM-DD)
- `guest_qty` - Number of guests
- `room_qty` - Number of rooms
- `rows` - Results per page (max 1000)
- `accommodation_types` - Filter: 201=Apartments, 204=Hotels, 208=Vacation Homes

#### 2. Get City IDs

```
GET /cities?name=Paris&country=fr
```

#### 3. Get Hotel Details

```
GET /hotels?hotel_ids=10004
```

Returns: photos, facilities, description, location, reviews

#### 4. Get Room Availability & Pricing

```
GET /blockAvailability?hotel_ids=10004&checkin=2026-03-15&checkout=2026-03-18
```

Returns: room types, prices, cancellation policies

### Accommodation Types (for filtering)

```
201 = Apartments
204 = Hotels
205 = Hostels
206 = Guest houses
208 = Holiday homes / Vacation rentals
212 = Bed and breakfasts
213 = Villas
219 = Motels
220 = Resorts
```

### Sample Response

```json
{
  "result": [
    {
      "hotel_id": 10004,
      "name": "Grand Plaza Hotel",
      "address": "123 Main Street",
      "city": "Paris",
      "country_code": "fr",
      "location": { "latitude": 48.8566, "longitude": 2.3522 },
      "review_score": 8.5,
      "review_nr": 1250,
      "class": 4,
      "accommodation_type_id": 204,
      "currency_code": "EUR",
      "min_rate": 150.0,
      "max_rate": 450.0,
      "photos": [{ "url_original": "..." }]
    }
  ]
}
```

### Booking Flow

```
1. User: "Find apartments in Barcelona for March 15-18"

2. Get city_id:
   GET /cities?name=Barcelona&country=es
   → city_id: -372490

3. Search accommodations:
   GET /hotels?city_ids=-372490&checkin=2026-03-15&checkout=2026-03-18
              &accommodation_types=201,208,213
   → List of apartments, vacation homes, villas

4. Get availability/pricing:
   GET /blockAvailability?hotel_ids=123,456,789&checkin=...
   → Real prices and room options

5. Present to user with booking links (affiliate URLs)
```

---

## Google APIs (Routes + Places)

Use Google for:

- **Routes**: Directions, travel times, transit info
- **Places**: Restaurants, attractions, airports (NOT primary accommodation)

## Overview

| API                  | Purpose                              | Use Case                                         |
| -------------------- | ------------------------------------ | ------------------------------------------------ |
| **Places API (New)** | Find and get details about locations | Restaurants, attractions, airports               |
| **Routes API**       | Calculate routes between locations   | Directions, travel times, multi-stop itineraries |

## Places API (New)

Base URL: `https://places.googleapis.com/v1/`

### Key Endpoints

#### 1. Text Search - Find places by query

```
POST https://places.googleapis.com/v1/places:searchText
```

Use for: "Find hotels near Times Square", "restaurants in Paris"

```json
{
  "textQuery": "hotels near Times Square New York",
  "maxResultCount": 10
}
```

#### 2. Nearby Search - Find places near a location

```
POST https://places.googleapis.com/v1/places:searchNearby
```

Use for: Finding amenities near a hotel, attractions near coordinates

```json
{
  "locationRestriction": {
    "circle": {
      "center": { "latitude": 40.758, "longitude": -73.985 },
      "radius": 1000.0
    }
  },
  "includedTypes": ["restaurant", "tourist_attraction"]
}
```

#### 3. Place Details - Get full info about a place

```
GET https://places.googleapis.com/v1/places/{placeId}
```

Use for: Getting hotel contact info, reviews, photos, hours

#### 4. Autocomplete - Suggest places as user types

```
POST https://places.googleapis.com/v1/places:autocomplete
```

Use for: Helping users find destinations, airports, hotels

### Important Place Types for Travel

```
Accommodation: lodging, hotel, motel, bed_and_breakfast, hostel, resort_hotel
Food: restaurant, cafe, bar, bakery, meal_takeaway
Transport: airport, train_station, bus_station, car_rental, taxi_stand
Attractions: tourist_attraction, museum, art_gallery, amusement_park, zoo
Services: atm, bank, pharmacy, hospital, gas_station
```

### Fields to Request (use field masks to control cost)

Essential fields for travel:

- `displayName` - Place name
- `formattedAddress` - Full address
- `location` - Lat/lng coordinates
- `rating` - User rating (1-5)
- `priceLevel` - Cost indicator
- `types` - Place categories
- `photos` - Place images
- `regularOpeningHours` - Operating hours
- `internationalPhoneNumber` - Contact
- `websiteUri` - Website
- `reviews` - User reviews

## Routes API

Base URL: `https://routes.googleapis.com/`

### Key Endpoints

#### 1. Compute Routes - Get directions

```
POST https://routes.googleapis.com/directions/v2:computeRoutes
```

```json
{
  "origin": {
    "address": "JFK Airport, New York"
  },
  "destination": {
    "address": "Times Square, New York"
  },
  "travelMode": "DRIVE",
  "computeAlternativeRoutes": true
}
```

#### 2. Compute Route Matrix - Multiple origins/destinations

```
POST https://routes.googleapis.com/distanceMatrix/v2:computeRouteMatrix
```

Use for: Comparing travel times between multiple hotels and attractions

### Travel Modes

```
DRIVE - Car/taxi
TRANSIT - Public transportation (includes transfers)
WALK - Walking directions
BICYCLE - Bike routes
TWO_WHEELER - Motorcycle/scooter
```

### Route Options for Travel Agents

```json
{
  "routeModifiers": {
    "avoidTolls": false,
    "avoidHighways": false,
    "avoidFerries": false
  },
  "departureTime": "2026-03-15T09:00:00Z",
  "routingPreference": "TRAFFIC_AWARE"
}
```

### Response Fields to Request

```
routes.duration - Total travel time
routes.distanceMeters - Total distance
routes.polyline - Route line for display
routes.legs - Segments between waypoints
routes.legs.steps - Turn-by-turn directions
routes.travelAdvisory - Toll info, restrictions
```

## Building Travel Agent Tools

### Tool: search_hotels

```python
def search_hotels(location: str, check_in: str, check_out: str, guests: int):
    """
    Search for hotels near a location.

    Parameters:
    - location: City, address, or landmark
    - check_in: Check-in date (YYYY-MM-DD)
    - check_out: Check-out date (YYYY-MM-DD)
    - guests: Number of guests
    """
    # Use Places Text Search
    # Filter by type: "lodging"
    # Return: name, rating, price_level, address, photos
```

### Tool: search_attractions

```python
def search_attractions(location: str, category: str = None):
    """
    Find tourist attractions near a location.

    Parameters:
    - location: City or coordinates
    - category: Optional filter (museum, park, etc.)
    """
    # Use Places Nearby Search
    # Filter by type: "tourist_attraction"
    # Include reviews and photos
```

### Tool: get_route

```python
def get_route(origin: str, destination: str, mode: str = "DRIVE"):
    """
    Get directions between two places.

    Parameters:
    - origin: Starting location
    - destination: End location
    - mode: DRIVE, TRANSIT, WALK, BICYCLE
    """
    # Use Routes Compute Routes
    # Return: duration, distance, steps
```

### Tool: plan_day_itinerary

```python
def plan_day_itinerary(hotel: str, attractions: list[str]):
    """
    Optimize visiting order for multiple attractions.

    Parameters:
    - hotel: Starting/ending point
    - attractions: List of places to visit
    """
    # Use Routes with optimizeWaypointOrder: true
    # Return optimized order with travel times
```

## Common Travel Agent Workflows

### 1. Hotel Recommendation Flow

```
User: "Find me a hotel in Paris near the Eiffel Tower"

1. Text Search: "hotels near Eiffel Tower Paris"
   → Get list of hotels with ratings, prices

2. For top 3 results, get Place Details
   → Reviews, photos, amenities

3. For each hotel, compute route to Eiffel Tower
   → Walking time to attraction

4. Present options sorted by rating/distance
```

### 2. Day Trip Planning Flow

```
User: "Plan a day visiting museums in Rome"

1. Nearby Search: museums in Rome
   → Get list with ratings, hours

2. Filter by opening hours for requested date

3. Compute Route Matrix between all museums
   → Get travel times between each pair

4. Optimize route order
   → Minimize total travel time

5. Generate itinerary with times
```

### 3. Multi-City Trip Flow

```
User: "Plan 5 days: Paris, Amsterdam, Brussels"

1. For each city:
   - Search hotels
   - Search top attractions

2. Between cities:
   - Compute routes (train/drive options)

3. Allocate days based on:
   - Travel time between cities
   - Number of attractions

4. Generate day-by-day itinerary
```

## API Cost Optimization

### Use Field Masks

Only request fields you need - you're billed per field category.

```
# Cheap: Basic info only
fields=places.displayName,places.formattedAddress

# More expensive: Includes contact, atmosphere
fields=places.displayName,places.rating,places.reviews
```

### Session Tokens for Autocomplete

Group autocomplete requests to reduce billing:

```
# Generate session token
# Use same token for all autocomplete requests
# Token expires after Place Details call
```

### Cache Place IDs

Place IDs don't change - store them to avoid repeated searches.

## Error Handling

### Common Errors

| Error              | Cause                 | Solution                  |
| ------------------ | --------------------- | ------------------------- |
| `ZERO_RESULTS`     | No places match query | Broaden search area/terms |
| `OVER_QUERY_LIMIT` | Too many requests     | Add delays, check quota   |
| `REQUEST_DENIED`   | API key issue         | Check key restrictions    |
| `INVALID_REQUEST`  | Bad parameters        | Validate inputs           |

### Graceful Degradation

```
if no_hotels_found:
    expand_search_radius()
    suggest_nearby_areas()

if route_unavailable:
    try_alternative_mode()
    suggest_nearby_starting_point()
```

## Building the Full Travel Agent

### Tool: search_accommodations (Booking.com)

```python
def search_accommodations(
    city: str,
    country: str,
    check_in: str,
    check_out: str,
    guests: int,
    accommodation_type: str = "any"  # hotel, apartment, villa, vacation_home
):
    """
    Search for accommodations including vacation rentals.

    Parameters:
    - city: City name
    - country: Country code (us, fr, es, etc.)
    - check_in: Check-in date (YYYY-MM-DD)
    - check_out: Check-out date (YYYY-MM-DD)
    - guests: Number of guests
    - accommodation_type: Filter by type
    """
    # 1. Get city_id from Booking.com
    # 2. Search with accommodation_type filter
    # 3. Get availability/pricing
    # 4. Return: name, type, rating, price, photos, booking_url
```

### Tool: search_restaurants (Google Places)

```python
def search_restaurants(location: str, cuisine: str = None, price_level: int = None):
    """
    Find restaurants near a location.
    Uses Google Places API.
    """
```

### Tool: search_attractions (Google Places)

```python
def search_attractions(location: str, category: str = None):
    """
    Find tourist attractions near a location.
    Uses Google Places API.
    """
```

### Tool: get_route (Google Routes)

```python
def get_route(origin: str, destination: str, mode: str = "DRIVE"):
    """
    Get directions between two places.
    Uses Google Routes API.
    """
```

---

## Combined Workflow Example

```
User: "Find me a nice apartment in Paris near the Louvre for March 15-18,
       2 guests. Also suggest some restaurants nearby."

1. ACCOMMODATION (Booking.com):
   - Get Paris city_id
   - Search: accommodation_types=201 (apartments)
   - Filter by location near Louvre (lat/lng)
   - Get pricing for dates
   → Return top 5 apartments with prices

2. RESTAURANTS (Google Places):
   - Nearby Search around Louvre coordinates
   - Type: restaurant
   → Return top restaurants with ratings

3. ROUTE (Google Routes):
   - For each apartment, calculate walk time to Louvre
   → Add "5 min walk to Louvre" to apartment listings

4. PRESENT:
   "Here are 5 apartments near the Louvre:
    1. Charming Studio - €120/night (8.9★) - 3 min walk to Louvre
    2. Modern Loft - €180/night (9.2★) - 7 min walk to Louvre
    ...

    Nearby restaurants:
    1. Le Fumoir (French) - €€€ - 4.5★
    2. Café Marly (French) - €€€€ - 4.3★
    ..."
```

---

## Affiliate Revenue Model

When users book through your links:

| Platform               | Commission                                    |
| ---------------------- | --------------------------------------------- |
| Booking.com            | 25-40% of their commission (varies by volume) |
| Travelpayouts partners | Varies by brand                               |

Your travel agent can actually **make money** while helping users!

---

## Alternative: Travelpayouts (Multi-Brand)

If you want access to multiple brands through ONE platform:

Travelpayouts aggregates:

- Booking.com
- Hostelworld
- 12Go (Asia transport)
- GetYourGuide (tours)
- Viator (activities)
- Rentalcars.com
- And 90+ more

Single dashboard, single payout. Good for expanding beyond accommodation later.
