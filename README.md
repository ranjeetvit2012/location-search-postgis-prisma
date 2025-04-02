
# **User Registration and Location-based Search Using PostGIS and Prisma**

This document outlines the functionality for **user registration** and **location-based search** using **PostGIS** for geospatial queries in a PostgreSQL database. The code is written using **Prisma** as an ORM for interacting with the database and performs operations like inserting user data and querying nearby users/shops based on coordinates.

---

## **1. Prerequisites**

Before starting, ensure the following are installed and set up:

- **PostgreSQL** with the **PostGIS** extension enabled.
- **Prisma ORM** set up to interact with your PostgreSQL database.
- **Node.js** installed to run the application.

---

## **2. Database Setup**

### **Step 1: Install PostGIS**

PostGIS is a spatial database extender for PostgreSQL. It provides geospatial objects to be stored in the database and functions to perform geospatial queries.

Run the following SQL query to enable the PostGIS extension:

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

### **Step 2: Add `coordinates` column to `User` table**

In this step, we add a new column `coordinates` of type `geography` to store the user’s geographical location.

```sql
ALTER TABLE "User"
ADD COLUMN coordinates geography(Point, 4326);
```

This adds a `coordinates` column that can store latitude and longitude in the **WGS 84 (SRID 4326)** coordinate system.

---

## **3. Function to Query Nearby Locations**

This function finds nearby users (or shops) within a specified radius from a given location (latitude and longitude). The `ST_DWithin` PostGIS function is used to filter locations within a given distance (in meters).

### **Code Example:**

```typescript
static async userList(coordinates: any) {
  try {
    const radius = 20000; // 20 km in meters
    const locations: any = await db.$queryRaw`
      SELECT 
        id, name, email, ST_AsText(coordinates) AS coordinates, 
        status, created_at, updated_at
      FROM "User"
      WHERE ST_DWithin(
        coordinates, 
        ST_SetSRID(ST_MakePoint(${coordinates?.lon}, ${coordinates?.lat}), 4326), 
        ${radius}
      );
    `;

    let shopLstRes: any = {};
    if (locations.length) {
      shopLstRes.status = 200;
      shopLstRes.data = locations;
      shopLstRes.message = "shop list";
    } else {
      shopLstRes.status = 400;
      shopLstRes.data = [];
      shopLstRes.message = "shop list not found";
    }
    return shopLstRes;
  } catch (err: any) {
    throw new Error(err);
  }
}
```

### **Explanation:**

- **`coordinates`**: The geographical coordinates (latitude and longitude) of the center point.
- **`radius`**: The search radius in meters (20 km in this example).
- **`ST_SetSRID(ST_MakePoint(...), 4326)`**: This creates a `Point` geometry with **SRID 4326** (WGS 84) and the specified latitude and longitude.
- **`ST_DWithin(...)`**: This PostGIS function checks if the given coordinates are within the specified radius (20 km).

---

## **4. Function to Register a User**

This function registers a new user in the `User` table, including their **name**, **email**, **coordinates**, and other details.

### **Code Example:**

```typescript
static async userRegister(data: any) {
  try {
    data.id = uuidv4(); // Generate a unique UUID for the user

    let userRes = await db.$executeRaw`
      INSERT INTO "User" (
        id, name, email, password, mobile, landmark, state, city, coordinates, role, extra_info, created_at, updated_at
      ) 
      VALUES (
        ${data?.id},
        ${data?.name},
        ${data?.email},
        ${data?.password},
        ${data?.mobile},
        ${data?.landmark || null},
        ${data?.state},
        ${data?.city},
        ST_SetSRID(ST_MakePoint(${data?.coordinates?.lon}, ${data?.coordinates?.lat}), 4326),
        ${data?.role}::"UserRole",
        ${data?.extra_info} || null,
        now(),
        now()
      )
    `;

    let userResData = {};
    if (userRes) {
      userResData = {
        status: 200,
        message: "User registered successfully",
      };
    } else {
      userResData = {
        status: 400,
        message: "Bad request",
      };
    }
    return userResData;
  } catch (err: any) {
    throw new Error(err);
  } finally {
    await db.$disconnect(); // Ensure the database connection is closed
  }
}
```

### **Explanation:**

- **`uuidv4()`**: A unique identifier is generated for each user using the **UUID** format.
- **`ST_SetSRID(ST_MakePoint(...), 4326)`**: This sets the SRID for the coordinates to **4326** (WGS 84).
- **`coordinates`**: The user’s **longitude** and **latitude** are stored in the `coordinates` column of the `User` table.

The `INSERT` query stores user data, including their location in the database.

---

## **5. Prisma Model for User**

Ensure your **Prisma schema** is set up correctly to interact with the `User` table and handle `coordinates` as a `geography` type.

### **Prisma Schema:**

```prisma
model User {
  id          String     @id @default(uuid())
  name        String
  email       String?    @unique
  password    String?
  mobile      String?
  landmark    String?
  state       String?
  city        String?
  status      UserStatus @default(ACTIVE)
  role        UserRole   @default(USER)
  extra_info  Json?
  coordinates Unsupported("geography")? // PostGIS geography column
  created_at  DateTime   @default(now())
  updated_at  DateTime   @updatedAt
}
```

Make sure the **`coordinates`** column is marked as `Unsupported("geography")` to allow Prisma to interact with it.

---

## **6. Error Handling and Response Format**

Both functions handle errors and return a consistent response format for success or failure.

- **Success Response**: Contains `status`, `data`, and `message`.
- **Error Handling**: Errors are caught and thrown as `Error` objects, with detailed messages.

---

## **7. How to Run and Test the Functions**

### **Step 1: Install Dependencies**

Run `npm install` to install all required packages, including **Prisma** and **PostgreSQL**.

### **Step 2: Migrate Database**

Run `npx prisma migrate deploy` to apply any migrations and set up the database schema.

### **Step 3: Run the Application**

Use **Node.js** to run your application and test user registration and location-based queries.

---

## **8. Conclusion**

This setup allows you to:
- **Register users** with geospatial coordinates.
- **Query nearby locations** using PostGIS functions like `ST_DWithin`.
- Integrate Prisma for easy interaction with a PostgreSQL database.

Make sure to test the functions thoroughly to ensure that the queries work correctly in your environment.

---



