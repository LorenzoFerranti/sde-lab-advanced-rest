# Advanced REST lab

A NestJS API server that combines three local resources stored in MySQL (cities, flights, hotels) with two external services: [Open-Meteo](https://open-meteo.com/en/docs) for the weather and [Frankfurter](https://frankfurter.dev/) for exchange rates.

## Structure

```text
sde-lab-03/
├── .env                              lab credentials
├── README.md                         lab description
├── ASSIGNMENT.md                     assignment for the students
├── docker-compose.yml                backend, MySQL and Redis
├── start-lab.sh                      starts Docker and seeds the database (Linux/macOS)
├── start-lab.ps1                     same as start-lab.sh, for Windows PowerShell
└── nest-backend/                     Node.js environment for the lab
    ├── Dockerfile                    
    ├── package.json
    ├── tsconfig.json
    ├── public/                       web interface to try the APIs
    │   ├── index.html
    │   ├── style.css
    │   ├── app.js
    │   ├── plane.svg
    │   └── sun.svg
    ├── seed/                         demo data and seeding script
    │   ├── seed.ts                   entry point (npm run seed)
    │   ├── run-seed.ts               clears the database and inserts the data
    │   └── seed-data.ts              cities, flights and hotels
    └── src/
        ├── main.ts                   starts the app and serves public/
        ├── app.module.ts             database and modules configuration
        ├── controller/
        │   └── travel.controller.ts  HTTP routes (one service call each)
        ├── service/
        │   ├── travel.service.ts     all the logic: database (TypeORM), validation, exercises
        │   └── external.service.ts   Open-Meteo, Frankfurter and the Redis cache
        ├── model/
        │   ├── city.entity.ts
        │   ├── flight.entity.ts
        │   └── hotel.entity.ts
        └── module/
            └── travel.module.ts      NestJS module
```

The database contains only `City`, `Flight` and `Hotel`. Flights and hotels refer to existing cities. **The price of a flight is in EUR** (`priceEur`); **the price of a hotel is in the currency of its city** (`city.currency`: London in GBP, Tokyo in JPY, ...). The weather is cached in Redis for 10 minutes. 

## Setup

You only need Docker and Docker Compose. Node.js and npm are included in the backend container.

The `.env` file with the lab credentials is included in the repository. From the project root, start the environment and seed the database:

```bash
./start-lab.sh
```

The script runs these commands, which can also be used separately:

```bash
docker compose down -v
docker compose up -d --build --renew-anon-volumes
docker compose exec backend npm run seed
docker compose logs -f backend
```

The backend waits for MySQL to be ready. Inside the Docker network it uses `mysql` and `redis` as hosts, regardless of the local hosts in `.env`. The tables are created automatically by TypeORM (`synchronize: true`), which is convenient for the lab.

### Trying the APIs

Once the environment is running, the API answers on `http://localhost:3000`. You can try them on the web interface.


**Web interface.** Choose the endpoint from the menu, fill in the fields that appear (ID, currency) and press **Show**. The endpoints of the exercises and of the assignment are in the menu. The tables at the bottom of the page (click to open them) show the existing IDs.


### Changing the code during the exercises

The `nest-backend` folder is mounted in `/app` through a **bind mount**: changing the files with your own editor updates them immediately in the container too, and vice versa. `npm run start:dev` recompiles and restarts the backend when the sources change. Polling lets it detect changes even on the shared file systems of Docker Desktop.

`node_modules` uses a separate volume, so the Linux dependencies of the container are not overwritten by those possibly installed on your computer.

`Ctrl+C` only stops showing the logs. To stop the environment use `docker compose down`: the MySQL data stays in the `mysql_data` volume. To start again from scratch run `./start-lab.sh`.

## Local API

| Resource | Calls |
| --- | --- |
| City | `GET /cities`, `GET /cities/:id` |
| Flight | `GET /flights`, `GET /flights/:id` |
| Hotel | `GET /hotels`, `GET /hotels/:id` |

The other endpoints (examples, exercises and assignment) are listed in [Examples, exercises and submission](#examples-exercises-and-submission). IDs are positive integers. A non-existent ID produces `404`, an invalid ID (`abc`, `0`) produces `400`. The calls with `?currency=` require a three-letter code: if it is missing or invalid, the response is `400`.

### Database ready for the lab

After executing `start-lab.sh` the project is started and **9 cities, 21 flights and 20 hotels** are prepared.

The flight times are generated on every run, within the next 48 hours, with the durations given in the dataset: this way every flight is in the future and has a weather forecast available at its arrival.

### Calling the external services

Both services are public REST APIs that need **no API key**. They are called only from `nest-backend/src/service/external.service.ts`, through the `getJsonFromUrl(url)` helper: a `fetch` GET with a 3-second timeout that throws on an HTTP error status. The service validates the answer and returns only the values we need; `TravelService` wraps each call in `available(...)` to handle errors.

| | Frankfurter | Open-Meteo |
| --- | --- | --- |
| **What it gives** | exchange rates between currencies | hourly weather forecast for some coordinates |
| **How to call it** | `GET https://api.frankfurter.dev/v2/rate/{from}/{to}` | `GET https://api.open-meteo.com/v1/forecast?latitude=..&longitude=..&hourly=temperature_2m,weather_code&start_date=..&end_date=..` |
| **Our method** | `rate(from, to)` returns the numeric rate (`1.08`) | `weather(latitude, longitude, date?)` returns `{ time, temperature_2m, temperatureUnit, weather_code }` |
| **Our endpoint** | `GET /rate/:from/:to` (Example 1) | `GET /flights/:id/weather` (Example 2) |
| **Cache** | none (updated daily) | Redis, 10 minutes |

You can also try them directly from the terminal:

```bash
curl -sS 'https://api.frankfurter.dev/v2/rate/EUR/USD'
curl -sS 'https://api.open-meteo.com/v1/forecast?latitude=41.9&longitude=12.5&hourly=temperature_2m,weather_code&timezone=GMT&forecast_days=1'
```

#### Frankfurter

`rate(from, to)` returns the exchange rate from the source currency to the target currency. `GET /rate/:from/:to` exposes it, for example `/rate/EUR/USD`: the response is `{ "status": "ok", "data": 1.08 }`. The aggregated responses use the same format in the `exchange` section. The Frankfurter rate is updated daily, so it is not a guaranteed price for a future booking.

#### Open-Meteo

`weather(latitude, longitude, date?)` returns the temperature (`temperature_2m`, in °C) and the weather code (`weather_code`). Without a date it uses the current time; with a date (Example 2 passes the arrival time of the flight) it returns the forecast of the UTC hour that contains that instant: an arrival at 17:10Z gives the forecast of 17:00Z, and `weather.data.time` shows the hour actually used. A date outside the forecast range of Open-Meteo is an error, not a fallback to the current weather, so the section becomes `unavailable`.

The weather is cached in Redis for 10 minutes, with a key made of the coordinates (2 decimals) and the requested hour, so different dates never share a value. The cache is optional: if Redis is down, the API calls Open-Meteo directly.

`GET /flights/:id/weather` loads the flight first (internal data), then asks for the weather at the coordinates of `arrivalCity` at `arrivalTime`. The response is `{ "flight": {...}, "weather": { "status": "ok", "data": {...} } }`. This call depends on the flight, so the two steps run in sequence.

#### Errors and timeouts

If Open-Meteo or Frankfurter do not answer within 3 seconds or return an error, the response is still `200` with `status: "unavailable"` in the corresponding section. If the rate is missing, the converted amounts are `null`; the original prices remain available. Local data, instead, must be valid: a non-existent city, flight or hotel produces `404`. The timeout applies to each external call; when the calls do not depend on each other, they start in parallel.

## Examples and exercises

The endpoints are grouped as in the menu of the web interface: **Provided**, **Examples**, **Exercises** and **Assignment**. Requests are tried from the web interface. The detailed instructions of each example and exercise are in the comments of `nest-backend/src/service/travel.service.ts` (the controller only calls it) and `nest-backend/src/service/external.service.ts`.

### Provided

Already working, to explore the data: `GET /cities`, `GET /cities/:id`, `GET /flights`, `GET /flights/:id`, `GET /hotels` and `GET /hotels/:id`.

### Examples

Guided, to be followed step by step with the teacher.

1. **Example 1:** `GET /rate/EUR/:currency`. Call an external service: Frankfurter exchange rate, with timeout and `unavailable` handling.
2. **Example 2:** `GET /flights/:id/weather`. Flight (internal data) and weather of the arrival city at the arrival time (external service), in sequence.

### Exercises

To be written by the students, using the examples as a model.

1. **Exercise 1:** `GET /hotels/:id/price?currency=USD`. Hotel (internal data) and Frankfurter exchange rate: the original price, in the currency of the hotel's city, does not change.
2. **Exercise 2:** `GET /cities/:id/cheapest-trip?currency=USD`. Cheapest flight and hotel of a city, with the total in the requested currency.

## Assignment: cheapest hotel, converted price and weather

Implement `GET /cities/:id/cheapest-hotel-weather?currency=USD`. Given a city ID, return its cheapest hotel, the nightly price converted into the requested currency using Frankfurter, and the current weather in that city using Open-Meteo.

The logic goes in `TravelService` (`nest-backend/src/service/travel.service.ts`, method `cheapestHotelWeather`); 

In `travel.service.ts`, replace each `TODO` placeholder with the right expression; the code compiles before and after.

### Response

Example for a hotel whose city uses GBP (exchange rate and weather are illustrative):

```json
{
  "hotel": {
    "id": 7,
    "name": "Hotel Colosseo",
    "pricePerNight": 80,
    "city": {
      "id": 3,
      "name": "London",
      "latitude": 51.5074,
      "longitude": -0.1278,
      "currency": "GBP"
    }
  },
  "hotelExchange": { "status": "ok", "data": 1.25 },
  "convertedPrice": 100,
  "weather": {
    "status": "ok",
    "data": {
      "temperature_2m": 15,
      "temperatureUnit": "°C",
      "weather_code": 3,
      "time": "2026-11-02T12:00:00.000Z"
    }
  }
}
```

`convertedPrice` is the converted price for **one night**, the target currency is taken from the request.

### Behaviour

| Situation | Result |
| --- | --- |
| City has hotels | Select the lowest `pricePerNight`, then the lowest hotel ID on a tie |
| City has no hotels, or does not exist | `404`, `No hotel for this city` |
| ID is not an integer (e.g. `abc`) | `400` from `ParseIntPipe` |
| `currency` missing or not three letters, after a hotel is found | `400` |
| Frankfurter fails or does not support the currency (e.g. `ZZZ`) | `200`, `hotelExchange: { "status": "unavailable" }`, `convertedPrice: null`; still request weather |
| Open-Meteo fails | `200`, `weather: { "status": "unavailable" }`; keep the hotel and converted price |
| Both external services fail | `200`, both external results `unavailable`, `convertedPrice: null`; keep the hotel |
| TODO still present in the code | `500`, with a message telling you which TODO needs to be completed |

### Steps

1. get the conversion rates: complete the `hotelExchange` definition
2. get the weather: complete the `weather` definition
3. submit a screenshot of the (working) API page. Show the result with parameters `ID=4` and `currency=USD`


### Test

You can check city IDs and currencies with `GET /cities`, or the tables in the web interface. You can test the endpoint by selecting the entry `Assignment: GET /cities/:id/cheapest-hotel-weather`.

