

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

![Data Exchange Diagram 1](./attachments/CPSDiagram2.jpg)

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

# Bookings, Flights, Changes, & Errors

**Open Questions for CPS team:**

- (Alice) For vehicle_type (which they can specify if they want to force a specific vehicle type):
  - What are all the options that someone could specify? I see "Van / Minivan" as one example in the example booking.
  - Can someone specify multiple options? Or do they have to pick 1?
- (Alice) Please share an example of this optional field because it's not in the examples below.
  - force_pickup_time/ force_dropoff_time: [datetime] the expected date time of picking up and dropping off.
- (Dan) Check whether fields we don't use will throw an error if not included
- (Dan) Is booking_plan_status something TUI sends us? That doesn't make sense to me. Is this something TUI sends us to indicate whether it's a new booking or an updated booking?
  - booking_plan_status: [choice] in {Pending, Planned} if this booking is already planned.
- (Dan) What needs to be true of the format for us to be able to parse it? Is it a .json blob?
  - Does order of fields not matter? Just checking.
- (Jamie) How does it work if multiple errors are applicable? We return only the first one? Or the list of all applicable? Is it the same for bookings vs APIs?

## Bookings

A booking represents a need for a transfer (a ride in a vehicle) for a group of passengers (1 or more). A group represented in a single booking generally has booked a tour with TUI together, will be on the same flights, and will be staying at the same hotel.

For every group who books a tour with TUI together, there will generally be 2 bookings sent to Mobi because there are 2 transfers. For example, if the group is going to Cancun, there would be the following 2 bookings:

- **Arrival to a destination:** a transfer picks up the group at the Cancun airport & brings them to their hotel
- **Departure from a destination:** a transfer picks up the group at their hotel & brings them to the Cancun airport

Guests that have itineraries involving multiple hotels may have additional transfers, of the "Between Hotels" type.

### Required Fields for All Bookings

| Field            | Type   | Description                                                  | Example                                                      |
| ---------------- | ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| booking_id       | string | Unique id for each booking                                   | "ASX-5006-1813434-2"                                         |
| ext_booking      | string | External booking id. If bookings share this value, then they will be grouped together. | "61902535"                                                   |
| transfer_way     | enum   | Whether a booking is an arrival, a departure, or between hotels. Possible values: "Arrival", "Departure", "Between hotels" | "Departure"                                                  |
| operation_date   | date   | Date of transfer (YYYY-MM-DD)                                | "2024-01-13"                                                 |
| total_pax        | int    | Number of passengers as part of the booking who will need a seat? | 2                                                            |
| destination_id   | string | Associated destination                                       | "5006"                                                       |
| touroperator_id  | string | Associated tour operator, identifying rules this booking needs to obey in planning. Usually includes destination_id. | "5006-205747"                                                |
| combinable       | bool   | Whether the booking can be combined with other bookings (e.g. VIP bookings cannot be combined) | "true"                                                       |
| flight_exclusive | bool   | Whether the flight cannot be combined with other flights (e.g. ??) | "false"                                                      |
| welfare          | bool   | Whether the group needs a handicap-accessible vehicle. Handicap-accessible vehicles will only be assigned to bookings where this field is set to true. | "false"                                                      |
| passengers       | dict   | Includes passenger_id (int starting from 1), name (string), & age (int). Upon ingestion of the booking, ages of passengers are checked against min_age (specified in master data per tour operator). By default passengers with age under 2 are assumed to be infants in arms and not require a seat. Mobi does not use the passenger information aside from this age check. | "passengers":[{"passenger_id":1,"name":"Hendrik Rauh","age":54},{"passenger_id":2,"name":"Grit Berghof","age":50}] |

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

If these fields are not sent as part of a booking, we will not send an error. **(Check if this is true)**

