**DRAFT AND EXPERIMENTAL**

CheckPlease API
=================
CheckPlease provides a simple, secure API for ordering and managing pre-employment and background checks, including Criminal Record checks from the [NZ Ministry of Justice](https://www.justice.govt.nz/criminal-records/).

Examples of checks are:
- NZ Criminal Record check (MoJ check)
- Credit check
- Academic qualifications check
- etc.

Concepts
-----------
In CheckPlease, every check lives within a **bundle**.

A bundle combines the data requirements of all of its checks to create a personal details UI for the individual.

So for example, even if an individual is being asked to complete 3 separate checks, they will only be asked for their personal details once - but of one of the checks is a Driver history check, then the driver licence field will be mandatory. 


Endpoints
----------
The following endpoints are available:

| API        | Purpose   |
| -------- |--------|
| POST&nbsp;/checks | Order a single check. |
| GET&nbsp;/checks/byExternalRef/{externalRef} | Get the current state for a check and its bundle.  |
| Your web hook | Be alerted when a check or its bundle is updated. You pass your web hook's url in when you create a check. |
| GET&nbsp;/checks/byClientKey/{clientKey}/request | Fetch the request pdf for a check. | 
| GET&nbsp;/checks/byClientkey/{clientKey}/result | Fetch the result pdf for a check. |
| PATCH&nbsp;/checks/byExternalRef/{externalRef} | Modify a check |
| GET&nbsp;/account | Fetch your account details. |
| GET&nbsp;/mojChecks/metaData | Fetch metadata about the statuses that a check can be in. |
 
To use the CheckPlease API, you'll need a free account at https://www.checkplease.co.nz.
 
We're here to help you with your integration - for help using our API, email us at info@checkplease.co.nz.   


API authentication
----------
For APIs calls that you make to CheckPlease, include your account's API key in the Authorization header. You can get your API key from your Account settings page in CheckPlease.

For API calls that CheckPlease makes in to you (i.e. web hook calls), CheckPlease will attach your account's API key in the Authorization header. You should reject any incoming requests that don't have the API key present.

   
Ordering a check
====
Use the **POST /checks** API to order a check.

For example, to order an NZ Criminal Record check on Fred Flintstone with 3 day turnaround (from the command line, using curl):

````
curl -X POST https://api.checkplease.co.nz/api/checks \
-H 'Content-Type: application/json' \
-H 'Authorization: OmV81mPqcQ_5vb3R9UjTQPelTmcvfeAbgqO5ptkK' \
-d @- << EOF
{
  "bundle": {
    "individualFirstName": "Fred",
    "individualMiddleNames": "Rocky",
    "individualLastName": "Flintstone",
    "individualEmail": "fred@flintstone.com",
    "bundleKey": "120439"
  },
  "check": {
    "type": "MOJ",
    "priority": "GOLD",
    "externalRef": "100233",
    "webHook": "https://myserver.com/checkWebHook
  }
}
EOF

````
The response contains the check object, now with the server-assigned fields included for check and bundle:

````
{
  "bundle": {
    "individualFirstName": "Fred",
    "individualMiddleNames": "Rocky",
    "individualLastName": "Flintstone",
    "individualEmail": "fred@flintstone.com",
    "bundleKey": "120439",
    "reference": 200346653
  },
  "check": {
    "type": "MOJ",
    "priority": "GOLD",
    "externalRef": "100233",
    "webHook": "https://myserver.com/checkWebHook",
    "plan": "BUSINESS",
    "clientKey": "bb5f03a6-0e1e-6a2c-ab08-ba9461a09137",
    "canChangePriority": true,
    "eta": "2018-10-17T12:03:44.05Z",
    "status": "WHITE",
    "statusLabel": "Sent to MoJ",
    "statusImage": "https://example.com/imageblah?x=1",
    "msInStatus": "230435",
    "requestComplete": false,
    "resultComplete": false,
    "vendorReference": null
  }
}
````

CheckPlease may return one of the following http statuses:
- 200 if the check was successfully created
- 400 if the check could not be created (for example, because firstName or lastName were missing)

Getting current state for a check
====
Use the **GET /checks/byExternalRef/{externalRef}** API to get a check's current state.

For example, to get the details for the check created earlier, from the command line using curl:

````
curl https://api.checkplease.co.nz/api/mojChecks/byExternalRef/100233 \
-H 'Authorization: OmV81mPqcQ_5vb3R9UjTQPelTmcvfeAbgqO5ptkK'
````
The response contains the check object.

CheckPlease may return one of the following http statuses:
- 200 if the check was successfully fetched

Polling vs web hooks
-------
If at all possible, avoid calling this GET API repeatedly to see if your check has changed (i.e. polling). Instead, pass a web hook in when you create the check. Then your integration will be notified in real time as soon as your check changes.

Using a web hook instead of polling is:
- more real-time: your code will be instantly alerted when your checks change their state, and your integration will be always up to date
- more efficient: since checks tend to change slowly (e.g. many days may elapse before the MoJ provides the result), web hooks require far less network traffic than polling
- more reliable: your API calls will never fail due to rate limiting (the API is not tightly rate limited but may be in the future)

Fields in the check object
===========
| Field        | Set by         | Mandatory? | Notes  |
| ------------- |:-------------:|:-----:|---|
| type | client | Y | What type of check this is, i.e. one of: <ul><li>MOJ (NZ Criminal Record check)<li>EQUICREDIT (Credit check via Equifax)<li>DRIVER (Comprehensive Driver check)<li>QUAL (Qualification check)<li>ACC (ACC records check)</ul> |
| priority | client | N | The possible values for priority depend on the check type. For NZ Criminal Record checks, the values are are: <ul><li>GOLD (3 working days turnaround)<li>SILVER (10 working days turnaround)<li>BRONZE (15 working days turnaround)</ul> If you don't provide a value for priority, then the check will be created using your CheckPlease account's default priority. |
| externalRef | client | N | The optional externalRef  field can be used to store your own reference (up to 512 characters) on the check. This is useful if you intend to update the check's priority after it has been created. |
| webHook | client | N | The optional webHook field can be used to store a url (up to 1024 characters) to your own web hook endpoint. This is useful if you want to be updated as the check progresses. |
| plan | server | Y | The plan field indicates the check's plan. This is copied from the account's plan when the check is created, and never subsequently updated. The possible values are:<ul><li>BUSINESS (CheckPlease handles verification, payment in advance with credit card)<li>CREDIT (CheckPlease handles verification, monthly invoicing)<li>MEMBER (Account owner provides credentials and handles verification, monthly invoicing)</ul> |
| clientKey | server | Y | Automatically generated by CheckPlease and can be used to fetch the result and request pdfs. |
| canChangePriority | server | Y | true if the check's priority can be changed with the PATCH /checks/byExternalRef/{ref} API. A check's priority can not usually be changed once payment is complete or the request has been submitted to the vendor. |    
| eta | server | Y | The date and time (as defined by [RFC 3339, section 5.6.](http://tools.ietf.org/html/rfc3339#section-5.6)) that the check is expected to be complete.<br /><br />This can change up until the request is made to the vendor, e.g. due to delays in verification.<br /><br />Null when the check is complete.|
| status | server | Y | The status field indicates the stage the check is currently at. Until the bundle is verified, this will be the bundle's status (see end of this document), then it will be the status of the check itself. |
| statusLabel | server | Y | The status label contains a human-readable statement of the check's current status. Until the bundle is verified, this will be the bundle's status label (see end of this document), then it will be the status label of the check itself. |
| statusImage | server | Y | The url of an image that conveys the status visually. |
| msInStatus | server | Y | The number of millseconds that the check has been in its current status. |
| requestComplete | server | Y | true once the request pdf (i.e. the document that CheckPlease sends to the vendor) has been built (not necessarily sent). This happens towards the end of the process, after the individual has uploaded their data and their ID has been verified.<br /><br />When true, you can fetch the request pdf with GET /checks/requests/{clientKey}/request. | 
| resultComplete | server | Y | true once the result pdf (i.e. the document that the vendor provides) has been received. | 
| vendorReference | server | N | The reference code (if any) allocated by the vendor once the request has been submitted to them - until then the field is null. |


Fields in the bundle object
===========
| Field        | Set by         | Mandatory? | Notes  |
| ------------- |:-------------:|:-----:|---|
| reference | server | Y | A unique 8 digit numeric reference code allocated by CheckPlease. |
| individualFirstName | client | Y | First name for the individual (person having the check done on them).|
| individualLastName | client | Y ||
| individualEmail | client | Y ||
| individualMiddleNames | client | N | When there are multiple middle names then comma separate them. |
| bundleKey | client | Y | The bundle key must be provided by the client to control which bundle the check will be created in. If in doubt, consider using the 
individual's email address. |


Fetching the request and result documents
==========================
A check is performed by sending a pdf to the vendor (e.g. the Ministry of Justice for a  request document). Later, the vendor sends back a result pdf. Both of these documents (when available) can be accessed via API.

Use the **GET /checks/byClientkey/{clientKey}/result** API to fetch the result pdf for a check.

Use the matching **GET /checks/byClientKey/{clientKey}/request** API to get the request pdf. 

You can obtain the client key from the check object.

For example, to stream the result pdf for the check with client key bb5f03a6-0e1e-6a2c-ab08-ba9461a09137:  

````
curl https://api.checkplease.co.nz/api/checks/byClientkey/bb5f03a6-0e1e-6a2c-ab08-ba9461a09137/result \
-H 'Authorization: OmV81mPqcQ_5vb3R9UjTQPelTmcvfeAbgqO5ptkK'

````

Use the check's **requestComplete** (true or false) field to learn whether the request document is available.  

Use the check's **resultComplete** (true or false) field to learn whether the result document is available.

**Security:** Accessing these links will return 400 unless the Authorization header includes the API key, so they can't be used directly from a browser. Instead, your app should present its own links for the documents to the end user (protected by your own authentication) and then proxy incoming requests to these APIs, with your API key in the Authorization header. This protects these sensitive documents from any unauthorized access. 


Using web hooks to track changes to your checks
====
When you create a check with a web hook specified, whenever the check's status is modified, CheckPlease will POST to your web hook endpoint, passing the updated check object in the request body.

This lets your server stay updated as the check progresses through each stage, eventually leading up to a result. For example, your server might change its own progress indicator on each incoming web hook call, to keep your end user informed.

The web hook is not called for the very first transition (i.e. when a check is created) - instead the full check is available to you in the response to your POST /api/checks API call.  

**Response status:** When your web hook endpoint succeeds, you should return http status 200. However CheckPlease currently only calls your web hook once per update, regardless of the response - there is no e.g. retry with exponential backoff if your endpoint returns a non-200 response.  

**Security:** CheckPlease attaches your API key in the Authorization header in its requests to your web hook. For security reasons, you should always check that the incoming Authorization header matches your own API key, otherwise an attacker with knowledge of your urls could theoretically call your server with false updates on your checks.  


Patching a check
============
Use the **PATCH /checks/byExternalRef/{externalRef}** API to modify a check's priority (currently on an MoJ check only).

The priority is the only value that you can modify on a check, and you can only modify it:
- on MoJ checks
- before payment (unless your check was created while your account is on a CREDIT or MEMBER plan)
- before submission to the MoJ (all checks)

You can use the canChangePriority field on the check to test whether this API call can be made.

The externalRef is the value that you passed earlier when calling POST /mojChecks to create the check.

For example, to update the check created earlier to SILVER priority, from the command line using curl:

````
curl -X PATCH https://api.checkplease.co.nz/api/checks/byExternalRef/100233 \
-H 'Content-Type: application/merge-patch+json' \
-H 'Authorization: OmV81mPqcQ_5vb3R9UjTQPelTmcvfeAbgqO5ptkK' \
-d @- << EOF
{
  "priority": "SILVER"
}
EOF
````

Note you must set the Content-Type header correctly.
 
CheckPlease may return one of the following http statuses:
- 200 if the check was successfully updated
- 400 if the check could not be updated (for example, because the check has already been submitted to MoJ)
- 404 if the check could not be found  
   

Fetching your account details
===========
Use the **GET /account** API to fetch your account details.

This simple call does not modify CheckPlease, and is useful when testing connectivity, e.g. that you have the correct API key.
 
For example, to fetch your account details, from the command line using curl:

````
curl https://api.checkplease.co.nz/api/account \
-H 'Authorization: OmV81mPqcQ_5vb3R9UjTQPelTmcvfeAbgqO5ptkK'
````
The response contains your account details.

````
{
  "companyName": "Big Store Enterprises",
  "shortCode": "bigstore",
  "accountOwnerFirstName": "Stephanie",
  "accountOwnerLastName": "Higgins",
  "accountOwnerEmail": "stephanie.higgins@bigstore.com",
  "defaultPriority": "BRONZE",
  "demo": true,
  "plan": "BUSINESS"
}
````


Bundle statuses
==========
The following shows the statuses that a bundle can pass through as:
- the check is ordered
- the individual consents and provides their personal data, ID document and online signature
- the verifier checks the individual's ID document
- the check is possibly returned to the individual to remedy problems (e.g illegible ID document)
- eventually the identity is verified
 
Your web hook is called each time a check transitions to a new status. 

````
* Diagram does not show all possible transitions

        POST /checks
             |
             /\Is this a credit account? 
            /  \         +------------------+
           /    \--------| PAYMENT_REQUIRED |
           \    / no     +------------------+
            \  /           |
             \/            |
             | yes         |
             |<------------+
             |
       +---------------+
       | ORDERED       |
       +---------------+
             |
       +-----|---------+   +----------+
       |     \/        |-->| DECLINED |
  +----->I_DETAILS     |   +----------+
  |    |     \/        |
  |    | I_DOCUMENT    | 
  |    |     \/        |
  |    | I_CONFIRM     |
  |    |     \/      Individual
  |    | I_SIGNATURE  experience
  |    |     |         |      +----------
  |    |     |         |----->|
  |    +-----|---------+      | CANCELLED_BY_CLIENT
+----------+ |            +-->|
| BOUNCED  | |            |   +---------
+----------+ |            |
  /\         |            |
  |    +-----|-----------------+
  |    |    \/                 |
  |    | V_ID_TYPE             |
  +----|     \/                |   +--------+
       | V_ID_NAME             |-->| BAD_ID |
       |     \/                |   +--------+
       | V_DOB                 |
       |     \/                |
       | V_EXPIRY              |
       |     \/             Verify
       | V_SIGNATURE        experience
       |     \/                |
       | V_3RD_PARTY_SIGNATURE |
       |     |                 |
       +-----|-----------------+
             |
             \/
     +---------------+
     | VERIFIED      |
     +---------------+
```` 
