# delete slug


**Brief Description** 

- delete slug info by external id

**Request URL** 
- `/route-mma/tvu-search/public/v1/media/delete-external-slug`

**Request Method**
- POST 

**Request Example**

```JSON
{
   "externalId": "xxxxx"
}
```

**Parameter** 

|Parameter|Required|Type|Description|
|:----    |:---|:----- |-----   |
| externalId         | Yes       | String | external id |

**Example of Response**

```JSON
{
    "errorCode": "0x0",
    "errorInfo": "success",
    "result": null
}
{
    "errorCode": "0x80100018",
    "errorInfo": "Not exist",
    "result": null
}
```


**Parameters of Response** 

|Parameter|Type|Description|
|:-----  |:-----|----- |
|errorCode |string  | `0x0` means success |
|errorInfo |string  | `success` means success|



