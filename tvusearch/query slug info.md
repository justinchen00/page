# query slug


**Brief Description** 

- query slug info by time

**Request URL** 
- `/route-mma/tvu-search/public/v1/media/slug-info`

**Request Method**
- POST 

**Request Example**

```JSON
{
    "peerId":"1998ffef1f409d3c0000000000000001",
    "startTime":1552708800000,
    "endTime":1636713000000,
    "sort":"DESC"
}
```

**Parameter** 

|Parameter|Required|Type|Description|
|:----    |:---|:----- |-----   |
| peerId         | Yes       | String | source Id |
| startTime            | No       | Long | start time by slug, default previous week  |
| endTime            | No       | Long | end time by slug, default  current time |
| sort            | No       | String |ASC/DESC default ASC |

**Example of Response**

```JSON
{
    "errorCode": "0x0",
    "errorInfo": "success",
    "result": [
        {
            "peerId": "1998ffef1f409d3c0000000000000001",
            "slug": "Attorney Ben Crump, George Floyd's family address federal charges against 4 former officers",
            "createTime": 1620483300000,
            "endTime": 1620486900000
        },
        {
            "peerId": "1998ffef1f409d3c0000000000000001",
            "slug": "White House COVID-19 response team holds news conference",
            "createTime": 1620397800000,
            "endTime": 1620483300000
        },
        {
            "peerId": "1998ffef1f409d3c0000000000000001",
            "slug": "NYC skyline from the Empire State Building to 1 WTC",
            "createTime": 1552708800000,
            "endTime": 1620397800000
        }
    ]
}
```


**Parameters of Response** 

|Parameter|Type|Description|
|:-----  |:-----|----- |
|errorCode |string  | `0x0` means success |
|errorInfo |string  | `success` means success|
|createTime |Long  | slug start time |
|endTime |Long  | slug end time |
|slug |String  | slug info |


