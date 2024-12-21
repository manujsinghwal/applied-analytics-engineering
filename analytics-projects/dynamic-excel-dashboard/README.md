First Class Handicrafts - Logistics and Supply Chain Tracking Dashboard

### Project Overview
This project involves building a dynamic Excel dashboard to track the operational efficiency and the performance of various carriers used by First Class Handicrafts, a small handicraft company based in Delhi, India. The dashboard provides real-time insights into shipment deliveries, delays, and performance comparisons across multiple carriers.
\
The purpose is to help the Ops Team to measure the operational efficiency and Last Mile Team to evaluate carrier efficiency, identify bottlenecks, and make data-driven decisions to improve operations.

### Data Source:
Data is pulled from a MySQL database using Power Query (ODBC connection) into MS Excel.
\
![image](https://github.com/user-attachments/assets/c2f425ad-bc87-42ca-a4ac-ef2d2c42f645)
\
![image](https://github.com/user-attachments/assets/450913bf-fe66-476c-a3b5-873cc50169e0)

The dataset includes the following fields:
1. **ShipmentId:** An unique identification for each shipment.
2. **Carrier:** Carrier name, through which "First Class Handicraft" shipping their products across different regions in India.
3. **CarrierService:** A shipping service type.
4. **OriginCity:** Origin city of the order.
5. **OriginState:** Origin state of the order.
6. **OriginCountry:** Origin country of the order.
7. **DestinationCity:** Destination city for the order.
8. **DestinationState:** Destination state for the order.
9. **DestinationCountry:** Destination country for the order.
10. **ShipmentImportedTimestamp:** When shipment gets imported through website.
11. **ShipmentFulfilledSLATimestamp:** Every imported order have a SLA timestamp, time by which shipment should be fulfilled (picked and packed).
12. **ShipmentFulfilledTimestamp:** Actual shipment fulfilled timestamp, actual time by which shipment was packed and had a label for the carrier and carrier service.
13. **ShipmentCarrierPickUpTimestamp:** Timestamp when shipment picked by the carrier for delivery.
14. **ShipmentExpectedDeliveryTimestamp:** Expected delivery timestamp for shipment based on the carrier service used.
15. **ShipmentDeliveredTimestamp:** Timestamp when shipment gets delivered to the customer.
16. **OrderStatus:** Fulfilled or Cancelled
17. **DeliveryStatus:** Delivered or InTransit
18. **DeliverySubStatus:** Delayed, OnTime or Pending
19. **ImportToFulfilledHours:** Hour difference between order import and order fulfill time.
20. **ImportToPickUpHours:** Hour difference between order import and carrier pick up time.
21. **ImportToDeliverHours:** Hour differece between order import and deliver (to the end customer) time.
22. **FulfilledToPickUpHours:** Hour difference between order fulfill and carrier pick up time.
23. **FulfilledToDeliverHours:** Hour difference between order fulfill and deliver (to the end customer) time.
24. **PickUpToDeliverHours:** Hour difference between carrier pick up and deliver (to the end customer) time.
25. **DeliveryDelayHour:** In case of late delivery, then how many hour we missed of.

### Problem Statement
The Ops and Last Mile Team at First Class Handicraft needs a way to:
1. Monitor every operational activity to understand the average time taken to fulfill orders.
2. Measure compliance with FulfillmentSLA (12 hours) and CarrierPickUpSLA (18 hours). Highlight any deviations from these SLAs to improve process efficiency.
3. Track carrier performance to understand how well different carriers are delivering shipments.
4. Identify delayed shipments and compare the performance of carriers across various regions and service types.
5. Dynamically update and refresh data daily to focus on shipments imported three days before the current date.
6. Filter data interactively to analyze the best and worst performing carriers.

### Dashboard Features
* **Real-Time Data Updates:**
1. Data shown in the dashboard is based on shipments imported three days before the current date to align with carrier service levels (Next Day, Two Day, and Standard (Three Day) Delivery).
2. A refresh button (available in Excel) allows the team to update the data dynamically without needing to re-enter manually.
<img width="959" alt="image" src="https://github.com/user-attachments/assets/9a72ec8f-533a-48a4-a228-4a603992f893" />


* **Carrier Performance Evaluation:**
1. Analyze key metrics:
  \
  i. Total orders delivered vs. total delayed orders.
  \
  ii. Average delay hours for each carrier.
  \
  iii. Best and worst performing carriers in terms of timely deliveries.

### Workflow
1. **Data Import:**
The following SQL query is used to fetch the required last mile shipment data for the dashboard. The query pulls data from the MySQL database for shipments imported three days prior to the current date.

```sql
-- Query: Shipment Data
-- Writer: Manuj Singhwal
-- Description: Fetch last mile data for last 3 days

with shipment_data as (
select
    o.ShipmentId,
    c.Carrier,
    cs.CarrierService,
    l1.City as OriginCity,
    l1.State as OriginState,
    l1.Country as OriginCountry,
    l2.City as DestinationCity,
    l2.State as DestinationState,
    l2.Country as DestinationCountry,
    o.ShipmentImportedTimestamp,
    o.ShipmentFulfilledSLATimestamp,
    o.ShipmentFulfilledTimestamp,
    o.ShipmentCarrierPickUpTimestamp,
    o.ShipmentDeliveredTimestamp,
    os.OrderStatus,
    case
        when o.CarrierServiceId = 11 then date_add(ShipmentCarrierPickUpTimestamp, interval 1 day) -- Next Day Delivery
        when o.CarrierServiceId = 13 then date_add(ShipmentCarrierPickUpTimestamp, interval 2 day) -- Two Day Delivery
        when o.CarrierServiceId = 12 then date_add(ShipmentCarrierPickUpTimestamp, interval 3 day) -- Standard (Three) Day Delivery
    end as ShipmentExpectedDeliveryTimestamp
from fact_orders as o
join dim_carriers as c
	on c.CarrierId = o.CarrierId
join dim_carrier_services as cs
	on cs.CarrierServiceId = o.CarrierServiceId
join dim_locations as l1
	on l1.LocationId = o.OriginLocationId
join dim_locations as l2
	on l2.LocationId = o.DestinationLocationId
join dim_order_status as os
	on os.StatusId = o.StatusId
where date(ShipmentImportedTimestamp) = '2024-10-01' -- We can use date_sub(curdate(), interval 3 day) to get data for general use case
),

final as (
select
    *,
    case
	when ShipmentDeliveredTimestamp is null then 'InTransit'
	else 'Delivered'
    end as DeliveryStatus,
    case
	when ShipmentDeliveredTimestamp is null then 'InTransit'
        when ShipmentDeliveredTimestamp > ShipmentExpectedDeliveryTimestamp then 'Delayed'
        when ShipmentDeliveredTimestamp <= ShipmentExpectedDeliveryTimestamp then 'OnTime'
    end as DeliverySubStatus,
	round(timestampdiff(minute, ShipmentImportedTimestamp, ShipmentFulfilledTimestamp) / 60.0, 2) as ImportToFulfilledHours,
    round(timestampdiff(minute, ShipmentImportedTimestamp, ShipmentCarrierPickUpTimestamp) / 60.0, 2) as ImportToPickUpHours,
    round(timestampdiff(minute, ShipmentImportedTimestamp, ShipmentDeliveredTimestamp) / 60.0, 2) as ImportToDeliverHours,
    round(timestampdiff(minute, ShipmentFulfilledTimestamp, ShipmentCarrierPickUpTimeStamp) / 60.0, 2) as FulfilledToPickUpHours,
    round(timestampdiff(minute, ShipmentFulfilledTimestamp, ShipmentDeliveredTimestamp) / 60.0, 2) as FulfilledToDeliverHours,
    round(timestampdiff(minute, ShipmentCarrierPickUpTimestamp, ShipmentDeliveredTimestamp) / 60.0, 2) as PickUpToDeliverHours
from shipment_data
)

select
	*,
	case
	    when DeliverySubStatus = 'Delayed' then round(timestampdiff(minute, ShipmentExpectedDeliveryTimestamp, ShipmentDeliveredTimestamp) / 60.0, 2)
	    else 0
	end as DeliveryDelayHour
from final;
```

2. **Data Refresh:**
Users can refresh the entire report with a single click using Excel’s Refresh All button to pull updated data from the MySQL database.

3. **Performance Insights:**
The dashboard provides clear insights into which carriers are consistently on time and which are contributing to delivery delays. This enables the team to take corrective actions based on real-time data.

### Final View

1. **Uber View:**
![image](https://github.com/user-attachments/assets/05b47d6a-9ec9-4440-b3f7-5cb0db87f6fe)

2. **Detailed View:**
![image](https://github.com/user-attachments/assets/991a245e-2bff-46fa-8ff6-10154a33ef4b)
