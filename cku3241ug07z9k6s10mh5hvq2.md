## Low Cost Step Functions with CDK

Step Functions are great. They let you orchestrate your Lambda functions in a declarative manner, allowing you to avoid combine those functions without directly chaining them together (and thus compounding your costs). However, they are expensive. The first 4000 transitions are free, but the rest are [$0.025 per 1,000 state transitions](https://aws.amazon.com/step-functions/pricing/). You could use [Express Workflows](https://aws.amazon.com/blogs/aws/new-aws-step-functions-express-workflows-high-performance-low-cost/) instead, but they can only run for five minutes of wall-clock time. So how can we have a cheap, long-running way of easily orchestrating Lambda functions? Perhaps CDK can help us build such a thing.

> Note, Step Functions also have error-handling, retries, parallel processing and more very useful functionality that we won't be trying to replicate here. Well, at least not yet 😉

## TL;DR

Using CDK, Lambda functions, SNS, and DynamoDB, it is possible to build a simple analog of Step Functions. See the [GitHub repo](https://github.com/andybalham/blog-source-code/tree/master/low-cost-step-functions) for the full code and working examples.

## The Aim

The aim is to have a single orchestrator Lambda function that uses SNS topics to send asynchronous requests to Lambda functions that perform the various tasks. The orchestrator function then subscribes to a response topic in order to process the output from those tasks. A DynamoDB table is to be used to hold the state of the orchestration between the asynchronous calls. The resulting architecture should look something like the following.

![blog-low-cost-step-functions-aim.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1632761660698/yUoPBN1Wa.jpeg)

## CDK Best Practices

The following is taken from [Best practices for developing and deploying cloud infrastructure with the AWS CDK ](https://docs.aws.amazon.com/cdk/latest/guide/best-practices.html) and will inform how we build the solution. I would recommend anyone interested in CDK to read the whole thing.

> #### Infrastructure and runtime code live in the same package
>
> A construct that is self-contained, in other words that completely describes a piece of functionality including its infrastructure and logic, makes it easy to evolve the two kinds of code together, test them in isolation, share and reuse the code across projects, and version all the code in sync.
>
> #### Model your app through constructs, not stacks
>
> When breaking down your application into logical units, represent each unit as a descendant of Construct and not of Stack. Stacks are a unit of deployment, and so tend to be oriented to specific applications. By using constructs instead of stacks, you give yourself and your users the flexibility to build stacks in the way that makes the most sense for each deployment scenario.

## Thinking In Constructs

With this advice in mind, the components are to be organised as follows.

![blog-low-cost-step-functions-aim-constructs.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1632761671710/ufeafRn_L.jpeg)

An orchestration is to be composed of a single orchestrator construct and one or more task constructs. The orchestrator and task constructs are each made up of a Lambda function and an SNS topic. The topics are to used for the request/response communication between the orchestrator and the tasks. The trick is going to be how we make wiring up these interdependent constructs as straightforward as possible.

Note that the orchestration state is not part of these constructs. This is in line with the following guidance from the [best practices](https://docs.aws.amazon.com/cdk/latest/guide/best-practices.html).

> #### Separate your application into multiple stacks as dictated by deployment requirements
>
> - Consider keeping stateful resources (like databases) in a separate stack from stateless resources. You can then turn on termination protection on the stateful stack, and can freely destroy or create multiple copies of the stateless stack without risk of data loss.

If we have long-running orchestrations, then we may have state that needs to persist between deployments of the orchestration implementation. Perhaps there was a bug-fix that required a patch release. We want to be careful that such state is not deleted in such scenarios. Given this, the decision is to keep the state external.

## The Orchestrator Construct

The Orchestrator construct is an abstract class that provides the base functionality for Orchestrators. The first thing to consider with the Orchestrator construct is the inputs and the outputs. For constructs, the inputs are passed in as a `props` object and the outputs are properties exposed by the construct itself. For the Orchestrator construct, these are as follows.

```TypeScript
export interface OrchestratorProps {
  executionTable: dynamodb.ITable;
  handlerFunction: lambda.Function;
}

export default abstract class Orchestrator extends cdk.Construct {
  readonly responseTopic: sns.ITopic;
  readonly handlerFunction: lambda.Function;
}
```

Inputs:

- `executionTable` is a reference to the DynamoDB table that will be used to store the orchestration state. The construct could create this itself, but as we saw from the best practices, it can be wise to keep stateful resources external.
- `handlerFunction` is a reference to the Lambda function that will do the orchestration. This resource will be instantiated by the concrete subclass, as it will provide functionality specific to the concrete orchestration.

Outputs:

- `responseTopic` is the SNS topic that tasks use in order to publish their responses back to the orchestrator.
- `handlerFunction` is the same function as passed in via the inputs. We expose a reference to it, as it is needed in order to interact with the orchestration.

With the inputs and outputs defined, we move on to the constructor where we create and wire up the resources.

```TypeScript
constructor(scope: cdk.Construct, id: string, props: OrchestratorProps) {
  super(scope, id);

  this.handlerFunction = props.handlerFunction;

  props.executionTable.grantReadWriteData(props.handlerFunction);
  props.handlerFunction.addEnvironment(
    OrchestratorEnvVars.EXECUTION_TABLE_NAME,
    props.executionTable.tableName
  );

  this.responseTopic = new sns.Topic(this, `ResponseTopic`);
  this.responseTopic.addSubscription(
    new snsSubs.LambdaSubscription(props.handlerFunction)
  );
}
```

Here the `handlerFunction` is exposed. It is then granted access to the state table and an environment variable is added to provide it with the name. The response topic is then created and the `handlerFunction` subscribed to it to receive the response messages.

## The Task Construct

As with the orchestration construct, the first thing to define are the inputs and outputs.

```TypeScript
export interface AsyncTaskProps<TReq, TRes> {
  handlerType: new () => AsyncTaskHandler<TReq, TRes>;
  handlerFunction: lambda.Function;
}

constructor(
  orchestrator: Orchestrator,
  id: string,
  props: AsyncTaskProps<TReq, TRes>
) {
  readonly requestTopic: sns.ITopic;
}
```

Inputs:

- `handlerType` is a parameterless constructor function that is used to retrieve the name of the concrete implementation, see `props.handlerType.name`.
- `handlerFunction` is a reference to the Lambda function that will do the orchestration. This function will delegate the handling to a subclass of `AsyncTaskHandler`.

Outputs:

- `requestTopic` is the SNS topic created that the orchestration will use to send requests to the task function.

```TypeScript
constructor(orchestrator: Orchestrator, id: string, props: AsyncTaskProps<TReq, TRes>) {
  super(orchestrator, id);

  this.requestTopic = new sns.Topic(this, 'RequestTopic');
  this.requestTopic.addSubscription(
    new snsSubs.LambdaSubscription(props.handlerFunction)
  );

  this.requestTopic.grantPublish(orchestrator.handlerFunction);
  orchestrator.handlerFunction.addEnvironment(
    `${props.handlerType.name.toUpperCase()}_REQUEST_TOPIC_ARN`
    this.requestTopic.topicArn
  );

  orchestrator.responseTopic.grantPublish(props.handlerFunction);
  props.handlerFunction.addEnvironment(
    AsyncTaskEnvVars.RESPONSE_TOPIC_ARN,
    orchestrator.responseTopic.topicArn
  );
}
```

The constructor first creates the `requestTopic` and subscribes the task `handlerFunction` to it to receive requests.

Next it uses the `orchestrator` parameter to access underlying `handlerFunction`. It grants this function access to publish requests to the task, then it adds an environment variable to the function. The environment variable is named following a convention based on the name of the `handlerType`. The orchestrator function will follow the same convention in order to derive the SNS topic ARN for a particular `handlerType`. Finally, the task `handlerFunction` is granted access to the orchestrator response topic and an environment variable added with the SNS topic ARN.

## The Constructs In Action

### Overview

To demonstrate the constructs in action, we are going to build a simple orchestration that takes three numbers and adds them together. It is going to do this using a sequence of two tasks, each adding two numbers together.

1. Take the inputs x, y, and z and store them
1. Set the running total to 0
1. Call a task to add a and b together
1. Store the result as the running total
1. Call a task to add c and the running total together
1. Store the result as the running total
1. Return the running total as the output

### Add Two Number Task

The first thing to do is define the request and response for the task. This is done by creating two interfaces as follows.

```TypeScript
export interface AddTwoNumbersRequest {
  value1: number;
  value2: number;
}

export interface AddTwoNumbersResponse {
  total: number;
}
```

Next, a subclass of `AsyncTaskHandler` is created to handle the request and return the response. `AsyncTaskHandler` is doing the heavy lifting of handling SNS events and turning them into `AddTwoNumbersRequest` instances, then taking `AddTwoNumbersResponse` and publishing them to the orchestrator response topic.

```TypeScript
export class AddTwoNumbersHandler extends AsyncTaskHandler<
  AddTwoNumbersRequest,
  AddTwoNumbersResponse
> {
  async handleRequestAsync(
    request: AddTwoNumbersRequest
  ): Promise<AddTwoNumbersResponse> {
    return {
      total: request.value1 + request.value2,
    };
  }
}
```

Finally, a handler function is exported. This simply despatches the incoming event to the `handleAsync` method on the `AsyncTaskHandler` base class.

```TypeScript
export const handler = async (event: any): Promise<void> =>
  new AddTwoNumbersHandler().handleAsync(event);
```

The CDK best practices guide mentions the following:

> The AWS CDK not only generates AWS CloudFormation templates for deploying infrastructure, it also bundles runtime assets like Lambda functions and deploys them alongside your infrastructure.

We can take advantage of this by using the [`NodejsFunction`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda_nodejs.NodejsFunction.html) construct and noting the following convention.

> If the `NodejsFunction` is defined in stack.ts with my-handler as id (new NodejsFunction(this, 'my-handler')), the construct will look at stack.my-handler.ts and stack.my-handler.js.)

So if we put following code in `AddTwoNumbers.ts`, then the CDK will look in `AddTwoNumbers.AddTwoNumbersHandler.ts` for a `handler` function.

```TypeScript
export default class AddTwoNumbers extends AsyncTask<
  AddTwoNumbersRequest,
  AddTwoNumbersResponse
> {
  constructor(orchestrator: Orchestrator, id: string) {
    super(orchestrator, id, {
      handlerType: AddTwoNumbersHandler,
      handlerFunction: new lambdaNodejs.NodejsFunction(
        orchestrator,
        AddTwoNumbersHandler.name
      ),
    });
  }
}
```

Here we are again using the constructor for `AddTwoNumbersHandler`. Once to pass in to the base class and again as a convention for the Lambda function id. This means that if we structure the code into the following two files then the CDK will bundle the code using [esbuild](https://esbuild.github.io/), which is one proven way to minimise cold starts.

The resulting files look as follows:

![lambda-bundling.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1632688196592/OgQ20eiN9.png)

### Simple Sequence Orchestration

Now we have a task to call, we come to defining the orchestration itself. First up, we need to define the inputs, the outputs, and the structure of the data the orchestration works upon. These are all defined as interfaces as follows.

```TypeScript
export interface SimpleSequenceInput {
  x: number;
  y: number;
  z: number;
}

export interface SimpleSequenceOutput {
  total: number;
}

export interface SimpleSequenceData {
  x: number;
  y: number;
  z: number;
  total: number;
}
```

Next we need to define how to get the initial data, based on the inputs, and how we get the output based on the data. This is done as follows, providing a `getData` function for the former and a `getOutput` function for the latter.

```TypeScript
const orchestrationProps: OrchestrationBuilderProps<
  SimpleSequenceInput,
  SimpleSequenceOutput,
  SimpleSequenceData
> = {
  getData: (input): SimpleSequenceData => ({
    ...input,
    total: 0,
  }),
  getOutput: (data): SimpleSequenceOutput => ({ total: data.total }),
};
```

The next step, no pun intended, is to define the steps of our orchestration. This is done using the [fluent builder pattern](https://singhsharp.hashnode.dev/c-design-patterns-fluent-builder-pattern) and the `OrchestrationBuilder` class. Each step has a unique id, a reference to the type of handler, and two functions. The `getRequest` function returns a request instance based on the current data. This is the request that is sent to the task handler. The `updateData` function takes the response returned by the task and updates the data. In contrast to Step Functions, this approach has some level of type safety thanks to TypeScript.

```TypeScript
const orchestration = new OrchestrationBuilder<
  SimpleSequenceInput,
  SimpleSequenceOutput,
  SimpleSequenceData
>(orchestrationProps)

  .invokeAsync({
    stepId: 'AddX&Y',
    HandlerType: AddTwoNumbersHandler,
    getRequest: (data) => ({
      value1: data.x,
      value2: data.y,
    }),
    updateData: (data, response) => {
      data.total = response.total;
    },
  })

  .invokeAsync({
    stepId: 'AddZ&Total',
    HandlerType: AddTwoNumbersHandler,
    getRequest: (data) => ({
      value1: data.z,
      value2: data.total,
    }),
    updateData: (data, response) => {
      data.total = response.total;
    },
  })

  .build();
```

Now we have our orchestration defined, we need to subclass `OrchestratorHandler` as follows and export a `handler` function to despatch events to it.

```TypeScript
export class SimpleSequenceHandler extends OrchestratorHandler<
  SimpleSequenceInput,
  SimpleSequenceOutput,
  SimpleSequenceData
> {
  constructor() {
    super(orchestration);
  }
}

export const handler = async (event: any): Promise<any> =>
  new SimpleSequenceHandler().handleAsync(event);
```

`OrchestratorHandler` is doing a lot of heaving lifting here behind the scenes. It handles the events to start the orchestration and it steps through the orchestration, pausing when an asynchronous task is called. When a response event is received, it then resumes stepping through.

The final piece in the puzzle is the orchestrator construct as follows.

```TypeScript
export default class SimpleSequence extends Orchestrator {

  constructor(scope: cdk.Construct, id: string, props: SimpleSequenceProps) {
    super(scope, id, {
      ...props,
      handlerFunction: new lambdaNodejs.NodejsFunction(
        scope,
        SimpleSequenceHandler.name
      ),
    });

    AddTwoNumbers(this, AddTwoNumbersHandler.name);
  }
}
```

Again, we use the `NodejsFunction` construct and the convention to wire it up to the appropriate `handler`. We also wire up the `AddTwoNumbers` to the orchestrator with one line of code. I hope you can see how the `AddTwoNumbers` code could easily be packaged and reused across orchestrations. This might be useful if a task held its own state, perhaps a call to an external service with a circuit breaker.

## Summary

We have seen that we can create a framework for creating serverless orchestrations without Step Functions. By using the CDK, we can take advantage of its compositional abilities and how it can combine the code and the infrastructure. Admittedly, the result lacks several features, such as error-handling, that you would need for production. However, this shows what is possible and the full code can be found in the [GitHub repo](https://github.com/andybalham/blog-source-code/tree/master/low-cost-step-functions), along with a set of working examples and unit tests.
