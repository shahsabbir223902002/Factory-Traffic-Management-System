# Factory Traffic Management System — Backend

Backend implementation for the CSI Smart Tech Factory Traffic Management System assessment.

## Stack

- Node.js + TypeScript
- Express REST API
- SQLite + better-sqlite3
- Zod validation
- Vitest

## Architecture

`HTTP API -> Application Service -> Traffic Engine -> State Machine/Scheduler -> Repository + Controller Port`

The traffic domain does not depend on Express, SQLite, MQTT, or the frontend.

## Safety rules

1. Junction A has two non-conflicting phases: NORTH_SOUTH and EAST_WEST.
2. A phase can move from GREEN to YELLOW, then ALL_RED, before the other phase becomes GREEN.
3. Conflicting GREEN states are never requested by the domain.
4. Emergency traffic has higher priority than normal/manual traffic, but emergency preemption still follows the safe transition sequence.
5. `event_id` is idempotent.
6. Controller-confirmed state is separate from desired state.
7. Controller commands have unique `command_id` values.
8. Every important decision is recorded in audit history.

## Assumptions

- Queue is based on active valid vehicles.
- CLEAR for an unknown vehicle is rejected and audited.
- `sequence_no` is stored and used as a useful ordering signal, but `event_id` remains the idempotency key.
- Manual mode remains until an explicit return-to-automatic command.
- Conflicting emergencies are handled deterministically by earliest valid emergency event.
- ACK timeout moves the junction toward DEGRADED handling; the backend never treats a sent command as physical confirmation.
- Restart does not blindly assume the physical controller matches the desired state.
- REST controller simulation is used now; a controller adapter can later be replaced by MQTT.

## API

### Junctions

- `GET /api/junctions`
- `GET /api/junctions/:id`
- `POST /api/junctions`

### Status and traffic

- `GET /api/junctions/:id/status`
- `POST /api/sensor-events`
- `GET /api/junctions/:id/history`

### Commands/controller

- `POST /api/junctions/:id/commands`
- `POST /api/controller-events`

## Run

```bash
npm install
npm run dev
```

Server: `http://localhost:3000`

Create a junction:

```bash
curl -X POST http://localhost:3000/api/junctions \
  -H "Content-Type: application/json" \
  -d '{"id":"A","name":"Junction A"}'
```

Vehicle arrival:

```bash
curl -X POST http://localhost:3000/api/sensor-events \
  -H "Content-Type: application/json" \
  -d '{
    "event_id":"evt-001",
    "junction_id":"A",
    "direction":"NORTH",
    "event_type":"ARRIVAL",
    "vehicle_id":"truck-01",
    "vehicle_type":"TRUCK",
    "sequence_no":1,
    "timestamp":"2026-10-08T10:00:00.000Z"
  }'
```

Manual command:

```bash
curl -X POST http://localhost:3000/api/junctions/A/commands \
  -H "Content-Type: application/json" \
  -d '{"command":"REQUEST_PHASE","phase":"EAST_WEST"}'
```

Controller ACK:

```bash
curl -X POST http://localhost:3000/api/controller-events \
  -H "Content-Type: application/json" \
  -d '{"command_id":"<command-id>","event_type":"ACK","junction_id":"A"}'
```

## Tests

```bash
npm test
npm run typecheck
npm run build
```

The test suite covers phase safety, transition sequencing, duplicate events, queue behavior, emergency priority, and controller state separation.


