Planned structure of documentation:

1. Overview
2. Bookings & Flights (already reviewed)
3. Regular Planning (ready for review now)
4. APIs (ready for review now)
5. Master Data

[TOC]



# Product Overview

The Continuous Planning Service enables TUI’s transfer service to operate efficiently by scheduling, optimizing, and routing TUI’s fleet from advanced pre-planning all the way to day-of operation. 

**[Charlie: Insert product overview diagram]**

# Glossary (in order of appearance)

Transfer Service

Cost Function

Business Rules

**Master Data** includes information about physical places and the business rules that should apply to relevant bookings during planning. 

Regular Planning

AWS Kinesis

# What We're Optimizing For

## Cost Functions

The Continuous Planning System optimizes for cost while following business rules. In order to optimize for cost, we must first define what cost is. Each destination has a specific cost function.

Example 1: Mallorca

Cost = distance travelled



Example 2: Zakynthos

Cost = (# of Vehicle A x cost of vehicle A) + (# of Vehicle B x cost of vehicle B)



In general, destinations where TUI owns their own fleet usually have a simple "distance traveled" cost function like Mallorca, and other destinations that are primarily relying on suppliers have the more complex cost function like Zakynthos.

## Business Rules

The Continuous Planning System has been configured with business rules for each destination, in order to comply with local legislation, plan trips that are physically possible given the constraints of the geography, and ensure a great customer experience. Most of these rules are specified in Master Data, which includes information about physical places and the business rules that should apply to relevant bookings during planning. 



# Bookings & Flights

## Bookings & Flights Overview

A **Booking** represents a need for a transfer (a ride in a vehicle) for a group of passengers (1 or more). A group represented in a single **Booking** generally has booked a tour with TUI together, will be on the same flights, and will be staying at the same **Hotel**.

For every group who books a tour with TUI together, there will generally be 2 **Bookings** sent to Mobi because there are 2 transfers. For example, if the group is going to Cancun, there would be the following 2 **Bookings**:

- **Arrival to a destination:** a transfer picks up the group at the Cancun airport & brings them to their hotel
- **Departure from a destination:** a transfer picks up the group at their hotel & brings them to the Cancun airport

Guests that have itineraries involving multiple **Hotels** may have additional transfers, so they would have 1 or more additional **Bookings** with the "Between Hotels" type.

Each **Flight** represents a real flight in the world that corresponds to an **Arrival** to a destination or a **Departure** from a destination where 1 or more groups of passengers will need a transfer. Multiple **Bookings** may correspond to a given **Flight.**



## Sending Bookings & Flights via AWS Kinesis

Bookings & Flights are sent as records in a AWS Kinesis data stream. Records include metadata and a payload. Fields within the record can be sent in any order. 

When a booking comes in via the Kinesis stream, it gets ingested but not planned until the "planning window" for the relevant destination. See [When Bookings Get Planned](#when-bookings-get-planned) for further details.

**Metadata specifies:**

- content-type
  - 'vnd.booking-event.v1' for bookings
  - 'vnd.flight-event.v1' for flights
- operation
  - saved (creating a new booking or flight, or updating it) - requires the booking or flight object with all required fields.
    - If a booking or flight comes in with a new booking_id or flight_id, it will be created
    - If a booking or flight comes in with an existing booking_id or flight_id, it will be updated
    - Booking_id & flight_id are globally unique across destinations & dates
  - deleted (deleting a booking or flight) - requires just the booking_id or flight_id
    - If a booking has already been assigned to a trip, deleting it will remove it from the relevant trip
  - locked (locking a booking) - requires just the booking_id
    - Locking a booking means it will not be replanned & assigned to a new trip
  - unlocked (unlocking a booking) - requires just the booking_id
    - Unlocking a booking means it can be replanned & assigned to a new trip



**Payload fields for bookings & flights are specified in the following sections:**

[Fields for Bookings](#fields-for-bookings)

[Fields for Flights](#fields-for-flights)



**Example Booking record:**

```
{'metadata': {'content-type': 'vnd.booking-event.v1', 'X-B3-TraceId': '0', 'operation': 'saved'}, 'payload': {'booking_id': 'ASX-5082-5178600-1', 'touroperator_id': 'TOP 1', 'ext_booking': 'BETWEENHT', 'lead_pax_name': 'ASELA ASELA', 'destination_id': '1', 'total_pax': 2, 'combinable': True, 'transfer_way': 'between hotels', 'force_pickup_datetime': '2023-01-31T15:00:00+01:00', 'origin_point_type': 'Hotel', 'origin_guest_hotel_id': '1', 'origin_stop_hotel_id': '1', 'destination_point_type': 'Hotel', 'destination_guest_hotel_id': '2', 'destination_stop_hotel_id': '2', 'flight_exclusive': False, 'booking_plan_status': 'Pending', 'passengers': [{'passenger_id': 1, 'name': 'ASELA ASELA', 'age': 30}, {'passenger_id': 2, 'name': 'ASELA ASELA', 'age': 30}], 'operation_date': '2023-01-31', 'welfare': False}}
```

**Example Flight record:**

```
{'metadata': {'content-type': 'vnd.flight-event.v1', 'operation': 'saved'}, 'payload': {'flight_id': '10027', 'flight_number': 'VY3832', 'flight_date': '2019-07-01T14:00:00+02:00', 'flight_way': 'Departure', 'origin_terminal_id': '1', 'first_flight': False, 'destination_terminal_id': 'MUC'}}
```



## Fields for Bookings

### Required Fields for All Bookings

| Field               | Type   | Description                                                  | Example                                                      |
| ------------------- | ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| booking_id          | string | Unique id for each booking                                   | "ASX-5006-1813434-2"                                         |
| ext_booking         | string | External booking id. If bookings share this value, then they will be grouped together. | "61902535"                                                   |
| transfer_way        | enum   | Whether a booking is an arrival, a departure, or between hotels. Possible values: "Arrival", "Departure", "Between hotels" | "Departure"                                                  |
| operation_date      | date   | Date of transfer (YYYY-MM-DD)                                | "2024-01-13"                                                 |
| total_pax           | int    | Number of passengers as part of the booking who will need a seat? | 2                                                            |
| destination_id      | string | Associated destination                                       | "5006"                                                       |
| touroperator_id     | string | Associated tour operator, identifying rules this booking needs to obey in planning. Usually includes destination_id. | "5006-205747"                                                |
| combinable          | bool   | Whether the booking can be combined with other bookings (e.g. VIP bookings cannot be combined) | "true"                                                       |
| flight_exclusive    | bool   | Whether the flight cannot be combined with other flights (e.g. **TODO: provide example of when this is useful**) | "false"                                                      |
| welfare             | bool   | Whether the group needs a handicap-accessible vehicle. Handicap-accessible vehicles will only be assigned to bookings where this field is set to true. | "false"                                                      |
| booking_plan_status | enum   | "Pending" or "Planned" ***(This field is not used by Mobi - does TUI use it?)*** | "Pending"                                                    |
| passengers          | dict   | Includes passenger_id (int starting from 1), name (string), & age (int). Upon ingestion of the booking, ages of passengers are checked against min_age (specified in master data per tour operator). By default passengers with age under 2 are assumed to be infants in arms and not require a seat. Mobi does not use the passenger information aside from this age check. | "passengers":[{"passenger_id":1,"name":"Hendrik Rauh","age":54},{"passenger_id":2,"name":"Grit Berghof","age":50}] |

### Required Fields for Arrivals

| Field                     | Type   | Description                                                  | Example             |
| ------------------------- | ------ | ------------------------------------------------------------ | ------------------- |
| origin_flight_id          | string | Associated flight id. Usually includes destination_id.       | "ASX-5175-347722-1" |
| destination_stop_hotel_id | string | Hotel id for the hotel where the transfer should drop the guests off. Usually includes destination_id. | "5175-64985"        |

### Required Fields for Departures

| Field                    | Type   | Description                                                  | Example            |
| ------------------------ | ------ | ------------------------------------------------------------ | ------------------ |
| destination_flight_id    | string | Associated flight id. Usually includes destination_id.       | "ASX-5006-1333547" |
| origin_stop_hotel_id     | string | Hotel id for the hotel where the transfer should pick up the guests. Usually includes destination_id. | "5006-7729"        |
| presentation_window_from | int    | How many minutes **at most** the transfer can arrive at the airport before the flight departure time. e.g. 180 means the transfer can arrive at most 3 hours before the flight departure. | 180                |
| presentation_window_to   | int    | How many minutes **at least** the transfer can arrive at the airport before the flight departure time. e.g. 120 means the transfer can arrive at most 2 hour before the flight departure. | 120                |

### Optional Fields

| Field                  | Type     | Description                                                  | Example                     |
| ---------------------- | -------- | ------------------------------------------------------------ | --------------------------- |
| force_pickup_datetime  | datetime | Force planning to use a specific time for pickup for this booking | "2024-05-05T18:15:00-04:00" |
| force_dropoff_datetime | datetime | Force planning to use a specific time for dropoff for this booking | "2024-05-05T18:15:00-04:00" |
| vehicle_type           | enum     | Force planning to use a specific type of vehicle for this booking. Any vehicle type specified in the master data for the associated destination is valid. | "Van / Minivan"             |

### Fields TUI Sends but Mobi Does Not Use

If these fields are not sent as part of a booking, we will not send an error. **(TODO: need to check if this is true)**

| Field                                           | Type   | Description                                                  |
| ----------------------------------------------- | ------ | ------------------------------------------------------------ |
| destination_guest_hotel_id                      | string | Hotel id for the hotel where the guests are staying, provided for arrivals. This is not used, because destination_stop_hotel_id is used instead. |
| origin_guest_hotel_id                           | string | Hotel id for the hotel where the guests are staying, provided for departures. This is not used, because origin_stop_hotel_id is used instead. |
| lead_pax_name                                   | string | Lead passenger for the booking                               |
| origin_point_type/destination_point_type        | enum   | "Hotel" or "Terminal". These are not used because transfer_way already defines what the origin & destination point types are. |
| orgin_terminal_type / destination_terminal_type | enum   | "Airport"                                                    |



### Example Bookings

**Arrival Booking**

```
{"booking_id":"ASX-5175-347722-1","touroperator_id":"5175-212553","ext_booking":"61902535","lead_pax_name":"HENDRIK,  RAUH","destination_id":"5175","total_pax":2,"combinable":true,"transfer_way":"Arrival","operation_date":"2024-01-24","origin_flight_id":"ASX-5175-31293","origin_point_type":"Terminal","origin_terminal_type":"Airport","destination_point_type":"Hotel","destination_guest_hotel_id":"5175-64985","destination_stop_hotel_id":"5175-64985","flight_exclusive":false,"presentation_window_from":0,"presentation_window_to":0,"booking_plan_status":"Pending","passengers":[{"passenger_id":1,"name":"Hendrik Rauh","age":54},{"passenger_id":2,"name":"Grit Berghof","age":50}],"welfare":false}}]}
```

**Departure Booking**

```
{"booking_id":"ASX-5006-1813434-2","touroperator_id":"5006-205747","ext_booking":"WRC1A1BU","lead_pax_name":"SR  NICOLE GRUSZYNSKI","destination_id":"5006","total_pax":2,"combinable":true,"transfer_way":"Departure","operation_date":"2024-01-13","origin_point_type":"Hotel","origin_guest_hotel_id":"5006-7729","origin_stop_hotel_id":"5006-7729","destination_flight_id":"ASX-5006-1333547","destination_point_type":"Terminal","destination_terminal_type":"Airport","flight_exclusive":false,"presentation_window_from":180,"presentation_window_to":180,"booking_plan_status":"Planned","passengers":[{"passenger_id":1,"name":"SR  NICOLE GRUSZYNSKI","age":30},{"passenger_id":2,"name":"SR  NICOLE GRUSZYNSKI","age":30}],"welfare":false}}]}
```

**Between Hotels Booking**

```
{"booking_id":"ASX-5006-1835811-1","touroperator_id":"5006-42180","ext_booking":"22077510","lead_pax_name":"MALJONEN  EERO  (L)","destination_id":"5006","total_pax":2,"combinable":false,"transfer_way":"Between hotels","vehicle_type":"Van / Minivan","operation_date":"2024-01-12","origin_point_type":"Hotel","origin_guest_hotel_id":"5006-8183","origin_stop_hotel_id":"5006-8183","destination_point_type":"Hotel","destination_guest_hotel_id":"5006-61586","destination_stop_hotel_id":"5006-61586","flight_exclusive":false,"booking_plan_status":"Pending","passengers":[{"passenger_id":1,"name":"MALJONEN  EERO  (L)","age":30},{"passenger_id":2,"name":"MALJONEN  EERO  (L)","age":30}],"welfare":false}}]
```



## Fields for Flights

### Required Fields for All Flights

| Field                   | Type     | Description                                                  | Example                     |
| ----------------------- | -------- | ------------------------------------------------------------ | --------------------------- |
| flight_id               | string   | Unique id for each flight                                    | "10027"                     |
| flight_number           | string   | Flight number used in the real world for this flight         | "VY3832"                    |
| flight_way              | enum     | Whether a flight is an arrival to a Destination, or a departure from a Destination. Possible values: "Arrival", "Departure". | "Departure"                 |
| flight_date             | datetime | Date/time of flight arrival or departure, depending on the flight_way | "2019-07-01T14:00:00+02:00" |
| destination_terminal_id | string   | Terminal that flight arrives in. If an arrival, use this as the terminal for the booking. | "MUC"                       |
| original_terminal_id    | string   | Terminal that flight departs from. If a departure, use this as the terminal for the booking. | "1"                         |



### Example Flights

**Example Departure Flight:**

```
{'flight_id': '10027', 'flight_number': 'VY3832', 'flight_date': '2019-07-01T14:00:00+02:00', 'flight_way': 'Departure', 'origin_terminal_id': '1', 'destination_terminal_id': 'MUC'}}
```



## Booking & Flight Errors

If there is an error with a booking, the error is sent back to TUI via AWS SNS. 

Timing of errors depends on the type of error: 

- An error may be sent immediately after the booking is received, if the booking could not be stored. These errors are specified in the next section, [Kinesis Rejection Errors](#kinesis-rejection-errors).
- If the booking can be stored but an error occurs later during planning, the error will be sent when planning occurs. These errors are specified in [Regular Planning](#regular-planning).

### Kinesis Rejection Errors

There are multiple categories of errors. The endpoint **GET /tui-cps/v1/messages** can be used to retrieve a complete set of possible error messages, across all categories. This section describes one category of errors: **kinesis_rejection** errors.

**kinesis_rejection** error messages indicate that a booking or a flight has been sent into the system, but the booking or flight has issues which would make it impossible to process. 

These messages are sent out at the time that the booking or flight is sent in, via an AWS SNS topic.  

Currently, if multiple **kinesis_rejection** error messages are applicable, multiple SNS messages will be sent. ***In the future, a single SNS message will be sent with all the applicable error messages for the booking or flight.***

### Kinesis Rejection Errors for Bookings

| message_id                      | Message                                                      | Description                                                  |
| ------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| "KR_unsupported_transfer_type"  | "Kinesis record for booking %(booking_id)s discarded: booking has unsupported transfer type %(transfer_type)s" | transfer_type must be "Arrival", "Departure", or "Between Hotels". Any other value for transfer_type will cause this error message. |
| "KR_no_existing_tour_operator"  | "Kinesis record for booking %(booking_id)s discarded: Touroperator %(tour_operator_id)s has not been defined in Master Data" | touroperator_id must match a touroperator_id that has been defined in the Master Data for the Destination. If it does not, that will cause this error message. |
| "KR_between_hotels_no_pickup"   | "Kinesis record for booking %(booking_id)s discarded: booking between hotels without specified pickup time." | A booking with transfer_way "Between Hotels" must have a specified pickup time. If it does not, that will cause this error message. |
| "KR_between_hotels_same_hotels" | "Kinesis record for booking %(booking_id)s discarded: the origin and destination hotels are the same: %(hotel_id)s" | A booking with transfer_way "Between Hotels" must specify 2 different hotels, one as origin and one as destination. If the hotel_id is the same for the origin and destination, that will cause this error message. |
| "KR_no_terminal_exists"         | "Kinesis record for flight %(flight_id)s discarded: the flight is non-existent and no flight can be created because no terminal is found in master data" | **TODO: revisit this error message, since this error seems to be a booking discard when we can't make a dummy flight. Not informative for TUI.** |

### Kinesis Rejection Errors for Flights

| message_id                 | Message                                                      | Description                                            |
| -------------------------- | ------------------------------------------------------------ | ------------------------------------------------------ |
| "KR_non_existing_terminal" | "Kinesis record for flight %(flight_id)s discarded: Non-existing terminal %(terminal_id)s referenced" | Terminal_id did not match a terminal_id in master data |
|                            |                                                              |                                                        |

### (Kinesis Rejection Error To Dos)

**TODO: Is this the complete set of kinesis rejection errors? Isn't it possible that any of the required fields is missing at this point? Do we not validate we have all the required fields?** 

**TODO: Document new format for single SNS message with multiple error messages.**

**TODO: Auto-generate error messages & descriptions, so that this won't get out of date. Add descriptions from this documentation into the code.** 

**TODO: Cross-check kinesis rejection errors with which fields we've specified as required vs optional.**

**TODO: Document these error messages in another place & recategorize in our code - they seem to be not actually kinesis_rejection errors**

 {    "category": "kinesis_rejection",    "message_id": "KR_booking_not_discarded_unassigned",    "message": "Solver was not able to plan booking %(booking_id)s: it was not discarded, but does not have a trip assigned!",    "description": **null**,    "solution": **null**  }

 {    "category": "kinesis_rejection",    "message_id": "KR_booking_vehicle_assignment_issue",    "message": "Solver expects to be able to assign a vehicle to booking %(booking_id)s but none is valid.",    "description": **null**,    "solution": **null**  }



# Regular Planning

## When Bookings Get Planned

- When a booking comes in via the Kinesis stream, it gets ingested but not planned until the "planning window" for the relevant destination. 
- The planning window varies by destination. For many destinations, the planning window begins 7 days prior to the date of the transfer, and ends 24 hours before the transfer.
- During the planning window for a particular destination & operation date, we check every 5 minutes if there have been any changes to bookings or flights. If we do see changes, we do a replan including all the bookings for the destination & operation date.
- If a booking is locked, it will not be replanned. 

## Where The Planning Window is Specified

- The start of the planning window for a particular destination is specified by the first_planning_time fields, as part of **Parameters**
  - First planning time pickups
  - First planning days pickups
  - First planning time dropoffs
  - First planning days dropoffs
- The end of the planning window for a particular destination is specified by the stop_free_replan fields, as part of the **Destination**
  - Stop free replan time pickups
  - Stop free replan days pickups
  - Stop free replan time dropoffs
  - Stop free replan time dropoffs

## Overview of How Planning Works

1. Mobi collects data as it streams in via Kinesis (bookings & flights)
2. Mobi performs data validation, making sure the data makes sense and is plannable. An error message will be sent if there is an issue with bookings such that they can't be planned.
3. Every 5 minutes, the solver will check to see if anything needs to be planned, and plan if needed. If the planning window for bookings begins, that will trigger those bookings to be planned. If there are changes to bookings or flights within their planning window, that will also trigger planning.
4. The solver breaks the plan into “atomic” parts, so that it can use a divide-and-conquer approach to decrease runtime while also not compromising optimality.
5. The solver makes an base initial solution that satisfies the business constraints
6. The solver rapidly uses a mix of algorithms built from Mobi’s AI expertise and algorithms designed from insights from planning experts to make perturbations to the current solution, improving it until we can no longer do so
7. Once we have the base plan, we fill out any auxiliary information such as timetables, pickup/dropoff data, etc
8. We perform validation checks to ensure our solution meets basic functional requirements as well as conforms to the clients business constraints
9. We send our updated plans live back to the client

## Pre-planning Data Validation

There are multiple categories of issues that will prevent planning. The endpoint **GET /tui-cps/v1/messages** can be used to retrieve a complete set of possible error messages, across all categories. This section describes one category of errors: **preprocessing** errors.

At the start of planning, the system runs preprocessing checks to ensure that the bookings are viable for planning. If an error is found, a Booking Discard message is sent via AWS SNS. These messages generally begin with "BD_" where BD stands for Booking Discards.

When a preprocessing error occurs & a Booking Discard message is sent, the booking is not actually discarded. The booking is ignored during regular planning until it is altered (an updated version is sent either via the Kinesis stream or via API call) or a related booking needs to be planned. **(TODO: define this, give example.**)



| message_id                             | Message                                                      |
| -------------------------------------- | ------------------------------------------------------------ |
| "BD_no_hotel"                          | "Missing needed hotel"                                       |
| "BD_zero_pax_booking"                  | "Listed with 0 seated passengers."                           |
| "BD_no_area_price"                     | "Booking %(booking_id)s has no price object. Needs price from origin %(origin)s to destination %(destination)s." |
| "BD_missing_ferry_bookings"            | "Associated ferry trip legs are missing"                     |
| "BD_missing_ferry_terminal"            | "Missing the destination/origin point associated with the ferry stop." |
| "BD_missing_feeder_meeting_point"      | "Booking is a feeder but area has no feeder meeting point."  |
| "BD_missing_feeder_main_meeting_point" | "Main booking for feeder is missing the main meeting point"  |
| "BD_dummy_hotel"                       | "Hotel %(hotel_name)s of id %(hotel_id)s is unknown to the planner. It is using the Dummy hotel placeholder" |
| "BD_wrong_coordinates"                 | "Hotel %(hotel_name)s of id %(hotel_id)s has coordinates of 0,0. It is using the Dummy hotel placeholder" |
| "BD_duplicated_locations"              | "Hotel %(hotel_name)s of id %(hotel_id)s has the same location of %(coordinates)s as some other hotels" |
| "BD_dummy_flight"                      | "Flight %(flight_id)s of booking %(booking_id)s is unknown to the planner. It is using flight placeholder info" |
| "BD_invalid_vehicle_type"              | "message": "Vehicle type %(vehicle_type)s does not exists."  |
| "BD_too_large_for_hotel"               | "Listed with %(pax_amount)s seated passengers, but its hotel can only take buses of size %(hotel_capacity)s, vehicles constrained to capacity %(vehicle_capacity)s. Available seats are: %(available_seats)s" |
| "BD_no_vehicle"                        | "There is no vehicle that can accommodate it."               |
| "BD_split_needed"                      | "There is no vehicle that can accommodate it."               |
| "BD_no_ferry"                          | "Unable to assign a ferry to booking %(booking_id)s"         |
| "BD_hotel_invalid"                     | "booking's hotel order is in a circular"                     |
| "BD_feeder_criteria_setup"             | "booking hotel area has two feeder criteria set"             |
| "BD_between_hotels_same_hotels"        | "booking %(booking_id)s between hotels with same destination and origin" |
| "PRIVATE_FEEDER_MAIN_DISCARDED"        | "Not using feeders with the following bookings as they are marked as exclusive: %(bookings)s" |
| "BD_no_more_vehicles"                  | "Unable to satisfy vehicle quantity limits for booking %(booking_id)s" |
| "BD_no_prev_next"                      | "Previous or next transfer booking discarded"                |
| "BD_missing_feeder_booking"            | "Main booking is missing the associated feeder booking."     |
| "BD_missing_main_booking"              | "Feeder booking is missing the associated main booking."     |
| "BD_main_not_configured_for_feeder"    | "Feeder booking is missing a properly-configured main booking" |
| "BD_ftaa_flaw"                         | "Some flaw with flight_terminal_airport_area."               |

## Plan Output

Plans are sent back as records in a AWS Kinesis data stream, each record representing a trip. Records include metadata and a payload. 

Each trip has a unique transfer_id. If a trip is updated or deleted during regular planning or via API calls, a record will be sent with the same transfer_id as a trip that has been sent previously.

**Metadata specifies:**

- content-type: vnd.response-planning-event.v1
- operation: saved or deleted

**Data includes the following fields:**

| Field                | Type   | Description                                                  | Example                                |
| -------------------- | ------ | ------------------------------------------------------------ | -------------------------------------- |
| transfer_id          | string | Unique id for each trip                                      | "8496be06-656f-4403-ae67-b38fa8c5874c" |
| date                 | string | Date of transfer (YYYY-MM-DD)                                | "2024-01-25"                           |
| destination_id       | string | Associated destination                                       | "5006"                                 |
| bookings             | list   | List of bookings, by booking_id                              | ["ASX-5006-1834847-3"]                 |
| vehicle_id           | int    | Type of vehicle                                              | "5006-VAN 8-MX0-V-10080"               |
| vehicle_sign         | string | Sign number for the vehicle (auto-generated 1-99)            | "29"                                   |
| transfer_way         | enum   | Whether a flight is an arrival to a Destination, or a departure from a Destination. Possible values: "arrival", "departure". | "arrival"                              |
| rules                | [int]  | The set of parameters applicable to this trip                | [233]                                  |
| duration             | int    | Duration of trips in minutes, estimated based on local speed limits | 55                                     |
| distance             | int    | Distance in meters                                           | 58922                                  |
| routes               | list   | List of routes. Routes are defined below.                    |                                        |
| remarks              |        |                                                              |                                        |
| combinable           | bool   | Whether this trip contains combinable bookings               | "true"                                 |
| welfare              | bool   | Whether the group needs a handicap-accessible vehicle. Handicap-accessible vehicles will only be assigned to bookings where this field is set to true. | "false"                                |
| exclusive_to         | bool   | Indicates if the trip was planned by an exclusive tour operator | "false"                                |
| change_origin        | enum   | What triggered the planning that output this plan. Options: Solver, Solver-trigger, Solver-replan, Insertion-replan, Dashboard, Supplier, Booking change response, Flight change response | "Dashboard"                            |
| username             | string | Username that made the change that triggered the plan, if applicable | "tui-adfs thamara.villagran@tui.com"   |
| locked               | bool   | Whether this trip is locked and will not be changed in regular planning | "false"                                |
| feasible             | bool   | Whether the trip conforms to applicable business rules. Trips can be made that violate business rules using force_infeasible. | "false"                                |
| infeasibility_reason | string | Infeasible messages relevant to this trip                    | see example below                      |
| total_pax            | int    | Total passengers on this trip                                | 5                                      |
| free_seats           | int    | Number of seats not occupied in vehicle                      | 3                                      |
| total_seats          | int    | Total seats in the vehicle                                   | 8                                      |
| available_seats      | int    | Number of free_seats available for passenger usage. When COVID-19 restrictions were in place limiting the % of capacity that could be filled, available_seats was less than free_seats. | 3                                      |



**Routes include the following fields:**

| Field                   | Type     | Description                                                  | Example                                |
| ----------------------- | -------- | ------------------------------------------------------------ | -------------------------------------- |
| stop_order              | int      | Number of the stop, in order. First stop is 0, then stops increment by 1. | 0                                      |
| date_time               | datetime | Datetime of stop                                             | "2024-01-25T15:55:00+00:00"            |
| stop_id                 | string   | Unique id for this stop on this trip                         | "b1db04fa-719f-470f-b06f-55b03af0c260" |
| feeder_meeting_point    | bool     | Whether the stop is a feeder meeting point, where other vehicles will pick up or drop off passengers for these bookings | false                                  |
| pickup_bookings         | list     | List of bookings, by booking_id, that will be picked up at this stop | ["ASX-5006-1834847-3"]                 |
| dropoff_bookings        | list     | List of bookings, by booking_id, that will be dropped off at this stop | ["ASX-5006-1834847-3"]                 |
| point_type              | enum     | One of the following: Shuttle, Hotel, Terminal.              | "Terminal"                             |
| terminal_type           | enum     | If point_type is "Terminal", the type of terminal. Currently "Airport" is the only value. | "Airport"                              |
| terminal_id             | string   | If point_type is Terminal, the id of the terminal            | "CUN-NA"                               |
| stop_hotel_id           | string   | If point_type is Hotel, the id of the hotel. This generally includes the destination. | "5006-15902"                           |
| distance_from_last_stop | int      | Distance from last stop (km)                                 | 59                                     |



**Example Plan Record With 1 Booking:**

{'Data': '{"metadata": {"content-type": "vnd.response-planning-event.v1", "operation": "saved"}, "payload": {"transfer_id": "8496be06-656f-4403-ae67-b38fa8c5874c", "date": "2024-01-25", "destination_id": 5006, "bookings": ["ASX-5006-1834847-3"], "vehicle_id": "5006-VAN 8-MX0-V-10080", "vehicle_sign": "29", "transfer_way": "arrival", "rules": ["233"], "duration": 55, "distance": 58922, **"routes": [{"stop_order": 0, "date_time": "2024-01-25T15:55:00+00:00", "stop_id": "b1db04fa-719f-470f-b06f-55b03af0c260", "feeder_meeting_point": false, "pickup_bookings": ["ASX-5006-1834847-3"], "point_type": "Terminal", "terminal_type": "Airport", "terminal_id": "CUN-NA", "distance_from_last_stop": 0}, {"stop_order": 1, "date_time": "2024-01-25T16:45:00+00:00", "stop_id": "3bf35cb0-7f14-485c-a73c-80c02a085ed7", "feeder_meeting_point": false, "dropoff_bookings": ["ASX-5006-1834847-3"], "point_type": "Hotel", "stop_hotel_id": "5006-15902", "distance_from_last_stop": 59}],** "remarks": [], "combinable": true, "welfare": false, "exclusive_to": false, "change_origin": "Dashboard", "username": "[tui-adfs_thamara.villagran@tui.com](mailto:tui-adfs_thamara.villagran@tui.com)", "locked": true, "feasible": true, "infeasibility_reason": [], "total_pax": 5, "free_seats": 3, "total_seats": 8, "available_seats": 3}



**Example Plan Record with Multiple Bookings:**

[{'Data': '{"metadata": {"content-type": "vnd.response-planning-event.v1", "operation": "saved"}, "payload": {"transfer_id": "26b4036c-4d43-437c-9df9-69b6dd4c1976", "date": "2024-02-01", "destination_id": 5006, "bookings": ["ASX-5006-1852517-1", "ASX-5006-1726004-4", "ASX-5006-1851818-2", "ASX-5006-1849003-2", "ASX-5006-1716845-2", "ASX-5006-1797645-2", "ASX-5006-1849895-2", "ASX-5006-1834914-2"], "vehicle_id": "5006-MINIVAN LUXE-MX0-W-5155", "vehicle_sign": "60", "transfer_way": "departure", "rules": ["236"], "duration": 65, "distance": 43128, **"routes": [{"stop_order": 0, "date_time": "2024-02-01T17:15:00+00:00", "stop_id": "802b983f-9895-42aa-a37c-812781a6afbe", "feeder_meeting_point": false, "pickup_bookings": ["ASX-5006-1726004-4", "ASX-5006-1852517-1"], "point_type": "Hotel", "stop_hotel_id": "5006-443319", "distance_from_last_stop": 0}, {"stop_order": 1, "date_time": "2024-02-01T17:25:00+00:00", "stop_id": "ad97a5c1-0236-48c5-a000-ee3ac0ef7a0a", "feeder_meeting_point": false, "pickup_bookings": ["ASX-5006-1716845-2", "ASX-5006-1797645-2", "ASX-5006-1849003-2", "ASX-5006-1851818-2"], "point_type": "Hotel", "stop_hotel_id": "5006-18925", "distance_from_last_stop": 1}, {"stop_order": 2, "date_time": "2024-02-01T17:35:00+00:00", "stop_id": "a89b5af1-1637-4aa9-9415-694b808bc77f", "feeder_meeting_point": false, "pickup_bookings": ["ASX-5006-1834914-2", "ASX-5006-1849895-2"], "point_type": "Hotel", "stop_hotel_id": "5006-89556", "distance_from_last_stop": 0}, {"stop_order": 3, "date_time": "2024-02-01T18:15:00+00:00", "stop_id": "d59dbae9-90a0-429e-8000-6aada3cced1e", "feeder_meeting_point": false, "dropoff_bookings": ["ASX-5006-1716845-2", "ASX-5006-1726004-4", "ASX-5006-1797645-2", "ASX-5006-1834914-2", "ASX-5006-1849003-2", "ASX-5006-1849895-2", "ASX-5006-1851818-2", "ASX-5006-1852517-1"], "point_type": "Terminal", "terminal_type": "Airport", "terminal_id": "SJD", "distance_from_last_stop": 42}],** "remarks": [], "cost": {"amount": 87.39, "currency_id": "USD"}, "combinable": true, "welfare": false, "exclusive_to": false, "change_origin": "Dashboard", "username": "[tui-adfs_ramon.estrada@ds.destinationservices.com](mailto:tui-adfs_ramon.estrada@ds.destinationservices.com)", "locked": true, "feasible": true, "infeasibility_reason": [], "total_pax": 15, "free_seats": 1, "total_seats": 16, "available_seats": 1}



**Example Plan Marked as Infeasible, with Infeasible Reasons:**

[{'Data': '{"metadata": {"content-type": "vnd.response-planning-event.v1", "operation": "saved"}, "payload": {"transfer_id": "1b0b3358-f7fb-4d89-b736-a4fe33c1a4e7", "date": "2024-01-30", "destination_id": 5002, "bookings": ["ASX-5002-1573584-1", "ASX-5002-1615065-1", "ASX-5002-1475614-1", "ASX-5002-1552413-1"], "vehicle_id": "5002-PENDFLIGHT-DO0-B-4507", "vehicle_sign": "777", "transfer_way": "arrival", "rules": ["163"], "duration": 70, "distance": 39516, **"routes": [{"stop_order": 0, "date_time": "2024-01-30T04:00:00+00:00", "stop_id": "29c4717c-7e19-44ba-ba1b-dbfc73ef1507", "feeder_meeting_point": false, "pickup_bookings": ["ASX-5002-1475614-1", "ASX-5002-1552413-1", "ASX-5002-1573584-1", "ASX-5002-1615065-1"], "point_type": "Terminal", "terminal_type": "Airport", "terminal_id": "PUJ", "distance_from_last_stop": 0}, {"stop_order": 1, "date_time": "2024-01-30T04:20:00+00:00", "stop_id": "d1cee5b7-bacd-4d83-83a1-4779f7490509", "feeder_meeting_point": false, "dropoff_bookings": ["ASX-5002-1573584-1"], "point_type": "Hotel", "stop_hotel_id": "5002-110057", "distance_from_last_stop": 17}, {"stop_order": 2, "date_time": "2024-01-30T04:50:00+00:00", "stop_id": "4d1e6e06-5062-44bf-bdcb-1142f9ccba6c", "feeder_meeting_point": false, "dropoff_bookings": ["ASX-5002-1615065-1"], "point_type": "Hotel", "stop_hotel_id": "5002-89230", "distance_from_last_stop": 19}, {"stop_order": 3, "date_time": "2024-01-30T05:00:00+00:00", "stop_id": "8d040015-550e-474c-8010-70b4d39ea2ca", "feeder_meeting_point": false, "dropoff_bookings": ["ASX-5002-1475614-1", "ASX-5002-1552413-1"], "point_type": "Hotel", "stop_hotel_id": "5002-428818", "distance_from_last_stop": 3}]**, "remarks": [], "combinable": true, "welfare": false, "exclusive_to": false, "change_origin": "Dashboard", "username": "[tui-adfs_aneury.calderon@tui.com](mailto:tui-adfs_aneury.calderon@tui.com)", "locked": true, "feasible": false, "infeasibility_reason": ["T_ERR_015\\tBookings {\'ASX-5002-1475614-1\', \'ASX-5002-1615065-1\', \'ASX-5002-1552413-1\', \'ASX-5002-1573584-1\'} have incompatible areas.", "T_ERR_025\\tTrip 1b0b3358-f7fb-4d89-b736-a4fe33c1a4e7 with 4 flights violates maximum number of grouped flights"], "total_pax": 11, "free_seats": 766, "total_seats": 777, "available_seats": 766}



# APIs: Requesting a Replan, Manual Changes, Warnings, & Errors

## Requesting a Replan

After the freeze date that is configured for the relevant destination (e.g. <2 days prior to planned date), Mobi will not replan automatically even if new bookings, changes to bookings, or changes to flights are received. If a replan is needed then it must be requested via an API call. These replan API calls are triggered by buttons in the Ermes interface.

| Ermes Name | Description                                                  | Use Cases                                                    | API Call                                                     |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| New Plan   | Create new vehicles to handle these bookings. Don’t affect anything already created. | Hub: ?<br />Airport: ?                                       | [POST /tui-cps/v1/bookings/replan](https://shiny-enigma-qklzoe7.pages.github.io/#/bookings/bookings_replan_create) |
| Full Plan  | Using existing vehicles, make as many changes as you want, domino effect is fine | Hub: ?<br />Airport: ?                                       | POST /tui-cps/v1/bookings/domino_replan                      |
| Light Plan | System only inserts bookings into existing vehicles with enough space, but doesn't add vehicles or change existing bookings. | Airport: ?                                                   | POST /tui-cps/v1/bookings/insertion_replan                   |
| [TBD]      | Replan all bookings for the rest of the day without changing vehicles, affecting only passengers & vehicles that have not yet arrived at the airport. | Airport: When a flight gets delayed, ensure the affected passengers get an updated transfer plan, moving other passengers around as needed while minimizing the need to request new vehicles. | TBD                                                          |



## Manual Changes

If the replan buttons cannot meet TUI staff's needs in some circumstances, then they can make manual changes to bookings & trips. These manual changes are made via API calls, which are triggered by buttons & other actions in Ermes.

| Ermes Name                         | Description                                                  | Use Cases                                                    | API Call                                                     |
| ---------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Bulk Assign                        | Assign bookings to existing trips manually                   | Hub: ?<br />Airport: ?                                       | [POST /tui-cps/v1/trips/{id}/bulk_assign_bookings](https://shiny-enigma-qklzoe7.pages.github.io/#/trips/trips_bulk_assign_bookings_create) |
| Edit Parking Number or Sign Number | Using existing vehicles, make as many changes as you want, domino effect is fine | Hub: ?<br />Airport: Assign parking number or change sign number when vehicle arrives to parking, so that airport employees in other places can direct guests to the right parking area and sign number. Especially useful for big airports, replaces the need for calling on walkie talkies and mobile phones to communicate things like this. | [PATCH /tui-cps/v1/trips/{id}](https://shiny-enigma-qklzoe7.pages.github.io/#/trips/trips_partial_update) |
| Create trip                        | Create new trip                                              | Hub: When receiving new bookings or changes to bookings after the freeze date, create a new trip when needed then use bulk assign to assign bookings to that new trip.<br />Airport: ? | [POST /tui-cps/v1/trips](https://shiny-enigma-qklzoe7.pages.github.io/#/trips/trips_create) |
| Lock                               | Lock a trip so that changes cannot be made by replans        | Hub: ?<br />Airport: ?                                       | POST /tui-cps/v1/trips/lock_trips                            |
| Bulk Unassign                      | Unassign bookings from trips manually                        | Hub: ?<br />Airport: ?                                       | [POST /tui-cps/v1/bookings/bulk_unassign](https://shiny-enigma-qklzoe7.pages.github.io/#/bookings/bookings_bulk_unassign_create) |
| Delete Trip                        | Delete a specific trip                                       | Hub: ?<br />Airport: ?                                       | [DELETE /tui-cps/v1/trips/{id}](https://shiny-enigma-qklzoe7.pages.github.io/#/trips/trips_destroy) |

## Invalid & Infeasible Messages

When these APIs are called, there are 3 possible types of responses:

- Success (200) - The API call succeeded and will take effect
- Invalid (?) - The request cannot be completed, because a required field is missing or a critical constraint would be violated. More information about the specific issue will be provided in the response.
- Infeasible (?) - The request will not be completed, because it violates business rules. If the request is re-sent with "force" set to True, the request can be completed. More information about the specific issue will be provided in the response.

Success Example:

Invalid Example:

Infeasible Example:



The endpoint **GET /tui-cps/v1/messages** can be used to retrieve a complete set of possible error messages, across all categories.

### Invalid Messages

**[Internal note: We recently got feedback from Asela that she wants to change which category certain issues are in - is that completed? We should document the new state.]**

| message_id  | Message                                                      | Description |
| ----------- | ------------------------------------------------------------ | ----------- |
| "T_ERR_020" | "Trip %(id)s does not have a vehicle."                       |             |
| "T_ERR_033" | "Trip %(id)s has no start time."                             |             |
| "T_ERR_039" | "Trip %(id)s visits a stop with no coordinates."             |             |
| "T_ERR_029" | "Trip %(id)s does not have a correct route."                 |             |
| "T_ERR_034" | "Trip %(id)s violates the presentation window for booking/s. Correct start time is %(correct_start_time)s given is %(real_start_time)s." |             |
| "T_ERR_042" | "Trip %(id)s has stops that are out of order in time."       |             |
| "T_ERR_030" | "Trip %(id)s does not match the departure/arrival type of its bookings." |             |
| "T_ERR_035" | "Trip %(id)s violates the presentation window for booking %(booking_id)s. Correct latest time is %(correct_last_time)s given is %(real_last_time)s." |             |
| "T_ERR_038" | "Trip %(id)s services %(terminal_amount)s terminal locations instead of 1." |             |
| "T_ERR_043" | "Trip %(id)s has mixed ferry dropoff/pickup bookings."       |             |
| "T_ERR_032" | "Trip %(id)s has booking/s not assigned a dropoff and pickup." |             |

### Infeasible Messages

| message_id                    | Message                                                      | Description                                                  |
| ----------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| "SAME_FLIGHT_HOTEL_VIOLATION" | "The combination %(flight)s %(hotel)s violates the same flight/hotel/trip constraint." | For each destination, you may specify whether all bookings with the same flight and the same hotel should be grouped together in the same trip, where possible. This message indicates that bookings with the same flight and hotel are spread across different trips. This issue should arise only when bookings are manually assigned to a trip. |
| "T_ERR_024"                   | "Trip %(id)s with %(real_time)s violates maximum time limit %(max_time)s." |                                                              |
| "T_ERR_020"                   | "Trip %(id)s does not have a vehicle."                       |                                                              |
| "T_ERR_028"                   | "Trip %(id)s with non-combinable vehicle has combinable booking %(bookings_id)s." |                                                              |
| "T_ERR_038"                   | "Bookings %(ids)s have different terminal locations."        |                                                              |
| "T_ERR_021"                   | "Trip %(id)s has %(pax)s passengers but %(seats)s available seats" |                                                              |
| "T_ERR_025"                   | "Trip %(id)s with %(flights)s flights violates maximum number of grouped flights" |                                                              |
| "T_ERR_031"                   | "Trip %(id)s violates the first stop requirement for booking %(booking_id)s." |                                                              |
| "T_ERR_027"                   | "Trip %(id)s with %(trip_type)s vehicle type and Booking %(booking_id)s requiring %(booking_type)s do not match." |                                                              |
| "T_ERR_037"                   | "Trip %(id)s violates the force dropoff time for bookings %(booking_id)s." |                                                              |
| "T_ERR_022"                   | "Trip %(id)s with %(real_stop)s stops violates the maximum %(max_stop)s stops constraint." |                                                              |
| "T_ERR_026"                   | "Trip %(id)s has a welfare check vehicle but non-welfare bookings." |                                                              |
| "T_ERR_036"                   | "Trip %(id)s violates the force pickup time for booking %(booking_id)s." |                                                              |
| "T_ERR_023"                   | "Trip %(id)s with %(seats)s seats violates hotel %(hotels_name)s seat limit %(limits)s." |                                                              |
| "Infeasible_Booking"          | "Booking %(bid)s in Trip %(tid)s is incompatible because of %(reasons)s" |                                                              |

### Incompatible Messages (a subset of Infeasible Messages)

| message_id  | Message                                                      |
| ----------- | ------------------------------------------------------------ |
| "T_ERR_005" | "Bookings %(ids)s have mismatched transfer ways."            |
| "T_ERR_006" | "Bookings %(ids)s do not have overlapping intervals."        |
| "T_ERR_007" | "Bookings %(ids)s violate hotel exclusivity."                |
| "T_ERR_008" | "Bookings %(ids)s violate hotel area exclusivity."           |
| "T_ERR_009" | "Bookings %(ids)s violate tour operator exclusivity."        |
| "T_ERR_010" | "Bookings %(ids)s violate tour operator hotel exclusivity."  |
| "T_ERR_011" | "Bookings %(ids)s violate tour operator group exclusivity."  |
| "T_ERR_012" | "Bookings %(ids)s violate flight exclusivity."               |
| "T_ERR_013" | "Bookings %(ids)s violate hotel area group exclusivity."     |
| "T_ERR_014" | "Bookings %(ids)s violate flight-cooperator exclusivity."    |
| "T_ERR_015" | "Bookings %(ids)s have incompatible areas."                  |
| "T_ERR_016" | "Bookings %(ids)s have incompatible dossier codes."          |
| "T_ERR_000" | "Bookings %(ids)s are the same booking."                     |
| "T_ERR_001" | "Bookings %(ids)s are uncombinable"                          |
| "T_ERR_017" | "Bookings %(ids)s have incompatible flights due to first flight feature." |
| "T_ERR_002" | "Bookings %(ids)s have incompatible flight times."           |
| "T_ERR_040" | "Bookings %(ids)s have incompatible ferry leg types."        |
| "T_ERR_018" | "Bookings %(ids)s have incompatible force pickup %(attribute)s." |
| "T_ERR_003" | "Bookings %(ids)s are too far apart to be grouped"           |
| "T_ERR_041" | "Feeder bookings %(ids)s have different corresponding main trips." |
| "T_ERR_019" | "Bookings %(ids)s have incompatible force dropoff %(attribute)s." |
| "T_ERR_004" | "Bookings %(ids)s violate terminal exclusivity."             |
