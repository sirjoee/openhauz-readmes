# Project Backend

Openhauz GraphQL backend service built with Node.js, TypeScript, Express, and Apollo Server.

## Prerequisites

- [Node.js](https://nodejs.org/) (Check `.nvmrc` or project requirements for specific version, if applicable)
- [Yarn](https://yarnpkg.com/) (Version 4+ recommended, using Corepack)

## Installation

1.  **Clone the repository:**

    ```bash
    git clone git@github.com:sirjowee/openhauz-backend.git
    cd openhauz-backend
    ```

2.  **Enable Corepack (if not already enabled):**

    ```bash
    corepack enable
    ```

3.  **Install dependencies:**
    Yarn PnP will install dependencies based on the `yarn.lock` file.

    ```bash
    yarn install
    ```

4.  **Set up environment variables:**
    Copy the example environment file and fill in the required values:
    ```bash
    cp .env.example .env
    ```
    Now, edit the `.env` file with your specific configuration (e.g., PORT).

## Available Scripts

This project uses a `Makefile` for common tasks. In the project directory, you can run:

- **`make build`**: Transpiles TypeScript to JavaScript in `dist/`.
- **`make dev`**: Runs the app in development mode using `ts-node-dev` (with auto-reload).
- **`make start`**: Starts the production server from `dist/` (requires `make build` first).
- **`make migrate-create name=<YourName>`**: Creates a new SQL migration file pair in `src/database/migrations/`.
- **`make migrate-up`**: Applies all pending UP migrations.
- **`make migrate-down`**: Reverts the last applied migration batch.
- **`make migrate-to target=<Timestamp>`**: Migrates UP TO a specific timestamp prefix.
- **`make migrate-down-to target=<Timestamp>`**: Migrates DOWN TO (before) a specific timestamp prefix.
- **`make migrate-up-count count=<Number>`**: Applies the next `n` pending UP migrations.
- **`make migrate-down-count count=<Number>`**: Reverts the last `n` applied migrations.
- **`make help`**: Displays all available `make` commands.

(These `make` commands are shortcuts for the underlying `yarn` scripts defined in `package.json`.)

## GraphQL Endpoint

Once the server is running (e.g., using `yarn dev`), the GraphQL endpoint is available at:

- **URL:** `http://localhost:4000/graphql` (or the port defined in your `.env` file)

You can use tools like Apollo Studio Sandbox or Postman to interact with the API.

### Example Registration Mutation

```graphql
mutation Register {
  register(
    input: {
      email: "user@example.com"
      password: "securepassword123"
      first_name: "John"
      last_name: "Doe"
      role_name: HOMEBUYER
    }
  ) {
    id
    email
    first_name
    last_name
  }
}
```

### Example Unified Listings Query

The `getUnifiedListings` query now returns both internal and external listings in the same unified `Listing` structure:

```graphql
query SafeListingQuery {
  getUnifiedListings(
    from_date: "2024-01-01T00:00:00.000Z"
    limit: 20
    location: "San Francisco, CA"
    includeExternal: true
    filters: {
      price_min: 500000
      price_max: 2000000
      bedrooms_min: 2
      bedrooms_max: 4
      bathrooms_min: 2.0
      property_type: SINGLE_FAMILY
      square_footage_min: 1500
      year_built_min: 2000
      listing_type: FOR_SALE
    }
  ) {
    listings {
      id
      listingType
      status
      price
      listed_at
      property {
        id
        address_line_1
        city
        state
        property_type
        bedrooms
        bathrooms
        square_footage
        living_area
        year_built
        image_urls
        owner {
          id
          first_name
          last_name
          email
        }
      }
      lister {
        id
        first_name
        last_name
        email
      }
      open_houses {
        id
        start_time
        end_time
        max_participants
      }
      created_at
      updated_at
    }
    hasMore
    totalCount
    nextCursor
  }
}
```

### Query Parameters

The `getUnifiedListings` query supports the following parameters:

- **`from_date`**: Start date for filtering internal open houses (required)
- **`limit`**: Maximum number of listings to return (default: 10)
- **`cursor`**: Pagination cursor for getting the next page of **internal listings only** (optional)
  - This cursor is based on the date of the last internal listing
  - It does **not** affect external API pagination
- **`external_page`**: Page number for external RapidAPI listings (default: 1)
  - This is **independent** of the cursor and only affects RapidAPI calls
  - Increment this to get the next page of external listings
- **`location`**: Location for external listings search (default: "New York, NY")
- **`includeExternal`**: Boolean flag to control whether external listings are included (default: true)
  - When `true`: Returns ~60% internal + ~40% external listings
  - When `false`: Returns only internal listings, using the full limit for internal results
- **`filters`**: Object containing filtering options that work with both internal and external listings:
  - **`price_min`/`price_max`**: Price range in dollars
  - **`bedrooms_min`/`bedrooms_max`**: Number of bedrooms range
  - **`bathrooms_min`/`bathrooms_max`**: Number of bathrooms range (supports decimals like 2.5)
  - **`property_type`**: Property type (`SINGLE_FAMILY`, `MULTI_FAMILY`, `CONDO`, `TOWNHOUSE`, `LAND`, `OTHER`)
  - **`square_footage_min`/`square_footage_max`**: Square footage range (uses `square_footage` or `living_area`)
  - **`year_built_min`/`year_built_max`**: Year built range
  - **`listing_type`**: Listing type (`FOR_SALE` or `FOR_RENT`)

### Dual Pagination System

The unified listings system uses **two independent pagination mechanisms**:

1. **Internal Listings**: Date-based cursor pagination

   - Use `cursor` parameter (date string)
   - Based on the `created_at` date of internal listings
   - Returned in response as `nextCursor`

2. **External Listings (RapidAPI)**: Page-based pagination
   - Use `external_page` parameter (integer, starts at 1)
   - Independent of the cursor value
   - Returned in response as `nextExternalPage`

### Pagination Examples:

```graphql
# First request - get page 1 of both internal and external
query {
  getUnifiedListings(
    from_date: "2024-01-01T00:00:00Z"
    limit: 20 # cursor: null (first page) # external_page: 1 (default)
  ) {
    listings {
      id
    }
    hasMore
    nextCursor
    nextExternalPage
  }
}

# Next request - get next page of internal listings, keep external at page 1
query {
  getUnifiedListings(
    from_date: "2024-01-01T00:00:00Z"
    limit: 20
    cursor: "2024-01-15T10:30:00.000Z" # from previous response
    external_page: 1 # keep external at page 1
  ) {
    listings {
      id
    }
    hasMore
    nextCursor
    nextExternalPage
  }
}

# Next request - get next page of external listings, keep internal cursor same
query {
  getUnifiedListings(
    from_date: "2024-01-01T00:00:00Z"
    limit: 20
    cursor: "2024-01-15T10:30:00.000Z" # same cursor
    external_page: 2 # increment external page
  ) {
    listings {
      id
    }
    hasMore
    nextCursor
    nextExternalPage
  }
}
```

**Filter Mapping**: These filters are automatically mapped to appropriate RapidAPI parameters for external listings and database query conditions for internal listings, ensuring consistent filtering across both data sources.

**⚠️ Important Query Notes:**

- **Separate Pagination**: `cursor` only affects internal listings, `external_page` only affects external listings
- **Independent Progression**: You can advance one pagination system without affecting the other
- **Client Responsibility**: The client must track both `nextCursor` and `nextExternalPage` to properly paginate through all results

**Avoid deep circular nesting** like `property.listings.open_houses.listing` as it creates circular references
**Use this safe pattern instead**:

```graphql
# ✅ SAFE: Query listings and their properties separately
query SafeListingQuery {
  getUnifiedListings(...) {
    listings {
      id
      listingType
      property {
        id
        address_line_1
        # Don't query property.listings here
      }
      open_houses {
        id
        start_time
        # Don't query open_houses.listing here
      }
    }
  }
}

# ❌ AVOID: Deep circular nesting
property {
  listings {
    open_houses {
      listing {  # This creates circular reference
        property { ... }
      }
    }
  }
}
```

- External listings will have empty `open_houses` arrays
- External listings are identifiable by IDs prefixed with `ext-listing-`, `ext-prop-`, `ext-user-`
- For performance, limit the depth of related field queries

**Note:** External listings are converted to the same `Listing` structure with synthetic `Property` and `User` objects to ensure a consistent response format. External listings will have:

- IDs prefixed with `ext-listing-`, `ext-prop-`, and `ext-user-`
- Synthetic lister information (External Agent)
- Empty `open_houses` arrays (external APIs typically don't provide open house data)
- Property descriptions indicating they are external listings

## Environment Variables

The application uses environment variables for configuration. Copy the `.env.example` file to `.env` and fill in the required values:

```bash
cp .env.example .env
```

**Required Variables:**

- `PORT`: Port for the HTTP server.
- `DB_HOST`: PostgreSQL database host.
- `DB_PORT`: PostgreSQL database port.
- `DB_USERNAME`: PostgreSQL database username.
- `DB_PASSWORD`: PostgreSQL database password.
- `DB_DATABASE`: PostgreSQL database name.
- `NODE_ENV`: Set to `development`, `production`, or `test`.

Environment variables are validated on startup using `envalid`. See `src/config/env.ts` for details.

## Database

This project uses PostgreSQL with TypeORM as the ORM. Entity definitions mapping to database tables are located in the `src/entity/` directory.

**Managing Schema Changes:**

Database schema changes (e.g., adding tables, modifying columns) **must** be managed using **TypeORM migrations**, especially for staging and production environments. The sections below detail how to use migrations.

**Development Setting (`synchronize: true`):**

For convenience during early **local development only**, the `synchronize` option is currently enabled when `NODE_ENV` is set to `development` (see `src/data-source.ts`). This setting makes TypeORM automatically compare your entity definitions (`src/entity/*`) to the connected database schema on application startup (`yarn dev`). If they differ, TypeORM will attempt to alter the database schema directly to match the entities.

**Warnings about `synchronize: true`:**

- **Potential Data Loss:** Automatic synchronization can lead to data loss if entity changes involve dropping columns or tables.
- **Not Suitable for Production:** It is unsafe for production or collaborative environments where schema changes need to be controlled, reviewed, and applied consistently.
- **Bypasses Migrations:** It ignores the migration history, potentially leading to inconsistencies if you later switch to using migrations without careful database preparation.

**Recommendation:** Use migrations as the primary way to manage schema changes even during development once the initial setup is stable. Ensure `synchronize: false` in `src/data-source.ts` for production and staging environments.

## Database Migrations

TypeORM migrations provide a reliable way to version control your database schema changes using raw SQL files located in `src/database/migrations/`.

**Using Make Commands:**

The recommended way to manage migrations is via the `Makefile` targets:

- **Create:** `make migrate-create name=YourMigrationName`
- **Apply All:** `make migrate-up`
- **Revert Last:** `make migrate-down`
- **Apply Up To:** `make migrate-to target=<TimestampPrefix>`
- _(See `make help` for all migration commands)_

**Prerequisites for Migrations:**

1.  **Database Connection:** PostgreSQL server running and accessible.
2.  **Environment Variables:** Correct `PG*` variables set in `.env`.
3.  **Build (Sometimes):** While the `make` commands handle `dotenv`, commands interacting with TypeORM entities _indirectly_ (like potentially future auto-generation, though we removed it) might require a `make build` first. Running migrations themselves (`make migrate-up`) typically does _not_ require a build as `node-pg-migrate` reads SQL files directly.

**Workflow:**

1.  **Create Migration File:**
    ```bash
    make migrate-create name=YourMigrationName
    ```
    This creates `[timestamp]_yourmigrationname.sql` and `[timestamp]_yourmigrationname.down.sql` in `src/database/migrations/`.
2.  **Write SQL:** Edit the generated `.sql` files. Add your schema changes to the `up` file and the corresponding revert logic to the `down` file.
3.  **Review SQL:** Carefully check your SQL in both files.
4.  **Apply Migration:**
    ```bash
    make migrate-up
    ```
    (Or use `make migrate-to target=...` / `make migrate-up-count count=...` for more control).

**Reverting Migrations:**

```bash
make migrate-down
# Or: make migrate-down-to target=...
# Or: make migrate-down-count count=...
```

**Tips:**

- Always run `yarn build` before migration commands.
- Review generated migrations carefully.
- Keep migrations small and focused on a single logical change where possible.
- Ensure `synchronize: false` in `src/data-source.ts` when using migrations in production/staging.
