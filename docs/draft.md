





| Category            | Business Rule       | Definition                                                   | Usual Value | Specified Where? |
| ------------------- | ------------------- | ------------------------------------------------------------ | ----------- | ---------------- |
| Customer Experience | Max time in vehicle | A trip can only take X amount of time. Does not include time from the first/last stop to the airport. Only affects time between first and last hotel stops. (Minutes) | 40          | Master Data      |
|                     |                     |                                                              |             |                  |
|                     |                     |                                                              |             |                  |

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

### Master Data Processing Issues

| message_id | Message | Description |
| ---------- | ------- | ----------- |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |
|            |         |             |

# (Notes to self)

We can link to other docs in this repo [another_file.md](./another_file.md)

Miscellaneous questions - check if we covered these:

- Once TUI sends us bookings, how will they know that we successfully planned or didn’t?
- How & when & in what format do we send error messages? (I think we send these via SNS).
- How do we send them back the plans? 
- What happens if they send us too many bookings at once? (like that one time they did this) How many do we expect they will send at maximum in what span of time, so they give us a heads up if they ever plan to send more?