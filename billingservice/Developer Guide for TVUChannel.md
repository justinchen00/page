# How to integrate Billing Service

## Change logs:

20211025: the first version of this document



## 1. for developer

### essential APIs
[query permission](https://justinchen00.github.io/page/billingservice/API%20definition.html#query-permission)

### Step by Step:

1. [APP] APP uses `query permission` to check permission. 

   If has permission(`permission.code` == 0x0), go to DONE.

   If no permission(`permission.redirect` is not null), open the URL provided by the API in IFrame.

2. [USER] user finished his operations and closes the IFrame 

3. [APP] checks the permission again.

4. DONE

### Samples

a sample of the case that user has permission to carry on with the operation in APP.

```json
{
	"permission": {
		"code": "0x0",
		"description": "you subscribed",
		"subscriptionId": "2e8d761fb5b448f1a04226e4d67ec989",
		"encryptedStr": "2e8d761fb5b448f1a04226e4d67ec989",
		"remainHours": null,
		"permission": null,
		"redirect": null
	},
	"status": 200
}
```

a sample of #2. 

```json
{
	"permission": {
		"code": "80E00103",
		"description": "you don't subscribe to the plan",
		"redirect": {
			"iframeSrc": "https://bill.tvunetworks.com/billing?product=Partyline",
			"iframeWidth": "90%",
			"iframeHeight": "820"
		}
	},
	"status": 200
}
```



## 2. for PM:

* using [API](https://rapidapi.com/tvu-networks-tvu-networks-default/api/billing-service/) to maintain plan/addon

## 3. More documents

FakeApp: http://paywall-app.tvunetworks.com:51000/ 

Playground: https://bill.tvunetworks.com/route-billing/billing-service/base/graphql 

Rapid API: https://rapidapi.com/provider/5600093/apis/billing-service/definition/overview

API document for engineer: https://justinchen00.github.io/page/billingservice/New%20API%20definition.html

BillingService: https://bill.tvunetworks.com 

#### design documents:

https://docs.google.com/document/d/108J1mX22vblwRx7OgXGxaNQHYypfyKKP7kBFjGzusg0/edit

https://docs.google.com/document/d/1kbozAs6vIr7dGajmTbhbqR4O2u3WCQUzsZfP4NBSUnE/edit

https://miro.com/app/board/o9J_lbF7xGI=/?moveToWidget=3074457363982308077&cot=14

https://miro.com/app/board/o9J_le1TEwU=/?moveToWidget=3074457363168410532&cot=14