| Field                                           | Type   | Description                                                  |
| ----------------------------------------------- | ------ | ------------------------------------------------------------ |
| destination_guest_hotel_id                      | string | Hotel id for the hotel where the guests are staying, provided for arrivals. This is not used, because destination_stop_hotel_id is used instead. |
| origin_guest_hotel_id                           | string | Hotel id for the hotel where the guests are staying, provided for departures. This is not used, because origin_stop_hotel_id is used instead. |
| lead_pax_name                                   | string | Lead passenger for the booking                               |
| origin_point_type/destination_point_type        | enum   | "Hotel" or "Terminal". These are not used because transfer_way already defines what the origin & destination point types are. |
| orgin_terminal_type / destination_terminal_type | enum   | "Airport"                                                    |

### Example Bookings

**Arrival Booking**

{"booking_id":"ASX-5175-347722-1","touroperator_id":"5175-212553","ext_booking":"61902535","lead_pax_name":"HENDRIK,  RAUH","destination_id":"5175","total_pax":2,"combinable":true,"transfer_way":"Arrival","operation_date":"2024-01-24","origin_flight_id":"ASX-5175-31293","origin_point_type":"Terminal","origin_terminal_type":"Airport","destination_point_type":"Hotel","destination_guest_hotel_id":"5175-64985","destination_stop_hotel_id":"5175-64985","flight_exclusive":false,"presentation_window_from":0,"presentation_window_to":0,"booking_plan_status":"Pending","passengers":[{"passenger_id":1,"name":"Hendrik Rauh","age":54},{"passenger_id":2,"name":"Grit Berghof","age":50}],"welfare":false}}]}

**Departure Booking**

{"booking_id":"ASX-5006-1813434-2","touroperator_id":"5006-205747","ext_booking":"WRC1A1BU","lead_pax_name":"SR  NICOLE GRUSZYNSKI","destination_id":"5006","total_pax":2,"combinable":true,"transfer_way":"Departure","operation_date":"2024-01-13","origin_point_type":"Hotel","origin_guest_hotel_id":"5006-7729","origin_stop_hotel_id":"5006-7729","destination_flight_id":"ASX-5006-1333547","destination_point_type":"Terminal","destination_terminal_type":"Airport","flight_exclusive":false,"presentation_window_from":180,"presentation_window_to":180,"booking_plan_status":"Planned","passengers":[{"passenger_id":1,"name":"SR  NICOLE GRUSZYNSKI","age":30},{"passenger_id":2,"name":"SR  NICOLE GRUSZYNSKI","age":30}],"welfare":false}}]}

**Between Hotels Booking**

{"booking_id":"ASX-5006-1835811-1","touroperator_id":"5006-42180","ext_booking":"22077510","lead_pax_name":"MALJONEN  EERO  (L)","destination_id":"5006","total_pax":2,"combinable":false,"transfer_way":"Between hotels","vehicle_type":"Van / Minivan","operation_date":"2024-01-12","origin_point_type":"Hotel","origin_guest_hotel_id":"5006-8183","origin_stop_hotel_id":"5006-8183","destination_point_type":"Hotel","destination_guest_hotel_id":"5006-61586","destination_stop_hotel_id":"5006-61586","flight_exclusive":false,"booking_plan_status":"Pending","passengers":[{"passenger_id":1,"name":"MALJONEN  EERO  (L)","age":30},{"passenger_id":2,"name":"MALJONEN  EERO  (L)","age":30}],"welfare":false}}]

## Flights

Each flight represents a real flight in the world on a specific day. Multiple bookings may correspond to a given flight. Flights generally have the following fields:

- (List out fields)

## Changes

**[Delegate: How do booking changes work? What does it look like when they get sent? Send a booking & the id is the same & we just replace it?]**

vehicle_type error recent (make system tolerate it)

Operations: ([from stream_processors.py])

- Saved
- Deleted
- Locked
- Unlocked

**[To delegate: example booking & booking change]**

## When Bookings Get Planned

What determines when bookings get planned? A field in the master data? What are the most common values?

We plan bookings with the same date & destination together.

## Errors

If there is an error with a booking, this error is sent back to TUI via AWS SNS. An error may be sent immediately after the booking is received, if the booking could not be stored. If the booking can be stored but an error occurs later during planning, the error will be sent when planning occurs (either 14 days prior to the planned date, or soon after receipt of the booking if sent less than 14 days prior to the planned date).



