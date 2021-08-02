# Developer Guide for Partyline

## Client of Charge Service:

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
   provides `a hosted page` for customer to finish checkout/create new subscription for given valid plan ID. 
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
  Based on the provided number of charge items and formula, generates a new subscription. Then provides `a hosted page` for customer to finish checkout/create new subscription for given valid plan ID. 
  [TUTORIALS](https://www.chargebee.com/checkout-portal-docs/checkout-new-tutorial.html) or consult my team for how to use `the hosted page`



## PM:

* need to  preset a plan with price $0 for custom plan.  add all optional add-on to it. providing readable `Description`

* need a custom field to tag the custom plan. I created a CF named  `Custom`

* select `custom` for custom plan.  add `formula` in `JSON meta data` . 
  e.g. 
  `JSON meta data`  of `partyline-common-custom`

  ```json
  {"formula": "$partyline-hour*0.25*(2*$partyline-participant*(1+225/100)+$partyline-output*(1+225/100))"}
  ```

  the attribute  `mandatory`/`Recommended` of addon could be utilized in business layer.

* the client of paywalld needs to save and understand the meaning of `billing type` and create the mapping relationship between the business action with its `billing type`.

* when there are several addons with the same `billing type`, addon's name will be parsed for some purposes. 





## Dev of Charge Service:

1. Miro
   https://miro.com/app/board/o9J_lbF7xGI=/?moveToWidget=3074457361905206409&cot=14
2. `pl-hour` is a dynamic value. 

