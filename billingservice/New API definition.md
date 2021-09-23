# new API and its internal logic

JustinChen@TVU



## Change logs:

1. 20210915 the first version
2. 20210922 add a new API `mutation updateSubscription`; modify `query permission` and add the recommended redirect URL

## Guideline 

Concept: https://docs.google.com/document/d/1kbozAs6vIr7dGajmTbhbqR4O2u3WCQUzsZfP4NBSUnE/edit

Data Structure: https://docs.google.com/document/d/108J1mX22vblwRx7OgXGxaNQHYypfyKKP7kBFjGzusg0/edit

Slack: https://join.slack.com/share/zt-w9dqrpc4-BvDkRy4pzXumeIiDmWf_4A

Miro: https://miro.com/app/board/o9J_lbF7xGI=/?moveToWidget=3074457363982308077&cot=14

Engineering Document: https://docs.google.com/document/d/1LJjxVGk3X9yfuTIjHIq8q6FSnxWk298Fo4CQmwLiFRQ/edit#

## Requirement

The new Billing Service should implement those features:

1. just rely on Chargebee to do two things:  (1) maintain payment information  (2) do transaction 
2. give up using the tools(plan/sub/addon) provided by Chargebee. Those logic should be implemented by ourselves.
3. Chargebee customer account should be saved in UserService's Group object. User has no its own CB customer account.

## Notes:

- router: `api/billing/v1/`

## Billing Service API

### query permission

> used by APP & HouseKeeper

For Partyline, its `partyline-hour` saves the static maximum credit. This user's real usage is obtained from Usage Service, which calculates from the moment that subscription was activated or updated to the moment when API is invoked. 
If all `permission.allow`=`true` , APP is allowed to proceed.
If the API returns the user has an unpaid invoice, App should guide user to open customer portal to maintain their payment information. `query customerPortal` can get the `customer portal`.

```
query{
  permission (
        customerId: "justinchen@tvunetworks.com",
        product: "Partyline",				// Partyline user just has one plan at a time. So no plan argument here.
        chargeItems: "string in JSON",		// sample listed below 				
                                            {
                                                "version":1,
                                                "partyline-participant":8,
                                                "partyline-output":4
                                            }  
                                            
                                            OR:
                                            {
                                            	"version":2,
                                            	"chargeItems": [
                                                {
                                                "id": "partyline-participant",
                                                "quantity": 8,
                                                },
                                                {
                                                "id": "partyline-output",
                                                "quantity": 4,
                                                }
                                            	]
                                            }
    ) {
	    remainHours			// The quantity of chargeItems - usage 
		permission {		// []
			id			// partyline-participant
			ceiling		// 8
			allow		// true/false
		},
		redirectUrl: "the URL of the default plan list, or a self-service portal"
		redirectReason: "need to upgrade plan, or there is an unpaid invoice"
    }
}
```

Usage Service API: 
https://showdoc.tvunetworks.com/web/#/111?page_id=3313
https://usageservice.tvunetworks.com/usageservice-backend/graphql

```
query{
    partylineJoinTimeSummary(
        startTime: "0",
        endTime: "1631571056",
        email: "justinchen@tvunetworks.com"
    )
}
```


The backend DB of Usage Service: https://usageservice.tvunetworks.com/#/infoView

### query plans

> used by embedded web

fetch the existing plans and their details for a given product.

```
query{
  plans(
        product: "TVUSearch"
        planType: "all"  // "all"/"common"/"custom", default is "all";
    ) {
        "number": 2,
        "planIds":[ "mm_fednet5_subppu_199", "mm_fednet_special_knbc_0" ]
    }
}
```

### query previewCustomSubscription

calculate the price of the custom subscription. 

```
query {
    previewCustomSubscription (
        customerId: "justinchen@tvunetworks.com",	# reserved
		product: "Partyline",						# case insensitive
    	planId: "partyline-common-custom",
    	chargeItems: "string in JSON",				# sample listed below 				
                                            {
                                                "version":1,
                                                "partyline-hour":10,
                                                "partyline-participant":8,
                                                "partyline-output":4
                                            }  
                                            
                                            OR:
                                            {
                                            	"version":2,
                                            	"chargeItems": [
                                                {
                                                "id": "partyline-hour",
                                                "quantity": 10,
                                                },
                                                {
                                                "id": "partyline-participant",
                                                "quantity": 8,
                                                },
                                                {
                                                "id": "partyline-output",
                                                "quantity": 4,
                                                }
                                            	]
                                            }

    ) {
      	price	# $100
    }
}
```

