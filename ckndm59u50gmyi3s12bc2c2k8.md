## Easier Step Functions with the AWS Toolkit for VS Code

If you are developing [AWS Step Functions](https://aws.amazon.com/step-functions/)  and you are using  [VS Code](https://code.visualstudio.com/), then the [AWS Toolkit for Visual Studio Code](https://aws.amazon.com/visualstudiocode/) makes your life so, so much easier.

The AWS guide [Working with AWS Step Functions](https://docs.aws.amazon.com/toolkit-for-vscode/latest/userguide/building-stepfunctions.html) provides a comprehensive guide to installing and using the extension. This is my write-up of the features that I have found to be the most useful so far. These are:

- State Machine Templates
- Code snippets
- Code completion and validation
- State machine graph visualization

Note that the extension also has functionality to download definitions from AWS, create state machines in AWS, and to update the definition for an existing state machine. I have found these less useful, as I prefer to deploy everything using a tool such as [SAM](https://aws.amazon.com/serverless/sam/). However, your mileage may vary.

The key to using the features above, is to save your definition file with the extension `.asl.json` or `.asl.yaml` (more on this later). This is when the magic of the extension kicks in. If you use the 'AWS: Create a new Step Functions state machine' option from the VS Code Command Palette, then this will be done automatically when you save the created file.

The State Machine Templates are accessed via the 'AWS: Create a new Step Functions state machine' option. This presents you with a list of starting points and provides a nice way to create your initial definition.

![Screenshot of list of State Machine Templates](https://cdn.hashnode.com/res/hashnode/image/upload/v1618169060732/23SRpygWu.png "State Machine Templates")

To build up your definition, you will need to add states. This is where the code snippets come in very handy. They provide a guide to creating the different types of states, saving you from having to remember the specifics.

![Screenshot of list of code snippets](https://cdn.hashnode.com/res/hashnode/image/upload/v1618168979553/ENOsotDD-.png "Code Snippets")

For me the best feature of all is the code completion and validation. The code completion adapts to the type of state and prompts you for properties specific to that type. In addition, when entering values for **Next**, **StartAt**, or **Default** properties, you will be prompted for state names. The code validation highlights the following errors:

- Missing properties
- Incorrect values
- No terminal state
- Non-existent states that are pointed to

![Screenshot of code validation errors](https://cdn.hashnode.com/res/hashnode/image/upload/v1618172421977/06Q38vBPr.png "Code Validation Errors")

This feature makes a world of difference in reducing the sort of simple typo errors that can eat up your time deploying to and testing in AWS. Yes, there are ways of testing locally that could help, but seeing and fixing the errors in the editor is always going to be more efficient.

Finally, there is the ability to render a graph of your state machine. I find following the flow of the state machine much easier to follow and validate when done by eye. The graph is rendered when you select the 'Render graph' and appears alongside your definition, as shown below:

![Screenshot of rendered state machine](https://cdn.hashnode.com/res/hashnode/image/upload/v1618168796205/s_sY4MB4T.png "Rendered State Machine")

A very recent addition to the toolkit was announced in March this year and is that [AWS Step Functions now has tooling support for YAML](https://aws.amazon.com/about-aws/whats-new/2021/03/aws-step-functions-adds-tooling-support-for-yaml/). This means that instead of the bracket-heavy definitions of yore, e.g.:

![Definition in JSON with many brackets](https://cdn.hashnode.com/res/hashnode/image/upload/v1618170337831/WIjxsrwIQ.png "Definition in JSON")

We can now express ourselves more cleanly and - hey - maybe add some comments if we are feeling louche. For example, the snippet above becomes:

![Definition in YAML](https://cdn.hashnode.com/res/hashnode/image/upload/v1618170534315/dwAS_VWkW.png "Definition in YAML")

For my investigations into the toolkit, I have created a working project that emulates a basic loan processing flow. This complete code for this can be found on GitHub [here](https://github.com/andybalham/blog-source-code/tree/master/step-functions-aws-toolkit). My next challenge is to take this project and convert it to use the [AWS Cloud Development Kit (CDK)](https://aws.amazon.com/cdk/), and see how that compares with the [SAM](https://aws.amazon.com/serverless/sam/) approach.