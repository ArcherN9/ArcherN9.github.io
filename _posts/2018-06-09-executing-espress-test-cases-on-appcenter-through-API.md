---
title: "Execute UI Tests on App Center through APIs"
author: ArcherN9
date: 2018-06-09 10:00:00 +0000
description: "App center has an API first policy. Easier said than done."
category: [Android]
tags: [AppCenter, API, Integrations, UI Testing]
image:
  src: "/assets/img/azure-featureimage.png"
---

Visual Studio App center (VSAC) is great! It virtually eliminates the humongous
effort it usually requires to setup a DevOps pipeline. As an end user, I am
completely satisfied with what is on offer. As a developer trying to drive it
through APIs, its a nightmare!

**Why a nightmare?**
It's because it had me stuck on a problem for upto 2 days that would have never
risen had there been proper documentation available. Albeit, they do have a
swagger notation website ([here](https://openapi.appcenter.ms/)) which attempts
to extensively document all aspects of rest APIs available. The platform is still
in its incessant development stage. I hope the documentation will become better
with time.

Pro tip : Read the swagger notation in JSON and not the UI. The UI rendered
leaves out a lot of information. I suggest leveraging
[Json Editor Online](https://jsoneditoronline.org/) to make it human readable.

This post aims to guide on executing Espresso tests on VSAC using rest APIs.
As of writing this post, the API version is tagged `v0.1`.

This article assumes you have carried out the following basic steps:

- Creating a user account;
- Creating an organization;
- Either signing up for a trial or have premium access to VSAC;

## Creating a test series

A test series is a logical group that holds certain UI tests. They are used to
define the nature of UI tests that are executed under this group. To create a
new test series, execute the following:

```javascript
POST https://api.appcenter.ms/v0.1/apps/<organization_name>/<application_name>/test_series

Headers:
- X-API-Token 	: <API_token>
- Content-Type	: application/json

Body (application/json):
{ name: <test_series_name> }

Example Response:
{
    "name": "Master New",
    "slug": "master-new",
    "mostRecentActivity": "2018-06-09T13:19:23.395Z",
    "testRuns": []
}
```

The API will generate a `slug` for your test-series. This `slug` is used to
refer to this test-series in suceeding calls.

## Get test series

Check if the newly created series reflects on requesting for the entire set.
This API may also be used to get the number of tests executed per test-series
and their results.

```javascript
GET https://api.appcenter.ms/v0.1/apps/<organization_name>/<application_name>/test_series

Headers: 
- X-API-Token 	: <API_token>
- Content-Type	: application/json

Example Response:
[
    {
        "name": "Master New",
        "slug": "master-new",
        "mostRecentActivity": "2018-06-09T13:19:12.236Z",
        "testRuns": []
    },
    {
        "name": "Master",
        "slug": "master",
        "mostRecentActivity": "2018-06-08T12:35:19.936Z",
        "testRuns": [
            {
                "date": "2018-06-08T12:35:19.936Z",
                "statusDescription": "Done",
                "failed": 0,
                "passed": 1,
                "completed": true
            }
        ]
    }
]
```

## Executing tests on VSAC is a four step process

- **First**: Creating a test run. This creates a `test_run_id` for you. Every
`test_run_id` has a state which may be retrieved from `/state` end point.
- **Second**:  Creating file hashes. This API is to intimate VSAC that you wish
to upload APKs for testing. For the platform to accept an upload, it needs to
know the `SHA-1` hashes of the files. 
- **Third**: Uploading the APKs as a `multipart/form-data`
- **Fourth**: Executing the test


### First : Create a new test run

```javascript
POST https://api.appcenter.ms/v0.1/apps/<organization_name>/<application_name>/test_runs

Headers:
- X-API-Token 	: <API_token>
- Content-Type	: application/json

Response:
HTTP Status Code : 201 Created
Headers :
location →/v0.1/apps/{application_id}/test_runs/<test_run_id>
```

Notice the API does not respond with any body, neither the usual 200 HTTP status
code. Instead, you receieve a [201 HTTP statuscode](https://httpstatuses.com/201).

### Second: Creating file hashes

To create a file hash, ensure you have successfully generated 2 APK files -
Build APK & an Android test APK that contains your Espresso test cases. To do
that, navigate to your Android project and execute the following

```sh
$ ./gradlew assembleDebug assembleAndroidTest
```

The APKs will be generated in `app/build/` folder. To find the `SHA-1` hash,
navigate to each of the folder that contains your APKs and execute:

```sh
$ shasum <file-name>.apk

# Example output
4eb758d8d46da4711bb814ce6e6099c696411111  <file-name>.apk
```

On executing each of the commands, the terminal will print a cryptic value.
These values will be used in the `/hashes/batch` API.

```javascript
POST https://api.appcenter.ms/v0.1/apps/<organization_name>/<application_name>/test_runs/<test_run_id>/hashes/batch

Headers:
- X-API-Token 	: <API_token>
- Content-Type	: application/json

Example Body (application/json): 
[
	{
		"file_type":"app-file", //Specifies Build APK
		"checksum":"df798b4d07597db804546b8ca723780992811111", //Checksum for the build APK
		"relative_path":"app-debug.apk" // No need to give the entire path, just the file name is adequate
        //Note : If key name 'relative_path' does not work, try 'relativePath'. 
	}, {
		"file_type":"test-file", // Specifies Build APK
		"checksum":"4eb758d8d46da4711bb814ce6e6099c696411111", // Checksum for the test APK
		"relative_path":"app-debug-androidTest.apk" // No need to give the entire path, just the file name is adequate
        //Note : If key name 'relative_path' does not work, try 'relativePath'.
	}
]

Example Response:
[
    {
        "fileType": "app-file",
        "checksum": "df798b4d07597db804546b8ca723780992811111",
        "relativePath": "app-debug.apk",
        "uploadStatus": {
            "statusCode": 412,
            "location": "https://testcloud.xamarin.com/v0.1/direct_uploads?token=<VSAC_generated_token_here>"
        }
    },
    {
        "fileType": "test-file",
        "checksum": "4eb758d8d46da4711bb814ce6e6099c696411111",
        "relativePath": "app-debug-androidTest.apk",
        "uploadStatus": {
            "statusCode": 412,
            "location": "https://testcloud.xamarin.com/v0.1/direct_uploads?token=<VSAC_generated_token_here>"
        }
    }
]
```

### Third: Uploading the APKs

The response of `/hashes/batch` API carries a `location` key for each of the
files for which hashes have been submitted. This is the URL where the respective
APKs need to be uploaded. Ensure you upload each APK to their respective URLs.

```javascript
POST https://testcloud.xamarin.com/v0.1/direct_uploads?token=<VSAC_generated_token_here>

Headers: 
- X-API-Token 	: <API_token>
- Content 	: multipart/form-data
- Content-Type	: application/x-www-form-urlencoded

Body (form-data):
- relative_path : <relative_path_of_APK_being_uploaded_as_received_in_previous_API>
- file 		: <Your APK File>
- file_type	: <fileType_as_received_in_previous_API>

Response:
HTTP Status Code : 201 Created
```

If you get HTTP status code 201 as a response for each of the files, you are set
to execute your test cases. 

### Fourth: Execute the test case

To execute a test case, you need to specify which devices the test should run on.
To create a new device set, please refer VSAC's swagger page. I created a set
from the portal. To specify a device, you need a `device_slug` retrieved by
`/device_sets` end point.

```javascript
GET https://api.appcenter.ms/v0.1/apps/<organization_name>/<application_name>/owner/device_sets

Headers:
- X-API-Token   : <API_token>
- Content-Type  : application/json

Example response :
[
    {
        "id": "a7a0b57e-0d51-4129-9834-fe17ff5c0a63",
        "name": "Galaxy Note",
        "slug": "galaxy-note",
        "osVersionCount": 1,
        "manufacturerCount": 1,
        "owner": { //Owner information },
        "deviceConfigurations": [
            {
                "id": "f243fd17-e403-43b7-af55-680d8cdf86b4",
                "image": { //Device image URL },
                "os": "7.1.1",
                "osName": "Android 7.1.1",
                "model": { //Model Information }
            }
        ]
    }
]
```

With a test run created, APK files uploaded & a `device_slug` retrieved, we are
ready to execute it on Appcenter's devices! 

```javascript
POST https://api.appcenter.ms/v0.1/apps/<organization_name>/<application_name>/test_runs/<test_run_id>/start

Headers: 
- X-API-Token 	: <API_token>
- Content-Type	: application/json

Body (application/json):
{
	"test_framework" 	: "espresso",
	"device_selection" 	: <organization_name>/<device_slug>,
	"language" 		: "java",
	"locale" 		: "en_US",
	"test_series" 		: <test_series_name>,
	"test_parameters" 	: {} //You may add custom test parameters here.
}

Example Response:
{
    "accepted_devices": [
        "Samsung Galaxy Note 8 (7.1.1)"
    ],
    "rejected_devices": []
}
```

If you receieve a similar response from VSAC, congratulations! You have
successfully executed your UI test case through REST APIs! To confirm, you may
execute `/state` API or simply head over to VSAC to have a visual confirmation.

![VSAC Visual confirmation](/assets/img/VSAC_Test_run.png)

#### Update Log

1. 10th June : Added `device_sets` API
