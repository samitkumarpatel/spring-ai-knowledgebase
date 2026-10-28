# Spring AI 2.x Demo

**Scenario:** build a store support assistant that answers policy questions, checks orders, and evaluates its responses.

**Versions checked: 28 October 2026.** Stable: **2.0.2**. Preview: **2.1.0-RC1**. Jev and SkillsJars are community integrations.

These are demo snippets, not a runnable application. Imports and bean wiring are omitted. Use an injected `ChatClient.Builder`; configure the vector store, memory, and credentials for the demos you select. Examples have not been compiled or run.

[Full reference, setup details, and sources](docs/spring-ai-2x-reference.md).

## Demo order

| Step | Show | Main point |
|---|---|---|
| 1 | ChatClient + structured output | Text or Java objects from one fluent API |
| 2 | Tools + tool search | Call application code |
| 3 | Memory | Remember an earlier turn |
| 4 | Modular RAG | Answer using your documents |
| 5 | MCP | Expose and consume remote tools |
| 6 | Jev | Classify requests and judge answers |
| 7 | SkillsJars | Load reusable instructions from a dependency |
| 8 | 2.1 preview | Sessions, confirmation, Responses, and vectors |

## Setup

Use a Spring Boot 4 application and the Spring AI BOM. Select modules for each demo:

| Demo | Modules / prerequisites |
|---|---|
| Chat | `spring-ai-starter-model-openai` |
| Tools | Chat starter; optional `spring-ai-starter-tool-search-advisor` |
| Memory | Configured `ChatMemory` bean |
| RAG | `spring-ai-rag`, vector store + embedding model; PDF reader for ingestion |
| MCP | Server: `spring-ai-starter-mcp-server-webmvc`; client: `spring-ai-starter-mcp-client` |
| Jev | `spring-ai-starter-typesafe` + `typesafe-spring-ai`, version `0.4.0` |
| Skills | `spring-ai-agent-utils:0.13.0` + a SkillsJar dependency |
| Preview sessions | `spring-ai-starter-model-session-jdbc` + a data source |

```properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.model=gpt-5-mini
```

Use a fresh builder for each client. `conversationId` must identify the current user's conversation.

## 1. ChatClient: text, streaming, and Java objects

```java
ChatClient client = builder
    .defaultSystem("You are a store support assistant. Keep answers brief.")
    .build();

String answer = client.prompt()
    .user("Explain refunds in one sentence.")
    .call().content();

Flux<String> stream = client.prompt()
    .user("Explain the return process.")
    .stream().content();
```

**Show:** a complete answer versus text arriving in chunks.

```java
record Ticket(String category, String summary) {}

Ticket ticket = client.prompt()
    .user("My card was charged twice.")
    .call()
    .entity(Ticket.class, spec -> spec
        .useProviderStructuredOutput()
        .validateSchema());
```

**Show:** `Ticket("PAYMENT", "Duplicate card charge")` as an illustrative result. Schema validation checks structure; factual and business checks remain separate.

## 2. Tools: check an order

```java
class OrderTools {
    @Tool(description = "Look up an order's delivery status")
    public String deliveryStatus(String orderId) {
        return "Order " + orderId + " arrives Friday.";
    }
}

String answer = client.prompt()
    .user("When will order A123 arrive?")
    .tools(new OrderTools())
    .call().content();
```

**Show:** model requests a tool → Java executes → model explains the result. In 2.0, ChatClient's `ToolCallingAdvisor` manages this loop.

For a large tool catalog, add the tool-search starter and enable:

```properties
spring.ai.chat.client.tool-search-advisor.enabled=true
```

**Show:** relevant tools are discovered instead of sending the whole catalog with every request.

## 3. Memory: remember the order number

```java
ChatClient memoryClient = builder
    .defaultAdvisors(
        MessageChatMemoryAdvisor.builder(chatMemory).build())
    .build();

memoryClient.prompt()
    .user("My order number is A123.")
    .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
    .call().content();

String answer = memoryClient.prompt()
    .user("What order number did I give you?")
    .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
    .call().content();
```

