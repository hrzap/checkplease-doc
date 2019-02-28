NZ Criminal Record check API
=================
CheckPlease provides a simple, secure API for ordering and managing Criminal Record checks from the [NZ Ministry of Justice](https://www.justice.govt.nz/criminal-records/).

An individual's criminal record is subject to the Criminal Records (Clean Slate) Act 2004, and covers criminal and traffic convictions, but does not include charges that haven't gone to court yet, infringements and charges where the individual was not convicted.

The following endpoints are available:

| API        | Purpose   |
| -------- |--------|
| POST&nbsp;/mojChecks | Order a new check |
| GET&nbsp;/mojChecks/byExternalRef/{externalRef} | Get the current state for a check.  |
| Your web hook | Be alerted when a check is updated. You pass your web hook's url in when you create a check. |
| GET&nbsp;/mojChecks/byClientKey/{clientKey}/request | Fetch the request pdf for a check (sent to MoJ). | 
| GET&nbsp;/mojChecks/byClientkey/{clientKey}/result | Fetch the result pdf for a check (received from MoJ). |
| PATCH&nbsp;/mojChecks/byExternalRef/{externalRef} | Modify a check's priority. |
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
Use the **POST /mojChecks** API to order a check.

For example, to order a check on Fred Flintstone with 3 day turnaround (from the command line, using curl):

````
curl -X POST https://api.checkplease.co.nz/api/mojChecks \
-H 'Content-Type: application/json' \
-H 'Authorization: OmV81mPqcQ_5vb3R9UjTQPelTmcvfeAbgqO5ptkK' \
-d @- << EOF
{
  "individualFirstName": "Fred",
  "individualMiddleNames": "Rocky",
  "individualLastName": "Flintstone",
  "individualEmail": "fred@flintstone.com",
  "priority": "GOLD",
  "externalRef": "100233",
  "webHook": "https://myserver.com/checkWebHook
}
EOF

````
The response contains the check object, now with the server-assigned fields included:

````
{
  "individualFirstName": "Fred",
  "individualMiddleNames": "Rocky",
  "individualLastName": "Flintstone",
  "individualEmail": "fred@flintstone.com",
  "priority": "GOLD",
  "plan": "BUSINESS",
  "externalRef": "100233",
  "webHook": "https://myserver.com/checkWebHook",
  "status": "ORDERED",
  "lastStatusChange": "2018-10-02T15:05:26.05Z",
  "lastUpdate": "2018-10-02T15:05:26.05Z",
  "eta": "2018-10-17T12:03:44.05Z",
  "reference": 200346653,
  "mojReference": null,
  "hasCriminalRecord": null,
  "requestComplete": false,
  "canChangePriority": true,
  "clientKey": "bb5f03a6-0e1e-6a2c-ab08-ba9461a09137",
  "verifierKey": "ba560021-1e52-6452-756d-74edc53b0123"
}
````

CheckPlease may return one of the following http statuses:
- 200 if the check was successfully created
- 400 if the check could not be created (for example, because firstName or lastName were missing)

Getting current state for a check
====
Use the **GET /mojChecks/byExternalRef/{externalRef}** API to get a check's current state.

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
If possible, you should try and avoid calling this API repeatedly to see if your check has changed (i.e. polling). Instead, pass a web hook in when you create the check. Then your integration will be notified in real time as soon as your check changes.

Using a web hook instead of polling is:
- more real-time: your code will be instantly alerted when your checks change their state, and your integration will be always up to date
- more efficient: since checks tend to change slowly (e.g. many days may elapse before the MoJ provides the result), web hooks require far less network traffic than polling
- more reliable: your API calls will never fail due to rate limiting (the API is not tightly rate limited but may be in the future)

Fields in the check object
===========
| Field        | Set by         | Mandatory? | Notes  |
| ------------- |:-------------:|:-----:|---|
| individualFirstName | client | Y | First name for the individual (person having the check done on them).|
| individualLastName | client | Y ||
| individualEmail | client | Y ||
| individualMiddleNames | client | N | When there are multiple middle names then comma separate them. |
| priority | client | N | The possible values for priority are: <ul><li>GOLD (3 working days turnaround)<li>SILVER (10 working days turnaround)<li>BRONZE (15 working days turnaround)</ul> If you don't provide a value for priority, then the check will be created using your CheckPlease account's default priority. |
| externalRef | client | N | The optional externalRef  field can be used to store your own reference (up to 512 characters) on the check. This is useful if you intend to update the check's priority after it has been created. |
| webHook | client | N | The optional webHook field can be used to store a url (up to 1024 characters) to your own web hook endpoint. This is useful if you want to be updated as the check progresses. |
| status | server | Y | The status field indicates the stage the check is currently at. The possible statuses are shown at the end of this document. |
| lastStatusChange | server | Y | The date (as defined by [RFC 3339, section 5.6.](http://tools.ietf.org/html/rfc3339#section-5.6)) that the check entered its current status. |
| lastUpdate | server | Y | The date (as defined by [RFC 3339, section 5.6.](http://tools.ietf.org/html/rfc3339#section-5.6)) that the check last changed - i.e.:<ul><li>entered its current status<li>an email from the individual was received<li>an email from the MoJ was received<li>The priority was changed</ul> |
| eta | server | Y | The date and time (as defined by [RFC 3339, section 5.6.](http://tools.ietf.org/html/rfc3339#section-5.6)) that the check is expected to be complete.<br /><br />This can change up until the request is made to MoJ, e.g. due to delays in verification.<br /><br />Null when the check is complete.|
| reference | server | Y | A unique 8 digit numeric reference code allocated by CheckPlease. |
| mojReference | server | N | The reference code allocated by MoJ once the request has been submitted to them - until then the field is null. |
| requestComplete | server | Y | true once the request pdf (i.e. the document that CheckPlease sends to MoJ) has been built. This happens towards the end of the process, after the individual has uploaded their data and their ID has been verified.<br /><br />When true, you can fetch the request pdf with GET /mojChecks/requests/{clientKey}/request. | 
| hasCriminalRecord | server | N | Present and non-null at the end of the process, once CheckPlease has received the result pdf back from the MoJ for the individual. True means that the individual has a criminal record, false means they don't.<br /><br />When the field is present and non-null, you can fetch the pdf result document itself (containing specific details of the individual's convictions, or lack of) with GET /mojChecks/requests/{clientKey}/result. |
| canChangePriority | server | Y | true if the check's priority can be changed with the PATCH /mojChecks/byExternalRef/{ref} API. A check's priority can not be changed once payment is complete or the request has been submitted to MoJ. |    
| clientKey | server | Y | Automatically generated by CheckPlease and can be used to fetch the result and request pdfs. |
| verifierKey | server | Y | Automatically generated by CheckPlease and can be used to perform verification (Checks on Member plan only) |
| plan | server | Y | The plan field indicates the check's plan. Copied from the account's plan when the check is created, and never subsequently updated. The possible values are:<ul><li>BUSINESS (CheckPlease handles verification, payment in advance with credit card)<li>CREDIT (CheckPlease handles verification, monthly invoicing)<li>MEMBER (MoJ CCH online member, account owner handles verification, monthly invoicing)</ul> |


Fetching the request and result documents
==========================
A criminal record check is performed by sending a pdf to the Ministry of Justice (the request document). Later, MoJ send back a result pdf. Both of these documents (when available) can be accessed via API.

Use the **GET /mojChecks/byClientkey/{clientKey}/result** API to fetch the result pdf for a check.

Use the matching **GET /mojChecks/byClientKey/{clientKey}/request** API to get the request pdf. 

Get the client key from the check object.

For example, to stream the result pdf for the check with client key bb5f03a6-0e1e-6a2c-ab08-ba9461a09137:  

````
curl https://api.checkplease.co.nz/api/mojChecks/byClientkey/bb5f03a6-0e1e-6a2c-ab08-ba9461a09137/result \
-H 'Authorization: OmV81mPqcQ_5vb3R9UjTQPelTmcvfeAbgqO5ptkK'

````

Use the check's **requestComplete** (true or false) field to learn whether the request document is available.  

Use the check's **hasCriminalRecord** (null or not null) field to learn whether the result document is available.

**Security:** Accessing these links will return 400 unless the Authorization header includes the API key, so they can't be used directly from a browser. Instead, your app should present its own links for the documents to the end user (protected by your own authentication) and then proxy incoming requests to these APIs, with your API key in the Authorization header. This protects these sensitive documents from any unauthorized access. 


Using web hooks to track changes to your checks
====
When you create a check with a web hook specified, whenever the check's status is modified, CheckPlease will POST to your web hook endpoint, passing the updated check object in the request body.

This lets your server stay updated as the check progresses through each stage, eventually leading up to a result. For example, your server might change its own progress indicator on each incoming web hook call, to keep your end user informed.

The web hook is not called for the very first transition (i.e. when a check is created) - instead the full check is available to you in the response to your POST /api/mojChecks API call.  

**Response status:** When your web hook endpoint succeeds, you should return http status 200. However CheckPlease currently only calls your web hook once per update, regardless of the response - there is no e.g. retry with exponential backoff if your endpoint returns a non-200 response.  

**Security:** CheckPlease attaches your API key in the Authorization header in its requests to your web hook. For security reasons, you should always check that the incoming Authorization header matches your own API key, otherwise an attacker with knowledge of your urls could theoretically call your server with false updates on your checks.  


Patching a check
============
Use the **PATCH /mojChecks/byExternalRef/{externalRef}** API to modify a check's priority.

The priority is the only value that you can modify on a check, and you can only modify it:
- before payment (unless your check was created while your account is on a CREDIT or MEMBER plan)
- before submission to the MoJ (all checks)

You can use the canChangePriority field on the check to test whether this API call can be made.

The externalRef is the value that you passed earlier when calling POST /mojChecks to create the check.

For example, to update the check created earlier to SILVER priority, from the command line using curl:

````
curl -X PATCH https://api.checkplease.co.nz/api/mojChecks/byExternalRef/100233 \
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
  "plan": "BUSINESS"
}
````

Fetching status metadata
=================
Use the **GET /api/mojChecks/metaData** to fetch metadata for a check's status. 

If you are building a sophisticated integration where you want to keep the user fully informed as checks progress through each stage, eventually leading up to a result, then you can use this API to fetch metadata about any status, such as its description, whether it requires the user's attention, etc.

As always you need to pass the API key in the Authorization header. However the metadata is universal, applies to every CheckPlease acccount, and changes very infrequently, so you only need to fetch the metadata once e.g. when your integration first starts up.  

For example, to fetch metadata from the command line using curl:

````
curl https://api.checkplease.co.nz/api/mojChecks/metadata \
-H 'Authorization: OmV81mPqcQ_5vb3R9UjTQPelTmcvfeAbgqO5ptkK'

Response:
{
  "statuses": [
    ...
    {
      "name": "V_ID_TYPE",
      "statusGroupMappings": [
        {
          "plan": "BUSINESS",
          "statusGroup": "VERIFYING"
        },
        {
          "plan": "MEMBER",
          "statusGroup": "VERIFY_NOW"
        }
      ]
    },
    ...
  ],
  "statusGroups": [
    ...
    {
      "name": "VERIFYING",
      "attentionRequired": true,
      "complete": false,
      "imagePrefix": "verifying",
      "description": "The individual's ID is being verified by CheckPlease."
    },
    {
      "name": "VERIFY_NOW",
      "attentionRequired": true,
      "complete": false,
      "imagePrefix": "pleaseVerify",
      "description": "Waiting for you to verify the individual's ID. Please click the tracking link and verify now."
    },
    ...
  ]
}
````

Each status belongs in a **status group** which is a rollup that makes sense to the user at a higher level. For example there are 4 separate statuses (I_DETAILS onwards) used as an individual is entering their data, however they all map to a single status group (INDIVIDUAL). The user may not care that the individual is uploading their ID document - they just need to know that the check is with the individual.    

The mapping from any status to its status group depends on the check's plan. There are three plans (BUSINESS, CREDIT and MEMBER) that a check can have, but you should treat CREDIT as BUSINESS for the purpose of this API.
 
For example, if we are working on a check that is in the V_ID_TYPE status, and is on the CREDIT plan, then we can see from the metadata above that:
- the status group is VERIFYING
- the user's attention is not required (i.e. things are rolling along)
- the check is not complete (there are more status transitions to come)
- the image prefix is "verifying" (see more below)  
- the user-facing description is "The individual's ID is being verified by CheckPlease."
 
Status images
---------
The CheckPlease UI presents an image next to each check, that shows the user at a glance the check's status group (e.g. VERIFYING) and priority (e.g. GOLD).

You can also link to these images directly from your own user interface, to give the user a consistent experience.

You assemble the url for a check image like this:
````
https://checkplease.co.nz/img/status/<imagePrefix>_<lowerCasePriority>.png
````
So if you have a check which is:
- status == V_ID_TYPE status
- plan == MEMBER
- priority == GOLD

..then the imagePrefix from the status group is *pleaseVerify* and the image url will be:
````
https://checkplease.co.nz/img/status/pleaseVerify_gold.png
````
![Image for check: V_ID_TYPE, MEMBER, GOLD](https://checkplease.co.nz/img/status/pleaseVerify_gold.png)

In the CheckPlease UI we use CSS to add a border and drop shadow effect to the image. 


Details of the different statuses
==========
The following shows the statuses that a criminal record check can pass through as:
- the check is ordered
- the individual consents and provides their personal data, ID document and online signature
- the verifier checks the individual's ID document
- the check is possibly returned to the individual to remedy problems (e.g illegible ID document)
- the request pdf is generated and set to the Ministry of Justice
- the result is received back from the Ministry of Justice
 
Your web hook is called each time a check transitions to a new status. 

````
* Diagram does not show all possible transitions

        POST /mojChecks
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
             /\account.manualRelease == true 
            /  \         +----------------+
           /    \--------| MANUAL_RELEASE |
           \    / yes    +----------------+
            \  /           |
             \/            |
             | no          |
             |<------------+
             |
             \/
     +---------------+   +--------------
     | M_REQUEST     |-->| M_BOUNCED
     +---------------+   +-------------
             |
             \/
     +-------------------+
     | M_ACKNOWLEDGEMENT | 
     +-------------------+
        |      |
        |      \/   
        |   +-------------------------+
        |   | COMPLETE_WITH_NO_RECORD |
        |   +-------------------------+
        \/
     +----------------------+
     | COMPLETE_WITH_RECORD | 
     +----------------------+
```` 
