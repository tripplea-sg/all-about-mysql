# MySQL HeatWave

## HeatWave Performance
```
wget https://downloads.mysql.com/docs/airport-db.tar.gz

tar xvzf airport-db.tar.gz

mysqlsh admin@10.0.1.151:3306 -e 'util.loadDump("airport-db", {threads: 16, deferTableIndexes: "all", ignoreVersion: true, resetProgress:true})'

# on mysql shell

show databases;

use airportdb;

show tables;

SELECT airline.airlinename label, COUNT(*) AS value FROM booking, flight, airline, passengerdetails 
WHERE booking.flight_id = flight.flight_id AND airline.airline_id = flight.airline_id
        AND booking.passenger_id = passengerdetails.passenger_id
        AND country IN ('SWITZERLAND' , 'FRANCE', 'ITALY')
GROUP BY airline.airlinename limit 10;

explain SELECT airline.airlinename label, COUNT(*) AS value FROM booking, flight, airline, passengerdetails 
WHERE booking.flight_id = flight.flight_id AND airline.airline_id = flight.airline_id
        AND booking.passenger_id = passengerdetails.passenger_id
        AND country IN ('SWITZERLAND' , 'FRANCE', 'ITALY')
GROUP BY airline.airlinename;

SELECT NAME, LOAD_STATUS FROM performance_schema.rpd_tables,performance_schema.rpd_table_id WHERE rpd_tables.ID = rpd_table_id.ID;

CALL sys.heatwave_load (JSON_ARRAY("airportdb"),NULL);

SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'rapid_load_progress';

SELECT NAME, LOAD_STATUS FROM performance_schema.rpd_tables,performance_schema.rpd_table_id WHERE rpd_tables.ID = rpd_table_id.ID;

SELECT airline.airlinename label, COUNT(*) AS value FROM booking, flight, airline, passengerdetails 
WHERE booking.flight_id = flight.flight_id AND airline.airline_id = flight.airline_id
        AND booking.passenger_id = passengerdetails.passenger_id
        AND country IN ('SWITZERLAND' , 'FRANCE', 'ITALY')
GROUP BY airline.airlinename limit 10;

explain SELECT airline.airlinename label, COUNT(*) AS value FROM booking, flight, airline, passengerdetails 
WHERE booking.flight_id = flight.flight_id AND airline.airline_id = flight.airline_id
        AND booking.passenger_id = passengerdetails.passenger_id
        AND country IN ('SWITZERLAND' , 'FRANCE', 'ITALY')
GROUP BY airline.airlinename;

SELECT /*+ set_var(use_secondary_engine=off) */ airline.airlinename label, COUNT(*) AS value FROM booking, flight, airline, passengerdetails 
WHERE booking.flight_id = flight.flight_id AND airline.airline_id = flight.airline_id
        AND booking.passenger_id = passengerdetails.passenger_id
        AND country IN ('SWITZERLAND' , 'FRANCE', 'ITALY')
GROUP BY airline.airlinename limit 10;

explain SELECT /*+ set_var(use_secondary_engine=off) */ airline.airlinename label, COUNT(*) AS value FROM booking, flight, airline, passengerdetails 
WHERE booking.flight_id = flight.flight_id AND airline.airline_id = flight.airline_id
        AND booking.passenger_id = passengerdetails.passenger_id
        AND country IN ('SWITZERLAND' , 'FRANCE', 'ITALY')
GROUP BY airline.airlinename limit 10;

explain SELECT airline.airlinename label, COUNT(*) AS value FROM booking, flight, airline, passengerdetails 
WHERE booking.flight_id = flight.flight_id AND airline.airline_id = flight.airline_id
        AND booking.passenger_id = passengerdetails.passenger_id
        AND country IN ('SWITZERLAND' , 'FRANCE', 'ITALY')
GROUP BY airline.airlinename limit 10;

SELECT airline.airlinename label, COUNT(*) AS value FROM booking, flight, airline, passengerdetails 
WHERE booking.flight_id = flight.flight_id AND airline.airline_id = flight.airline_id
        AND booking.passenger_id = passengerdetails.passenger_id
        AND country IN ('SWITZERLAND' , 'FRANCE', 'ITALY')
GROUP BY airline.airlinename limit 10;

\ts

runSql(
"SELECT airline.airlinename label, COUNT(*) AS value FROM booking, flight, airline, passengerdetails WHERE booking.flight_id = flight.flight_id AND airline.airline_id = flight.airline_id AND booking.passenger_id = passengerdetails.passenger_id AND country IN ('SWITZERLAND' , 'FRANCE', 'ITALY') GROUP BY airline.airlinename limit 10", 
, function (result) {
    var options: IGraphOptions = {
        series: [
            {
                id: "myFirstGraph",
                type: "bar",
                yLabel: "Airline Name",
                data: result as IJsonGraphData,
                marginLeft: 10,
                marginBottom: 30,
                yDomain: [0, 30000]
            },
        ],
    };
    Graph.render(options);
});

\sql

SELECT
    airline.airlinename,
    SUM(booking.price) AS price_tickets,
    COUNT(*) AS nb_tickets
FROM
    booking,
    flight,
    airline,
    airport_geo
WHERE
    booking.flight_id = flight.flight_id
        AND airline.airline_id = flight.airline_id
        AND flight.from = airport_geo.airport_id
        AND airport_geo.country = 'UNITED STATES'
GROUP BY airline.airlinename
ORDER BY nb_tickets DESC , airline.airlinename
LIMIT 10;

SELECT /*+ set_var(use_secondary_engine=off) */
    airline.airlinename,
    SUM(booking.price) AS price_tickets,
    COUNT(*) AS nb_tickets
FROM
    booking,
    flight,
    airline,
    airport_geo
WHERE
    booking.flight_id = flight.flight_id
        AND airline.airline_id = flight.airline_id
        AND flight.from = airport_geo.airport_id
        AND airport_geo.country = 'UNITED STATES'
GROUP BY airline.airlinename
ORDER BY nb_tickets DESC , airline.airlinename
LIMIT 10;
```
## HeatWave ML
Login to bastion and download dataset
```
scp opc@150.136.8.32:/home/opc/currency.csv .
```
Login to database using MySQL Shell and import csv
```
mysqlsh <user>@<ip>:<port> --sql

create database forex;
create table forex.currency (trans_date text, indonesia text, thailand text,malaysia text, philipines  text,singapore  text,brunai  text,vietnam  text,myanmar  text,laos text,cambodia  text);

# change mode to JS
\js
util.importTable("currency.csv", {schema: "forex", table: "currency", dialect: "csv", skipRows: 1, showProgress: true})

# change mode to SQL
\sql
alter table forex.currency modify column cambodia float;
alter table forex.currency modify column laos float;
alter table forex.currency modify column vietnam float;
alter table forex.currency modify column brunei float;
alter table forex.currency modify column singapore float;
alter table forex.currency modify column philipines float;
alter table forex.currency modify column malaysia float;
alter table forex.currency modify column thailand float;
alter table forex.currency modify column indonesia float;
alter table forex.currency modify column trans_date

alter table forex.currency add column transdate date;
update forex.currency set transdate=STR_TO_DATE(trans_date, "%d/%m/%y");
alter table forex.currency drop column trans_date;
```
Create train data and test data
```
create table forex.currency_train select * from forex.currency where transdate < '2023-07-06';
create table forex.currency_test select * from forex.currency where transdate >= '2023-07-06';
create table forex.currency_test_result select * from forex.currency where transdate >= '2023-07-06';
alter table forex.currency_test drop column singapore;
```
Train data
```
CALL sys.ML_TRAIN('forex.currency_train', 'singapore',JSON_OBJECT('task', 'regression'), @forex_model);
```
Test model by predicting SG currency rate
```
CALL sys.ML_MODEL_LOAD(@forex_model, NULL);

CALL sys.ML_PREDICT_TABLE('forex.currency_test',@forex_model, 'forex.currency_predictions',NULL);
```
Validate and compare
```
select * from forex.currency_predictions;

select * from forex.currency_test_result;
```