**Show:** the follow-up can answer `A123`. This uses the 2.0 memory API; see the session replacement in the preview below.

## 4. Modular RAG: answer from a policy document

Ingest once:

```java
var reader = new PagePdfDocumentReader(
    new ClassPathResource("return-policy.pdf"));
var splitter = TokenTextSplitter.builder().withChunkSize(800).build();
vectorStore.add(splitter.apply(reader.get()));
```

Retrieve context for each question:

```java
var rag = RetrievalAugmentationAdvisor.builder()
    .documentRetriever(VectorStoreDocumentRetriever.builder()
        .vectorStore(vectorStore)
        .similarityThreshold(0.70)
        .build())
    .build();

String answer = client.prompt()
    .user("Can I return an item after 45 days?")
    .advisors(rag)
    .call().content();
```

**Show:** put a 30-day deadline in the PDF and inspect the answer against it. Adjust retrieval thresholds for your data.

Optional query rewriting—add this to the advisor builder:

```java
.queryTransformers(RewriteQueryTransformer.builder()
    .chatClientBuilder(builder.build().mutate())
    .build())
```

Optional filtered lookup:

```java
List<Document> matches = vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("return deadline")
        .filterExpression("country == 'DK'")
        .topK(3).build());
```

**Show:** rewrite an informal query or restrict retrieval to documents tagged `country=DK`.

## 5. MCP: expose the order lookup to another application

Server component:

```java
@Component
class OrderMcpTools {
    @McpTool(description = "Look up delivery status")
    public String deliveryStatus(
            @McpToolParam(required = true) String orderId) {
        return "Order " + orderId + " arrives Friday.";
    }
}
```

Server configuration:

```properties
spring.ai.mcp.server.protocol=STREAMABLE
```

Client connection:

```properties
spring.ai.mcp.client.streamable-http.connections.orders.url=http://localhost:8080
```

Client request, using the configured MCP tool callback provider:

```java
String answer = client.prompt()
    .user("When will order A123 arrive?")
    .toolCallbacks(mcpToolCallbackProvider.getToolCallbacks())
    .call().content();
```

**Show:** ChatClient → MCP client → order server. Run the client separately from the server. Configure endpoint security before exposing it beyond the local demo.

## 6. Jev: classify and evaluate

```properties
spring.ai.typesafe.api-key=${TYPESAFE_API_KEY}
```

Classify a support ticket using the injected `TypeSafeClient`:

```java
SystemOneResponse result = typeSafeClient.systemOne(
    "My card was charged twice. Please fix it today!",
    Map.of(
        "urgent", Noul.of("Does the customer express urgency?"),
        "team", Choice.builder()
            .instructions("Which team should handle this?")
            .option("billing", "Charges, invoices, and refunds")
            .option("technical", "Broken features and integrations")
            .build()));

String route = result.choice("team").confidence() >= 0.80
    ? result.choiceValue("team") : "human-review";
```

**Show:** typed decisions with a confidence-based review path. The threshold is an example policy, not a universal recommendation.

Judge an answer against supplied policy:

```java
JevJudge judge = JevJudge.builder(typeSafeClient)
    .noul("supported", Noul.builder()
        .instructions("Is `assistant_answer` supported by `supporting_context`?")
        .whenFalse("The answer contradicts or invents policy.")
        .build(), 0.85)
    .failOnInconclusive(true)
    .failOnError(true)
    .build();

JevVerdict verdict = judge.judge(JevJudgeInput.builder()
    .question("Can I return it after 45 days?")
    .answer("Yes, returns are allowed for 60 days.")
    .context("Return requests must be made within 30 days.")
    .build());

System.out.println(verdict.passed());
System.out.println(verdict.feedback());
```

**Show:** the conflicting answer should fail the check. Jev evaluates supplied content; the chat model generates the answer.

