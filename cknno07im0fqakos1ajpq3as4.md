## Converting an AWS Step Function to use CDK - Part 2

In [Part 1](https://www.10printiamcool.com/converting-an-aws-step-function-to-use-cdk-part-1), we started to convert a state machine from an [ASL](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-amazon-states-language.html) definition used by [SAM](https://aws.amazon.com/serverless/sam/), to a fluent definition written in [CDK](https://aws.amazon.com/cdk/). The full graph of the state machine is shown below:

![process-loan-application-graph.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1618779676238/7Dhl0Lx2i.png)

So far, I managed, thanks to the `payloadResponseOnly` property, to convert the initial states that perform and aggregate identity checks.

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
        payloadResponseOnly: true,
        })
      )
      .next(
        new sfnTasks.LambdaInvoke(this, 'AggregateIdentityResults', {
        lambdaFunction: aggregateIdentityResultsFunction,
        payloadResponseOnly: true,
        inputPath: '$.identityResults',
        resultPath: '$.overallIdentityResult',
        })
      )
    ),
  }
);
```

The next step was to add a `Choice` state to decide whether to continue with the application or to decline it and perform the associated tasks. To start, I put in some placeholder `Pass` states and did a test deployment to check I had the syntax correct.

```TypeScript
.next(
  new sfn.Choice(this, 'EvaluateIdentityResults')
	.when(
	  sfn.Condition.booleanEquals('$.overallIdentityResult', false),
	  new sfn.Pass(this, 'PerformDeclineTasks')
	)
	.otherwise(new sfn.Pass(this, 'PerformAffordabilityCheck'))
)
```

Now I needed to replace the `PerformDeclineTasks` `Pass` state with the real one. Looking at the original graph, the `PerformDeclineTasks` state needs to be referenced from two different states in the graph.

![double-reference.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1618779701093/IWVKnkfqC.png)

This meant that I couldn't declare the state in line with the rest of the definition, unless I wanted to duplicate the states. To be honest, the definition was getting quite unwieldy anyway, so I started refactoring and creating constants to hold the states. For example, the `Map` state to perform the identity checks became the following.

```TypeScript
const performIdentityChecks = new sfn.Map(this, 'PerformIdentityChecks', {
  inputPath: '$.application',
  itemsPath: '$.applicants',
  resultPath: '$.identityResults',
}).iterator(
  new sfnTasks.LambdaInvoke(this, 'PerformIdentityCheck', {
    lambdaFunction: performIdentityCheckFunction,
    payloadResponseOnly: true,
  })
);
```

In addition to creating constants for the states, we can create constants for the conditions as well. This gives us the added benefit of being able to create meaningful names for the conditions too, e.g.:

```TypeScript
const overallIdentityResultIsFalse = sfn.Condition.booleanEquals(
  '$.overallIdentityResult',
  false
);
```

The overall result was a much more succinct definition:

```TypeScript
const processApplicationStateMachine = new sfn.StateMachine(
  this,
  'ProcessApplicationStateMachine',
  {
    definition: sfn.Chain.start(
      performIdentityChecks
        .next(aggregateIdentityResults)
        .next(
          new sfn.Choice(this, 'EvaluateIdentityResults')
            .when(overallIdentityResultIsFalse, performDeclineTasks)
            .otherwise(new sfn.Pass(this, 'PerformAffordabilityCheck')) // Placeholder
        )
    ),
  }
);
```

One thing that caused a stumble was that the original definition for the `PerformDeclineTasks` state used the `Parameters` property for the `SendDeclineEmail` state to pass in a combination of static and dynamic values.

```YAML
PerformDeclineTasks:
  Type: Parallel
  End: true
  Branches:
  - StartAt: SendDeclineEmail
    States:
      SendDeclineEmail:
        Type: Task
        Resource: "${SendEmailFunctionArn}"
        Parameters:
          emailType: Decline
          application.$: "$.application"
        End: true
```

It wasn't immediately obvious to me how to do this with CDK, as there was no `Parameters` property on `LambdaInvokeProps`. A bit of digging led me to the following AWS article: [Task parameters from the state JSON](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-stepfunctions-tasks-readme.html#task-parameters-from-the-state-json)

This pointed me towards the `payload` property and the `TaskInput` class. Using these I could replicate what was being achieved in the original flow, as shown below:

```TypeScript
const performDeclineTasks = new sfn.Parallel(this, 'PerformDeclineTasks').branch(
  new sfnTasks.LambdaInvoke(this, 'SendDeclineEmail', {
    lambdaFunction: sendEmailFunction,
    payloadResponseOnly: true,
    payload: sfn.TaskInput.fromObject({
      emailType: 'Decline',
      'application.$': '$.application',
    }),
  })
);
```

After refactoring all states and conditions, I ended up with the following state machine definition. The full source code can be found [here](https://github.com/andybalham/blog-source-code/tree/master/step-functions-cdk).

```TypeScript
const processApplicationStateMachine = new sfn.StateMachine(
  this,
  'ProcessApplicationStateMachine',
  {
    definition: sfn.Chain.start(
      performIdentityChecks
        .next(aggregateIdentityResults)
        .next(
          new sfn.Choice(this, 'EvaluateIdentityResults')
            .when(overallIdentityResultIsFalse, performDeclineTasks)
            .otherwise(
              performAffordabilityCheck.next(
                new sfn.Choice(this, 'EvaluateAffordabilityResult')
                  .when(affordabilityResultIsBad, performDeclineTasks)
                  .when(affordabilityResultIsPoor, performReferTasks)
                  .otherwise(performAcceptTasks)
              )
            )
        )
    ),
  }
);
```

Here are my thoughts on the end result:

- The nesting is getting quite deep, and that is with only a couple of decisions.
- I don't find it particularly readable to my eye, despite my refactoring attempts.
- The hierarchical nature doesn't lend itself to easy editing. It is easy to cut and paste the wrong part and get lost with all the brackets.
- There is no local visualisation. I had to deploy to AWS and then use the AWS Toolkit to get the definition.

Overall, I was not as impressed as I would hoped with using this approach for step functions. The fluent syntax promised something, but didn't quite deliver for me. I use the [Prettier](https://prettier.io/) extension for VS Code for formatting, and this syntax didn't seem to play well with it.

However, I have some ideas to help address my concerns. First up, will be a CDK `Construct` to generate the ASL for local visualisation.

### P.S. Evaluate Expression

On this journey, I stumbled across [Evaluate Expression](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-stepfunctions-tasks-readme.html#evaluate-expression) tasks, described by the AWS documentation as follows:

> Use the `EvaluateExpression` to perform simple operations referencing state paths. The `expression` referenced in the task will be evaluated in a Lambda function (`eval()`). This allows you to not have to write Lambda code for simple operations.

Armed with this knowledge I was able replace the `aggregateIdentityResults` Lambda function with the following:

```TypeScript
const aggregateIdentityResults = new sfnTasks.EvaluateExpression(
  this,
  'AggregateIdentityResultsExpression',
  {
    expression: '($.identityResults).every((r) => r.success)',
    resultPath: '$.overallIdentityResult',
  }
);
```

Note the brackets around `$.identityResults` in the expression. Without these, the engine tries to replace a placeholder called `$.identityResults.every` and gets very upset indeed.