`formula` presents in JSON and is saved in Billing Service.

```json
{"formula": "$partyline-hour*0.25*(2*$partyline-participant*(1+225/100)+$partyline-output*(1+225/100))"}
```

### query customerPortalStatus

> used by embedded web

client can use it to check the final status of a checkout carried out by a hosted page.

```
query{
  customerPortalStatus (
        hostedPageID: "b9Z5Sce6AxFBNBMLu4qcuJHFthSF4VI0I"
    ) {
        type        # "checkout_one_time"
        url
        created_at  # 1622021085
        invoice {   # type invoice
            id
            customer_id
            status  # "payment_due", check below
            total
            amount_paid
            amount_due
            amount_to_collect
        }
    }
}
```

`invoice.status`: 
If the payment succeeds, it is marked as `paid`. If the payment fails, the invoice is marked as `payment_due` and retry settings are taken into account. If no retry attempts are configured, the invoice is marked as `not_paid`. If the amount due is zero or negative, the invoice is immediately marked as paid and the balance if any is carried forward to the next term of the invoice.

ref: [https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#retrieve_a_hosted_page](https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#retrieve_a_hosted_page)[https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#invoice_status](https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#invoice_status)

### mutation checkoutNewSubscription

> used by embedded web

providing a hosted page for customers to finish checkout/create new subscriptions for given valid plan ID.  Return `success` when the customer has an active subscription of the plan already.

```
mutation {
    checkoutNewSubscription (
        customerId: "justinchen@tvunetworks.com",
		product: "Partyline",
    	planId: "partyline-common-advance",
    	chargeItems: "same JSON as query previewCustomSubscription",	# optional. It's meaningful while `plan` is `custom`
    ) {
      hostedPage {
          ...
	  }
    }
}
```

Internal logic:

1. need to create a Chargebee customer account if no account for the group of this user exists. And pass its customer account ID to User Service;
2. checkout OneTime Invoice(https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#checkoutOneTime-usecases), return a hosted page;
3. save hostedPageID & the string of chargeItems to the database table `payment` to log 
4. our embedded web uses `query customerPortalStatus` to get the payment status. The embedded web should inform App of the status. 
5. At the same time, the backend should wait for the callback from Chargebee. We can check the payment status in its handler.
6. Chargebee will pass the invoice status to `mutation eventNotify` via web hook.
7. The payment status and invoiceID can be obtained from the previous three steps.  They have the same logic. The relationship between `subscriptionID` and `invoiceID` will be created there and saved in table `payment`.
8. Based on the record saved in table `payment` , insert a new record to `subscription` table and set the status of subscription to `valid` , and table `subaddon` should be updated too. 

#### `cron` task

Two cron task is created to scan the valid subscription and abnormal invoices each day.

### query customerPortal

The end-user can use the self-service portal to maintain their billing information / invoice / subscription. You

```
query{
  customerPortal (
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        redirectUrl: "https://search.tvunetworks.com"	# optional. URL to redirect when the user logs out from the portal.
    ) {
        id          # "portal_AzZlxDSWcJOT1FvC",
        token       # aRNcIRmnenKuV67D07JlT8tC07caVpPw",
        access_url  # https://tvunetworks-test.chargebee.com/portal/v2/authenticate?token=aRNcIRmnenKuV67D07JlT8tC07caVpPw"
    }
}
```

### mutation eventNotify

> used by CB's webhook

use it to get the status of the invoice. The format is defined by Chargebee.

### mutation updateSubscription

> used by embedded web

providing a hosted page for customers to finish checkout/create new subscriptions for given valid plan ID.  Return `success` when the customer has an active subscription of the plan already.

```
mutation {
    checkoutNewSubscription (
        customerId: "justinchen@tvunetworks.com",
		product: "Partyline",
    	planId: "partyline-common-advance",
    	chargeItems: "same JSON as query previewCustomSubscription",	// optional. It's meaningful while `plan` is `custom`
    	subScriptionID: "..."		//optional for now.
    ) {
      hostedPage {
          ...
	  }
    }
}
```

Internal logic:

1. If no `subScriptionID`, use the only existing one.
2. no credit will be prorated for now!!!  Directly charge the customer the new price of new plan.

## HouseKeeper API

1. an internal Timer. It is created when an event is created in APP; It is destroyed when an event is stopped in APP. And notify App when credit runs out or is about to run out.
2. API: `startEvent` { session, user， `a new encrypted string got from query permission` ,   `callback_URL`}
3. API: `stopEvent` { session, user,  `a new encrypted string got from query permission` }