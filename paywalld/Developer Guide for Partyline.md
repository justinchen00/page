# Developer Guide for Partyline

## 1. for the Client of Charge Service:

### Common API

check operation permission.

query previewCustomSubscription

### Common Rate Card

workflow and related APIs

1. query plans
   fetch the existing plans with `type`="custom" for given product
2. query plan
   fetch the details of each plan for display.
3. mutation checkoutNewSubscription
   provides `a hosted page` for customers to finish checkout/create a new subscription for a given valid plan ID. 
   [TUTORIALS](https://www.chargebee.com/checkout-portal-docs/checkout-new-tutorial.html) or consult my team for how to use `the hosted page`

### Custom Rate Card

workflow and related APIs

1. query plans
   fetch the existing plans with `type`="custom" for given product
2. query plan
   fetch the details of each plan, 
3. query previewCustomSubscription
   return the price
4. mutation checkoutNewSubscription
   Based on the provided number of charge items and formula, generates a new subscription. Then provides `hosted page` for customers to finish checkout/create new subscriptions for a given valid plan ID. 
   [TUTORIALS](https://www.chargebee.com/checkout-portal-docs/checkout-new-tutorial.html) or consult my team for how to use `the hosted page`

## 2. for PM:

* need to  preset a plan with price $0 for custom plan.  add all optional add-on to it. providing readable `Description`

* need a custom field to tag the custom plan. I created a CF named  `Custom`

* select `custom` for custom plan.  add `formula` in `JSON meta data` . 
  e.g. 
  `JSON meta data`  of `partyline-common-custom`

  ```json
  {"formula": "$partyline-hour*0.25*(2*$partyline-participant*(1+225/100)+$partyline-output*(1+225/100))"}
  ```

  the attribute  `mandatory`/`Recommended` of addon could be utilized in the business layer.

* Partyline needs to save `chargeItem` and create the mapping relationship between the business features with them. 
  Such as,  `partyline-output`=2 means the user allows to have 2 outputs at most.

* It's not feasible to get the value of the addon attribute CF("cf_billing_type") via one API invoking. Using a loop to fetch attributes of each addon is time-consuming. It's fine for `TVUSearch`. Because there are just two addons. However, For Partyline, a plan will be attached with lots of addons, and those addons remain unchanged after the billing model is nailed down. So let's use the addon.id as the feature name and no need to change CF("cf_billing_type").  
  Please make sure the addon.id of Partyline is short and straightforward, such as `pl-output` or `partyline-output`.

## 3. More documents

1. Miro
   https://miro.com/app/board/o9J_lbF7xGI=/?moveToWidget=3074457361905206409&cot=14
2. 

