# add/update slug


**Brief Description** 

- add a slug info
- udpate slug info

**Request URL** 
- `/route-mma/tvu-search/public/v1/media/updateslug`

**Request Method**
- POST 

**Request Example**

- add slug request
```JSON
{
    "sourceId": "3e3e1aed3a04c7220000000000000001",
    "slug": "test slug111",
    "startTime": 1635772023122,
    "endTime": 1635781068323
}
```
- update slug request by slug id
```JSON
{
	"id": "75930545972C68BA",
	"slug": "test slug udpate",
	"startTime": 1635772023122,
	"endTime": 1635781068323
}
```
- update slug request by external id
```JSON
{
	"externalId": xxxx,
	"slug": xxxx,
	"startTime": xxxx,
	"endTime": xxxx
}
```

**Parameter** 

|Parameter|Required|Type|Description|
|:----    |:---|:----- |-----   |
| sourceId         | Yes       | String | source Id |
| slug            | Yes       | String | slug info  |
| startTime            | No       | Long | start time by slug, default used current time  |
| endTime            | No       | Long | end time by slug  |
| externalId            | No       | String | external id, slug ino can be updated based on this id(externalId) or the id returned by our system  |
| sourceName            | No       | String | source name, in view of the ext source |
| sourceType            | No       | int | Pack = 1,Grid = 100,GLink = 101,Ext = 200,YouTube = 201,LocalSDI = 300  |
| id            | No       | String | slug id, slug info can be updated based on this id  |

**Example of Response**

```JSON
{
    "errorCode": "0x0",
    "errorInfo": "success",
    "result": {
        "id": "75930545972C68BA",
        "deleteFlag": 0,
        "createTime": 1635772023122,
        "updateTime": 1635781068323,
        "peerId": "3e3e1aed3a04c7220000000000000001",
        "gridLinkMetadata": "test slug111",
        "globalGridMetadata": null,
        "sourceName": null,
        "sourceType": null,
        "endTime": 1635781068323,
        "externalId": null
    }
}
```


**Parameters of Response** 

|Parameter|Type|Description|
|:-----  |:-----|----- |
|errorCode |string  | `0x0` means success |
|errorInfo |string  | `success` means success|
|createTime |Long  | slug start time |
|endTime |Long  | slug end time |
|gridLinkMetadata |String  | slug info |
