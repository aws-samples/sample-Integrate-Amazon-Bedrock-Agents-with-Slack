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

[Architecture Diagram to be added]

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
   [Image to be added]

2. In the **Create an app pop-up**, choose **From scratch**
   [Image to be added]

3. For **App Name**, enter `virtual-meteorologist`
4. For **Pick a workspace to develop your app in**, choose the workspace where you want to use this application
5. Choose **Create App**
   [Image to be added]

After the application is created, you'll be taken to the **Basic Information** page.

6. In the navigation pane under **Features**, choose **OAuth & Permissions**
7. Navigate to the **Scopes** section and under **Bot Tokens Scopes**, add the following scopes by choosing **Add an OAuth Scope** and entering `im:read`, `im:write`, and `chat:write`
   [Image to be added]

8. On the **OAuth & Permissions** page, navigate to the **OAuth Tokens** section and choose **Install to {Workspace}**
9. Choose **Allow** to complete the process
    [Image to be added]

10. On the **OAuth & Permissions** page, navigate to OAuth Tokens and copy the value for **Bot User OAuth Token** that has been created. Save this in a notepad to use later when you’re deploying the CloudFormation template.
    [Image to be added]

11. In the navigation pane under **Settings**, choose **Basic Information**
    [Image to be added]

12. Navigate to **Signing Secret** and choose **Show**
    [Image to be added]

13. Copy and save this value to your notepad to use later when you’re deploying the CloudFormation template
    [Image to be added]
    
## Deploy the sample Amazon Bedrock agent resources with AWS CloudFormation

If you already have an Amazon Bedrock agent configured, you can copy its ID and alias from the agent details. Otherwise, deploy the sample agent using CloudFormation:

Resources deployed:

### Lambda Functions:
- GeoCoordinates
- Weather
- DateTime

### IAM Roles:
- GeoCoordinatesRole
- WeatherRole
- DateTimeRole
- BedrockAgentExecutionRole

### Lambda Permissions:
- GeoCoordinatesLambdaPermission
- WeatherLambdaPermission
- DateTimeLambdaPermission

### Bedrock Resources:
- BedrockAgent
- Action Groups:
  - obtain-latitude-longitude-from-place-name
  - obtain-weather-information-with-coordinates
  - get-current-date-time-from-timezone

Choose Launch Stack to deploy the resources:
[Launch Stack Button to be added]

After deployment, copy the BedrockAgentId and BedrockAgentAliasID from the Outputs tab.

## Deploy the Slack Integration

Deploy the following resources using CloudFormation:

### API Gateway:
- SlackAPI

### Lambda Functions:
- MessageVerificationFunction
- SQSIntegrationFunction
- BedrockAgentsIntegrationFunction

### IAM Roles:
- MessageVerificationFunctionRole
- SQSIntegrationFunctionRole
- BedrockAgentsIntegrationFunctionRole

### SQS Queues:
- ProcessingQueue
- DeadLetterQueue

### Secrets Manager:
- SlackBotTokenSecret

Choose Launch Stack and provide:
- Stack name
- Slack bot user OAuth token
- Signing secret
- BedrockAgentId
- BedrockAgentAliasID

Keep SendAgentRationaleToSlack set to False by default.

## Complete Slack Integration

1. Return to Slack API and select your application
2. Enable Events and configure:
   - Enter API Gateway URL
   - Subscribe to bot events:
     - app_mention
     - message.im
3. Save and reinstall the app

## Testing the Integration

Test the integration by:
1. Locating virtual-meteorologist in Slack's Apps section
2. Adding it to your channel
3. Using @virtual-meteorologist to get weather information

[Example interaction images to be added]

## Clean Up

To remove the solution:

1. Delete the Slack integration CloudFormation stack:
   - Navigate to AWS CloudFormation console
   - Select your stack
   - Choose Delete
2. If using the sample agent, delete that stack as well

## Considerations

When designing serverless architectures, separating Lambda functions by purpose offers significant advantages in terms of maintenance and flexibility. This design pattern allows for straightforward behavior modifications and customizations without impacting the overall system logic. Each request involves two Lambda functions: one for token validation and another for SQS payload processing. During high-traffic periods, managing concurrent executions across both functions requires attention to Lambda concurrency limits. For use cases where scaling is a critical concern, combining these functions into a single Lambda function might be an alternative approach, or you could consider using services such as Amazon EventBridge to help manage the event flow between components. Consider your use case and traffic patterns when choosing between these architectural approaches.

## Summary

This implementation demonstrates how to integrate Amazon Bedrock Agents with Slack, enabling AI-powered solutions that enhance user experience through contextual conversations. Organizations can track improvements in KPIs such as MTTR, first-call resolution rates, and overall productivity gains, showcasing the practical benefits of Amazon Bedrock Agents in enterprise collaboration settings.

## Additional Resources

To learn more about building Amazon Bedrock Agents, refer to the following resources:

- [Build a FinOps agent using Amazon Bedrock with multi-agent capability and Amazon Nova as the foundation model](#)
- [Building a virtual meteorologist using Amazon Bedrock Agents](#)
- [Build a gen AI–powered financial assistant with Amazon Bedrock multi-agent collaboration](#)
