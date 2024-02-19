# Product Overview

The Continuous Planning Service enables TUI’s transfer service to operate efficiently by scheduling, optimizing, and routing TUI’s fleet from advanced pre-planning all the way to day-of operation. 

**[Charlie: Insert product overview diagram]**

# What We're Optimizing For

## Cost Functions

The Continuous Planning System optimizes for cost while following business rules. In order to optimize for cost, we must first define what cost is. Each destination has a specific cost function.

Example 1: Mallorca

Cost = distance travelled



Example 2: Zakynthos

Cost = (# of Vehicle A x cost of vehicle A) + (# of Vehicle B x cost of vehicle B)



In general, destinations where TUI owns their own fleet usually have a simple "distance traveled" cost function like Mallorca, and other destinations that are primarily relying on suppliers have the more complex cost function like Zakynthos.

**[Ask Jacob: Is this how he'd represent Zakynthos cost function?]**

## Business Rules

The Continuous Planning System has been configured with business rules for each destination, in order to comply with local legislation, plan trips that are physically possible given the constraints of the geography, and ensure a great customer experience. The following business rules are applied in most destinations.

| Category            | Business Rule       | Definition                                                   | Usual Value | Specified Where? |
| ------------------- | ------------------- | ------------------------------------------------------------ | ----------- | ---------------- |
| Customer Experience | Max time in vehicle | A trip can only take X amount of time. Does not include time from the first/last stop to the airport. Only affects time between first and last hotel stops. (Minutes) | 40          | Master Data      |
|                     |                     |                                                              |             |                  |
|                     |                     |                                                              |             |                  |

**[To delegate: List all business rules & take a first pass at all values in the table.]**

# Data Exchange Overview

![Data Exchange Diagram 1](/Users/charliefarison/git/takemobi-test-doc/docs/attachments/CPSDiagram2.jpg)

1. **Bookings, Flights, & changes:** TUI sends bookings, flights, & changes to Mobi on a continuous basis via an AWS Kinesis stream. All bookings, flights, & changes are sent this way up until the freeze date (e.g. 2 days prior to planned date).
2. **Booking errors:** If there is an error with a booking, this error is sent back to TUI via AWS SNS. An error may be sent immediately after the booking is received, if the booking could not be stored. If the booking can be stored but an error occurs later during planning, the error will be sent when planning occurs (either 14 days prior to the planned date, or soon after receipt of the booking if sent less than 14 days prior to the planned date).
3. **Manual Changes:** After the freeze date (e.g. <2 days prior to planned date), changes must be made manually via API call. This includes new bookings, changes to bookings, new trips, assigning bookings to trips manually, and requesting a replan.
4. **Manual Change errors:** When manual changes are made via API call, there may be errors, which prevent planning & must be addressed. There may also be warnings (which can be ignored if desired) if the manual changes violate QA rules. Both errors and warnings are sent in the API response.
5. **Master Data Request:** Mobi requests master data from TUI from each destination every 15 minutes, via an API call. This ensures that if there are changes on the TUI side (e.g. *NEED EXAMPLE) the changes will be part of Mobi’s planning. Master data includes elements like (e.g. *NEED EXAMPLES).
6. **Master Data:** In response to Mobi’s API call, TUI sends updated master data. (*DOES TUI SEND ALL OF IT? OR ONLY CHANGES?)
7. **Master Data Update Indicator:** If TUI needs the master data to be updated faster than 15 minutes, TUI can call the Master Data Update Indicator API. (*DO THEY EVER CALL THIS?)
8. **Plans & changes:** Mobi sends plans & changes to TUI via an AWS Kinesis stream. TUI consumes this data and displays it in the Ermes system.
9. **Planning complete message:** Mobi sends a planning complete message via AWS SNS (*DOES TUI USE THIS? WHEN & WHY?)

Each item in this diagram is further described in a section below.

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



## Pre-planning Error Checks

There are multiple categories of errors. The endpoint **GET /tui-cps/v1/messages** can be used to retrieve a complete set of possible error messages, across all categories. This section describes one category of errors: **preprocessing** errors.

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

# Master Data

## Overview

Master Data includes information about physical places and the business rules that should apply to relevant bookings during planning. Business rules specified in the bookings themselves generally override business rules supplied in Master Data. 

## Key Concepts in Master Data

![Data Exchange Diagram 1](/Users/charliefarison/git/takemobi-test-doc/docs/attachments/CPSDiagram1.jpg)

As shown in the image above, one **Destination** can have multiple **Area Groups**. One **Area Group** can have multiple **Areas**, but an **Area** does not have to belong to an **AreaGroup**. One **Area** can have multiple **Airports** and multiple **Hotels**. **Vehicles** are specified per destination.

## Destinations

Destinations tend to be either islands or broad regions surrounding a major city. Island destinations include Mallorca and Zakynthos. Other destinations include Antalya and Cancún.

**Destination Example:** 
{'id': '5002', 'destination_name': 'Punta Cana', 'company': 'DO0', 'office': 3, 'transport_stations': ['AZS', 'EPS', 'HEX', 'JBQ', 'LRM', 'PLRMP', 'POP', 'PSDG', 'PSDQP', 'PUJ', 'SDQ', 'STI'], 'transport_setup': {'qa_rule_id': 163, 'time_zone': 'America/Santo_Domingo', 'solver_calculation': 'cost', 'presentation_time_average': 'max', 'combinable_sign_from': 1, 'combinable_sign_to': 99, 'non_combinable_sign_from': 200, 'non_combinable_sign_to': 299, 'stop_free_replan_time_pickups': '08:00:00', 'stop_free_replan_days_pickups': 1, 'stop_free_replan_time_dropoffs': '08:00:00', 'stop_free_replan_days_dropoffs': 2, 'real_cost': True, 'round_times': True, 'back_to_back_sign': 'original', 'group_by_hotelflight': True, 'private_veh_capacity_behaviour': 'none'}}

## Transport Stations: Airports & Ports

Airports & ports map to airports & ports in the real world. There may be multiple airports & ports in one destination, but many have just one airport. These are specified as "transport stations" in the master data.

**Airport Example:**
{'id': 'AAQ', 'name': 'ANAPA', 'transport_station_type': 'airport'}, {'id': 'AAR', 'name': 'AARHUS', 'transport_station_type': 'airport'}

**Port Example:** 

{'id': '0GRC2', 'name': 'Zakynthos Port', 'transport_station_type': 'harbour'}

### Terminals

Terminals map to airport terminals in the real world. One airport can have multiple terminals, while one terminal can only belong to one airport (specified as transport_station_id).

**Terminal Example:** 

{'id': 'AAT', 'transport_station_id': 'AAT', 'name': 'ALTAY'}



## Hotels

Hotels map to hotels in the real world. There are generally many hotels per destination.

**Hotel Example:**
{'id': '5016-100393', 'area_id': '5016-LAGANAS', 'name': 'Majestic Spa', 'address': ', LAGANAS, 291 00 ZAKYNTHOS, GRECIA', 'destination_id': '5016', 'cmd_id': 'AC18743631', 'transport_setup': {'location': [37.72782089288549, 20.86380683604017], 'exclusive_hotel': False, 'first_stop': False, 'hotel_pickup_setups': [], 'audit_date': '2023-07-26T06:51:40.487941Z'}}

**HotelGroup**: destination-level. One hotelGroup can have multiple hotels but hotel does not have to belong to a HotelGroup. It specifies hotel's wait time.

Example:
{'id': 18, 'name': 'CABINA CASA DE CAMPO', 'destination_id': '5002', 'checkpoint_waiting_time_arrival': 3, 'checkpoint_waiting_time_departure': 5}

**HotelStoppriorities**: destination-level. Determine which hotels are prioritized and visited earlier than other areas in a certain transfer_way trip.

Example:
{'id': '0ce0fb1d-a48b-49b5-9361-4477297d8a94', 'destination_id': '5016', 'transfer_way': 'departure', 'name': 'LAGANAS-KALAM', 'hotels': ['5016-194208', '5016-438487', '5016-438488', '5016-231664', '5016-466222', '5016-400419', '5016-432138', '5016-122142', '5016-183338', '5016-202842', '5016-194212', '5016-355229', '5016-15420', '5016-76715', '5016-15609', '5016-139864', '5016-139862', '5016-8107', '5016-136732', '5016-139863', '5016-356035', '5016-432320', '5016-194215', '5016-194216', '5016-356497', '5016-444382', '5016-354857', '5016-465879', '5016-125775', '5016-15638', '5016-154198', '5016-194219', '5016-15688', '5016-200473', '5016-194222', '5016-438790'], 'enabled': True}



## Vehicles

**Vehicles** map to vehicles available for transporting passengers in the real world, either part of TUI's fleet or available from suppliers.

Each **Destination** has **Vehicles** specified separately. Attributes of a vehicle include type of vehicle, number of seats, quantity of vehicles available. When quantity equals to -1, it means we can consider there are "unlimited" number of this vehicle.

Vehicle Example:
{'id': '5016-ADAPTED VEHICLE-GR0-E', 'destination_id': '5016', 'code': 'ADAPTED VEHICLE', 'name': 'ADAPTED VEHICLE RAMP', 'vehicle_type_id': '5016-E', 'transport_setup': {'show_sign_at': ['none']}}

### Suppliers

Each vehicle has one supplier, and suppliers generally have multiple vehicles.

Suppliers are specified per destination. If the same supplier provides vehicles in multiple destinations, it is specified as a spearate supplier per destination.

(What's the supplier name for TUI-owned fleet?)

**Supplier Example:**
{'id': '5016-4111', 'name': 'TUI HELLAS - ZTH', 'destination_id': '5016', 'sap_code': '0000224308', 'transport_setup': {'live_tracking_enabled': False, 'no_transfer': False, 'allow_multiple_b2b': False}}

### Prices

Many Destinations have cost functions structured like so:

Cost = (# of Vehicle A x cost of vehicle A) + (# of Vehicle B x cost of vehicle B)

In this case, Mobi needs a price for each vehicle. These prices are per X.

Vehicle can have multiple prices while price can also belong to multiple vehicles. It shows price for the linked vehicle to hold passengers (number of the passengers is greater than pax_min and less than pax_max) from origin_point to destination_point (usually area object or terminal object). 

**Price Example:**
{'id': 1262, 'destination_id': '5016', 'origin': 'ZTH', 'origin_type': 'airport', 'destination': '5016-BOCHALI', 'destination_type': 'area', 'date_from': '2022-04-22', 'date_to': '2022-12-31', 'price_type': 'unit', 'min_nro_pax': 4, 'max_nro_pax': 16, 'price': 39.24, 'currency_id': 'EUR'}

## Areas

**Area Example:**
{'id': '5016-AGDIMITRIS', 'description': 'Agios Dimitris', 'destination_id': '5016', 'destination_name': 'ZTH', 'transport_setup': {'exclusive_area': False, 'locks_setup': [], 'audit_date': '2023-04-12T06:36:39.349174Z', 'area_penalty': 0, 'feeder': False, 'feeder_time_in_advance': 0}}

**AreaIncompatibilities**: destination-level. Determine which areas are incompatible with each other when planning a certain transfer_way trip associated with a certain terminal.

Example:
{'id': 687, 'destination_id': '5016', 'transfer_way': 'arrival', 'incompatibility_areas': ['5016-ALYKANAS', '5016-AGSOSTIS'], 'date_from': '2022-11-16', 'date_to': '2030-12-31', 'enabled': True}

**AreaStopPriorities**: destination-level. Determine which areas are prioritized and visited earlier than other areas in a certain transfer_way trip.

Example:
{'id': 'a2851fa1-5cb1-48a1-9954-e03fa4a5bdb4', 'destination_id': '5016', 'transfer_way': 'arrival', 'name': 'ARRIVAL AREA ORDER', 'areas': ['5016-KALAMAKI', '5016-LAGANAS'], 'enabled': True}



## Tour Operators

Tour Operators are organizations that run tours in a specific destination. Business rules are primarily set on a tour operator level, by specifying a set of parameters by qa_rule_id for a given tour operator. Tour operators also specify (what is exclusive_to, what is private_veh_capacity_behaviour)?

Example:
{'id': '5016-212177', 'name': 'ABAX', 'short_name': 'ABAXS', 'destination_id': '5016', 'sap_code': '0001093579', 'client_type': '3p_b2b', 'transport_setup': {'qa_rule_id': 48, 'exclusive_to': 'both', 'private_veh_capacity_behaviour': 'none'}}

### Tour Operator Hotel Configurations

An optional piece of master data that can be sent to specify if this hotel is exclusive (not combinable with bookings from other hotels) or first_stop (the hotel should be prioritized).

**TourOperatorHotelConfigurations Example:**
 {'id': 589, 'touroperator_id': '5032-212614', 'hotel_id': '5032-2933', 'exclusive_hotel': True, 'first_stop': False, 'destination_id': '5032'}



## Parameters 

## Master Data Format

[Reference in .yaml format](https://musical-guide-gq4eyjv.pages.github.io/#/)

## Request

## Master Data Contents

## Master Data Update Indicator

# Plans & Changes

## Plans

Questions for Dan:

- Do the plans always have the same content-type (vnd.response-planning-event.v1)?
- Do the plans always have the same operation (saved)?
- How is duration defined? What are the units? What is the type (e.g. int, long)?
- How is distance defined? What are the units? What is the type (e.g. int, long)?
- How is combinable defined? Combinable bookings makes sense to me, but I'm not sure what it means as a property of a trip.
- What is exclusive_to? How is it defined?
- What are the options for change_origin?
- Is username always populated?
- What’s the difference between free_seats & available_seats?
- What are “rules”? Is that the same thing as Parameters?
- What are the possible values for point_type?
- What are the possible values for terminal_type?
- One of your examples is a locked trip. In what scenario would we send a locked trip to TUI as part of a plan? In my mind locking trips is something that primarily happens on the day of travel via API call
- Can I get an example trip that contains multiple bookings?
- Can I get an example trip that contains remarks?
- Can I get an example trip that has contains infeasible reasons, and ideally multiple infeasible reasons? Presumably a trip will contain infeasible reasons if feasible: false.
- If we replan & send new trips, how does TUI know the old trips are no longer relevant? Struggling to understand how they turn our stream of plan records into a set of things they are displaying.

Plans are sent back as records in a AWS Kinesis data stream. Records include metadata and a payload.

**Metadata specifies:**

- content-type: vnd.response-planning-event.v1
- operation: saved

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
| rules                |        |                                                              | [233]                                  |
| duration             |        |                                                              | 55                                     |
| distance             |        |                                                              | 58922                                  |
| routes               | list   | List of routes. Routes are defined below.                    |                                        |
| remarks              |        |                                                              |                                        |
| combinable           | bool   | Whether this trip can be combined with other trips           |                                        |
| welfare              | bool   | Whether the group needs a handicap-accessible vehicle. Handicap-accessible vehicles will only be assigned to bookings where this field is set to true. | "false"                                |
| exclusive_to         | bool   |                                                              |                                        |
| change_origin        |        |                                                              |                                        |
| username             |        |                                                              |                                        |
| locked               |        |                                                              |                                        |
| feasible             |        |                                                              |                                        |
| infeasibility_reason |        |                                                              |                                        |
| total_pax            |        |                                                              |                                        |
| free_seats           |        |                                                              |                                        |
| total_seats          |        |                                                              |                                        |
| available_seats      |        |                                                              |                                        |



**Routes include the following fields:**

| Field                   | Type     | Description                                                  | Example                                |
| ----------------------- | -------- | ------------------------------------------------------------ | -------------------------------------- |
| stop_order              | int      | Number of the stop, in order. First stop is 0, then stops increment by 1. | 0                                      |
| date_time               | datetime | Datetime of stop                                             | "2024-01-25T15:55:00+00:00"            |
| stop_id                 |          |                                                              | "b1db04fa-719f-470f-b06f-55b03af0c260" |
| feeder_meeting_point    | bool     | Whether the stop is a feeder meeting point, where other vehicles will pick up or drop off passengers for these bookings | false                                  |
| pickup_bookings         | list     | List of bookings, by booking_id, that will be picked up at this stop | ["ASX-5006-1834847-3"]                 |
| dropoff_bookings        | list     | List of bookings, by booking_id, that will be dropped off at this stop | ["ASX-5006-1834847-3"]                 |
| point_type              | enum     |                                                              | "Terminal"                             |
| terminal_type           | enum     | If point_type is "Terminal", the type of terminal            | "Airport"                              |
| terminal_id             | string   | If point_type is Terminal, the id of the terminal            | "CUN-NA"                               |
| stop_hotel_id           | string   | If point_type is Hotel, the id of the hotel. This generally includes the destination. | "5006-15902"                           |
| distance_from_last_stop | int      | Distance from last stop (km)                                 | 59                                     |



**Example Plan Record:**

{'Data': '{"metadata": {"content-type": "vnd.response-planning-event.v1", "operation": "saved"}, "payload": {"transfer_id": "8496be06-656f-4403-ae67-b38fa8c5874c", "date": "2024-01-25", "destination_id": 5006, "bookings": ["ASX-5006-1834847-3"], "vehicle_id": "5006-VAN 8-MX0-V-10080", "vehicle_sign": "29", "transfer_way": "arrival", "rules": ["233"], "duration": 55, "distance": 58922, **"routes": [{"stop_order": 0, "date_time": "2024-01-25T15:55:00+00:00", "stop_id": "b1db04fa-719f-470f-b06f-55b03af0c260", "feeder_meeting_point": false, "pickup_bookings": ["ASX-5006-1834847-3"], "point_type": "Terminal", "terminal_type": "Airport", "terminal_id": "CUN-NA", "distance_from_last_stop": 0}, {"stop_order": 1, "date_time": "2024-01-25T16:45:00+00:00", "stop_id": "3bf35cb0-7f14-485c-a73c-80c02a085ed7", "feeder_meeting_point": false, "dropoff_bookings": ["ASX-5006-1834847-3"], "point_type": "Hotel", "stop_hotel_id": "5006-15902", "distance_from_last_stop": 59}],** "remarks": [], "combinable": true, "welfare": false, "exclusive_to": false, "change_origin": "Dashboard", "username": "[tui-adfs_thamara.villagran@tui.com](mailto:tui-adfs_thamara.villagran@tui.com)", "locked": true, "feasible": true, "infeasibility_reason": [], "total_pax": 5, "free_seats": 3, "total_seats": 8, "available_seats": 3}

# Next Tasks To Delegate

- Flights (1 hour ish). Alice? Fields with type, description, example - call out if certain fields are required vs optional, and if there are any TUI sends us but does not use. Example flight.

# (Notes to self)

We can link to other docs in this repo [another_file.md](./another_file.md)

Miscellaneous questions - check if we covered these:

- Once TUI sends us bookings, how will they know that we successfully planned or didn’t?
- How & when & in what format do we send error messages? (I think we send these via SNS).
- How do we send them back the plans? 
- What happens if they send us too many bookings at once? (like that one time they did this) How many do we expect they will send at maximum in what span of time, so they give us a heads up if they ever plan to send more?