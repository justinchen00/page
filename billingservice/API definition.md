# API and its internal logic of Billing Service

JustinChen@TVU

## Change logs:

1. 20210424 the first version that have been deployed for TVUSearch
2. 20210525 add a new API `mutation checkoutNewSubscriptionInvoice` for Partyline
3. 20210526 enhance the feature of `query plan` for Partyline; 
added a new API `mutation cancelSubscription`
4. 20210527 added a new API `query customerPortalStatus`
5. 20210531 `query subscription` and `query plan` support checking `remaining credit`
6. 20210607 added a new API `mutation cancelInvoice`
enhance the feature of `query customer` to support getting the unpaid invoices.
7. 20210801 new argument `type` of `query plans`; 
    new argument `chargeItems` of `mutation checkoutNewSubscription `;
    new API `query previewCustomSubscription`; 
    new API `query permission`
8. 20210820 clarify the argument `planId` in `query subscription`; 


## Some terms in the document:

- frontend --- the web page of the client of paywalld, such as the webpage of TVUSearch
- backend -- the web service of the client of paywalld, such as the web service backend of TVUSearch.
- customer -- who uses TVU service, such as "NY Times".
- service/client/product --- refer to TVU service, such as Mediamind/Producer/Partyline.
- billing type --- a kind of operation that is not free or should be logged. `mm_download` represents the operation of  the downloading in TVUSearch.
- charge Item — applicable `addons` .
- SUB — subscription
- PPU — pay per usage
- PK — credit

## PM/Solution need to know:

When having an existing and active plan, the customer isn't allowed to subscribe to the plan again.

Before a new service begins to use paywall, PM had to figure out the `plan` and `charge Items` that they need. 

