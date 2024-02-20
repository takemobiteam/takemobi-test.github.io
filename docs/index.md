Planned structure of documentation:

1. Overview
2. Bookings & Flights (already reviewed)
3. Regular Planning (ready for review now)
4. APIs (ready for review now)
5. Master Data

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



# Regular Planning

## When Bookings Get Planned

- When a booking comes in via the Kinesis stream, it gets ingested but not planned until the "planning window" for the relevant destination.
- The start of the planning window is specified by the first_planning_time fields, as part of **Parameters**
  - First planning time pickups
  - First planning days pickups
  - First planning time dropoffs
  - First planning days dropoffs
- The end of the planning window is specified by the stop_free_replan fields, as part of the **Destination**
  - Stop free replan time pickups
  - Stop free replan days pickups
  - Stop free replan time dropoffs
  - Stop free replan time dropoffs
- During the planning window for a particular destination & operation date, we check every 5 minutes if there have been any changes to bookings or flights. If we do see changes, we do a replan including all the bookings for the destination & operation date.
- If a booking is locked, it will not be replanned. 

**TODO: What are the most common values for the start & end of planning window?**

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
