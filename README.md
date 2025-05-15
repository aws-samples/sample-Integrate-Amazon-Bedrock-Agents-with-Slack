> [!NOTE]
> The content presented here serves as an example intended solely for educational objectives and should not be implemented in a live production environment without proper modifications and rigorous testing.

# Integrate Amazon Bedrock Agents with Slack

As companies increasingly adopt [generative AI](https://aws.amazon.com/ai/generative-ai/) applications, AI agents capable of delivering tangible business value have emerged as a crucial component. In this context, integrating custom-built AI agents within chat services such as [Slack](https://slack.com/) can be transformative, providing businesses with seamless access to AI assistants powered by sophisticated [foundation models (FMs)](https://aws.amazon.com/what-is/foundation-models/). After an AI agent is developed, the next challenge lies in incorporating it in a way that provides straightforward and efficient use. Organizations have several options: integration into existing web applications, [development of custom frontend interfaces](https://github.com/aws-samples/sample-cognito-integrated-bedrock-agents-chat-ui), or integration with communication services such as Slack. The third option—integrating custom AI agents with Slack—offers a simpler and quicker implementation path you can follow to summon the AI agent on-demand within your familiar work environment.

This solution drives team productivity through faster query responses and automated task handling, while minimizing operational overhead. The pay-per-use model optimizes cost as your usage scales, making it particularly attractive for organizations starting their AI journey or expanding their existing capabilities.

There are numerous practical business use cases for AI agents, each offering measurable benefits and significant time savings compared to traditional approaches. Examples include a knowledge base agent that instantly surfaces company documentation, reducing search time from minutes to seconds. A compliance checker agent that facilitates policy adherence in real time, potentially saving hours of manual review. Sales analytics agents provide immediate insights, alleviating the need for time consuming data compilation and analysis. AI agents for IT support help with common technical issues, often resolving problems faster than human agents.

These AI-powered solutions enhance user experience through contextual conversations, providing relevant assistance based on the current conversation and query context. This natural interaction model improves the quality of support and helps drive user adoption across the organization. You can follow this implementation approach to provide the solution to your Slack users in use cases where quick access to AI-powered insights would benefit team workflows. By integrating custom AI agents, organizations can track improvements in key performance indicators (KPIs) such as mean time to resolution (MTTR), first-call resolution rates, and overall productivity gains, demonstrating the practical benefits of AI agents powered by [large language models (LLMs)](https://aws.amazon.com/what-is/large-language-model/).

In this post, we present a solution to incorporate [Amazon Bedrock Agents](https://aws.amazon.com/bedrock/agents/) in your Slack workspace. We guide you through configuring a Slack workspace, deploying integration components in [Amazon Web Services](https://aws.amazon.com/) (AWS), and using this solution.

## Solution Overview

The solution consists of two main components: the Slack to Amazon Bedrock Agents integration infrastructure and either your existing [Amazon Bedrock](https://aws.amazon.com/bedrock/) agent or a sample agent we provide for testing. The integration infrastructure handles the communication between Slack and the Amazon Bedrock agent, and the agent processes and responds to the queries.

The solution uses [Amazon API Gateway](https://aws.amazon.com/api-gateway/), [AWS Lambda](https://aws.amazon.com/lambda/), [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/), and [Amazon Simple Queue Service](https://aws.amazon.com/sqs/) (Amazon SQS) for a serverless integration. This alleviates the need for always-on infrastructure, helping to reduce overall costs because you only pay for actual usage.

Amazon Bedrock agents automate workflows and repetitive tasks while securely connecting to your organization's data sources to provide accurate responses.

An [action group](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-action-create.html) defines actions that the agent can help the user perform. This way, you can integrate business logic with your backend services by having your agent process and manage incoming requests. The agent also maintains context throughout conversations, uses the process of chain of thought, and enables more personalized interactions.

The following diagram represents the solution architecture, which contains two key sections:

- **Section A** – The Amazon Bedrock agent and its components are included in this section. With this part of the solution, you can either connect your existing agent or deploy our sample agent using the provided [AWS CloudFormation](http://aws.amazon.com/cloudformation) template
  
- **Section B** – This section contains the integration infrastructure (API Gateway, Secrets Manager, Lambda, and Amazon SQS) that’s deployed by a CloudFormation template.

<img width="2192" alt="1-Solutions-Overview-Slack-Integration-with-Amazon-Bedrock-Agents" src="https://github.com/user-attachments/assets/723f99bd-a705-4e8f-8685-3f72f343c880" />

The request flow consists of the following steps:

1. A user sends a message in Slack to the bot by using `@appname`.
2. Slack sends a webhook POST request to the API Gateway endpoint.
3. The request is forwarded to the verification Lambda function.
4. The Lambda function retrieves the Slack signing secret and bot token to verify request authenticity.
5. After verification, the message is sent to a second Lambda function.
6. Before putting the message in the SQS queue, the Amazon SQS integration Lambda function sends a “🤔 Processing your request…” message to the user in Slack within a thread under the original message.
7. Messages are sent to the FIFO (First-In-First-Out) queue for processing, using the channel and thread ID to help prevent message duplication.
8. The SQS queue triggers the Amazon Bedrock integration Lambda function.
9. The Lambda function invokes the Amazon Bedrock agent with the user’s query, and the agent processes the request and responds with the answer.
10. The Lambda function updates the initial “🤔 Processing your request…” message in the Slack thread with either the final agent’s response or, if debug mode is enabled, the agent’s reasoning process.

## Prerequisites

You must have the following in place to complete the solution in this post:

- An [AWS account](https://signin.aws.amazon.com/signin?redirect_uri=https%3A%2F%2Fportal.aws.amazon.com%2Fbilling%2Fsignup%2Fresume&client_id=signup)
- A [Slack account](https://slack.com/get-started?entry_point=help_center#/createnew) (two options):
  - For company Slack accounts, work with your administrator to create and publish the integration application, or you can use a sandbox organization
  - Alternatively, create your own Slack account and workspace for testing and experimentation
- [Model access](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html) in Amazon Bedrock for Anthropic's Claude 3.5 Sonnet in the same [AWS Region](https://docs.aws.amazon.com/glossary/latest/reference/glos-chap.html#region) where you'll deploy this solution (if using your own agent, you can skip this requirement)
- The accompanying CloudFormation templates provided in [GitHub repo](https://github.com/aws-samples/sample-Integrate-Amazon-Bedrock-Agents-with-Slack):
  - Sample Amazon Bedrock agent (`virtual-meteorologist`)
  - Slack integration to Amazon Bedrock Agents

## Create a Slack application in your workspace

Creating applications in Slack requires specific permissions that vary by organization. If you don’t have the necessary access, you’ll need to contact your Slack administrator. The screenshots in this walkthrough are from a personal Slack account and are intended to demonstrate the implementation process that can be followed for this solution.

1. Go to [Slack API](https://api.slack.com/apps) and choose **Create New App**
<img width="1482" alt="2-SlackAPI-Create-New-App" src="https://github.com/user-attachments/assets/23515843-7854-4652-9b1a-802e47d3431c" />

2. In the **Create an app pop-up**, choose **From scratch**
<img width="638" alt="3-Create-an-app-from-scratch" src="https://github.com/user-attachments/assets/69268bc1-f5f7-4769-96f3-e154f61ce059" />

3. For **App Name**, enter `virtual-meteorologist`
4. For **Pick a workspace to develop your app in**, choose the workspace where you want to use this application
5. Choose **Create App**
<img width="635" alt="4-Name-app-and-choose-workspace" src="https://github.com/user-attachments/assets/4c402cbc-a9ea-400c-be59-bdcc7ed61d54" />

After the application is created, you'll be taken to the **Basic Information** page.

6. In the navigation pane under **Features**, choose **OAuth & Permissions**
7. Navigate to the **Scopes** section and under **Bot Tokens Scopes**, add the following scopes by choosing **Add an OAuth Scope** and entering `im:read`, `im:write`, and `chat:write`
![5-Scopes-Compressed-2](https://github.com/user-attachments/assets/c8c6234e-1f55-45a7-ae9e-710c484c6a54)

8. On the **OAuth & Permissions** page, navigate to the **OAuth Tokens** section and choose **Install to {Workspace}**
9. Choose **Allow** to complete the process
![6-Install-Compressed](https://github.com/user-attachments/assets/8317e5b7-ba2d-45e7-b041-cbb89885d6f3)

10. On the **OAuth & Permissions** page, navigate to OAuth Tokens and copy the value for **Bot User OAuth Token** that has been created. Save this in a notepad to use later when you’re deploying the CloudFormation template.
<img width="1502" alt="7-Copy-OAuthToken" src="https://github.com/user-attachments/assets/8bd20e86-3be5-4ce7-a1ab-2138853a46bf" />

11. In the navigation pane under **Settings**, choose **Basic Information**
12. Navigate to **Signing Secret** and choose **Show**
13. Copy and save this value to your notepad to use later when you’re deploying the CloudFormation template
<img width="1506" alt="8-SigningSecret" src="https://github.com/user-attachments/assets/c73ee4b4-9318-4082-a25c-69f580403c0c" />

## Deploy the sample Amazon Bedrock agent resources with AWS CloudFormation

If you already have an Amazon Bedrock agent configured, you can copy its [ID](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-view.html) and [alias](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-alias-view.html) from the agent details. If you don’t, then when you run the CloudFormation template for the sample Amazon Bedrock agent (`virtual-meteorologist`), the following resources are deployed (costs will be incurred for the AWS resources used):

- Lambda Functions:
  - **GeoCoordinates** – Converts location names to latitude and longitude coordinates
  - **Weather** – Retrieves weather information using coordinates
  - **DateTime** – Gets current date and time for specific time zones

- AWS Identity and Access Management IAM roles:
  - **GeoCoordinatesRole** – Role for `GeoCoordinates` Lambda function
  - **WeatherRole** – Role for `Weather` Lambda function
  - **DateTimeRole** – Role for `DateTime` Lambda function
  - **BedrockAgentExecutionRole** – Role for Amazon Bedrock agent execution

- Lambda permissions:
  - **GeoCoordinatesLambdaPermission** – Allows Amazon Bedrock to invoke the `GeoCoordinates` Lambda function
  - **WeatherLambdaPermission** – Allows Amazon Bedrock to invoke the `Weather` Lambda function
  - **DateTimeLambdaPermission** – Allows Amazon Bedrock to invoke the `DateTime` Lambda function

- Amazon Bedrock agent:
  - **BedrockAgent** – Virtual meteorologist agent configured with three action groups
  
- Amazon Bedrock agent action groups:
  - `obtain-latitude-longitude-from-place-name`
  - `obtain-weather-information-with-coordinates`
  - `get-current-date-time-from-timezone`

Choose **Launch Stack** to deploy the resources:

[![cloudformation-launch-stack-slack-bedrockagent-integration](https://github.com/user-attachments/assets/2d5c3173-7f45-4a32-9e49-2e436573883e)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=virtual-meteorologist&templateURL=https://aws-blogs-artifacts-public.s3.us-east-1.amazonaws.com/artifacts/ML-18564/cfn-virtual-meteorologist-agent.yaml)

After deployment is complete, navigate to the **Outputs** tab and copy the `BedrockAgentId` and `BedrockAgentAliasID` values. Save these to a notepad to use later when deploying the Slack integration to Amazon Bedrock Agents CloudFormation template.

<img width="845" alt="9-virtual-meteorologist-cfn-output" src="https://github.com/user-attachments/assets/df6003d4-1ae2-43d2-b397-920237231e73" />

## Deploy the Slack integration to Amazon Bedrock Agents resources with AWS CloudFormation

When you run the CloudFormation template to integrate Slack with Amazon Bedrock Agents, the following resources are deployed (costs will be incurred for the AWS resources used):

- API Gateway:
  - **SlackAPI** – A REST API for Slack interactions

- Lambda functions:
  - **MessageVerificationFunction** – Verifies Slack message signatures and tokens
  - **SQSIntegrationFunction** – Handles message queueing to Amazon SQS
  - **BedrockAgentsIntegrationFunction** – Processes messages with the Amazon Bedrock agent

- IAM roles:
  - **MessageVerificationFunctionRole** – Role for `MessageVerificationFunction` Lambda function permissions
  - **SQSIntegrationFunctionRole** – Role for `SQSIntegrationFunction` Lambda function permissions
  - **BedrockAgentsIntegrationFunctionRole** – Role for `BedrockAgentsIntegrationFunction` Lambda function permissions

- SQS queues:
  - **ProcessingQueue** – FIFO queue for ordered message processing
  - **DeadLetterQueue** – FIFO queue for failed message handling

- Secrets Manager secret:
  - **SlackBotTokenSecret** – Stores Slack credentials securely

Choose **Launch Stack** to deploy these resources:

[![cloudformation-launch-stack-slack-bedrockagent-integration](https://github.com/user-attachments/assets/2d5c3173-7f45-4a32-9e49-2e436573883e)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=slack-bedrock-agent-integration&templateURL=https://aws-blogs-artifacts-public.s3.us-east-1.amazonaws.com/artifacts/ML-18564/cfn-amazon-bedrock-agents-slack-integration.json)


Provide your preferred stack name. When deploying the CloudFormation template, you’ll need to provide four values: the Slack bot user OAuth token, the signing secret from your Slack configuration, and the `BedrockAgentId` and `BedrockAgentAliasID` values saved earlier. If your agent is in [draft version](https://docs.aws.amazon.com/bedrock/latest/userguide/deploy-agent.html), use `TSTALIASID` as the `BedrockAgentAliasID`. Although our example uses a draft version, you can use the alias ID of your published version if you’ve already published your agent.

<img width="2055" alt="10-SlackIntegrationCFNDeployment" src="https://github.com/user-attachments/assets/b0d6691b-32c5-46be-87f2-9ee01729fc5c" />

Keep `SendAgentRationaleToSlack` set to `False` by default. However, if you want to troubleshoot or observe how Amazon Bedrock Agents processes your questions, you can set this to `True`. This way, you can receive detailed processing information in the Slack thread where you invoked the Slack application.

When deployment is complete, navigate to the **Outputs** tab and copy the `WebhookURL` value. Save this to your notepad to use in your Slack configuration in the next step.

<img width="906" alt="11-SlackIntegrationCFNOutput" src="https://github.com/user-attachments/assets/230c5e03-c3d7-4b3a-9820-8026ca9606f5" />

## Integrate Amazon Bedrock Agents with your Slack workspace

Complete the following steps to integrate Amazon Bedrock Agents with your Slack workspace:

1. Go to [Slack API](https://api.slack.com/apps) and choose the `virtual-meteorologist` application

<img width="1251" alt="12-Slack-Select-YourApps" src="https://github.com/user-attachments/assets/cca61b60-f0fd-4625-a8db-7394f42568c2" />

2. In the navigation pane, choose **Event Subscriptions**
3. On the **Event Subscriptions** page, turn on **Enable Events**
4. Enter your previously copied API Gateway URL for **Request URL**—verification will happen automatically
5. For **Subscribe to bot events**, select **Add Bot User Event** button and add `app_mention` and `message.im`
6. Choose **Save Changes**
7. Choose **Reinstall your app** and choose **Allow** on the following page
![13-SlackEventSubscriptions-Compressed-2](https://github.com/user-attachments/assets/9b206224-d27e-42b3-acaf-a833fcdb59db)

## Test the Amazon Bedrock Agents bot application in Slack

Return to Slack and locate `virtual-meteorologist` in the **Apps** section. After you add this application to your channel, you can interact with the Amazon Bedrock agent by using `@virtual-meteorologist` to get weather information.
![14-SlackAddApp-Compressed-2](https://github.com/user-attachments/assets/e28a1219-c3b7-4897-8767-ea8bcec3f324)

Let’s test it with some questions. When we ask about today’s weather in Chicago, the application first sends a “🤔 Processing your request…” message as an initial response. After the Amazon Bedrock agent completes its analysis, this temporary message is replaced with the actual weather information.

![15-FirstQuestion-Compressed](https://github.com/user-attachments/assets/96184784-d9d7-4597-bb97-793eff469e79)

You can ask follow-up questions within the same thread, and the Amazon Bedrock agent will maintain the context from your previous conversation. To start a new conversation, use `@virtual-meteorologist` in the main channel instead of the thread.

![16-FollowupQuestion-Compressed](https://github.com/user-attachments/assets/7994ad8d-2d1b-4f25-8dd1-25ec031a0038)

## Clean Up

If you decide to stop using this solution, complete the following steps to remove it and its associated resources deployed using AWS CloudFormation:

1. Delete the Slack integration CloudFormation stack:
  - On the AWS CloudFormation console, choose **Stacks** in the navigation pane
  - Locate the stack you created for the Slack integration for Amazon Bedrock Agents during the deployment process (you assigned a name to it)
  - Select the stack and choose **Delete**

2. If you deployed the sample Amazon Bedrock agent (`virtual-meteorologist`), repeat these steps to delete the agent stack

## Considerations

When designing serverless architectures, separating Lambda functions by purpose offers significant advantages in terms of maintenance and flexibility. This design pattern allows for straightforward behavior modifications and customizations without impacting the overall system logic. Each request involves two Lambda functions: one for token validation and another for SQS payload processing. During high-traffic periods, managing concurrent executions across both functions requires attention to Lambda [concurrency](https://docs.aws.amazon.com/lambda/latest/dg/lambda-concurrency.html) limits. For use cases where scaling is a critical concern, combining these functions into a single Lambda function might be an alternative approach, or you could consider using services such as [Amazon EventBridge](https://aws.amazon.com/eventbridge/) to help manage the event flow between components. Consider your use case and traffic patterns when choosing between these architectural approaches.

## Summary

This post demonstrated how to integrate Amazon Bedrock Agents with Slack, a widely used enterprise collaboration tool. After creating your specialized Amazon Bedrock Agents, this implementation pattern shows how to quickly integrate them into Slack, making them readily accessible to your users. The integration enables AI-powered solutions that enhance user experience through contextual conversations within Slack, improving the quality of support and driving user adoption. You can follow this implementation approach to provide the solution to your Slack users in use cases where quick access to AI-powered insights would benefit team workflows. By integrating custom AI agents, organizations can track improvements in KPIs such as mean time to resolution (MTTR), first-call resolution rates, and overall productivity gains, showcasing the practical benefits of Amazon Bedrock Agents in enterprise collaboration settings.

We provided a sample agent to help you test and deploy the complete solution. Organizations can now quickly implement their Amazon Bedrock agents and integrate them into Slack, allowing teams to access powerful generative AI capabilities through a familiar interface they use daily. Get started today by developing your own agent using Amazon Bedrock Agents.

## Additional Resources

To learn more about building Amazon Bedrock Agents, refer to the following resources:

- [Build a FinOps agent using Amazon Bedrock with multi-agent capability and Amazon Nova as the foundation model](https://aws.amazon.com/blogs/machine-learning/build-a-finops-agent-using-amazon-bedrock-with-multi-agent-capability-and-amazon-nova-as-the-foundation-model/)
- [Building a virtual meteorologist using Amazon Bedrock Agents](https://aws.amazon.com/blogs/machine-learning/building-a-virtual-meteorologist-using-amazon-bedrock-agents/)
- [Build a gen AI–powered financial assistant with Amazon Bedrock multi-agent collaboration](https://aws.amazon.com/blogs/machine-learning/build-a-gen-ai-powered-financial-assistant-with-amazon-bedrock-multi-agent-collaboration/)
