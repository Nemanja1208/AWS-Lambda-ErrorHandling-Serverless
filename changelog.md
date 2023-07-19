### Documentation source - `https://aws.amazon.com/tutorials/handle-serverless-application-errors-step-functions-lambda/`

# Implementation steps

1. Create a `Lambda` function to mock an API

   In this step, you will create a Lambda function that will mock a few basic API interactions. The Lambda function raises exceptions to simulate responses from a fictitious API, depending on the error code that you provide as input in the event parameter.

   - Open the AWS Management Console, so you can keep this step-by-step guide open. When the screen loads, enter your user name and password to get started. Next, enter lambda in the search bar and select Lambda to open the service console. Choose Create function.

   - Leave `Author` from scratch selected. Next, configure your `Lambda` function as follows:

     - For `Function name`, enter MockAPIFunction.
     - For `Runtime`, choose Python 3.9.
     - For `Architecture`, choose arm64.
     - For `Role`, select Create a new role with basic `Lambda` permissions.

   - Choose Create function.

   - On the MockAPIFunction screen, scroll down to the Code source section. In this tutorial, you will create a function that uses the programming model for authoring `Lambda` functions in Python. In the code window, replace all of the code with the following, then choose Deploy.

     ```
         class TooManyRequestsException(Exception): pass
         class ServerUnavailableException(Exception): pass
         class UnknownException(Exception): pass

         def lambda_handler(event, context):
             statuscode = event["statuscode"]
             if statuscode == "429":
             raise TooManyRequestsException('429 Too Many Requests')
             elif statuscode == "503":
             raise ServerUnavailableException('503 Server Unavailable')
             elif statuscode == "200":
             return '200 OK'
         else:
         raise UnknownException('Unknown error')
     ```

   - Make note of the function `Amazon Resource Name (ARN)`. ARNs uniquely identify AWS resources, and help you track and use AWS items and policies across AWS services and API calls. We require an ARN when you need to reference a specific resource from `Step Functions`. The ARN is found in the Function overview section of the Function page.

2. Create an AWS Identity and Access Management `(IAM)` role

   AWS `Step Functions` can run code and access other AWS resources (for example, data stored in Amazon S3 buckets). To maintain security, you must grant Step Functions access to these resources using AWS Identity and Access Management `(IAM)`.

   - In another browser window, navigate to the AWS Management Console and enter IAM in the search bar. Select `IAM` to open the service console. Select Roles, then choose Create role.

   - In the Select trusted entity section, leave the default of AWS service. Under Use case, select `Step Functions` from the Use cases for other AWS services dropdown, then choose Next. On the Add permissions page, choose Next.

   - On the Name, review, and create page, enter `step_functions_basic_execution` for Role name and choose Create role. Your new IAM role is created and appears in the list.

3. Create a `Step Functions` state machine

   Now that you’ve created your simple Lambda function that mocks an API response, you can create a `Step Functions` state machine to call the API and handle exceptions.

   In this step, you will use the Step Functions console to create a state machine that uses a Task state with a Retry and Catch field to handle the various API response codes. You will use a Task state to invoke your mock API `Lambda` function, which will return the API status code you provide as input into your state machine.

   - Open the `AWS Step Functions console`. On the Create state machine page, select Write your workflow in code.

   - Next, you will design a state machine that will take different actions depending on the response from your mock API. If the API can’t be reached, the workflow will try again. Retries are a helpful way to address transient errors. The workflow also catches different exceptions thrown by the mock API. Replace the contents of the state machine definition section with the following code:

     ```
     {
     "Comment": "An example of using retry and catch to handle API responses",
     "StartAt": "Call API",
     "States": {
         "Call API": {
             "Type": "Task",
             "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:FUNCTION_NAME",
             "Next" : "OK",
             "Comment": "Catch a 429 (Too many requests) API exception, and resubmit the failed request in a rate-limiting fashion.",
             "Retry" : [ {
                 "ErrorEquals": ["TooManyRequestsException" ],
                 "IntervalSeconds": 1,
                 "MaxAttempts": 2
             } ],
             "Catch": [
                 {
                     "ErrorEquals": ["TooManyRequestsException"],
                     "Next": "Wait and Try Later"
                 }, {
                     "ErrorEquals": ["ServerUnavailableException"],
                     "Next": "Server Unavailable"
                 }, {
                     "ErrorEquals": ["States.ALL"],
                     "Next": "Catch All"
                 }
             ]
         },
         "Wait and Try Later": {
             "Type": "Wait",
             "Seconds" : 1,
             "Next" : "Change to 200"
         },
         "Server Unavailable": {
             "Type": "Fail",
             "Error":"ServerUnavailable",
             "Cause": "The server is currently unable to handle the request."
         },
         "Catch All": {
             "Type": "Fail",
             "Cause": "Unknown error!",
             "Error": "An error of unknown type occurred"
         },
         "Change to 200": {
         "Type": "Pass",
         "Result": {
             "statuscode" :"200"
         },
         "Next": "Call API"
         },
         "OK": {
             "Type": "Pass",
             "Result": "The request has succeeded.",
             "End": true
         }
         }
     }
     ```

   - Select the recycle button to update the flow diagram:

   - Find the “Resource” line in the “Call API” Task state (line 7). To update this ARN to the ARN of the mock API `Lambda` function you just created, retrieve the `ARN` from Step 1e earlier and replace the placeholder. Choose Next to continue.

   - Fill in the following details:

   - State machine name: MyAPIStateMachine
   - Permissions: Select Choose an existing role and select step_functions_basic_execution.
   - Choose Create state machine.