All pre-configuration listed below can be added in the console of Chargebee.  You can refer to [plan](https://www.chargebee.com/docs/1.0/plans.html)/[custom field](https://www.chargebee.com/docs/2.0/custom_fields.html#creating-custom-fields) for details.

For a new service, we should do:

1. append a new option to the custom field `TVU Service Type` of `plan` and `subscription`, which includes `TVUSearch`/`Producer`/`Partyline` now.  
2. append new options to the custom field `Billing Type` of `Addons`for each charge Item.  
For `TVUSearch`, it includes `mm_download`/`mm_live`.  
For `Partyline`, it includes `pl_partitant`/`pl_output`/`pl_hours`. 
3. create a non-recurring `addon` for each charge Item and choose `Billing Type`. `addon` is still required even if it's free. Its price could be $0. Normally, `Pricing Model` is `Per Unit`.
If just PPU billing model, that's it.
e.g.
For the charge item of `download`  in `TVUSearch` , there are multiple `addon` for different price: 

	![image](https://user-images.githubusercontent.com/18137639/126103389-5a7bd14e-ce49-4c3c-ac1b-5026ea1322cf.png)

	![image](https://user-images.githubusercontent.com/18137639/126103445-3bb4cf54-11b9-4014-a07f-b0f6e517b970.png)

1. If project needs to bill customer periodically, you need to create `plan` ; set `Restricted addons`; attach those related `addon` to it in the way of `On Demand`.
2. If needing PK, please assign a string to the custom field `billing credit` of `plan`, which follows
    - JSON representation
    - object key is the same value of `Billing Type`
    e.g. 
    For `TVU Search` ,  the customer who subscribes to the plan will have 50 seconds credits for downloading.

        ```
        {
            "mm_download": 50,
            "mm_live": 0
        }
        ```

3. share the new added `TVU Service Type` and `Billing Type` with your engineer
4. done

## Notes:

- router: `api/paywall/v2/`
- When paywalld's API is invoked, paywalld will create a new customer account if the account does not exist.

## API

### query customer

Retrieve a customer's information of payment method and those exceptional invoices. All exceptional invoices and its content can be achieved by this API.

```
query{
  customer(customerId:"justinchen@tvunetworks.com",
           product: "TVUSearch"
           ) {
    cardStatus              # valid / expiring / expired / no card
    unbilledCharges         # Total unbilled charges for this customer. in cents, min=0
    exceptionalInvoices     # status == payment_due / not_paid
    {
        id
        customer_id
        status              # https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#invoice_status
        total
        due_date
        ...
    }
  }
}
```

`card_status`  == `valid` means the customer filed payment way.

ref: [https://apidocs.chargebee.com/docs/api/customers#retrieve_a_customer](https://apidocs.chargebee.com/docs/api/customers#retrieve_a_customer)

e.g. curl -L -g -X GET '[https://tvunetworks-test.chargebee.com/api/v2/invoices?customer_id](https://tvunetworks-test.chargebee.com/api/v2/invoices?customer_id)[is]=[emmasun4@admin.com](mailto:emmasun4@admin.com)&status[is_not]=not_paid'

### query plan

Retrieve all information of `plan`.  This API will be frequently invoked, so developer should pay attention to its performance.
paywalld's client can use this API to

1. get its quoted price,
2. the ceiling of the feature, which is presented in attachedItems.quantity.
3. get its remaining credit

```
query{
  plan(
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        planId: "mm_exsource_subpkppu_50"
    ) {
        planId
        price   # 19900 cents
        status  # "active"...
        ...
        chargeItems         # [] ="applicable_addons". Just show those addons whose "status" is "active"
        {
            id              # "mm_fednet4_ppu_addon_300_nr_live",
            product         # "TVUSearch",      == cf_tvu_service_type
            billingType     # e.g. "mm_live",   == cf_billing_type
            pricing_model   # "per_unit",
            price
            ...
        }
        attachedItems       # [] =attached_addons, it is used for Feature Control for now.
        {
            id
            quantity        # user can use it to decide the ceiling of the charge item.
            billingType     # = cf_billing_type. Users can use it to map to the feature name of business logic.
            ...
        }
        credit              # = cf_billing_credit. e.g. "{\"pl_hours\": 5}"
        remainingCredit     # remaining credit. this valus is from subscription.     e.g. "{\"pl_hours\": 5}"
    }
}
```

ref: [https://apidocs.chargebee.com/docs/api/plans?prod_cat_ver=1#retrieve_a_plan](https://apidocs.chargebee.com/docs/api/plans?prod_cat_ver=1#retrieve_a_plan)[https://apidocs.chargebee.com/docs/api/addons?prod_cat_ver=1#retrieve_an_addon](https://apidocs.chargebee.com/docs/api/addons?prod_cat_ver=1#retrieve_an_addon)[https://apidocs.chargebee.com/docs/api/addons?prod_cat_ver=1#list_addons](https://apidocs.chargebee.com/docs/api/addons?prod_cat_ver=1#list_addons)
For fetching the value of `billingType` of addon, please use the filter, such as cf_billing_type[is]=pl_participant&cf_tvu_service_type[is]=Partyline&status[is]=active

### query plans

fetch the existing plan list for given product

```
query{
  plans(
        product: "TVUSearch"
        planType: "all"  # "all"/"common"/"custom", default is "all";
    ) {
        "number": 2,
        "planIds":[ "mm_fednet5_subppu_199", "mm_fednet_special_knbc_0" ]
    }
}
```

ref: https://apidocs.chargebee.com/docs/api/plans?prod_cat_ver=1#list_plans

### query subscription

return the information of customer's subscription.  paywalld's client can use this API to check whether the customer has an active subscription of the plan and its remaining credit.

`paywalld` ensures that a user just has one active `subscription` for one `plan` in TVUSearch;  a user just has one active  `subscription` in `Partyline`.

```
query {
    subscription (
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        planId: "jj_plan_pk"	# In `Partyline`, this option will be optional 
    ) {
         subscriptionId
         status             # active/future/in_trial/non_renewing/paused/cancelled
         deleted            # true / false
         remainingCredit    # remaining credit. = cf_billing_credit. e.g. "{\"pl_hours\": 5}"
         ...
    }
}
```

ref:  https://apidocs.chargebee.com/docs/api/subscriptions?prod_cat_ver=1#list_subscriptions
The explanation of `status`: [https://apidocs.chargebee.com/docs/api/subscriptions#subscription_status](https://apidocs.chargebee.com/docs/api/subscriptions#subscription_status)

### query chargeItem

Retrieve all information of `addon`.  paywalld's client can use this API to get its quoted price.

```
query{
  chargeItem (
        customerId: "justinchen@tvunetworks.com",   //which is useless for paywalld now
        product: "TVUSearch",
        chargeItem: "mm_fednet4_ppu_addon_300_nr_live"
    ) {
        # ="applicable_addons". Just show those addons whose "status" is "active"
        id              # "mm_fednet4_ppu_addon_300_nr_live",
        product         # "TVUSearch",  == cf_tvu_service_type
        billingType     # "mm_live",    == cf_billing_type
        pricingModel    # "per_unit",
        price           # 300,
        ...
    }
}
```

ref: [https://apidocs.chargebee.com/docs/api/addons?prod_cat_ver=1#retrieve_an_addon](https://apidocs.chargebee.com/docs/api/addons?prod_cat_ver=1#retrieve_an_addon)

### query permission

For Partyline, its `pl-hour` saves the maximum credit and it's static. This user's real usage is got from Usage Service, which calculates from the moment that subscription was activated or updated. So it's a dedicated logic for Partyline.

```
query{
  permission (
        customerId: "justinchen@tvunetworks.com",
        product: "Partyline",				# Partyline user just has one plan at a time. So no plan argument here.
        chargeItems: "string in JSON",		# sample listed below 				
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
	    remainHours			# The quantity of chargeItems - usage 
		permission {		# []
			id		# partyline-participant
			ceiling		# 8
			allow		# true/false
		}	
    }
}
```

ref:  Usage Service:
https://showdoc.tvunetworks.com/web/#/111?page_id=3230
https://211.160.178.10:23256/usageService/#/infoView

e.g.  
the quantity of `partyline-participant` is 6, which means this user who has this subscription is allowed to convene a meeting with 6 participants at most.

```
"addons": [
                    {
                        "id": "partyline-hour",
                        "quantity": 5,
                        "unit_price": 0,
                        "amount": 0,
                        "object": "addon"
                    },
                    {
                        "id": "partyline-participant",
                        "quantity": 6,
                        "unit_price": 0,
                        "amount": 0,
                        "object": "addon"
                    },
                    {
                        "id": "partyline-output",
                        "quantity": 2,
                        "unit_price": 0,
                        "amount": 0,
                        "object": "addon"
                    }
                ],
```

### query customerPortal

The end-user can use the self-service portal to maintain their billing information / invoice / subscription. You 

```
query{
  customerPortal (
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        redirectUrl: "https://search.tvunetworks.com"	# URL to redirect when the user logs out from the portal.
    ) {
        id          # "portal_AzZlxDSWcJOT1FvC",
        token       # aRNcIRmnenKuV67D07JlT8tC07caVpPw",
        access_url  # https://tvunetworks-test.chargebee.com/portal/v2/authenticate?token=aRNcIRmnenKuV67D07JlT8tC07caVpPw"
    }
}
```

ref: [https://apidocs.chargebee.com/docs/api/portal_sessions?prod_cat_ver=1#create_a_portal_session](https://apidocs.chargebee.com/docs/api/portal_sessions?prod_cat_ver=1#create_a_portal_session)

### query customerPortalStatus

client can use it to check the final status of a checkout carried out by hosted page.

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
If the payment succeeds, it is marked as paid. If the payment fails, the invoice is marked as payment_due and retry settings are taken into account. If no retry attempts are configured, the invoice is marked as not_paid. If the amount due is zero or negative, the invoice is immediately marked as paid and the balance if any is carried forward to the next term of the invoice.

ref: [https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#retrieve_a_hosted_page](https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#retrieve_a_hosted_page)[https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#invoice_status](https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#invoice_status)

### query previewCustomSubscription

calculate the price of the subscription. 

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

`formula` saves in `JSON meta data`
e.g. 
 `JSON meta data`  of `partyline-common-custom`

```json
{"formula": "$partyline-hour*0.25*(2*$partyline-participant*(1+225/100)+$partyline-output*(1+225/100))"}
```

### mutation createCustomer

```
mutation{
  createCustomer(
    customerId:"justinchen@tvunetworks.com", //using TVU's account(email address) as Chargebee(CB)'s ID
    email:"justinchen@tvunetworks.com",	// normally same as the value of `id`
    firstName:"Justin",
    lastName:"Chen"
  ) {
      customer{
          id
      }
  }
}
```

ref: [https://apidocs.chargebee.com/docs/api/customers?prod_cat_ver=1#create_a_customer](https://apidocs.chargebee.com/docs/api/customers?prod_cat_ver=1#create_a_customer)

### mutation updatePayment

return hosted page. paywalld's client can use this API to allow customer to update his payment way.

```
mutation{
  updatePayment(customerId:"justinchen@tvunetworks.com")
  {
  	hostedPage {
  		...
	}
  }
}
```

ref: [https://apidocs.chargebee.com/docs/api/hosted_pages#manage_payment_sources](https://apidocs.chargebee.com/docs/api/hosted_pages#manage_payment_sources)

### mutation checkoutOneTimePageAmount

for PPU. return hosted page. no need for an add-on.

```
mutation {
    checkoutOneTimePageAmount (
        customerId:"justinchen@tvunetworks.com",
        product: "TVUSearch",
        description: "FedNet01 April 29 12:33:01 7 minutes download",
        amount: 84	//$ cent
        )
    {
      hostedPage {
          ...
	  }
    }
}
```

> How to check the status of payment?

Two ways:

- The frontend web page can handle it. The prerequisite is paywalld needs to enable `iframe_messaging` while getting the hosted page.
- The backend service uses `hosted_page.id` to query the payment status. If paid, the backend can save the state for future reference.
ref: [https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#retrieve_a_hosted_page](https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#retrieve_a_hosted_page)

### mutation checkoutOneTimePageQuantity

for PPU. return hosted page. The add-on should be created in advance, and its `Charge Type` must be `One-time`.  And share it with the client of paywalld, such as MediaMind.

```
mutation {
    checkoutOneTimePageQuantity (
    customerId:"justinchen@tvunetworks.com",
     product: "TVUSearch",
     description: "FedNet01 April 29 12:33:01 7 minutes download",

     chargeItem: "addon_name",  // create it in advance,
     quantity: 15
    )
    {
        hostedPage {
            ...
        }
	}
}
```

`hosted page`:
e.g.

```
{
    "hosted_page": {
        "id": "b3coJuK1RJkcoRCP69d9gdqCKMcOcwxy",
        "type": "checkout_one_time",
        "url": "https://tvunetworks-test.chargebee.com/pages/v3/b3coJuK1RJkcoRCP69d9gdqCKMcOcwxy/",
        "state": "created",
        "embed": false,
        "createdAt": 1618720670,
        "expiresAt": 1618731470,
        "object": "hosted_page",
        "updatedAt": 1618720670,
        "resourceVersion": 161872060898
  }
}
```

ref: [https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#checkoutOneTime-usecases](https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#checkoutOneTime-usecases)

### mutation checkoutOneTimeInvoiceAmount

for PPU. API returns the status of invoice. It applies to the case that there is no need for confirmation from the customer, such as `mm_takelive` of `tvusearch`. no add-on. Need to check customer's billing information before using it in business logic. 
ref: [https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#create-usecases](https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#create-usecases)

```
mutation {
    checkoutOneTimeInvoiceAmount (
        customerId:"justinchen@tvunetworks.com",
        product: "TVUSearch",
        description: "FedNet01 April 29 12:33:01 7 minutes download",
        amount: 84	//$ cent
        )
    {
        invoiceId: "xxxx",
        invoiceStatus: "paid",
        amountPaid: 84
        total: 84
    }
}
```

### mutation checkoutOneTimeInvoiceQuantity

`add-on` can be used here. It should be created in advance, and its `Charge Type` must be `One-time`. The client of paywalld, such as MediaMind.
ref: [https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#create-usecases](https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#create-usecases)

```
mutation {
    checkoutOneTimeInvoiceQuantity (
    customerId:"justinchen@tvunetworks.com",
     product: "TVUSearch",
     description: "FedNet01 April 29 12:33:01 7 minutes download",
     chargeItem: "addon_name",  // create it in advance,
     quantity: 15
    )
    {
        invoiceId: "xxxx",
        invoiceStatus: "paid",
        amountPaid: 84
        total: 84
    }
}
```

### mutation cancelSubscription

`paywalld` ensures that one `plan` just has one `subscription` for one user.  Return the same structure as `query subscription` did.

```
query {
    cancelSubscription (
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        planId: "MM_Fednet_Sub_98"
    ) {
         subscriptionId 		// subscription ID
         status		// active / future/in_trial/non_renewing/paused/cancelled
    	 deleted	// true / false,
         ...
	}
}
```

ref:  [https://apidocs.chargebee.com/docs/api/subscriptions?prod_cat_ver=1#cancel_a_subscription](https://apidocs.chargebee.com/docs/api/subscriptions?prod_cat_ver=1#cancel_a_subscription)

### mutation cancelInvoice

If a prepay invoice fails to charge, it should be canceled immediately.

```
query {
    cancelInvoice (
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        invoiceID: "735"
    ) {
         invoiceID
         status
	}
}
```

ref:

### mutation checkoutNewSubscription

providing a hosted page for customers to finish checkout/create new subscriptions for given valid plan ID.  Return `success` when the customer has an active subscription of the plan already.

```
mutation {
    checkoutNewSubscription (
        customerId: "justinchen@tvunetworks.com",
		product: "TVUSearch",
    	planId: "mm_exsource_subpkppu_50",
    	chargeItems: "same JSON as query previewCustomSubscription",	# optional
    ) {
      hostedPage {
          ...
	  }
    }
}
```

The argument of `chargeItems` will be meaningful as `plan` is `custom` and `formula` exists.

The original price of `plan` will be override by the new price calculated on two factors: (1) the formula of this plan; (2) the number of `charge item` in the argument of `chargeItems` .

[PK] copy the custom field `billing credit` of `plan` onto the custom field `billing credit` of `subscription`.  It follows JSON.
`iframe_messaging` should be `true`. `embed` should be `true`.
ref: [https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#checkoutNew-usecases](https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#checkoutNew-usecases)

[PK]  There is a periodic task for PK mode: resetting the `billing credit` at the end of every billing period.
paywalld should provide an API to receive the notification from Chargebee for event `subscription renewed`.  Its format is listed in Appendix 1.
ref: [https://tvunetworks-test.chargebee.com/apikeys_and_webhooks/webhooks](https://tvunetworks-test.chargebee.com/apikeys_and_webhooks/webhooks)

### mutation checkoutNewSubscriptionInvoice

If no suitable payment method, this API returns `failure` .  If the customer has an existing and active plan, API returns `success`.

```
mutation {
    checkoutNewSubscriptionInvoice (
        customerId: "justinchen@tvunetworks.com",
		product: "TVUSearch",
    	planId: "mm_exsource_subpkppu_50",
    ) {
        invoice {
            invoiceId: "xxxx",
            invoiceStatus: "paid",
            amountPaid: 84
            total: 84

            subscriptionId
        }
        subscription {		//subscription structure
            ...
        }
        customer {
            ...
        }
    }
}
```

[PK] copy the custom field `billing credit` of `plan` onto the custom field `billing credit` of `subscription`.  It follows JSON.
ref: [https://apidocs.chargebee.com/docs/api/subscriptions?prod_cat_ver=1#create_subscription_for_customer](https://apidocs.chargebee.com/docs/api/subscriptions?prod_cat_ver=1#create_subscription_for_customer)

### mutation checkoutExistingSubscriptionInvoice

checkout an existing subscription with billing_type(add_on). API returns the status of invoice. So it can use the same class as the return value of checkoutOnetimeInvoice.

paywalld's client can use this API to charge customer's add-on that attaches to an active subscription.

```
mutation {
    checkoutExistingSubscriptionInvoice (
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        planId: "mm_exsource_subpkppu_50",
        billingType: "mm_download",
        description: "FedNet01 April 29 12:33:01 7 minutes download",
        quantity: 120
    ) {
        invoice {
            invoiceId
            invoiceStatus	# "paid"
            amountPaid
            total
        }
    }
}
```

ref: [https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#create_invoice_for_a_non-recurring_addon](https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#create_invoice_for_a_non-recurring_addon)

Internal workflow:

1. check whether the customer has an active subscription. if has no, return error.
2. check whether the customer has no unbilled charges. If it has, return error.
3. get `plan` 's `applicable_addons` list.
ref: [https://apidocs.chargebee.com/docs/api/plans#retrieve_a_plan](https://apidocs.chargebee.com/docs/api/plans#retrieve_a_plan)
4. get the addon ID whose `cf_billing_type` has the same value as "billing_type" of the API input.
5. If the custom field `billing credit` of `subscription` is empty, no PK is involved.
directly invoices `addon` with the `quantity` in API to the customer.
Return the information of `invoice`. Most important value is "invoice"."amount_paid" and

    "invoice"."[status](https://apidocs.chargebee.com/docs/api/invoices#invoice_status)"

6. [PK] If `billing credit` has value, parse it and get the remaining `credit/quantity` of the addon of the subscription.
7. [PK] if (remained credit >= charging `quantity` in API)
invoice addon with the `quantity` in API, but its price should be $0(addon_unit_price=$0).
8. [PK] if (remaining credit < charging `quantity` in API)
invoice addon with those remaining credit in $0.
And an invoice addon with the non-free quantity(charging quantity - remaining credit) in the default price.
9. [PK] If invoice.status == paid, update the JSON of the custom field `billing credit` of `subscription`.
ref: [https://apidocs.chargebee.com/docs/api/subscriptions?prod_cat_ver=1#update_a_subscription](https://apidocs.chargebee.com/docs/api/subscriptions?prod_cat_ver=1#update_a_subscription)
10. [PK] Return the information of `invoice`.

### mutation checkoutExistingSubscriptionPage

typically in the trial state

ref: [https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#checkout_existing_subscription](https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#checkout_existing_subscription)

### mutation createPlan

### mutation createChargeItem

## Billing models and their essential APIs

### 1. PPU

#### case1: the business service knows the final price before carry out the action

mutation `checkoutOneTimePageAmount`

e.g.  Before downloading a file, we know its total price. 

#### case2: the business service knows the final price after carried out the action

query `customer`: use this API to check whether customer has correctly filed his payment way.

mutation `checkoutOneTimeInvoiceAmount`: charge customer when we know the final amount of the bill.

e.g. We didn't know the final price until the "takelive" stopped.

#### case3: the business service knows the count of charged item before carry out the action

mutation `checkoutOneTimePageQuantity`

#### case4: the business service knows the count of charged item after carried out the action

query `customer`: use this API to check whether customer correctly filed his payment way.

mutation `checkoutOneTimeInvoiceQuantity`: charge customers when we know the final amount of the bill.

### 2. SUB

#### case 5: charge customer a fixed amount each charging period

query `subscription`: use this API to check whether the customer has an active subscription of the plan.

mutation `checkoutNewSubscription`

mutation `checkoutExistingSubscriptionInvoice`:  this is optional for this case.

### 3. SUB+PPU

#### case 6: charge customer a fixed amount each charging period and extra charge customer when he carry out the action that is not free

query subscription: use this API to check whether the customer has an active subscription of the plan.

mutation `checkoutNewSubscription`

mutation `checkoutExistingSubscriptionInvoice`

### 4. SUB+PPU+PK

#### case 7: when customer subscribed a plan, he will get amount of free credits for the charge item

same as case 6. The details are hidden by paywalld.

## Appendix

1. message body of subscription_renewed

```
 {
    "id": "ev_Azz5oUSRtOs3v1CkB",
    "occurred_at": 1615960809,
    "source": "scheduled_job",
    "object": "event",
    "api_version": "v2",
    "content": {
        "subscription": {
            "id": "jj_sub_pk",
            "plan_id": "jj_plan_pk",
            "plan_quantity": 1,
            "plan_unit_price": 95000,
            "billing_period": 1,
            "billing_period_unit": "week",
            "customerId": "AzZltkSPvMtaRKGr",
            "plan_amount": 95000,
            "plan_free_quantity": 0,
            "status": "active",
            "current_term_start": 1615960800,
            "current_term_end": 1616565599,
            "next_billing_at": 1616565600,
            "created_at": 1615445442,
            "started_at": 1615359600,
            "activated_at": 1615359600,
            "updated_at": 1615960808,
            "has_scheduled_changes": false,
            "resource_version": 1615960808948,
            "deleted": false,
            "object": "subscription",
            "currency_code": "USD",
            "addons": [{
                "id": "-justin_sub_pk_ppu_recurring_week",
                "quantity": 1,
                "unit_price": 20,
                "amount": 20,
                "object": "addon"
            }],
            "due_invoices_count": 0,
            "mrr": 412876,
            "exchange_rate": 1.0,
            "base_currency_code": "USD",
            "shipping_address": {
                "first_name": "JUSTIN",
                "last_name": "CHEN",
                "email": "JUSTINCHEN@TVUNETWORKS.COM",
                "phone": "6504395678",
                "line1": "857 Maude Avenue",
                "city": "Mountain View",
                "state_code": "CA",
                "state": "California",
                "country": "US",
                "zip": "94043",
                "validation_status": "not_validated",
                "object": "shipping_address"
            }
        },
        "customer": { },
        "card": { },
        "invoice": {}
    },
    "event_type": "subscription_renewed",
    "webhook_status": "not_configured"
}
```

1. [engineer] The API of Chargebee has two big versions. V1 and V2.
For V2, there are two small versions, prod_cat_ver=1 and prod_cat_ver=2. we are using is `prod_cat_ver=1`, which was the stable version when we started developing paywalld. [https://apidocs.chargebee.com/docs/api/](https://apidocs.chargebee.com/docs/api/)
