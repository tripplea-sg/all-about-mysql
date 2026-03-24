# HeatWave Workshop
## Preparation
Open https://apexapps.oracle.com/pls/apex/dbpm/r/livelabs/view-workshop?wid=870 and follow instruction on: </br>
1. Lab-1 Task-1 to create VCN </br>
2. Lab-1 Task-2 to create Bastion Host </br>
3. Configure Bastion Host as per Lab-1 Task-3, and download the data dump using ssh opc@138.2.88.109:/home/opc/hw_workshop.tar.gz </br>
4. Lab-1 Task-4 to create OAC </br>
5. Lab-1 Task-5 to create MDS </br>
6. Lab-2 Task-1 to add HeatWave Cluster </br>
7. Lab-2 Task-2 to configure PAC - OAC </br>

## Run Query Using HeatWave

1. Extract data dump
```
tar -zxvf hw_workshop.tar.gz
```
2. Import to MDS
```
mysqlsh -u<mds_user> -p<mds_password> -h<mds_ip> -P<mds_port> -- util loadDump 'backup' 
```
3. Load Table to HeatWave <\br>
Perform Lab-3 Task-1 step 3 and step 4 </br>
Perform Lab-3 Task-2

## HeatWave Autopilot
Open https://apexapps.oracle.com/pls/apex/r/dbpm/livelabs/run-workshop?p210_wid=882&p210_wec=&session=10537407253097 </br>
Perform Lab-5

