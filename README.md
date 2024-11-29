Powered by [<img src="https://github.com/MuleSoft-AI-Chain-Project/mule-einstein1-connector/blob/master/icon/icon.svg" width="80" height="80"/>](https://github.com/MuleSoft-AI-Chain-Project/mule-einstein1-connector)


# Mule TokenAIzer Custom Policy

This custom policy enables tokenization and obfuscation of sensitive data in API requests and responses. By applying this policy, you can protect sensitive information based on various regulations (such as PCI DSS, GDPR, HIPAA, etc.), ensuring the original data structure is preserved while sensitive values are masked.

## Why?

A tokenizer policy is needed to enhance data security and compliance when handling sensitive information. Here are some key reasons why it is important:

1. Data Protection: Tokenization replaces sensitive data (e.g., credit card numbers, Social Security Numbers) with non-sensitive tokens. These tokens are meaningless outside a secure tokenization system, reducing the risk of exposure during transmission.
2. Compliance with Regulations: Many regulations, such as PCI DSS, HIPAA, and GDPR, require mechanisms to protect sensitive data. A tokenizer policy ensures compliance by obfuscating or protecting data, helping to meet these regulatory standards.
3. Risk Reduction: By replacing sensitive data with tokens, the original data is not exposed if there is a breach. This minimizes the impact and liability associated with data leaks.
4. Consistency in Data Security: Applying a policy ensures uniform protection across all systems and APIs, preventing gaps in the security of sensitive data.
5. Flexibility: A tokenizer policy allows customization for specific data protection needs. For example:
Masking credit card numbers as **"**********1234"**.
Tokenizing names or addresses based on the regulation (e.g., GDPR anonymization or PCI DSS requirements).
6. Improved Development Practices: A tokenizer policy abstracts the complexity of data protection from developers, making it easier to enforce security without requiring them to implement custom solutions.

## How?

The policy uses an integration with the [**Einstein Trust Layer**](https://www.salesforce.com/artificial-intelligence/trusted-ai/) for tokenization of sensitive payloads. The process works as follows:

1. **Context Enrichment**: The policy identifies which regulations apply to the payload (based on policy config) and enriches the context for tokenization.
2. **Tokenization**: The payload is sent to Einstein AI, which generates obfuscated values based on the specified regulations. For this the policy creates a dynamic prompt, optimized for this purpose. NOTE: Please take into account that the models API (used by the MAC Einstein Connector) applies [rate-limiting](https://developer.salesforce.com/docs/einstein/genai/guide/rate-limits.html) that vary depending on the type of org you are using.
3. **TO-DO: Detokenization (optional)**: If detokenization is enabled, the policy will flag the token and store it to allow future detokenization (not performed by this policy).

### Detailed Flow Explanation

1. **Context Enrichment Flow**: This flow is responsible for enriching the tokenization context based on the selected regulations. It generates a list of regulations to consider when processing the payload.
   ```xml
   <flow name="enrich-context-flow">
       <set-variable value='#[%dw 2.0
                            output application/java
                            ---
                            (
                                [
                                    if ({{{pciEnabled}}}) "PCI DSS: Focuses on protecting payment card and transaction data..." else "",
                                    if ({{{hipaaEnabled}}}) "HIPAA: Protects health information..." else "",
                                    if ({{{gdprEnabled}}}) "GDPR: Protects personal data..." else "",
                                    ...
                                ] 
                                filter ((item) -> item != "")
                                joinBy ". \n"
                            )]' variableName="promptContext"/>
   </flow>
   ```
2. **Tokenization Flow**: This flow communicates with Einstein AI to tokenize sensitive information based on the enriched context. The tokenized response is then processed and returned.

```xml
<flow name="ask-flow">
    <mac-einstein:chat-answer-prompt config-ref="Einstein_AI_Config" prompt="#[vars.prompt++'\nThe regulations to consider are '++ vars.promptContext ++ '\nThis is the payload to replace: \n' ++ write(payload, 'application/json')]" modelName="{{{model}}}"/>
    <set-payload value="#[output json --- read(payload.generation.generatedText, 'application/json')]"/>
</flow>
```

3. **TO-DO: Detokenization Flow**: If detokenization is allowed, this flow prepares the payload for the reversal of tokenization.

```xml
<flow name="detokenizer-prepare-flow">
    <logger level="DEBUG" message="#['Preparing detokenization for payload']" category="com.mule.policies.tokenAIzer" />
    <!-- TO-DO -->
</flow>
```

## Usage

Before publishing this policy to Exchange you will need:
1. Add anypoint-exchange as a valid server with credentials to Settings.xml in your ~./m2 folder

After publishing the policy to Exchange (see Publish to Exchange under Development section), follow these steps to apply it to an existing managed API:

1. Log into **Anypoint Platform**.
2. Enter **API Manager**.
3. Click on the API version for the application you want to apply the policy to.
4. Click on **Policies** (located on the left).
5. Click on **Apply New Policy**.
6. Filter by the **Custom** category and select **tokenAIzer**. Click on the **Configure Policy** button.
7. Configure the policy parameters:

| Parameter                | Description                                                                 |
| ------------------------ | --------------------------------------------------------------------------- |
| `requestProtected`        | **Tokenize Request Payload?** Check if you want to protect the request payload by tokenizing sensitive data. |
| `responseProtected`       | **Tokenize Response Payload?** Check if you want to protect the response payload by tokenizing sensitive data. |
| `detokenizationAllowed`   | **Do you want to support detokenization?** Check if you want to support detokenization. Enabling this may impact performance as detokenization will need to reverse the tokenization process. |
| `pciEnabled`              | **PCI DSS** Check if you want to protect request/response payloads following PCI DSS guidance. If enabled, it protects payment card and transaction data. |
| `hipaaEnabled`            | **HIPAA** Check if you want to protect request/response payloads following HIPAA guidelines. It ensures protection of health-related information. |
| `gdprEnabled`             | **GDPR** Check if you want to protect request/response payloads following GDPR regulations for personal data protection. |
| `ccpaEnabled`             | **CCPA** Check if you want to protect request/response payloads following CCPA regulations to protect personal data of California residents. |
| `ferpaEnabled`            | **FERPA** Check if you want to protect request/response payloads following FERPA guidelines to protect educational records in the US. |
| `glbaEnabled`             | **GLBA** Check if you want to protect request/response payloads following GLBA regulations to protect financial data in the US. |
| `model`                   | **Model** Supported models for tokenization using Einstein AI. Select the AI model to use for tokenization and obfuscation tasks. Options include: `Anthropic Claude 3 Haiku on Amazon`, `Azure OpenAI Ada 002`, `Azure OpenAI GPT 3.5 Turbo`, `Azure OpenAI GPT 3.5 Turbo 16k`, `Azure OpenAI GPT 4 Turbo` - `OpenAI Ada 002`, `OpenAI GPT 3.5 Turbo`, `OpenAI GPT 3.5 Turbo 16k`, `OpenAI GPT 4`, `OpenAI GPT 4 32k`, `OpenAI GPT 4o (Omni)`, `OpenAI GPT 4 Turbo`. [Reference](https://developer.salesforce.com/docs/einstein/genai/guide/supported-models.html) |
| `einsteinClientId`        | **Client ID** The client ID to authenticate with the external client app required by Einstein AI. This is a mandatory value for integrating with Einstein. |
| `einsteinClientSecret`    | **Client Secret** The client secret to authenticate with the external client app required by Einstein AI. This is a mandatory value and should be kept secure. |
| `einsteinSfOrg`           | **Einstein Salesforce Org** The Salesforce Organization (Org) where Einstein AI is running. This is required for the policy to interact with Einstein services. |

#### Example Configuration

Here's an example of how you might configure the policy parameters:

```yaml
requestProtected: true
responseProtected: true
detokenizationAllowed: false
pciEnabled: true
hipaaEnabled: false
gdprEnabled: false
ccpaEnabled: false
ferpaEnabled: false
glbaEnabled: false
model: "Azure OpenAI GPT 4 Turbo"
einsteinClientId: "your-client-id"
einsteinClientSecret: "your-client-secret"
einsteinSfOrg: "your-salesforce-org"
```

### Development

The following commands are required during the development phase:

| Task                     | Command                |
| ------------------------ | ---------------------- |
| Package policy           | `mvn clean install`    |
| Publish to Exchange      | `mvn deploy`           |

### Dependencies
- **Mule Runtime 4.x**: Required to run Mule 4 policies.
- **Maven**: Used for building and deploying the policy.
- [**MAC Einstein Connector**](https://mac-project.ai/docs/einstein-ai)
- **Einstein AI**: Required for tokenization and AI-based obfuscation.

---

## Original Developer

[GitHub Repository](https://github.com/panizzag)

## Contribution

Want to contribute? Great!

Just fork the repo, make your updates, and open a pull request!

## To-do
- Implement unit tests 
- Implement detokenization flow
- Add support for additional data formats
- Improve error handling for invalid input
- Improve AI Features: RAG, Chat with Memory
