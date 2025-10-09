# Protocol Documentation
<a name="top"></a>

## Table of Contents

- [Protocol Documentation](#protocol-documentation)
  - [Table of Contents](#table-of-contents)
  - [V2C/VehicleMessageHeader.proto](#v2cvehiclemessageheaderproto)
    - [VehicleMessageHeader](#vehiclemessageheader)
- [Vehicle Message Header](#vehicle-message-header)
  - [Message Orchestration](#message-orchestration)
  - [MQTT Topic Design](#mqtt-topic-design)
    - [lat\_long](#lat_long)
  - [V2C/VehiclePrecisionLocation.proto](#v2cvehicleprecisionlocationproto)
- [Precision Vehicle Location](#precision-vehicle-location)
  - [Message Orchestation](#message-orchestation)
  - [MQTT Topic Design](#mqtt-topic-design-1)
    - [PublishPreciseVehicleLocation](#publishprecisevehiclelocation)
    - [RequestPreciseVehicleLocation](#requestprecisevehiclelocation)
    - [ResponseCondition](#responsecondition)
    - [ResponsePreciseVehicleLocation](#responseprecisevehiclelocation)
    - [ResponseCondition.resultsEnum](#responseconditionresultsenum)
    - [gpsFixEnum](#gpsfixenum)
  - [Scalar Value Types](#scalar-value-types)



<a name="V2C_VehicleMessageHeader-proto"></a>
<p align="right"><a href="#top">Top</a></p>

## V2C/VehicleMessageHeader.proto



<a name="com-vehicle-messages-VehicleMessageHeader"></a>

### VehicleMessageHeader
# Vehicle Message Header

This message defines an application message header for messages past across the system. This is useful because the standard MQTT message headers are typically local to the broker of the system, so while the MQTT headers 
are useful for QoS assurances and message debugging they do not necessarily correlate the messages to the 
services deeper in the vehicle or the cloud services. 

## Message Orchestration
![HeaderMessage.puml](build/resources/main/V2C/images/HeaderMessage.png)

## MQTT Topic Design
| Direction        | Subscribe Topic | Publish Topic | 
| -----------      | -----           | --------      |
| Vehicle to Cloud | No Topic        | No Topic      |
| -----------      | -----           | --------      |
| Cloud to Vehicle | No Topic        |  No Topic     |


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| message_id | [int32](#int32) |  | Unique Application message_id. When initiated from channels like Mobile or API Gateways this should persist all the way to the vehilce, returning as a correlation id. |
| correlation_id | [int32](#int32) |  | For request/response and other multi-message patterns this should be populated with the message_id of the first message in the chain. |
| vehicle_id | [string](#string) |  | this can be any unique identifier for the vehicle, we recommend using the fingerprint on the client&#39;s unique x.509 certificate. |
| message_timestamp | [int64](#int64) |  | EPOCH timestamp when the message was created |
| protocol_version | [double](#double) |  | version of the protocol schema/data model being used. |
| location | [lat_long](#com-vehicle-messages-lat_long) |  | GNSS latitude and longtitude |






<a name="com-vehicle-messages-lat_long"></a>

### lat_long



| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| latitude | [double](#double) |  | GNSS latitude |
| longitude | [double](#double) |  | GNSS longitude |





 

 

 

 



<a name="V2C_VehiclePrecisionLocation-proto"></a>
<p align="right"><a href="#top">Top</a></p>

## V2C/VehiclePrecisionLocation.proto
# Precision Vehicle Location 
These messsages provide the complete recommendations for GNSS vehicle location data. This can be used to provide vehicle location for a number of back end services that require no other processing, such as looking up nearest points of      interest. For specific applications like Charge Station location and dynamic mapping a seperate service that requires   more data would be used and will be defined in this specification. 

## Message Orchestation
Coming Soon 

## MQTT Topic Design
| Direction        | Subscribe Topic | Publish Topic | 
| -----------      | -----           | --------      |
| Vehicle to Cloud | TBD             | TBD           |
| -----------      | -----           | --------      |
| Cloud to Vehicle | TBD             | TBD           |


<a name="com-vehicle-messages-PublishPreciseVehicleLocation"></a>

### PublishPreciseVehicleLocation



| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| vehicleMessageHeader | [VehicleMessageHeader](#com-vehicle-messages-VehicleMessageHeader) |  |  |
| latitude | [double](#double) |  | Position Latitude in degrees. Use NaN if not populated, during Privacy Mode |
| longitude | [double](#double) |  | Position Longitude in degrees. Use NaN if not populated, during Privacy Mode |
| estimated_position_error | [int32](#int32) |  | Estimated error in Meters in a radius around the lat/lon position. If GPS fix is not current, then estimated error should be an error value (e.g., -1 or 0x80000000). |
| position_altitude | [double](#double) |  | Position Altitude. |
| gps_fix_typeEnum | [gpsFixEnum](#com-vehicle-messages-gpsFixEnum) |  | Indicates the type of GPS Fix |
| is_gps_fix_available | [bool](#bool) |  | False when the FixType is ID_FIX_NO_POS[0] or ID_FIX_ESTIMATED[3] |
| estimated_altitude_error | [int32](#int32) |  | Estimated error in Meters above or below altitude if GPS Fix is current. |
| position_direction | [double](#double) |  | Direction the vehicle is traveling. |






<a name="com-vehicle-messages-RequestPreciseVehicleLocation"></a>

### RequestPreciseVehicleLocation



| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| vehicleMessageHeader | [VehicleMessageHeader](#com-vehicle-messages-VehicleMessageHeader) |  | The request when coming from the cloud, requires only basic header information |






<a name="com-vehicle-messages-ResponseCondition"></a>

### ResponseCondition



| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| result | [ResponseCondition.resultsEnum](#com-vehicle-messages-ResponseCondition-resultsEnum) |  |  |
| reason_code | [string](#string) |  |  |






<a name="com-vehicle-messages-ResponsePreciseVehicleLocation"></a>

### ResponsePreciseVehicleLocation



| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| vehicleMessageHeader | [VehicleMessageHeader](#com-vehicle-messages-VehicleMessageHeader) |  |  |
| response_condition | [ResponseCondition](#com-vehicle-messages-ResponseCondition) |  |  |
| latitude | [double](#double) |  | Position Latitude in degrees. Use NaN if not populated, during Privacy Mode |
| longitude | [double](#double) |  | Position Longitude in degrees. Use NaN if not populated, during Privacy Mode |
| estimated_position_error | [int32](#int32) |  | Estimated error in Meters in a radius around the lat/lon position. If GPS fix is not current, then estimated error should be an error value (e.g., -1 or 0x80000000). |
| position_altitude | [double](#double) |  | Position Altitude. |
| gps_fix_typeEnum | [gpsFixEnum](#com-vehicle-messages-gpsFixEnum) |  | Indicates the type of GPS Fix |
| is_gps_fix_available | [bool](#bool) |  | False when the FixType is ID_FIX_NO_POS[0] or ID_FIX_ESTIMATED[3] |
| estimated_altitude_error | [int32](#int32) |  | Estimated error in Meters above or below altitude if GPS Fix is current. |
| position_direction | [double](#double) |  | Direction the vehicle is traveling. |





 


<a name="com-vehicle-messages-ResponseCondition-resultsEnum"></a>

### ResponseCondition.resultsEnum


| Name | Number | Description |
| ---- | ------ | ----------- |
| SUCCESS | 0 |  |
| FAILED_SYSTEM_ERROR | 1 |  |
| FAILED_LOCATION_UNAVAILABLE | 2 |  |



<a name="com-vehicle-messages-gpsFixEnum"></a>

### gpsFixEnum


| Name | Number | Description |
| ---- | ------ | ----------- |
| ID_FIX_NO_POS | 0 | The GNSS fix position is not fixed. |
| ID_FIX_3D | 1 | 3-Dimensional position fix |
| ID_FIX_2D | 2 | 2-Dimensional position fix |
| ID_FIX_ESTIMATED | 3 | Estimated (i.e. forward predicted) position fix. |


 

 

 



## Scalar Value Types

| .proto Type | Notes | C++ | Java | Python | Go | C# | PHP | Ruby |
| ----------- | ----- | --- | ---- | ------ | -- | -- | --- | ---- |
| <a name="double" /> double |  | double | double | float | float64 | double | float | Float |
| <a name="float" /> float |  | float | float | float | float32 | float | float | Float |
| <a name="int32" /> int32 | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint32 instead. | int32 | int | int | int32 | int | integer | Bignum or Fixnum (as required) |
| <a name="int64" /> int64 | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint64 instead. | int64 | long | int/long | int64 | long | integer/string | Bignum |
| <a name="uint32" /> uint32 | Uses variable-length encoding. | uint32 | int | int/long | uint32 | uint | integer | Bignum or Fixnum (as required) |
| <a name="uint64" /> uint64 | Uses variable-length encoding. | uint64 | long | int/long | uint64 | ulong | integer/string | Bignum or Fixnum (as required) |
| <a name="sint32" /> sint32 | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int32s. | int32 | int | int | int32 | int | integer | Bignum or Fixnum (as required) |
| <a name="sint64" /> sint64 | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int64s. | int64 | long | int/long | int64 | long | integer/string | Bignum |
| <a name="fixed32" /> fixed32 | Always four bytes. More efficient than uint32 if values are often greater than 2^28. | uint32 | int | int | uint32 | uint | integer | Bignum or Fixnum (as required) |
| <a name="fixed64" /> fixed64 | Always eight bytes. More efficient than uint64 if values are often greater than 2^56. | uint64 | long | int/long | uint64 | ulong | integer/string | Bignum |
| <a name="sfixed32" /> sfixed32 | Always four bytes. | int32 | int | int | int32 | int | integer | Bignum or Fixnum (as required) |
| <a name="sfixed64" /> sfixed64 | Always eight bytes. | int64 | long | int/long | int64 | long | integer/string | Bignum |
| <a name="bool" /> bool |  | bool | boolean | boolean | bool | bool | boolean | TrueClass/FalseClass |
| <a name="string" /> string | A string must always contain UTF-8 encoded or 7-bit ASCII text. | string | String | str/unicode | string | string | string | String (UTF-8) |
| <a name="bytes" /> bytes | May contain any arbitrary sequence of bytes. | string | ByteString | str | []byte | ByteString | string | String (ASCII-8BIT) |

