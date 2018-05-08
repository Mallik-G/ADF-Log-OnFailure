# ADF Log OnFailure

ADFv2 allows for conditional branching, for instance OnSuccess and OnFailure. This sample shows how to build a Logic App that can accept an OnFailure condition and log it to Log Analytics.

## Steps

1. Create a Log Analytics instance in Azure.
2. Create a Data Factory v2 instance in Azure.
3. Create a Logic Apps instance in Azure.
4. Create a new connection in your Logic Apps Resource Group that points to your Log Analytics instance.
5. Use the "logicapp.json" file as the code for your Logic App, changing the connection to the appropriate connection.
6. Save the Logic App and inspect the HTTP Request Activity to get the appropriate URL.
7. Use the "pipeline.json" file as an example of an OnFailure Web Activity. You will specify the URL as what you obtained in step 6.

## OnFailure Web Activity

You will have one Web Activity per other Activity in your pipeline. You will wire each of them to the OnFailure event of the monitored Activity.

The Web Activity uses headers to send much of the categorical information because this is much easier to input when you have a lot of activities than trying to craft a JSON body.

The headers used in this example are (but of course, you could use whatever you want):

| Header       | Input                       | Notes                                                     |
| ------------ | ----------------------------|---------------------------------------------------------- |
| log-customer | Hardcoded or from parameter | The name of the customer this pipeline is executing for.  |
| log-factory  | @pipeline().DataFactory     | This expression automatically puts the Data Factory name. |
| log-pipeline | @pipeline().Pipeline        | This expression automatically puts the Pipeline name.     |
| log-activity | Hardcoded                   | Type the name of the activity that failed.                |
| log-status   | Hardcoded                   | The status, you wish to log, probably "Fail" or "Error".  |

The output of the failed Activity (which will be all the error details) is sent in the body to the Logic App.

## Logic App

The Logic App pulls out all the headers and merges them back into the message body and then submits the payload to the Log Analytics instance.

The message that is sent to Log Analytics will look something like this (notice the header fields appended to the bottom):

```json
{
    "copyDuration": 4,
    "errors": [
        {
            "Code": 9003,
            "Message": "ErrorCode=UserErrorInvalidStorageConnectionString,'Type=Microsoft.DataTransfer.Common.Shared.HybridDeliveryException,Message=Invalid storage connection string provided to &apos;UnknownLocation&apos;. Check the storage connection string in configuration.,Source=Microsoft.DataTransfer.Common,''Type=System.FormatException,Message=No valid combination of account information found.,Source=Microsoft.WindowsAzure.Storage,'",
            "EventType": 0,
            "Category": 5,
            "Data": {},
            "MsgId": null,
            "ExceptionType": null,
            "Source": null,
            "StackTrace": null,
            "InnerEventInfos": []
        }
    ],
    "effectiveIntegrationRuntime": "DefaultIntegrationRuntime (East US)",
    "usedCloudDataMovementUnits": 4,
    "usedParallelCopies": 1,
    "executionDetails": [
        {
            "source": {
                "type": "AzureBlob"
            },
            "sink": {
                "type": "AzureBlob"
            },
            "status": "Failed",
            "start": "2018-05-08T15:11:51.7993007Z",
            "duration": 4,
            "usedCloudDataMovementUnits": 4,
            "usedParallelCopies": 1,
            "detailedDurations": {
                "queuingDuration": 3,
                "transferDuration": 0
            }
        }
    ],
    "customer": "Microsoft",
    "factory": "DfIsProsIsCusDv2",
    "pipeline": "pipeline1",
    "activity": "Copy1",
    "status": "Fail"
}
```

## Log Analytics

Log Analytics receives the payload and stores it.

It will change the name you specified for the event to add "_CL".

It will change the field names to describe their datatype, for example, "customer" will become "customer_s".

Keep in mind that events may not show up in Log Analytics for some time (typically less than 15 minutes, but can sometimes be hours).