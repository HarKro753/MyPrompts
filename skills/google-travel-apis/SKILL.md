---
name: google-travel-apis
description: How to use Google Routes API and Places API for travel agent functionality - finding places, planning routes, and building itineraries. Use when building travel agents, planning routes, finding hotels/restaurants, or when the user mentions Google APIs, directions, or place searches.
---

# Google Travel APIs Skill

This skill covers the domain knowledge needed to build an AI travel agent using Google's Routes API and Places API.

## Overview

For an MVP travel agent, you need two core APIs:

| API                  | Purpose                              | Use Case                                         |
| -------------------- | ------------------------------------ | ------------------------------------------------ |
| **Places API (New)** | Find and get details about locations | Hotels, restaurants, attractions, airports       |
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

## Sample Travel Agent System Prompt Addition

```markdown
## Travel Planning

When helping with travel:

1. Ask for destination, dates, and preferences
2. Use search_places to find accommodations
3. Use search_places to find attractions
4. Use get_route to calculate travel times
5. Present options with ratings, prices, and travel times
6. Remember user preferences in memory (hotel chains, dietary needs)

Always provide:

- Estimated costs where available
- Travel times between locations
- Operating hours for attractions
- Alternative options at different price points
```