4. Test your error-handling workflow

   To test your error-handling workflow, you will invoke your state machine to call your mock API by providing the error code as input.

   -

   - A new execution dialog box appears, where you can enter input for your state machine. You will supply the error code that we want the mock API to return. Replace the existing text with the code below, then choose Start execution:

     `{"statuscode": "200"}`

   - On the Execution details screen, choose Execution input and output to see the input and the final output of the state machine. You can see that the workflow interpreted statuscode 200 as a successful API call.

   - Under Graph view, you can see the execution path of each execution, shown in green in the workflow. Select the Call API Task state and then expand the Input and Output fields in the Step details screen.

   - You can see that this Task state successfully invoked your mock API Lambda function with the input you provided, and captured the output of that `Lambda` function, showing “statusCode” : 200.

   - Next, select the OK Task state in the visual workflow. Under Step details you can see that the output of the previous step (the Call API Task state) has been passed as the input to this step. The OK state is a Pass state, which simply passed its input to its output, performing no work. Pass states are useful when constructing and debugging state machines.

5. Inspect the execution of your state machine

   - Scroll to the top of the Execution details screen and select MyAPIStateMachine.

   - Select Start execution again, and this time provide the following input and then choose the Start execution button. `{"statuscode": "503"}`

   - In the Execution event history section, expand each execution step to confirm that your workflow behaved as expected. We expected this execution to fail, so don’t be alarmed. You will notice that:

     - Step Functions captured your Input.
     - That input was passed to the Call API Task state.
     - The Call API Task state called your MockAPIFunction using that input.
     - The MockAPIFunction executed.
     - The MockAPIFunction failed with a ServerUnavailableException.
     - The catch statement in your Call API Task state caught that exception.
     - The catch statement failed the workflow.
     - Your state machine completed its execution.

   - Next, you will simulate a 429 exception. Scroll to the top of the Execution details screen and select MyAPIStateMachine. Select Start execution, provide the following input, and choose the Start execution button: `{"statuscode": "429"}`

   - Now you will inspect the retry behavior of your workflow. In the Execution event history section, expand each execution step once more to confirm that Step Functions tried calling the MockAPILambda function two more times, both of which failed. At that point, your workflow transitioned to the Wait and Try Later state (shown in the screenshot), in the hopes that the API was just temporarily unresponsive.

   - Next, the Wait state used brute force to change the response code to 200, and your workflow completed execution successfully. That probably wouldn’t be how you handled a 429 exception in a real application, but we’re keeping things simple for the sake of the tutorial.

   - Run one more instance of your workflow, and this time, provide a random API response that is not handled by your state machine: `{"statuscode": "999"}`

   - Inspect the execution again using the Execution event history. When complete, select MyAPIStateMachine once more. In the Executions pane, you can see the history of all executions of your workflow, and step into them individually as you like.

   ### Congatulations, you learned how to handle errors with AWS Lambda and Step Functions in a serverless application.

6. Clean up resources

   In this step, you will terminate resources related to AWS Step Functions and AWS Lambda. Terminating resources that are not actively being used reduces costs and is a best practice. Not terminating your resources can result in a charge.

   - At the top of the AWS Step Functions console window, select State machines.

   - In the State machines window, select MyAPIStateMachine and choose Delete. Confirm the action by choosing Delete state machine in the dialog box. Your state machine will be deleted in a minute or two once Step Functions has confirmed that any in-process executions have completed.

   - Next, you will delete your Lambda functions. Enter lambda into the search bar next to Services, then select Lambda.

   - On the Functions screen, select your MockAPIFunction, choose Actions, and then select Delete. Confirm the deletion by choosing Delete again.

   - Lastly, you will delete your IAM roles. Enter IAM into the search bar next to Services, then select IAM.

   - Select both of the IAM roles that you created for this tutorial, then choose Delete. Confirm the deletion by choosing Yes, delete in the dialog box.

   - You can now sign out of the AWS Management Console.
