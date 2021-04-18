## Converting an AWS Step Function to use CDK - Part 1

If you like fluent coding, then [AWS Cloud Development Kit](https://aws.amazon.com/cdk/) step function definitions looks right up your street. However, things are not as straightfoward as you might think.

In my previous post [Easier Step Functions with the AWS Toolkit for VS Code](https://www.10printiamcool.com/easier-step-functions-with-the-aws-toolkit-for-vs-code), I extolled the virtues of using the AWS Toolkit in conjunction with the [AWS Serverless Application Model (SAM)](https://aws.amazon.com/serverless/sam/). Now, I will propose a completely different way of doing things.

This post assumes that you have some familiarity with the CDK. If you are not, then the [Getting started with the AWS CDK](https://docs.aws.amazon.com/cdk/latest/guide/getting_started.html) guide is the best place to start. The following quote from the guide does a good job of providing an overview of the key concept behind CDK.

_"An AWS CDK app is an application written in TypeScript, JavaScript, Python, Java, or C# that uses the AWS CDK to define AWS infrastructure. An app defines one or more stacks. Stacks (equivalent to AWS CloudFormation stacks) contain constructs, each of which defines one or more concrete AWS resources, such as Amazon S3 buckets, Lambda functions, Amazon DynamoDB tables, and so on."_

For this post I took a copy of the original SAM-based [repo](https://github.com/andybalham/blog-source-code/tree/master/step-functions-aws-toolkit), and then amended it to use CDK. The result can be found  [here](https://github.com/andybalham/blog-source-code/tree/master/step-functions-cdk). 

Step functions in CDK require references to the functions they invoke. In the demo project all the functions are in the same file and follow a naming convention. This enabled me to create the following method in the `Stack` class:

```TypeScript
  private addFunction(functionName: string): lambda.Function {
    return new lambdaNodejs.NodejsFunction(this, `${functionName}Function`, {
      entry: path.join(__dirname, '..', 'src', 'functions', 'index.ts'),
      handler: `handle${functionName}`,
    });
  }
```
With this in place, I was able to declare all the required functions as follows:

```TypeScript
    const performIdentityCheckFunction = this.addFunction('PerformIdentityCheck');
    const aggregateIdentityResultsFunction = this.addFunction('AggregateIdentityResults');
    const performAffordabilityCheckFunction = this.addFunction('PerformAffordabilityCheck');
    const sendEmailFunction = this.addFunction('SendEmail');
    const notifyUnderwriterFunction = this.addFunction('NotifyUnderwriter');
```
I could now turn my attention to converting the step function itself. The step function is a simplified flow that processes a loan application. The first steps run an identity check for each applicant and then aggregates the results

With SAM, this was defined with the following YAML:

```YAML
  PerformIdentityChecks:
    Type: Map
    InputPath: "$.application"
    ItemsPath: "$.applicants"
    ResultPath: "$.identityResults"
    Iterator:
      StartAt: PerformIdentityCheck
      States:
        PerformIdentityCheck:
          Type: Task
          Resource: "${PerformIdentityCheckFunctionArn}"
          End: true
    Next: AggregateIdentityResults

  AggregateIdentityResults:
    Type: Task
    Resource: "${AggregateIdentityResultsFunctionArn}"
    InputPath: "$.identityResults"
    ResultPath: "$.overallIdentityResult"
    Next: EvaluateIdentityResults
```
CDK uses a fluent syntax with properties matching those above. So easy I thought, just replicate the same logic in TypeScript:

```TypeScript
const processApplicationStateMachine = new sfn.StateMachine(
  this,
  'ProcessApplicationStateMachine',
  {
    definition: sfn.Chain.start(
      new sfn.Map(this, 'PerformIdentityChecks', {
        inputPath: '$.application',
        itemsPath: '$.applicants',
        resultPath: '$.identityResults',
      })
        .iterator(
          new sfnTasks.LambdaInvoke(this, 'PerformIdentityCheck', {
            lambdaFunction: performIdentityCheckFunction,
          })
        )
        .next(
          new sfnTasks.LambdaInvoke(this, 'AggregateIdentityResults', {
            lambdaFunction: aggregateIdentityResultsFunction,
            inputPath: '$.identityResults',
            resultPath: '$.overallIdentityResult',
          })
        )
    ),
  }
);
```
The next step was for me to test it. Using the AWS Toolkit, I right-clicked on the step function and used one of the JSON test files in the project.

![aws-toolkit-start-execution.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1618509121534/CeFP_bjlD.png)

I then went in to the AWS console and was heartened to see it all green.

![surface-success.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1618509506727/54o5nY3tz.png)

__However...__ looking at the step output for PerformIdentityCheck, I saw the following output:

```JSON
{
  "ExecutedVersion": "$LATEST",
  "Payload": {
    "success": false
  },
  "SdkHttpMetadata": {
      <snip>
  },
  "SdkResponseMetadata": {
    "RequestId": "76e41976-672d-4be0-a4d2-a5b80e7f9afe"
  },
  "StatusCode": 200
}
```
This was not quite what I was expecting, but surely this is easily solved by using the `outputPath` property on the functions to select just the `Payload`. E.g.:

```TypeScript
new sfnTasks.LambdaInvoke(this, 'PerformIdentityCheck', {
  lambdaFunction: performIdentityCheckFunction,
  outputPath: '$.Payload',
})
```
With this change in place, I deployed again, and fired off my test. The result... __abject failure__.

![unexpected-failure.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1618510384828/UaIm_L3wFR.png)

I checked the output of the PerformIdentityCheck step, all was as expected, I checked the output of the map step, and again all was as expected. 

The problem was with the AggregateIdentityResults. It had executed as expected, outputting the following.

```JSON
{
  "resourceType": "lambda",
  "resource": "invoke",
  "output": {
    "ExecutedVersion": "$LATEST",
    "Payload": false,
    <snip>
    "StatusCode": 200
  },
  "outputDetails": {
    "truncated": false
  }
}
```
However, an `Invalid path '$.Payload' : No results for path: $['Payload']` error was being thrown after it had executed.

```JSON
{
  "error": "States.Runtime",
  "cause": "An error occurred while executing the state 'AggregateIdentityResults' (entered at the event id #13). Invalid path '$.Payload' : No results for path: $['Payload']"
}
```
Cue a lost hour trying to work out why `$.Payload` worked for one function task, but not for another. I did eventually get to the bottom of this (see the end of the post), but my investigations led me to the following issue from April 2020: [RunLambdaTask with outputPath not working](https://github.com/aws/aws-cdk/issues/7709) 

This pointed me in the direction of a solution. This was to use the `payloadResponseOnly` property, defined by the [docs](https://docs.aws.amazon.com/cdk/api/latest/docs/@aws-cdk_aws-stepfunctions-tasks.LambdaInvokeProps.html) as follows:
> 'Invoke the Lambda in a way that only returns the payload response without additional metadata.'

E.g.:

```TypeScript
new sfnTasks.LambdaInvoke(this, 'PerformIdentityCheck', {
  lambdaFunction: performIdentityCheckFunction,
  payloadResponseOnly: true,
})
```
With this in place, I re-ran the test, and checked the result in AWS.

```JSON
{
  "application": {
    <snip>
  },
  "identityResults": [
    {
      "success": false
    }
  ],
  "overallIdentityResult": false
}
```
**Hurrah!** This was exactly as expected, with no extraneous data being returned by the function invocations. 

What I did notice, by looking at the generated definitions, is that the generated ASL for invoking the functions different depending on the value of `payloadResponseOnly`. Without `payloadResponseOnly: true` the definition is as generated as follows:

```JSON
    "AggregateIdentityResults": {
      <snap>
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "arn:aws:lambda:eu-west-2:361728023653:function:ProcessApplicationStack-AggregateIdentityResultsFu-B7MG7QWC1VLN",
        "Payload.$": "$"
      }
    }
```
Whilst with `payloadResponseOnly: true`, we get the following that matches the original SAM-based definition:

```JSON
    "AggregateIdentityResults": {
      <snip>
      "Resource": "arn:aws:lambda:eu-west-2:361728023653:function:ProcessApplicationStack-AggregateIdentityResultsFu-B7MG7QWC1VLN"
    }```

This difference must be the reason for the difference in the response, but why that should be I don't know. However, I can now get the results I want, so I shall move on.

That concludes this part, I had anticipated getting further, but that outcome is pretty standard for software development. In part 2 I will continue converting the step function to CDK and record the challenges I encounter on the way.

Edit: The reason for the `Payload` error was due to my misunderstanding of how the paths are processed. The key bit I was missing is below:

_"The OutputPath is computed after applying ResultPath. All service integrations return metadata as part of their response. When using ResultPath, it's not possible to merge a subset of the task output to the input."_