# APIs: Requesting a Replan, Manual Changes, Warnings, & Errors

## Requesting a Replan

After the freeze date that is configured for the relevant destination (e.g. <2 days prior to planned date), Mobi will not replan automatically even if new bookings, changes to bookings, or changes to flights are received. If a replan is needed then it must be requested via an API call. These replan API calls are triggered by buttons in the Ermes interface.

| Ermes Name | Description                                                  | Use Cases                                                    | API Call                                                     |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| New Plan   | Create new vehicles to handle these bookings. Don’t affect anything already created. | Hub: ?<br />Airport: ?                                       | [POST /tui-cps/v1/bookings/replan](https://shiny-enigma-qklzoe7.pages.github.io/#/bookings/bookings_replan_create) |
| Full Plan  | Using existing vehicles, make as many changes as you want, domino effect is fine | Hub: ?<br />Airport: ?                                       | POST /tui-cps/v1/bookings/domino_replan                      |
| Light Plan | System only inserts bookings into existing vehicles with enough space, but doesn't add vehicles or change existing bookings. | Airport: ?                                                   | POST /tui-cps/v1/bookings/insertion_replan                   |
| TBD        | Replan all bookings for the rest of the day without changing vehicles, affecting only passengers & vehicles that have not yet arrived at the airport. | Airport: When a flight gets delayed, ensure the affected passengers get an updated transfer plan, moving other passengers around as needed while minimizing the need to request new vehicles. | TBD                                                          |



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

## Warnings

## Errors

# Master Data

![Data Exchange Diagram 1](./attachments/CPSDiagram1.jpg)

As shown in the image above, one **Destination** can have multiple **AreaGroups**. One **AreaGroup** can have multiple **Areas**, but an **Area** does not have to belong to an **AreaGroup**. One **Area** can have multiple **Airports** and multiple **Hotels**.

Destinations tend to be either islands or broad regions surrounding a major city. Island destinations include Mallorca and Zakynthos. Other destinations include Antalya and Cancún.

*Destination Example*: 
{'id': '5002', 'destination_name': 'Punta Cana', 'company': 'DO0', 'office': 3, 'transport_stations': ['AZS', 'EPS', 'HEX', 'JBQ', 'LRM', 'PLRMP', 'POP', 'PSDG', 'PSDQP', 'PUJ', 'SDQ', 'STI'], 'transport_setup': {'qa_rule_id': 163, 'time_zone': 'America/Santo_Domingo', 'solver_calculation': 'cost', 'presentation_time_average': 'max', 'combinable_sign_from': 1, 'combinable_sign_to': 99, 'non_combinable_sign_from': 200, 'non_combinable_sign_to': 299, 'stop_free_replan_time_pickups': '08:00:00', 'stop_free_replan_days_pickups': 1, 'stop_free_replan_time_dropoffs': '08:00:00', 'stop_free_replan_days_dropoffs': 2, 'real_cost': True, 'round_times': True, 'back_to_back_sign': 'original', 'group_by_hotelflight': True, 'private_veh_capacity_behaviour': 'none'}}

[Full .yaml contents](https://musical-guide-gq4eyjv.pages.github.io/#/)

## Request

## Master Data Contents

## Master Data Update Indicator

# Plans & Changes

## Plans

## Changes

# Next Tasks To Delegate

- Flights (1 hour ish). Alice? Fields with type, description, example - call out if certain fields are required vs optional, and if there are any TUI sends us but does not use. Example flight.

# (Notes to self)

We can link to other docs in this repo [another_file.md](./another_file.md)

Miscellaneous questions - check if we covered these:

- Once TUI sends us bookings, how will they know that we successfully planned or didn’t?
- How & when & in what format do we send error messages? (I think we send these via SNS).
- How do we send them back the plans? 
- What happens if they send us too many bookings at once? (like that one time they did this) How many do we expect they will send at maximum in what span of time, so they give us a heads up if they ever plan to send more?