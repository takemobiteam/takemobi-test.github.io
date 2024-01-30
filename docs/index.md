

[TOC]



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

 {    "category": "kinesis_rejection",    "message_id": "KR_booking_not_discarded_unassigned",    "message": "Solver was not able to plan booking %(booking_id)s: it was not discarded, but does not have a trip assigned!",    "description": **null**,    "solution": **null**  }, 

 {    "category": "kinesis_rejection",    "message_id": "KR_booking_vehicle_assignment_issue",    "message": "Solver expects to be able to assign a vehicle to booking %(booking_id)s but none is valid.",    "description": **null**,    "solution": **null**  },
