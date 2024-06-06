# Smart Parking Lot Backend System Design

## Objective

Design the low-level architecture for a backend system of a smart parking lot to handle vehicle entry and exit management, parking space allocation, and fee calculation.

## Problem Statement

Imagine a parking lot in an urban area with multiple floors and numerous parking spots. This system should efficiently manage the parking process by automatically assigning parking spots based on vehicle size and availability, tracking the time each vehicle spends in the parking lot, and calculating parking fees upon exit.

## Functional Requirements

1. **Parking Spot Allocation**: Automatically assign an available parking spot to a vehicle when it enters, based on the vehicle’s size (e.g., motorcycle, car, bus).
2. **Check-In and Check-Out**: Record the entry and exit times of vehicles.
3. **Parking Fee Calculation**: Calculate fees based on the duration of stay and vehicle type.
4. **Real-Time Availability Update**: Update the availability of parking spots in real-time as vehicles enter and leave.

## Design Aspects to Consider

- **Data Model**: Design a database schema to manage parking spots, vehicles, and parking transactions.
- **Algorithm for Spot Allocation**: Develop an algorithm to efficiently assign parking spots to incoming vehicles.
- **Fee Calculation Logic**: Implement logic to calculate fees based on parking duration and vehicle type.
- **Concurrency Handling**: Ensure the system can handle multiple vehicles entering or exiting simultaneously.

## Data Model

### Tables

1. **ParkingSpot**

   ```sql
   CREATE TABLE ParkingSpot (
    spot_id SERIAL PRIMARY KEY,
    floor_number INT,
    size VARCHAR(20),
    is_occupied BOOLEAN DEFAULT FALSE,
    vehicle_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (vehicle_id) REFERENCES Vehicle(vehicle_id)
   );
   ```

2. **Vehicle**

   ```sql
   CREATE TABLE Vehicle (
    vehicle_id SERIAL PRIMARY KEY,
    vehicle_type VARCHAR(20),
    license_plate VARCHAR(20),
    entry_time TIMESTAMP,
    exit_time TIMESTAMP,
    parking_spot_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (parking_spot_id) REFERENCES ParkingSpot(spot_id)
   );
   ```

3. **Transaction**
   ```sql
   CREATE TABLE Transaction (
    transaction_id SERIAL PRIMARY KEY,
    vehicle_id INT,
    entry_time TIMESTAMP,
    exit_time TIMESTAMP,
    total_fee DECIMAL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (vehicle_id) REFERENCES Vehicle(vehicle_id)
   );
   ```
4. **PricingRule**

```sql
   CREATE TABLE PricingRule (
  vehicle_type VARCHAR(20) PRIMARY KEY,
  rate_per_hour DECIMAL NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
);
```

## Algorithm for Spot Allocation

If multiple spots are available, a suitable strategy (e.g., nearest available, spread allocation for efficient utilization) can be implemented to choose a spot.

**Filtering by Additional Criteria**

1. Accessible spots (if some spots are designated for disabled vehicles)
2. Charging station availability (for electric vehicles)

**Spot Selection Strategy**

1. Nearest Available Spot: Minimizes walking distance for the driver.
2. Farthest Available Spot: Encourages filling up inner spots first for better space utilization.
3. Round-Robin Allocation: Distributes allocation across different areas of the parking lot for a fairer experience.

```javascript
const allocateParkingSpot = async (vehicleId, vehicleType) => {
  const availableSpots = await ParkingSpot.findAll({
    where: { size: vehicleType, isOccupied: false
    ...additionalCriteria, // Include additional filter,
  });

  if (!availableSpots.length) {
    throw new Error("No available parking spots for this vehicle type");
  }

  // Example strategy: Choose the nearest available spot (assuming spots are ordered by distance)
  const selectedSpot = availableSpots[0];
  await selectedSpot.update({ isOccupied: true, vehicleId });
  await Vehicle.update(
    { parkingSpotId: selectedSpot.spotId },
    { where: { vehicleId } }
  );

  return selectedSpot.spotId;
};
```

## Fee Calculation Logic

```javascript
const calculateParkingFee = async (vehicleId) => {
  const vehicle = await Vehicle.findByPk(vehicleId);

  if (!vehicle) {
    throw new Error("Vehicle not found");
  }

  const entryTime = vehicle.entryTime;
  const exitTime = new Date(); // Assuming exit time is now
  const durationInHours = (exitTime - entryTime) / 3600000; // Duration in hours

  const ratePerHour = getRate(vehicle.vehicleType);
  const totalFee = durationInHours * ratePerHour;

  await vehicle.update({ exitTime });
  await ParkingSpot.update(
    { isOccupied: false, vehicleId: null },
    { where: { spotId: vehicle.parkingSpotId } }
  );
  await Transaction.create({ vehicleId, entryTime, exitTime, totalFee });

  return totalFee;
};

const getRate = async (vehicleType) => {
  const rule = await PricingRule.findOne({ where: { vehicleType } });
  if (!rule) {
    throw new Error("Pricing rule not found for vehicle type: " + vehicleType);
  }
  return rule.ratePerHour;
};
```