[More Jev examples: retries, guardrails, RAG, and tool selection](docs/spring-ai-2x-reference.md#34-typesafe-jev-structured-judgment-with-java-examples).

## 7. SkillsJars: load instructions from a dependency

Example published skill dependency, alongside Agent Utils:

```xml
<dependency>
    <groupId>com.skillsjars</groupId>
    <artifactId>browser-use__browser-use__browser-use</artifactId>
    <version>2026_02_23-1d154e1</version>
</dependency>
```

```java
ChatClient skillsClient = builder
    .defaultToolCallbacks(SkillsTool.builder()
        .addSkillsResource(new ClassPathResource("META-INF/skills"))
        .build())
    .build();

String plan = skillsClient.prompt()
    .user("Use the browser-use skill to explain a workflow "
        + "for checking an order's delivery status.")
    .call().content();
```

**Show:** dependency → skill metadata → instructions loaded on demand. This describes a workflow; actual browser execution requires browser tools and a runtime.

[Custom skill definition, packaging, and extraction](docs/spring-ai-2x-reference.md#35-skillsjars-package-agent-instructions-as-java-dependencies).

## 8. Spring AI 2.1 preview

Use **2.1.0-RC1** dependencies for these snippets.

### Session memory

```java
ChatClient sessionClient = builder
    .defaultAdvisors(SessionAdvisors.builder(sessionService)
        .compaction(new TurnCountTrigger(20),
            WindowCompactionStrategy.builder().lastTurns(10).build())
        .build())
    .build();

String answer = sessionClient.prompt()
    .user("What did we decide about my return?")
    .advisors(SessionAdvisors.session(conversationId))
    .call().content();
```

**Show:** an event log with a compacted model view. Original events remain stored.

### Tool confirmation

```java
@Tool(description = "Cancel an order", requiresConfirmation = true)
public String cancelOrder(String orderId) {
    return orderService.cancel(orderId);
}
```

**Show:** requested action → pending decision → application approval UI → execution and resume. The annotation does not implement the UI or resumption.

### Responses API

```properties
spring.ai.openai.chat.api=responses
spring.ai.openai.responses.model=gpt-5-mini
spring.ai.openai.responses.reasoning-effort=medium
```

**Show:** use the same ChatClient API with the Responses integration.

### Chunk overlap

```java
var splitter = TokenTextSplitter.builder()
    .withChunkSize(800)
    .withChunkOverlap(100)
    .build();
```

**Show:** neighboring chunks share boundary context.

### Existing vectors

```java
vectorStore.upsert(List.of(new EmbeddedDocument(document, embedding)));

List<Document> matches = vectorStore.similaritySearch(
    queryEmbedding, SearchRequest.builder().topK(3).build());
```

**Show:** import and query vectors without generating them again. Requires a supporting store and compatible embeddings.

## Optional demos

| Feature | Quick demonstration |
|---|---|
| Semantic cache | Ask equivalent policy questions and inspect whether the second hits the cache |
| Observability | Inspect `response.getMetadata().getUsage()` and model/tool traces |
| Message parts, preview | Inspect `response.getResult().getOutput().getParts()` |
| Audio/image | Transcribe a recording, stream speech, or generate a help illustration |

## What's next

**2.1 GA target: 19 November 2026**, subject to change. The announced focus is stabilizing session memory and message parts. Legacy chat-memory APIs are deprecated in the preview for planned removal in 3.0.

## Links

- [Spring AI learning resources](https://spring.io/projects/spring-ai#learn)
- [2.0 release overview](https://spring.io/blog/2026/06/12/spring-ai-2-0-0-GA-available-now/)
- [2.1 preview and next steps](https://spring.io/blog/2026/10/23/spring-ai-2-1-0-RC1-available-now/)
- [TypeSafe structured judgment](https://spring.io/blog/2026/09/21/spring-ai-typesafe-structured-judgment/)
- [SkillsJars](https://www.skillsjars.com/docs)
- [Detailed examples and complete references](docs/spring-ai-2x-reference.md)