## HeatWave ML
Connect to Bastion Host and run MySQL Shell
```
mysqlsh <user>:<password>@<ip>:3306 --sql
```
Perform the following after connect with MySQL Shell
```
show databases; 
select * from ecommerce.sales_data;
select * from ecommerce.stock_master
create database ml;
```
Create Train Data
```
select StockCode, Country, date_format(InvoiceDate,'%M') vMonth, date_format(InvoiceDate, '%Y') vYear, sum(Quantity) from ecommerce.sales_detail group by StockCode, Country, date_format(InvoiceDate,'%M'), date_format(InvoiceDate, '%Y') having sum(Quantity)>0;
select StockCode, Country from (select StockCode, Country, count(1) from (select StockCode, Country, date_format(InvoiceDate,'%M') vMonth, date_format(InvoiceDate, '%Y') vYear, sum(Quantity) from ecommerce.sales_detail group by StockCode, Country, date_format(InvoiceDate,'%M'), date_format(InvoiceDate, '%Y') having sum(Quantity)>0) a group by StockCode, Country having count(1)=13) b;
create table ml.country_product as select StockCode, Country from (select StockCode, Country, count(1) from (select StockCode, Country, date_format(InvoiceDate,'%M') vMonth, date_format(InvoiceDate, '%Y') vYear, sum(Quantity) from ecommerce.sales_detail group by StockCode, Country, date_format(InvoiceDate,'%M'), date_format(InvoiceDate, '%Y') having sum(Quantity)>0) a group by StockCode, Country having count(1)=13) b;
select a.StockCode, a.Country, date_format(InvoiceDate,'%m') vMonth, date_format(InvoiceDate,'%y') vYear, UnitPrice, sum(Quantity) Quantity from ecommerce.sales_detail a, ml.country_product b where (a.StockCode, a.Country)=(b.StockCode, b.Country) group by a.StockCode, a.UnitPrice, a.Country, date_format(InvoiceDate,'%m'), date_format(InvoiceDate,'%y') having sum(Quantity)>0;
create table ml.train_data as select a.StockCode, a.Country, date_format(InvoiceDate,'%m') vMonth, date_format(InvoiceDate,'%y') vYear, UnitPrice, sum(Quantity) Quantity from ecommerce.sales_detail a, ml.country_product b where (a.StockCode, a.Country)=(b.StockCode, b.Country) group by a.StockCode, a.UnitPrice, a.Country, date_format(InvoiceDate,'%m'), date_format(InvoiceDate,'%y') having sum(Quantity)>0;
alter table ml.train_data modify column UnitPrice float; 
alter table ml.train_data modify column Quantity float;
alter table ml.train_data modify column vYear int;
alter table ml.train_data modify column vMonth int;
desc ml.train_data;
```
Train Data with HeatWave autoselect the ML algorithm (model 1)
```
CALL sys.ML_TRAIN('ml.train_data', 'Quantity', JSON_OBJECT('task', 'regression'), @sales_prediction_model);
select @sales_prediction_model;
```
Train Data with induced ML Algorithm (model 2)
```
CALL sys.ML_TRAIN('ml.train_data', 'Quantity', JSON_OBJECT('task', 'regression','model_list', JSON_ARRAY('RandomForestRegressor', 'XGBRegressor')), @sales_prediction_model_select);
select @sales_prediction_model_select;
```
Show ML metadata databases
```
desc ML_SCHEMA_admin.MODEL_CATALOG;
select * from ML_SCHEMA_admin.MODEL_CATALOG where model_metadata not like '%Error%';
```
Explain model 1:
```
CALL sys.ML_MODEL_LOAD('ml.train_data_admin_1670946770', NULL);
CALL sys.ML_EXPLAIN('ml.train_data', 'Quantity', 'ml.train_data_admin_1670946770', JSON_OBJECT('model_explainer', 'shap','prediction_explainer', 'permutation_importance'));
SELECT model_explanation FROM ML_SCHEMA_admin.MODEL_CATALOG WHERE model_handle = 'ml.train_data_admin_1670946770';
```
Explain Model 2:
```
CALL sys.ML_MODEL_LOAD('ml.train_data_admin_1670946900', NULL);
CALL sys.ML_EXPLAIN('ml.train_data', 'Quantity', 'ml.train_data_admin_1670946900', JSON_OBJECT('model_explainer', 'shap','prediction_explainer', 'permutation_importance'));
SELECT model_explanation FROM ML_SCHEMA_admin.MODEL_CATALOG WHERE model_handle = 'ml.train_data_admin_1670946900';
```
Prepare test data:
```
(select StockCode, Country, vMonth, vYear, UnitPrice from (select ROW_NUMBER() OVER(PARTITION BY vYear) AS row_num, a.StockCode, Country, vMonth, vYear, UnitPrice from ml.train_data a where a.Country='Germany') b limit 1) union (select StockCode, Country, vMonth, vYear, UnitPrice from (select ROW_NUMBER() OVER(PARTITION BY vYear) AS row_num, a.StockCode, Country, vMonth, vYear, UnitPrice from ml.train_data a where a.Country='France') b limit 1) union (select StockCode, Country, vMonth, vYear, UnitPrice from (select ROW_NUMBER() OVER(PARTITION BY vYear) AS row_num, a.StockCode, Country, vMonth, vYear, UnitPrice from ml.train_data a where a.Country='EIRE') b limit 1);
```
Create test data table:
```
create table ml.test_data as (select StockCode, Country, vMonth, vYear, UnitPrice from (select ROW_NUMBER() OVER(PARTITION BY vYear) AS row_num, a.StockCode, Country, vMonth, vYear, UnitPrice from ml.train_data a where a.Country='Germany') b limit 1) union (select StockCode, Country, vMonth, vYear, UnitPrice from (select ROW_NUMBER() OVER(PARTITION BY vYear) AS row_num, a.StockCode, Country, vMonth, vYear, UnitPrice from ml.train_data a where a.Country='France') b limit 1) union (select StockCode, Country, vMonth, vYear, UnitPrice from (select ROW_NUMBER() OVER(PARTITION BY vYear) AS row_num, a.StockCode, Country, vMonth, vYear, UnitPrice from ml.train_data a where a.Country='EIRE') b limit 1);
update ml.test_data set vMonth=1, vYear=12;
```
Predict table:
```
CALL sys.ML_MODEL_LOAD('ml.train_data_admin_1670946770', NULL); 
CALL sys.ML_PREDICT_TABLE('ml.test_data', 'ml.train_data_admin_1670946770', 'ml.predict_table_1');
select * from ml.predict_table_1;
```
Explain prediction
```
CALL sys.ML_EXPLAIN_TABLE('ml.test_data','ml.train_data_admin_1670946770','ml.explain_table_1',JSON_OBJECT('model_explainer', 'shap','prediction_explainer', 'permutation_importance'));
select * from ml.explain_table_1;

CALL sys.ML_MODEL_LOAD('ml.train_data_admin_1670946900', NULL);
select * from ml.explain_table_2;
```
## Oracle Analytic Cloud (OAC)
pen https://apexapps.oracle.com/pls/apex/dbpm/r/livelabs/view-workshop?wid=870 and follow instruction on Lab-4
