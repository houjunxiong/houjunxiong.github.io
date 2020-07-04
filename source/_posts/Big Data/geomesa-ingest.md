---
title: Geomesa 导入矢量数据
date: 2016-08-30 18:36:07
categories: Spatial-Temporal Big Data
tags:
 - GeoMesa
 - Distribute Computing
 - Spatial-Temporal Big Data
---
本文以纽约市出租车数据为例，描述了向Geomesa中导入矢量数据的过程。

## 编写数据导入配置文件

配置文件位于$GEOMESA_HOME/conf/sfts目录下，编写相应文件夹（例如nyctaxi）下的reference.conf即可，以纽约市出租车数据为例，其配置文件如下：

```json
geomesa {
  sfts {
    nyctaxi = {
      attributes = [
		{ name = "trip_id", type = String, index = true }
		{ name = "medallion", type = Integer }
		{ name = "hack_license", type = Integer }
		{ name = "vender_id", type = String }
		{ name = "rate_code", type = String }
		{ name = "pickup_dropoff", type = String } 
		{ name = "dtg", type = Date }
		{ name = "geom", type = Point, srid = 4326, index = true }
	]      
	user-data = {
        geomesa.table.sharing = "false"
        table.indexes.enabled = "records,z3,attr_idx"
      }
    }
    // alternative: both pickup and dropoff points in a single feature
    nyctaxi-single = {
      attributes = [
        { name = "trip_id", type = String, index = true }
        { name = "medallion", type = Integer }
        { name = "hack_license", type = Integer }
        { name = "vender_id", type = String }
        { name = "rate_code", type = String }
        { name = "pickup_dtg", type = Date, index = true }
        { name = "pickup_point", type = Point, srid = 4326, index = true }
        { name = "dropoff_dtg", type = Date }
        { name = "dropoff_point", type = Point, srid = 4326 }
      ]
    }
    nyctaxi-trip = {
      attributes = [
        { name = "medallion", type = String, index = true }
        { name = "hack_license", type = String, index = true }
        { name = "vender_id", type = String, index = true }
        { name = "rate_code", type = String, index = true }
	{ name = "store_and_fwd_flag", type = String, index = true }
	{ name = "passenger_count", type = Integer, index = true }
	{ name = "trip_time_in_secs", type = Integer, index = true }
	{ name = "trip_distance", type = Double, index = true }
        { name = "pickup_dtg", type = Date, index = true }
        { name = "pickup_point", type = Point, srid = 4326, index = true }
        { name = "dropoff_dtg", type = Date, index = true }
        { name = "dropoff_point", type = Point, srid = 4326, index=true }
      ]
    }
  }  
converters {
    // default converter is the pickup point only
    nyctaxi = {
      type = "delimited-text"
      format = "CSV"
      id-field = "$trip_segment_id"
      options {
        skip-lines = 1 // header per file
      }
      fields = [
        { name = "trip_segment_id", transform = "md5(stringToBytes(concatenate($0, ',pickup')))"}
        { name = "trip_id", transform = "md5(stringToBytes($0))" }
        { name = "medallion", transform = "$1::integer" }
        { name = "hack_license", transform = "$2::integer" }
        { name = "vender_id", transform = "$3" }
        { name = "rate_code", transform = "$4" }
        { name = "pickup_dropoff", transform = "'pickup'" }
        { name = "dtg", transform = "date('YYYY-MM-dd HH:mm:ss', $6)" } // for pickup $6, dropoff $7
        { name = "geom", transform = "point($11::double, $12::double)" } // for pickup $11,$12; for dropoff $13,$14
      ]
    }
  nyctaxi-drop = {
    type = "delimited-text"
    format = "CSV"
    id-field = "$trip_segment_id"
    options {
      skip-lines = 1 // header per file
    }
    fields = [
      { name = "trip_segment_id", transform = "md5(stringToBytes(concatenate($0, ',dropoff')))"}
      { name = "trip_id", transform = "md5(stringToBytes($0))" }
      { name = "medallion", transform = "$1::integer" }
      { name = "hack_license", transform = "$2::integer" }
      { name = "vender_id", transform = "$3" }
      { name = "rate_code", transform = "$4" }
      { name = "pickup_dropoff", transform = "'dropoff'" }
      { name = "dtg", transform = "date('YYYY-MM-dd HH:mm:ss', $7)" } // for pickup $6, dropoff $7
      { name = "geom", transform = "point($13::double, $14::double)" } // for pickup $11,$12; for dropoff $13,$14
    ]
  }
  nyctaxi-single = {
    type = "delimited-text"
    format = "CSV"
    id-field = "$trip_id"
    options {
      skip-lines = 1 // header per file
    }
    fields = [
      { name = "trip_id", transform = "md5(stringToBytes($0))" }
      { name = "medallion", transform = "$1::integer" }
      { name = "hack_license", transform = "$2::integer" }
      { name = "vender_id", transform = "$3" }
      { name = "rate_code", transform = "$4" }
      { name = "pickup_dtg", transform = "date('YYYY-MM-dd HH:mm:ss', $6)" }
      { name = "pickup_point", transform = "point($11::double, $12::double)" }
      { name = "dropoff_dtg", transform = "date('YYYY-MM-dd HH:mm:ss', $7)" }
      { name = "dropoff_point", transform = "point($13::double, $14::double)" }
    ]
  }
  nyctaxi-trip = {
    type = "delimited-text"
    format = "CSV"
    id-field = uuid()
    options {
      skip-lines = 1 // header per file
    }
    fields = [
      { name = "medallion", transform = "$1" }
      { name = "hack_license", transform = "$2" }
      { name = "vender_id", transform = "$3" }
      { name = "rate_code", transform = "$4" }
      { name = "store_and_fwd_flag", transform = "$5" }
      { name = "passenger_count", transform = "$8::integer" }
      { name = "trip_time_in_secs", transform = "$9::integer" }
      { name = "trip_distance", transform = "$10::double" }
      { name = "pickup_dtg", transform = "date('YYYY-MM-dd HH:mm:ss', $6)" }
      { name = "pickup_point", transform = "point($11::double, $12::double)" }
      { name = "dropoff_dtg", transform = "date('YYYY-MM-dd HH:mm:ss', $7)" }
      { name = "dropoff_point", transform = "point($13::double, $14::double)" }
    ]
  }
  } 
}
```

## 使用Ingest命令导入矢量数据

```bash
# 导入矢量数据
geomesa ingest -u root -p root -c cybergis.nyctaxi -s nyctaxi-trip -C nyctaxi-trip alldata/tripdata/trip_data/trip_data_test.csv
```