## Concurrency Handling

To handle concurrency and ensure proper processing order, we can use a queuing system (e.g., message queue) for incoming requests.

### Using a Message Queue

1. **Message Queue Setup**: Use a message queue like RabbitMQ, Kafka, or any other suitable queuing system.
2. **Producer**: When a vehicle arrives or leaves, an entry/exit request is sent to the queue.
3. **Consumer**: A worker service consumes messages from the queue, processes the requests, and updates the database accordingly.

### Implementation Example

```javascript
const amqp = require("amqplib/callback_api");

const handleVehicleEntry = async (vehicleDetails) => {
  amqp.connect("amqp://localhost", (error0, connection) => {
    if (error0) {
      throw error0;
    }
    connection.createChannel((error1, channel) => {
      if (error1) {
        throw error1;
      }
      const queue = "vehicle_entry";
      const msg = JSON.stringify(vehicleDetails);

      channel.assertQueue(queue, { durable: true });
      channel.sendToQueue(queue, Buffer.from(msg), { persistent: true });

      console.log(" [x] Sent %s", msg);
    });

    setTimeout(() => {
      connection.close();
    }, 500);
  });
};

const handleVehicleExit = async (vehicleId) => {
  amqp.connect("amqp://localhost", (error0, connection) => {
    if (error0) {
      throw error0;
    }
    connection.createChannel((error1, channel) => {
      if (error1) {
        throw error1;
      }
      const queue = "vehicle_exit";
      const msg = JSON.stringify({ vehicleId });

      channel.assertQueue(queue, { durable: true });
      channel.sendToQueue(queue, Buffer.from(msg), { persistent: true });

      console.log(" [x] Sent %s", msg);
    });

    setTimeout(() => {
      connection.close();
    }, 500);
  });
};
```

### Database Locking

In addition to using a message queue, we need to ensure database operations are atomic. This can be done by wrapping our operations in a transaction, which locks the rows involved in the allocation and fee calculation processes.

```javascript
const { Transaction: SequelizeTransaction } = require("sequelize");

const handleVehicleEntryWithTransaction = async (vehicleDetails) => {
  const transaction = await sequelize.transaction();

  try {
    const vehicle = await Vehicle.create(vehicleDetails, { transaction });
    const spotId = await allocateParkingSpot(
      vehicle.vehicleId,
      vehicle.vehicleType,
      transaction
    );

    await transaction.commit();
    return spotId;
  } catch (error) {
    await transaction.rollback();
    throw error;
  }
};

const handleVehicleExitWithTransaction = async (vehicleId) => {
  const transaction = await sequelize.transaction();

  try {
    const fee = await calculateParkingFee(vehicleId, transaction);
    await transaction.commit();
    return fee;
  } catch (error) {
    await transaction.rollback();
    throw error;
  }
};
```

## Communication Flow

1. **Vehicle Entry**:

   - Vehicle arrives at the entrance.
   - Entry details are sent to the message queue.
   - Worker service consumes the entry request from the queue.
   - Worker service allocates a parking spot and updates the database.
   - The system responds with the allocated parking spot ID.

2. **Vehicle Exit**:
   - Vehicle arrives at the exit.
   - Exit request is sent to the message queue.
   - Worker service consumes the exit request from the queue.
   - Worker service calculates the parking fee, updates the database, and frees the parking spot.
   - The system responds with the total parking fee.

## Express API Endpoints

```javascript
const express = require("express");
const app = express();
app.use(express.json());

app.post("/entry", async (req, res) => {
  try {
    await handleVehicleEntry(req.body);
    res.status(200).send({ message: "Vehicle entry request sent" });
  } catch (error) {
    res.status(500).send({ error: error.message });
  }
});

app.post("/exit", async (req, res) => {
  try {
    await handleVehicleExit(req.body.vehicleId);
    res.status(200).send({ message: "Vehicle exit request sent" });
  } catch (error) {
    res.status(500).send({ error: error.message });
  }
});

app.listen(3000, () => {
  console.log("Server is running on port 3000");
});
```

## Additional Considerations

- **Error Handling and Logging**: Implement error handling and logging mechanisms throughout the system to ensure any issues are captured and can be debugged efficiently.
- **Scalability**: Design the system to be scalable to accommodate a growing number of parking spots and vehicles. This includes considering load balancing and database sharding techniques.
- **Payment Integration**: Consider integrating with payment gateways for seamless fee collection and ensuring the entire payment process is secure and user-friendly.
- **Security Measures**: Implement security measures to protect user data and prevent unauthorized access. This includes using HTTPS, encrypting sensitive data, and employing robust authentication and authorization mechanisms.
