# Spring AI 2.x: Features, Simple Examples, and What Comes Next

Research date: **28 October 2026**.

This guide brings together the feature overview, explanations, examples, and roadmap discussed in our conversation. The examples use an **online-store support assistant**.

Starting point: [Spring AI project and learning resources](https://spring.io/projects/spring-ai#learn).

## 1. Release status and scope

| Release | Status on the research date | Meaning |
|---|---|---|
| **2.0.2** | Latest stable | Released functionality for production applications |
| **2.1.0-RC1** | Preview | Available for testing before the final release |
| **2.1.0 GA** | Target: **19 November 2026** | Planned date, subject to change |

Sources: [October release announcement](https://spring.io/blog/2026/10/23/spring-ai-2-1-0-RC1-available-now/), [release milestones](https://github.com/spring-projects/spring-ai/milestones).

**ChatClient, advisors, RAG, tools, conversation memory, and MCP existed before 2.0.** The 2.x line improves their architecture, consistency, and capabilities. This is a feature-level guide, grouping related changes rather than reproducing every bug fix, internal refactoring, or dependency update.

Version labels in this guide:

- **2.0:** available in the stable 2.0.x line; a patch release is named when relevant.
- **2.1 preview:** available in the 2.1 preview line, not yet stable on the research date.
- **Planned:** roadmap direction, not a released API commitment.

## 2. How to read the examples

These are focused, documentation-based snippets, not a complete application or an executable sample project. Imports, dependency management, credentials, and some application wiring are omitted. They have not been compiled or run as part of this guide.

Assumptions:

- `builder` is an injected, auto-configured `ChatClient.Builder`.
- `client` is a configured `ChatClient`.
- `vectorStore`, `chatMemory`, `semanticCache`, and `sessionService` are configured beans where used.
- `conversationId` is assigned and scoped by your application to the current user and conversation.
- Business services, external embeddings, and tool objects are supplied by your application.
- Answers and execution traces are illustrative, not guaranteed model responses.

Use separate builder instances or derive a fresh builder when combining examples. They are independent demonstrations, not instructions to progressively mutate one shared builder.

Useful modules, depending on the example:

| Capability | Dependency or prerequisite |
|---|---|
| OpenAI models | `spring-ai-starter-model-openai` |
| Simple vector-store RAG | `spring-ai-vector-store-advisor`, plus a configured vector store |
| Modular RAG | `spring-ai-rag`, plus a configured retriever/vector store |
| Tool search | `spring-ai-starter-tool-search-advisor` |
| Redis semantic cache | `spring-ai-starter-vector-store-redis-semantic-cache` and Redis |
| MCP HTTP server | `spring-ai-starter-mcp-server-webmvc` or the WebFlux counterpart |
| MCP client | `spring-ai-starter-mcp-client` or its WebFlux counterpart |
| JDBC session memory, preview | `spring-ai-starter-model-session-jdbc` and a `DataSource` |

Manage compatible Spring AI versions through its BOM. Stable and preview examples should be evaluated against their corresponding documentation versions.

## 3. ChatClient: ask a question

**Status: existing capability, central to the 2.0 architecture.**

```java
ChatClient client = builder
    .defaultSystem("You are a helpful store support assistant.")
    .build();

String answer = client.prompt()
    .user("Explain what a refund is in one sentence.")
    .call()
    .content();
```

Example answer:

```text
A refund returns money you paid for a purchase.
```

**Use it for:** the main entry point to your AI application. In 2.0, ChatClient is where Spring AI composes advisors, tool execution, and corrective loops. `ChatModel` remains the lower-level building block.

## 4. Streaming: display the answer as it arrives

**Status: existing capability available in 2.0.**

```java
Flux<String> chunks = client.prompt()
    .user("Explain the steps for returning a product.")
    .stream()
    .content();
```

The web application consumes the stream and displays chunks progressively:

```text
"To return"
" a product,"
" first..."
```

**Use it for:** chat interfaces where users should see text before the full answer finishes.

Source for sections 3–4: [ChatClient API](https://docs.spring.io/spring-ai/reference/api/chatclient.html).

## 5. Structured output: turn a message into a Java object

**Status: existing capability with improved reliability controls in 2.0.**

```java
record SupportTicket(String category, String summary) {}

SupportTicket ticket = client.prompt()
    .user("I paid for my order but received no confirmation.")
    .call()
    .entity(SupportTicket.class, spec -> spec
        .useProviderStructuredOutput()
        .validateSchema());
```

Example result:

```java
SupportTicket(
    "PAYMENT",
    "Payment completed without order confirmation"
)
```

What the two switches do:

- `useProviderStructuredOutput()` sends the schema as a provider-level constraint where supported.
- `validateSchema()` validates the output and enables corrective retries on schema failures.

The `StructuredOutputValidationAdvisor` supports the correction loop. Custom `StructuredOutputConverter` implementations can expose `getJsonSchema()` to participate in schema-based behavior.

**Use it for:** extracting information your application will process. Schema validation checks structure, not factual accuracy or business validity.

Sources: [structured output](https://docs.spring.io/spring-ai/reference/api/structured-output.html), [custom output converters](https://docs.spring.io/spring-ai/reference/api/structured-output/converters.html).

## 6. Tool calling: let the model request Java code

**Status: redesigned execution architecture in 2.0.**

```java
class OrderTools {

    @Tool(description = "Look up the delivery status of an order")
    public String deliveryStatus(String orderId) {
        return "Order " + orderId + " arrives Friday.";
    }
}
```

```java
String answer = client.prompt()
    .user("When will order A123 arrive?")
    .tools(new OrderTools())
    .call()
    .content();
```

Conceptual execution:

```text
Model requests: deliveryStatus("A123")
Java returns:   "Order A123 arrives Friday."
Model answers:  "Your order should arrive Friday."
```

What changes in 2.0:

- `ToolCallingAdvisor` manages execution consistently across providers.
- ChatClient automatically registers the tool-calling advisor.
- Provider-specific internal tool loops are removed.
- Extensions can observe and customize individual iterations.
- Direct `ChatModel` use requires external tool-loop management.

**Use it for:** live information and actions, such as looking up an order through an application service.

## 7. Advisors: combine behavior around the model call

**Status: existing abstraction with recursive composition improvements in 2.0.**

```java
ChatClient supportClient = builder
    .defaultAdvisors(
        MessageChatMemoryAdvisor.builder(chatMemory).build(),
        QuestionAnswerAdvisor.builder(vectorStore).build()
    )
    .build();

String answer = supportClient.prompt()
    .user("Does that return policy also cover opened products?")
    .advisors(a -> a.param(
        ChatMemory.CONVERSATION_ID, conversationId))
    .call()
    .content();
```

Conceptually:

```text
Read conversation → Retrieve policy → Ask model → Save conversation
```

Recursive advisors can re-enter the downstream chain. This supports tool loops, structured-output correction, and other iterative patterns.

Tool advisor hooks include initialization, before/after each iteration, and finalization, with synchronous and streaming variants. A custom implementation can emit progress events or apply application-specific budgets.

**Use it for:** composing reusable behavior around model calls.

Sources for sections 6–7: [tool calling](https://docs.spring.io/spring-ai/reference/api/tools.html), [recursive advisors](https://docs.spring.io/spring-ai/reference/api/advisors-recursive.html).

## 8. Tool search: select from a large collection

**Status: 2.0.**

Add `spring-ai-starter-tool-search-advisor`, then enable:

```properties
spring.ai.chat.client.tool-search-advisor.enabled=true
```

```java
String answer = client.prompt()
    .user("Where is order A123?")
    .tools(orderTools, billingTools, inventoryTools, shippingTools)
    .call()
    .content();
```

Conceptually:

```text
Available: hundreds of tools
Question:  "Where is order A123?"
Discover:  deliveryStatus, shipmentTracking
```

`ToolSearchToolCallingAdvisor` progressively discovers tools and replaces the default tool-calling advisor when configured.

**Use it for:** reducing tool definitions sent to the model per request.

## 9. Tool-call limits: stop excessive execution

**Status: added in 2.0.1.**

Behavior example, not a configuration snippet:

```text
Application configures a tool-call budget.

Model requests repeated searches:
    Search 1 → allowed
    Search 2 → allowed
    ...
    Budget reached → execution stops with a limit error
```

**Use it for:** preventing unbounded tool execution. A tool-call limit is different from an output-token limit.

Sources for sections 8–9: [tool calling](https://docs.spring.io/spring-ai/reference/api/tools.html), [2.0.1 release notes](https://github.com/spring-projects/spring-ai/releases/tag/v2.0.1).

## 10. Document ingestion: put a PDF into the knowledge base

**Status: existing ETL capability available in 2.0.**

```java
var reader = new PagePdfDocumentReader(
    new ClassPathResource("return-policy.pdf"));

var splitter = TokenTextSplitter.builder()
    .withChunkSize(800)
    .build();

List<Document> chunks = splitter.apply(reader.get());

vectorStore.add(chunks);
```

```text
PDF → Text documents → Smaller chunks → Embeddings → Vector database
```

The ETL building blocks are a document reader, a transformer, and a writer. A vector store acts as the writer in this example.

**Use it for:** preparing data before users ask questions.

## 11. Custom text splitting: recognize document punctuation

**Status: improvements in the 2.0 development line.**

```java
var splitter = TokenTextSplitter.builder()
    .withChunkSize(800)
    .withPunctuationMarks(List.of('.', '?', '!', ';', '。'))
    .build();
```

The splitter can consider punctuation such as `。` when finding boundaries. Tokenizer encoding controls are also available.

**Use it for:** multilingual documents and specialized text formats.

Source for sections 10–11: [ETL and text splitting](https://docs.spring.io/spring-ai/reference/api/etl-pipeline.html).

## 12. Simple RAG: answer from stored policy documents

**Status: existing capability available in 2.0.**

After ingesting the PDF:

```java
String answer = client.prompt()
    .user("How long do I have to return an item?")
    .advisors(
        QuestionAnswerAdvisor.builder(vectorStore).build()
    )
    .call()
    .content();
```

Illustrative flow:

```text
Find chunk: "Returns are accepted within 30 days."
Add chunk to the prompt.
Answer: "You have 30 days to return the item."
```

**Use it for:** straightforward question answering over documents. Add the `spring-ai-vector-store-advisor` dependency.

## 13. Modular RAG: improve the question before retrieval

**Status: existing modular architecture carried into 2.0.**

```java
var rag = RetrievalAugmentationAdvisor.builder()
    .queryTransformers(
        RewriteQueryTransformer.builder()
            .chatClientBuilder(builder.build().mutate())
            .build()
    )
    .documentRetriever(
        VectorStoreDocumentRetriever.builder()
            .vectorStore(vectorStore)
            .similarityThreshold(0.70)
            .build()
    )
    .build();

String answer = client.prompt()
    .user("Need info on returns for opened stuff")
    .advisors(rag)
    .call()
    .content();
```

Illustrative transformation:

```text
Original:  "Need info on returns for opened stuff"
Rewritten: "What is the return policy for opened products?"
Retrieve:  Relevant policy sections
Generate:  An answer using those sections
```

The modular architecture lets you customize query transformation, retrieval, document processing, and context augmentation. Add `spring-ai-rag`.

By default, the modular advisor instructs the model not to answer when retrieval returns no context. This is a prompt behavior, not a guarantee of factual correctness.

| Approach | Main component | Good starting point for |
|---|---|---|
| Simple RAG | `QuestionAnswerAdvisor` | Basic document lookup |
| Modular RAG | `RetrievalAugmentationAdvisor` | Customizable retrieval pipelines |

Source for sections 12–13: [RAG documentation](https://docs.spring.io/spring-ai/reference/api/retrieval-augmented-generation.html).

## 14. Metadata filters: search the right documents

**Status: existing vector-store capability with 2.x enhancements.**

```java
vectorStore.add(List.of(
    new Document(
        "Returns are accepted within 30 days.",
        Map.of("country", "DK", "type", "returns")
    )
));
```

Search Danish return documents:

```java
List<Document> matches = vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("Can I return this product?")
        .filterExpression("country == 'DK' && type == 'returns'")
        .topK(3)
        .build()
);
```

**Use it for:** selecting a country, category, or document type. Supported operations depend on the store. Access-control filters must be derived and enforced by application logic.

Source: [vector-store API](https://docs.spring.io/spring-ai/reference/2.1/api/vectordbs.html).

## 15. Conversation memory: understand follow-up questions

**Status: available in 2.0; legacy memory APIs are deprecated in the 2.1 preview.**

```java
ChatClient memoryClient = builder
    .defaultAdvisors(
        MessageChatMemoryAdvisor.builder(chatMemory).build()
    )
    .build();
```

Use the same server-assigned conversation ID for two turns:

```java
memoryClient.prompt()
    .user("My order number is A123.")
    .advisors(a -> a.param(
        ChatMemory.CONVERSATION_ID, conversationId))
    .call()
    .content();

String answer = memoryClient.prompt()
    .user("What order number did I give you?")
    .advisors(a -> a.param(
        ChatMemory.CONVERSATION_ID, conversationId))
    .call()
    .content();
```

Example answer: `A123`.

2.0 improvements include Redis persistence and turn-boundary eviction. Redis-backed memory survives application restarts when configured; in-memory storage does not. Legacy repositories differ in their support for tool-call messages, which is one reason session memory matters.

Conversation memory is the context supplied to the model. It is not necessarily the complete durable chat history.

**Use it for:** context across turns, with a distinct ID per user and conversation.

Source: [conversation memory](https://docs.spring.io/spring-ai/reference/api/chat-memory.html).

## 16. Semantic cache: reuse answers to similar questions

**Status: Redis semantic-cache addition in the 2.0 development line.**

```java
var cacheAdvisor = SemanticCacheAdvisor.builder()
    .cache(semanticCache)
    .build();

client.prompt()
    .user("What is the return policy?")
    .advisors(cacheAdvisor)
    .call()
    .content();

client.prompt()
    .user("Explain your policy for returning products.")
    .advisors(cacheAdvisor)
    .call()
    .content();
```

If similarity exceeds the configured threshold:

```text
First request  → Model call → Cache answer
Second request → Cache hit → Reuse answer
```

The Redis integration supports similarity thresholds, expiration, and system-prompt isolation. User-specific data needs appropriate additional isolation.

| Capability | Job | Example |
|---|---|---|
| Conversation memory | Supply earlier conversation context | Remember the order number |
| Semantic cache | Reuse a previous response | Reuse an equivalent return-policy answer |

**Use it for:** repeated, equivalent questions where reuse remains valid.

Source: [Redis semantic caching](https://docs.spring.io/spring-ai/reference/api/vectordbs/redis.html).

## 17. MCP tools: expose services to other AI applications

**Status: MCP existed earlier; annotations and transports are consolidated in 2.0.**

In the MCP server:

```java
@Component
class OrderMcpTools {

    @McpTool(description = "Look up an order's delivery status")
    public String deliveryStatus(
            @McpToolParam(required = true) String orderId) {
        return "Order " + orderId + " arrives Friday.";
    }
}
```

In an AI application with its MCP client connection configured:

```java
String answer = client.prompt()
    .user("When will order A123 arrive?")
    .toolCallbacks(mcpToolCallbackProvider.getToolCallbacks())
    .call()
    .content();
```

```text
ChatClient → MCP client → MCP server → deliveryStatus()
```

`@Tool` registers a method for Spring AI tool calling. `@McpTool` exposes a method through an MCP server. The client integration adapts discovered MCP tools into callbacks.

**Use it for:** sharing capabilities across applications using a standard protocol.

Source: [MCP client starter](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-client-boot-starter-docs.html).

## 18. MCP resources, prompts, and progress

**Status: annotation programming model consolidated in 2.0.**

A **resource** supplies readable content:

```java
@McpResource(uri = "policy://returns", name = "Return policy")
public String returnPolicy() {
    return "Returns are accepted within 30 days.";
}
```

A **prompt** supplies a reusable interaction template:

```java
@McpPrompt(name = "explain-return")
public GetPromptResult explainReturn() {
    return GetPromptResult.builder(List.of(
        new PromptMessage(
            Role.USER,
            TextContent.builder(
                "Explain the return policy in three simple steps."
            ).build()
        )
    )).build();
}
```

A tool can report **progress**:

```java
@McpTool(description = "Prepare a return report")
public String prepareReport(McpSyncRequestContext context) {
    context.info("Starting report");
    context.progress(p -> p
        .progress(1)
        .total(2)
        .message("Orders loaded"));

    return "Report ready";
}
```

These methods belong in scanned Spring components. Resource and prompt requests must be handled by the client; they are not automatically inserted into every ChatClient call.

Additional protocol interactions, illustrated conceptually:

| Capability | Simple interaction |
|---|---|
| Sampling | Server asks the client to perform an LLM operation |
| Elicitation | Server requests missing information through the client |
| Capability-change notification | Server announces a changed tool or resource list |

Unified synchronous/asynchronous request contexts and client callback annotations support these interactions, subject to negotiated client/server capabilities.

**Use it for:** content access, reusable prompts, interactive workflows, and feedback during long operations.

Source: [MCP annotations](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-annotations-server.html).

## 19. MCP transports: choose how applications connect

**Status: Streamable HTTP becomes the default in 2.0; SSE is deprecated.**

With the WebMVC server starter:

```properties
spring.ai.mcp.server.protocol=STREAMABLE
```

For its stateless variant:

```properties
spring.ai.mcp.server.protocol=STATELESS
```

For local process communication with the standard server starter:

```properties
spring.ai.mcp.server.stdio=true
```

| Situation | Example |
|---|---|
| Remote HTTP integration | Support application connects to an order MCP server |
| Stateless HTTP deployment | Server instances handle requests without MCP session state |
| Local process integration | Desktop client launches a Java MCP server and communicates through STDIO |

Stateless HTTP trades away session-dependent bidirectional capabilities. It does not make your underlying order database or business services stateless.

Spring AI 2.0 ships with MCP Java SDK 2.0.0 and support for the 2025-11-25 protocol specification. Spring-specific WebMVC/WebFlux transports move into Spring AI.

Authentication is separate: transport starters do not automatically secure the endpoint. OAuth/API-key support through `mcp-security` is a community extension.

Sources: [MCP server configuration](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-server-boot-starter-docs.html), [2.0 announcement](https://spring.io/blog/2026/06/12/spring-ai-2-0-0-GA-available-now/).

## 20. Observability: inspect latency and usage

**Status: existing capability with improvements in 2.0.x.**

```java
ChatResponse response = client.prompt()
    .user("Explain the return process.")
    .call()
    .chatResponse();

System.out.println(response.getMetadata().getUsage());
```

With Actuator and a configured metrics/tracing backend, an illustrative trace might show:

```text
Support request
    → Vector search: 40 ms
    → Model request: 900 ms
    → Tool execution: 80 ms
```

Use the auto-configured builder to retain observability wiring. 2.0.2 adds observations for additional speech, transcription, and image models. Prompt and completion content logging is off by default.

**Use it for:** identifying slow retrieval, expensive requests, or failing tools.

Source: [observability](https://docs.spring.io/spring-ai/reference/observability/index.html).

## 21. Immutable options and platform foundations

**Status: 2.0 foundations.**

```java
var options = OpenAiChatOptions.builder()
    .model("gpt-5-mini")
    .build();

String answer = client.prompt()
    .user("Explain refunds briefly.")
    .options(options)
    .call()
    .content();
```

Options are immutable after construction. Use builders to create another configuration. Option merging is handled consistently at the ChatClient layer; direct ChatModel configuration has different requirements.

| Foundation | What changed | Simple development example |
|---|---|---|
| Spring platform | Boot 4.0/4.1 and Framework 7.0 baseline for 2.0 | Start the application on Boot 4 |
| JSpecify | Nullability annotations across the API | Kotlin and analysis tools distinguish nullable results from required values |
| Jackson 3 | Framework JSON configuration moves toward Jackson 3 | Customize a Jackson 3 `JsonMapper` through the documented extension points |
| Option builders | Immutable options and consistent merging | Override one request's model options through the fluent API |
| Configuration properties | Simplified structure, including removal of the artificial `.options` segment | Consult upgrade notes when translating old property keys |
| Provider consolidation | OpenAI and Anthropic integrations use their official SDKs | Use the consolidated integration instead of choosing between overlapping implementations |

Core chat providers highlighted in the GA announcement are OpenAI, Anthropic, Amazon Bedrock, Google GenAI, Mistral AI, DeepSeek, and Ollama. Azure OpenAI uses the consolidated OpenAI integration. OCI and Azure Cosmos DB support move toward vendor-maintained integrations.

Sources: [OpenAI configuration](https://docs.spring.io/spring-ai/reference/api/chat/openai-chat.html), [2.0 foundations](https://spring.io/blog/2026/06/12/spring-ai-2-0-0-GA-available-now/).

## 22. Additional stable 2.0.x changes with simple examples

These are behavior examples, not complete configuration snippets. Some changes are provider-specific.

| Improvement | Simple example |
|---|---|
| Amazon S3 vector integration | Use an S3-backed vector index for document retrieval |
| Bedrock Knowledge Base integration | Retrieve context from an existing Bedrock knowledge base |
| PDF page ranges, 2.0.1 | Ingest policy pages while skipping unrelated sections of a manual |
| Redis text search | Find documents containing the exact product code `ABC-123` |
| Redis range search | Retrieve matches within the configured similarity/distance range |
| Redis HNSW tuning | Increase search effort when recall matters more than latency |
| Redis metadata-field configuration, 2.0.2 | Register a metadata field for indexed filtering |
| Custom Azure vector-store field names | Map an existing `body` field rather than requiring `content` |
| Null metadata filters | Find documents whose `category` metadata is missing |
| PGvector dimension validation | Reject a 768-dimensional vector when the table expects 1,536 dimensions |
| Configurable Neo4j filter converter | Adapt metadata filters for a custom integration |
| Filter-based SimpleVectorStore deletion | Remove vectors matching a document category |
| GemFire authentication | Connect using configured username/password credentials |
| OpenSearch document-ID handling | Allow server-generated IDs for a compatible serverless setup |
| Milvus Docker Compose integration | Connect a local application to its Compose-managed Milvus service |
| Speech streaming, 2.0.1 | Start playing a spoken answer while more audio arrives |
| Richer OpenAI transcription, 2.0.1 | Transcribe a support recording with additional options and metadata |
| Google image generation, 2.0.1 | Generate an illustration for a help article |
| Google tool choice, 2.0.1 | Require or restrict tool selection where supported |
| Typed media builders, 2.0.1 | Supply media through an overload matching its data type |
| Native structured-output enhancements | Request schema-constrained results with a supporting provider |
| Reasoning/thinking options | Set supported reasoning behavior for a complex support question |
| Thought/signature handling | Preserve provider-required signatures across tool interactions |
| Anthropic Skills/Files support | Use supported Claude workflows with uploaded files and skills |
| Unified cache usage metrics | Inspect provider-reported cached-token usage through a common abstraction |
| MCP stateless customizers, 2.0.2 | Customize the stateless server builder through a Spring bean |

Milestone notes describe features at that point in development. Some were later removed or consolidated: for example, Infinispan does not remain in the final core distribution, and the temporary `ToolSpec` consumer API was removed before GA.

Sources: [M1](https://github.com/spring-projects/spring-ai/releases/tag/v2.0.0-M1), [M2](https://spring.io/blog/2026/01/23/spring-ai-2-0-0-M2-available-now/), [M3](https://github.com/spring-projects/spring-ai/releases/tag/v2.0.0-M3), [M5](https://github.com/spring-projects/spring-ai/releases/tag/v2.0.0-M5), [M6](https://github.com/spring-projects/spring-ai/releases/tag/v2.0.0-M6), [M7](https://github.com/spring-projects/spring-ai/releases/tag/v2.0.0-M7), [2.0.1](https://github.com/spring-projects/spring-ai/releases/tag/v2.0.1), [2.0.2](https://github.com/spring-projects/spring-ai/releases/tag/v2.0.2).

## 23. OpenAI Responses API

**Status: 2.1 preview.**

```properties
spring.ai.openai.chat.api=responses
spring.ai.openai.responses.model=gpt-5-mini
spring.ai.openai.responses.reasoning-effort=medium
```

```java
String answer = client.prompt()
    .user("Compare these return-policy exceptions step by step.")
    .call()
    .content();
```

This selects `OpenAiResponsesChatModel` for the auto-configured client. Responses-specific options include supported hosted tools and reasoning summaries.

**Use it for:** Responses API capabilities through Spring AI. This integration sends context with requests; it does not automatically create a provider-stored conversation.

Source: [Responses API integration](https://docs.spring.io/spring-ai/reference/2.1/api/chat/openai-responses-chat.html).

## 24. Message parts: preserve different kinds of content

**Status: 2.1 preview.**

```java
ChatResponse response = client.prompt()
    .user("Explain how you can help with this order.")
    .call()
    .chatResponse();

AssistantMessage message = response.getResult().getOutput();

message.getParts().forEach(part ->
    System.out.println(part.getClass().getSimpleName())
);
```

Different requests and providers can produce parts such as:

```text
TextPart
ToolCallPart
ReasoningPart
MediaPart
```

Other part types include `ToolResultPart` and `UnknownPart`. Familiar accessors such as `getText()` and `getToolCalls()` remain views over this content.

**Use it for:** retaining ordered content and provider payloads across conversations. A reasoning part may contain opaque replay data rather than readable reasoning text. A plain question may return only a text part.

Source: [message parts](https://docs.spring.io/spring-ai/reference/2.1/api/chatmodel.html).

## 25. Session memory: retain an event log and compact its view

**Status: 2.1 preview; moves the community session foundation into Spring AI.**

Add `spring-ai-starter-model-session-jdbc` and configure a data source for the JDBC example.

```java
ChatClient sessionClient = builder
    .defaultAdvisors(
        SessionAdvisors.builder(sessionService)
            .compaction(
                new TurnCountTrigger(20),
                WindowCompactionStrategy.builder()
                    .lastTurns(10)
                    .build()
            )
            .build()
    )
    .build();
```

```java
String answer = sessionClient.prompt()
    .user("What did we decide about my return?")
    .advisors(SessionAdvisors.session(conversationId))
    .call()
    .content();
```

Conceptually:

```text
Stored log:       Conversation events
Model's view:     Recent complete turns after compaction
Original events: Remain in the log
```

Compaction keeps turn boundaries intact. Summarization can also be configured; it is not enabled by this window-only example. JDBC support includes PostgreSQL, MySQL, MariaDB, and H2. An in-memory repository is suitable for demos and tests.

**Use it for:** longer conversations and agents whose event history should outlive a particular model invocation or application instance.

Source: [session memory](https://docs.spring.io/spring-ai/reference/2.1/api/session.html).

## 26. Tool confirmation: obtain a decision before an action

**Status: 2.1 preview.**

```java
@Tool(
    description = "Cancel an order",
    requiresConfirmation = true
)
public String cancelOrder(String orderId) {
    return orderService.cancel(orderId);
}
```

Application flow:

```text
User:   "Cancel A123."
Model:  Requests cancelOrder("A123").
Spring: Returns pending calls without executing them.
UI:     Displays "Confirm cancellation?"
User:   Approves.
App:    Executes the allowed call and resumes the conversation.
```

The application collects `ToolCallDecision` values and uses the tool-calling manager to execute allowed calls before continuing. If a reply contains a call requiring confirmation, none of that reply's calls run before the decision stage.

Confirmation policies can cover tools you do not own, including MCP tools, or inspect arguments such as transaction amounts.

**Use it for:** actions that require an explicit decision. The annotation supplies metadata; your application supplies the UI, authorization, decisions, and resumption.

Source: [tool confirmation](https://docs.spring.io/spring-ai/reference/2.1/api/tools/tool-call-confirmation.html).

## 27. Precomputed embeddings: import existing vectors

**Status: 2.1 preview.**

```java
Document document = new Document(
    "8f14e45f-ceea-467a-9a3e-5b1c2d6f7a90",
    "Returns are accepted within 30 days.",
    Map.of("type", "returns")
);

// Produced by an external embedding pipeline.
float[] vector = externalEmbedding;

vectorStore.upsert(List.of(
    new EmbeddedDocument(document, vector)
));
```

`upsert` writes the provided vector without generating another embedding and replaces an existing document with the same ID. Store-specific ID rules still apply.

**Use it for:** migrating an existing vector dataset or importing batch-generated embeddings.

## 28. Search with an existing query embedding

**Status: 2.1 preview.**

```java
List<Document> matches = vectorStore.similaritySearch(
    externalQueryEmbedding,
    SearchRequest.builder()
        .topK(3)
        .similarityThreshold(0.70)
        .build()
);
```

**Use it for:** reusing a query vector generated elsewhere. It must use the same embedding space as the stored vectors.

These operations require a supporting store. The preview documentation lists pgvector, Redis, Elasticsearch, Qdrant, and OpenSearch support, with implementation-specific constraints. Invalid dimensions and non-finite values are rejected; unsupported operations fail explicitly.

Source for sections 27–28: [precomputed vectors and vector search](https://docs.spring.io/spring-ai/reference/2.1/api/vectordbs.html).

## 29. Overlapping chunks: preserve boundary context

**Status: 2.1 preview.**

```java
var splitter = TokenTextSplitter.builder()
    .withChunkSize(800)
    .withChunkOverlap(100)
    .build();
```

Conceptually:

```text
Chunk 1: [--------------------]
Chunk 2:                [--------------------]
                        shared context
```

**Use it for:** reducing context loss where chunks meet. Overlap counts toward chunk size and increases storage and embedding work. Default overlap is zero.

Source: [chunk overlap](https://docs.spring.io/spring-ai/reference/2.1/api/etl-pipeline.html).

## 30. Other preview additions and community extensions

| Addition | Simple example |
|---|---|
| Qxotic jinfer integration | Run a supported inference model inside the JVM |
| Google speech/transcription | Read a support answer aloud or transcribe a recording |
| PGvector column mapping | Reuse a table with custom content, metadata, and vector columns |
| Provider refinements | Preserve thinking text, customize credentials, or receive rate-limit metadata |
| Bedrock tool-result caching | Reuse eligible tool-result context through the provider integration |
| Anthropic citation-document caching | Include citation documents in supported prompt caching |

Community projects remain distinct from core Spring AI:

| Community project or area | Simple example |
|---|---|
| `spring-ai-agent-utils` | Give an agent reusable file, shell, web-fetch, or task tools |
| Agent Skills support | Load a reusable task skill when the agent needs it |
| MCP security | Add an OAuth or API-key security integration |
| A2A integration | Connect collaborating agents through an agent-to-agent protocol |

Availability and versions of community extensions must be checked independently of the core release.

Sources: [2.0 overview and community extensions](https://spring.io/blog/2026/06/12/spring-ai-2-0-0-GA-available-now/), [October preview announcement](https://spring.io/blog/2026/10/23/spring-ai-2-1-0-RC1-available-now/).

## 31. What comes next

1. **2.1 GA:** the announced immediate priority is feedback and stabilization, particularly session memory and message parts.
2. **2.0 maintenance:** the stable line continues with fixes and compatible improvements; the milestone list includes 2.0.3 work.
3. **Memory migration:** legacy chat-memory APIs are deprecated in the 2.1 preview, with removal planned for 3.0.
4. **More session storage coverage:** corresponding replacements for additional repositories/dialects are planned before that removal.

Example migration direction:

```text
Today: 2.0 application using Redis ChatMemory
    → Evaluate 2.1 session APIs
    → Adopt a suitable session repository when available
    → Complete migration before legacy API removal
```

The preview announcement explicitly keeps `VectorStoreChatMemoryAdvisor`, because sessions do not yet provide a corresponding semantic-recall replacement. Do not assume every legacy memory type has an available session replacement today.

Dates and roadmap items can change. The November 19 target is a milestone date, not a guarantee.

Sources: [announced next steps](https://spring.io/blog/2026/10/23/spring-ai-2-1-0-RC1-available-now/), [live milestones](https://github.com/spring-projects/spring-ai/milestones).

## 32. Upgrade checklist for an existing 1.x application

Consult the complete upgrade notes before changing dependencies. Notable changes include:

- Move to a compatible Spring Boot 4 baseline for 2.0.
- Update mutable option/setter code to builders.
- Review configuration-property changes and module renames.
- Replace removed `toolNames()`/bean-name resolution APIs with explicit tools or callbacks.
- Remove reliance on model-internal tool loops and `internalToolExecutionEnabled`.
- Review MCP annotation package and transport dependency changes.
- Migrate separate Azure OpenAI and OpenAI SDK modules to the consolidated integration where applicable.
- Supply explicit conversation IDs for memory operations.
- Verify provider migrations/removals, including Vertex chat, MiniMax-specific support, and externally maintained integrations.
- For the preview, plan memory migration without assuming all repository counterparts already exist.

Example tool migration direction:

```text
Old design: model implementation executes tools internally
New design: ChatClient → ToolCallingAdvisor → tool execution → model
```

Sources: [upgrade notes](https://docs.spring.io/spring-ai/reference/upgrade-notes.html), [2.0 RC1 architectural changes](https://spring.io/blog/2026/06/06/spring-ai-2-0-0-RC1-available-now/).

## 33. Put the pieces together

| Customer request | Components |
|---|---|
| "Explain your return policy" | ChatClient + modular RAG |
| "Where is order A123?" | Tool calling, optionally through MCP |
| "What about the order I mentioned earlier?" | Conversation/session memory |
| "Create a support ticket" | Structured output + application persistence |
| "Cancel my order" | Tool calling + confirmation in the 2.1 preview |
| Another equivalent policy question | Semantic cache, where reuse is appropriate |
| Investigate slow or expensive requests | Observability |

```text
Customer
    → ChatClient
        → Conversation/session context
        → RAG for policy documents
        → Tools/MCP for live order information
        → Model response
    → Answer or structured result

Selected actions can pause for confirmation in 2.1.
```

Suggested learning order:

1. ChatClient and streaming.
2. Structured output.
3. Local tools and the tool loop.
4. Document ingestion and simple RAG.
5. Modular RAG and metadata filters.
6. Memory and semantic caching.
7. MCP servers and clients.
8. Observability and advanced advisor composition.
9. Preview session memory, confirmation, Responses, and precomputed vectors.

## 34. TypeSafe Jev: structured judgment with Java examples

**Status: community integration, versioned independently of Spring AI core.**

The [September article](https://spring.io/blog/2026/09/21/spring-ai-typesafe-structured-judgment/) introduces Jev for typed decisions: classify a ticket, score an answer, or decide whether a request needs review. The article demonstrates `0.1.0`; the project's current documentation uses **0.4.0**. Examples below target the documented 0.4.0 API rather than reproducing historical defaults.

### 34.1 Three question types

| Primitive | Meaning | Store-support example |
|---|---|---|
| `Noul` | Truth value from 0 to 1 | Does this ticket express urgency? |
| `Choice` | A selected label, option probabilities, and confidence | Billing, technical, or sales? |
| `Score` | Continuous value on an ordered, zero-based rubric | Calm = 0, frustrated = 1, angry = 2 |

A `Noul` value near 0.5 indicates uncertainty. A `Choice` still picks a label when none fits well; include an applicability check or a review path. Confidence describes separation in the model's answer distribution, not a guarantee of correctness.

This complements structured text generation: `ChatClient.entity(...)` produces typed content; Jev evaluates a supplied state against typed questions.

Source: [question primitives](https://spring-ai-community.github.io/spring-ai-typesafe/latest/concepts/primitives/).

### 34.2 Standalone Java: classify a ticket and decide whether to route it

For this example, add only the SDK to an existing Java 17+ Maven project:

```xml
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>typesafe-java-sdk</artifactId>
    <version>0.4.0</version>
</dependency>
```

Set `TYPESAFE_API_KEY` in your environment. This is a Java application with a `main` method; it does not need a Spring context. A hosted request requires valid credentials and service access.

```java
import java.util.LinkedHashMap;
import java.util.Map;

import org.springaicommunity.typesafe.TypeSafeClient;
import org.springaicommunity.typesafe.question.Choice;
import org.springaicommunity.typesafe.question.Noul;
import org.springaicommunity.typesafe.question.Question;
import org.springaicommunity.typesafe.question.Score;
import org.springaicommunity.typesafe.response.SystemOneResponse;

public class TicketJudgmentExample {

    public static void main(String[] args) {
        TypeSafeClient jev = TypeSafeClient.builder().build();

        Map<String, Question> questions = new LinkedHashMap<>();
        questions.put("urgent", Noul.of(
            "Does the customer express urgency or an ongoing financial loss?"));
        questions.put("team", Choice.builder()
            .instructions("Which support department should handle this message?")
            .option("billing", "Charges, refunds, invoices, and payments")
            .option("technical", "Broken features and integration errors")
            .option("sales", "Pricing and new purchases")
            .build());
        questions.put("frustration", Score.of(
            "How frustrated does the customer sound?",
            "Calm", "Frustrated", "Very angry"));

        SystemOneResponse result = jev.systemOne(
            "My card was charged twice. Please fix it today!", questions);

        String team = result.choiceValue("team");
        double confidence = result.choice("team").confidence();
        double urgency = result.noulValue("urgent");

        // Example application policy; tune thresholds against real tickets.
        String route = confidence >= 0.80 ? team : "human-review";
        System.out.println("Route: " + route);
        System.out.println("Urgent: " + (urgency >= 0.80));
        System.out.println("Frustration: " + result.scoreValue("frustration"));
    }
}
```

Illustrative output, not a live result:

```text
Route: billing
Urgent: true
Frustration: 1.2
```

`LinkedHashMap` preserves question order. This example prints a routing decision; it does not create a ticket, issue a refund, or perform an external action.

Sources: [Java client](https://spring-ai-community.github.io/spring-ai-typesafe/latest/client/TypeSafeClient/), [upstream ticket demo and import packages](https://github.com/spring-ai-community/spring-ai-typesafe/blob/main/examples/src/main/java/org/springaicommunity/typesafe/demo/TicketTriageDemo.java).

### 34.3 Spring Boot setup for judging and advisors

In a compatible Boot 4 application with Spring AI configured, use:

```xml
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>spring-ai-starter-typesafe</artifactId>
    <version>0.4.0</version>
</dependency>
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>typesafe-spring-ai</artifactId>
    <version>0.4.0</version>
</dependency>
```

```properties
spring.ai.typesafe.api-key=${TYPESAFE_API_KEY}
```

The starter provides a `TypeSafeClient` bean. Current documentation requires Spring AI **2.0.1+** for `typesafe-spring-ai`. You still need your chat provider configuration when generating answers.

The following are focused Java snippets; imports and bean wiring are omitted as in the rest of this guide.

Source: [TypeSafe quick start](https://spring-ai-community.github.io/spring-ai-typesafe/latest/).

### 34.4 Judge an answer against evidence

```java
JevJudge judge = JevJudge.builder(typeSafeClient)
    .noul("supported_by_policy", Noul.builder()
        .instructions("Is every policy claim in `assistant_answer` supported "
            + "by `supporting_context`?")
        .whenFalse("The answer makes a claim unsupported by the supplied policy.")
        .build(), 0.85)
    .failOnInconclusive(true)
    .failOnError(true)
    .build();

JevVerdict verdict = judge.judge(JevJudgeInput.builder()
    .question("Can I return this after 45 days?")
    .answer("Yes, all returns are accepted within 60 days.")
    .context("Returns must be requested within 30 days of delivery.")
    .build());

if (!verdict.passed()) {
    System.out.println(verdict.feedback());
}
```

Expected behavior: the conflicting 60-day claim should fail the evidence check. A model judgment is probabilistic; evaluate it against your own examples. `JevJudgeInput` explicitly supplies the policy instead of asking the judge to infer it.

Source: [JevJudge and evidence fields](https://spring-ai-community.github.io/spring-ai-typesafe/latest/judge/JevJudge/).

### 34.5 Retry an answer using judgment feedback

The policy is in the system prompt so the generated answer and advisor's judge input have the necessary evidence. Use a separate criterion that refers to that input:

```java
JevJudge retryJudge = JevJudge.builder(typeSafeClient)
    .noul("follows_policy", Noul.builder()
        .instructions("Does `assistant_answer` follow the return policy "
            + "provided in `user_question`, including its system instructions?")
        .whenFalse("Correct the answer to match the supplied return policy.")
        .build(), 0.85)
    .failOnInconclusive(true)
    .failOnError(true)
    .build();

ChatClient checkedClient = builder
    .defaultSystem("Return policy: requests must be made within 30 days. "
        + "Do not invent exceptions.")
    .defaultAdvisors(JevSelfRefineAdvisor.builder()
        .judge(retryJudge)
        .maxRepeatAttempts(2)
        .failOnExhaustedAttempts(true)
        .build())
    .build();

String answer = checkedClient.prompt()
    .user("Can I return my purchase after 45 days?")
    .call()
    .content();
```

```text
Generate → Judge → Pass: return answer
                 → Fail: provide feedback and retry
```

Two repeat attempts means at most three generation attempts. Exhaustion throws instead of returning an unchecked best effort. Service-error handling is a separate advisor policy; configure it for the application's requirements. The example is for synchronous calls, not token streaming.

Source: [self-refinement advisor](https://spring-ai-community.github.io/spring-ai-typesafe/latest/judge/JevSelfRefineAdvisor/).

### 34.6 Screen inputs and outputs

```java
ChatClient screenedClient = builder
    .defaultAdvisors(
        JevGuardrailAdvisor.builder(typeSafeClient).build())
    .build();

String answer = screenedClient.prompt()
    .user("Explain the return process.")
    .call()
    .content();
```

A blocked input stops before chat generation. A blocked output is replaced with a refusal. `REVIEW` proceeds and is logged by default; it does not automatically pause for a human. Guardrail detection is not a substitute for deterministic authorization in business tools.

Source: [guardrail advisor](https://spring-ai-community.github.io/spring-ai-typesafe/latest/guardrails/JevGuardrailAdvisor/).

### 34.7 Filter and rerank modular RAG documents

Requires `spring-ai-rag` in addition to the TypeSafe integration:

```java
var judgedRag = RetrievalAugmentationAdvisor.builder()
    .documentRetriever(VectorStoreDocumentRetriever.builder()
        .vectorStore(vectorStore)
        .topK(10)
        .build())
    .documentPostProcessors(
        JevDocumentFilter.builder(typeSafeClient).build(),
        JevDocumentReranker.builder(typeSafeClient).topK(3).build())
    .build();

String answer = client.prompt()
    .user("What is the deadline for returns?")
    .advisors(judgedRag)
    .call()
    .content();
```

```text
Retrieve candidates → Screen passages → Rank survivors → Supply context
```

Filtering checks evidence, relevance, conflicting premises, and prompt-injection signals. Reranking asks whether a passage can answer the question. These perform per-document work: filter before reranking to reduce unnecessary calls.

Sources: [document filter](https://spring-ai-community.github.io/spring-ai-typesafe/latest/rag/JevDocumentFilter/), [document reranker](https://spring-ai-community.github.io/spring-ai-typesafe/latest/rag/JevDocumentReranker/).

### 34.8 Select tools using structured judgment

Requires the optional `spring-ai-tool-search-tool` module. This selects references; it does not execute the tools.

```java
JevToolIndex index = JevToolIndex.builder(typeSafeClient).build();

index.indexTools(conversationId, List.of(
    ToolReference.builder()
        .toolName("deliveryStatus")
        .summary("Look up an order's shipping and arrival status")
        .build(),
    ToolReference.builder()
        .toolName("requestRefund")
        .summary("Submit a refund request for an order")
        .build()
));

ToolSearchResponse selection = index.search(
    new ToolSearchRequest(conversationId,
        "When will my package arrive?", 2, null));

selection.toolReferences().forEach(tool ->
    System.out.println(tool.toolName()));
```

Illustrative selection: `deliveryStatus`. A separate applicability question lets the index return no tool when the catalog cannot serve the request.

Source: [JevToolIndex](https://spring-ai-community.github.io/spring-ai-typesafe/latest/toolsearch/JevToolIndex/).

## 35. SkillsJars: package agent instructions as Java dependencies

**Status: ecosystem packaging approach with a community Spring AI integration; independently versioned.**

[SkillsJars](https://www.skillsjars.com/docs) distributes agent skills as Maven dependencies. A skill contains instructions and optional supporting files, rather than being a model or an executable Java tool by itself.

| Piece | Store-support example |
|---|---|
| Skill | Instructions for drafting a return-response checklist |
| Java tool | A method that queries an order service |
| MCP | A protocol through which an application exposes that method |
| Jev | A component that judges whether the draft follows policy |

The composition above is an application-design example, not a built-in combined product.

### 35.1 Consume a published SkillsJar

The SkillsJars documentation provides this browser skill as a concrete dependency example:

```xml
<dependency>
    <groupId>com.skillsjars</groupId>
    <artifactId>browser-use__browser-use__browser-use</artifactId>
    <version>2026_02_23-1d154e1</version>
</dependency>
```

The documented classpath convention is:

```text
META-INF/skills/<org>/<repo>/<skill>/SKILL.md
```

For Spring AI, skills load from the classpath without an extraction step. The extraction plugin is for assistants expecting filesystem files.

Source: [SkillsJars documentation](https://www.skillsjars.com/docs).

### 35.2 Java: connect packaged skills to ChatClient

Add Agent Utils to a compatible Spring AI application. Its current repository documents `0.13.0`; the SkillsJars page shows an older `0.5.0` example. This section uses the current repository's version and the documented SkillsTool callback API.

```xml
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>spring-ai-agent-utils</artifactId>
    <version>0.13.0</version>
</dependency>
```

Source: [Agent Utils dependency](https://github.com/spring-ai-community/spring-ai-agent-utils).

Inside a Spring configuration class, with the `SkillsTool` import from the chosen Agent Utils release:

```java
@Bean
ChatClient skillsClient(ChatClient.Builder builder) {
    var skills = SkillsTool.builder()
        .addSkillsResource(new ClassPathResource("META-INF/skills"))
        .build();

    return builder
        .defaultToolCallbacks(skills)
        .build();
}
```

```java
String plan = skillsClient.prompt()
    .user("Use the browser-use skill to explain a workflow for checking "
        + "an order's delivery status in a browser.")
    .call()
    .content();
```

Conceptually:

```text
Load skill metadata → Model selects skill → Load instructions → Follow workflow
```

This example asks for a workflow description. Actually controlling a browser requires the corresponding browser tools and runtime. Likewise, a skill that runs Python needs a suitable execution tool and Python environment; adding its JAR does not supply those capabilities.

Skill selection is model-driven, not a guarantee on every invocation. Reference-file access and script execution require appropriate tools and accessible resources.

Source: [SkillsTool classpath loading and callback registration](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/docs/tools/SkillsTool.md).

### 35.3 Define a small skill of your own

Illustrative content for `skills/return-response/SKILL.md` in a separate skill-library project:

```markdown
---
name: return-response
description: Draft a customer reply about returns using the supplied policy and order facts.
---
# Return response

1. Read the provided policy and order facts.
2. State whether the request meets the policy deadline.
3. List missing facts without guessing them.
4. Draft a short reply explaining the next step.
5. Do not claim a refund was issued unless a tool result confirms it.
```

The file uses the Agent Skills convention: required `name` and `description` metadata followed by task instructions. This block is documentation only; it does not install or activate a skill in this workspace.

Source: [Agent Skills specification](https://agentskills.io/specification).

### 35.4 Package the skill into a JAR

In that skill-library project's Maven build:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.skillsjars</groupId>
            <artifactId>maven-plugin</artifactId>
            <version>0.0.7</version>
            <executions>
                <execution>
                    <goals>
                        <goal>package</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

```bash
./mvnw package
```

The plugin packages the project's `skills/` contents under `META-INF/skills/`. Consumers then add the resulting artifact through normal Maven coordinates. Publishing is a separate operation.

Source: [creating SkillsJars](https://www.skillsjars.com/docs#creating-skillsjars).

### 35.5 Filesystem-based coding assistants

For assistants expecting files, configure the extraction plugin with the desired skill dependencies and run:

```bash
./mvnw skillsjars:extract -Ddir=.agents/skills
```

Use the assistant's actual discovery directory. This extraction path is separate from the classpath-loading approach for your Spring AI application's runtime.

Source: [official Spring AI example](https://github.com/skillsjars/skillsjars-example-spring-ai).

### 35.6 Example composition with the rest of the guide

```text
Customer requests help with a late return
    → RAG retrieves policy
    → Order tool supplies purchase facts
    → return-response skill guides drafting
    → Chat model generates the response
    → Jev evaluates policy consistency
    → Application shows the response or requests review
```

A skill supplies procedure, RAG supplies evidence, tools supply capabilities, and judgment evaluates the result. Business rules and authorization still belong in application code.

Examples in sections 34–35 are documentation snippets, not compiled or live-service-verified demonstrations. Dependency versions are pinned to the inspected documentation; verify release compatibility when adopting them.

## 36. Primary reference index

- [Spring AI project and learning resources](https://spring.io/projects/spring-ai#learn)
- [Spring AI 2.0 GA announcement](https://spring.io/blog/2026/06/12/spring-ai-2-0-0-GA-available-now/)
- [Spring AI 2.0.1 release notes](https://github.com/spring-projects/spring-ai/releases/tag/v2.0.1)
- [Spring AI 2.0.2 release notes](https://github.com/spring-projects/spring-ai/releases/tag/v2.0.2)
- [Spring AI 2.1 M1 release notes](https://github.com/spring-projects/spring-ai/releases/tag/v2.1.0-M1)
- [Spring AI 2.1 RC1 release notes](https://github.com/spring-projects/spring-ai/releases/tag/v2.1.0-RC1)
- [October 2026 announcement and next steps](https://spring.io/blog/2026/10/23/spring-ai-2-1-0-RC1-available-now/)
- [Upgrade notes](https://docs.spring.io/spring-ai/reference/upgrade-notes.html)
- [ChatClient](https://docs.spring.io/spring-ai/reference/api/chatclient.html)
- [Tools](https://docs.spring.io/spring-ai/reference/api/tools.html)
- [Recursive advisors](https://docs.spring.io/spring-ai/reference/api/advisors-recursive.html)
- [Structured output](https://docs.spring.io/spring-ai/reference/api/structured-output.html)
- [ETL](https://docs.spring.io/spring-ai/reference/api/etl-pipeline.html)
- [RAG](https://docs.spring.io/spring-ai/reference/api/retrieval-augmented-generation.html)
- [Chat memory](https://docs.spring.io/spring-ai/reference/api/chat-memory.html)
- [Redis and semantic caching](https://docs.spring.io/spring-ai/reference/api/vectordbs/redis.html)
- [MCP annotations](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-annotations-server.html)
- [MCP server configuration](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-server-boot-starter-docs.html)
- [MCP client configuration](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-client-boot-starter-docs.html)
- [Observability](https://docs.spring.io/spring-ai/reference/observability/index.html)
- [Preview Responses API](https://docs.spring.io/spring-ai/reference/2.1/api/chat/openai-responses-chat.html)
- [Preview message parts](https://docs.spring.io/spring-ai/reference/2.1/api/chatmodel.html)
- [Preview session memory](https://docs.spring.io/spring-ai/reference/2.1/api/session.html)
- [Preview tool confirmation](https://docs.spring.io/spring-ai/reference/2.1/api/tools/tool-call-confirmation.html)
- [Preview vector operations](https://docs.spring.io/spring-ai/reference/2.1/api/vectordbs.html)
- [Preview chunk overlap](https://docs.spring.io/spring-ai/reference/2.1/api/etl-pipeline.html)
- [Release milestones](https://github.com/spring-projects/spring-ai/milestones)
- [TypeSafe structured judgment article](https://spring.io/blog/2026/09/21/spring-ai-typesafe-structured-judgment/)
- [TypeSafe Java and Spring AI documentation](https://spring-ai-community.github.io/spring-ai-typesafe/latest/)
- [SkillsJars documentation](https://www.skillsjars.com/docs)
- [SkillsJars Spring AI example](https://github.com/skillsjars/skillsjars-example-spring-ai)
- [Agent Utils SkillsTool](https://github.com/spring-ai-community/spring-ai-agent-utils/blob/main/docs/tools/SkillsTool.md)
- [Agent Skills specification](https://agentskills.io/specification)

Unversioned documentation links may track newer stable releases over time. Match the documentation selector to the Spring AI version used by your application.
