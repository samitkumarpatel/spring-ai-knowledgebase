# Spring AI 2.x: A Step-by-Step Walkthrough

Build a store support assistant step by step: generate answers, call tools, retrieve policy documents, and evaluate responses.

**Versions checked: 28 October 2026.** Stable: **2.0.2**. Preview: **2.1.0-RC1**. Jev and SkillsJars are community integrations.

The examples focus on individual features; imports and bean wiring are omitted. They assume an injected `ChatClient.Builder` and configured services where needed. These snippets have not been compiled or run.

[Full reference, setup details, and sources](docs/spring-ai-2x-reference.md).

## Walkthrough

| Step | Feature | What it adds |
|---|---|---|
| 1 | ChatClient + structured output | Text, Java objects, and response context |
| 2 | Tools + tool search | Call application code |
| 3 | Memory | Remember an earlier turn |
| 4 | Modular RAG | Answer using your documents |
| 5 | MCP | Expose and consume remote tools |
| 6 | Jev | Classify requests and judge answers |
| 7 | SkillsJars | Load reusable instructions from a dependency |
| 8 | 2.1 preview | Sessions, confirmation, Responses, and vectors |

## Setup

### Maven version management

> Start with a Spring Boot **4.0.x or 4.1.x** application. Add these sections to `pom.xml`; Spring Boot manages its own dependencies, and the Spring AI BOM manages `org.springframework.ai` modules.

```xml
<properties>
    <spring-ai.version>2.0.2</spring-ai.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>${spring-ai.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Chat, tools, and memory

> Add this inside `<dependencies>`. It provides the OpenAI integration and auto-configured ChatClient builder. Basic tool calling and default in-memory chat memory need no additional starter.

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

> For the tool-search example, also add:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-tool-search-advisor</artifactId>
</dependency>
```

### Modular RAG and PDF ingestion

> This example uses PostgreSQL with PGvector. Add these dependencies and configure a database with the required extensions. The OpenAI starter above supplies the embedding model.

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-rag</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pdf-document-reader</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

> For the reference guide's `QuestionAnswerAdvisor` example, also add `org.springframework.ai:spring-ai-vector-store-advisor`.

### MCP server and client

> Add the server starter to the **order-server application**:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
</dependency>
```

> Add the client starter to the **support-assistant application**, alongside the model starter:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client</artifactId>
</dependency>
```

### Jev and SkillsJars

> Community libraries use explicit versions, independently of the Spring AI BOM. Add the dependencies for the features you use:

```xml
<!-- Jev: typed judgments, judges, and advisors -->
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

<!-- SkillsTool integration -->
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>spring-ai-agent-utils</artifactId>
    <version>0.13.0</version>
</dependency>

<!-- Example packaged browser skill -->
<dependency>
    <groupId>com.skillsjars</groupId>
    <artifactId>browser-use__browser-use__browser-use</artifactId>
    <version>2026_02_23-1d154e1</version>
</dependency>
```

### Preview session memory

> For section 8, use the **2.1.0-RC1** Spring AI BOM with its compatible Spring Boot **4.2** preview baseline. Keep the preview setup separate from the stable examples. Add the JDBC session starter and a database driver; H2 is sufficient for a local example:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-session-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

### Application configuration

```properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.model=gpt-5-mini

# Jev examples
spring.ai.typesafe.api-key=${TYPESAFE_API_KEY}

# PGvector examples: provide a PostgreSQL database with vector support
spring.datasource.url=${DATABASE_URL}
spring.datasource.username=${DATABASE_USERNAME}
spring.datasource.password=${DATABASE_PASSWORD}
spring.ai.vectorstore.pgvector.initialize-schema=true
```

> Include only properties for the integrations you enable. Use a fresh builder for each client. `conversationId` identifies the current user's conversation.

> Dependency references: [Spring AI BOM](https://docs.spring.io/spring-ai/reference/getting-started.html), [PDF reader](https://docs.spring.io/spring-ai/reference/api/etl-pipeline.html#_pdf_page), [PGvector](https://docs.spring.io/spring-ai/reference/api/vectordbs/pgvector.html), [Jev](https://spring-ai-community.github.io/spring-ai-typesafe/latest/), [Agent Utils](https://github.com/spring-ai-community/spring-ai-agent-utils), [SkillsJars](https://www.skillsjars.com/docs).

## 1. ChatClient: text, streaming, Java objects, and response context

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

> `call().content()` returns the complete answer. `stream().content()` returns a `Flux<String>` that the interface can display as chunks arrive.

```java
record Ticket(String category, String summary) {}

Ticket ticket = client.prompt()
    .user("My card was charged twice.")
    .call()
    .entity(Ticket.class, spec -> spec
        .useProviderStructuredOutput()
        .validateSchema());
```

> An illustrative result is `Ticket("PAYMENT", "Duplicate card charge")`. The entity API maps the response into a Java record; schema validation checks its structure. Factual and business checks remain separate.

### ChatClientResponse: model response and advisor context

```java
ChatClientResponse result = client.prompt()
    .user("Explain refunds in one sentence.")
    .call()
    .chatClientResponse();

ChatResponse modelResponse = result.chatResponse();
if (modelResponse != null) {
    System.out.println(modelResponse.getResult().getOutput().getText());
    System.out.println(modelResponse.getMetadata().getUsage());
}

System.out.println(result.context());
```

> `chatClientResponse()` returns the model response together with the advisor execution context. `chatResponse()` exposes answer text and usage metadata; `context()` exposes data contributed by advisors, such as retrieved documents in a RAG flow. The context can be empty when no advisor adds data. The model response is nullable, so the example checks it before reading it.

> Source: [ChatClient return types](https://docs.spring.io/spring-ai/reference/api/chatclient.html#_call_return_values), [ChatClientResponse API](https://docs.spring.io/spring-ai/docs/current/api/org/springframework/ai/chat/client/ChatClientResponse.html).

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

> The model requests `deliveryStatus("A123")`, Spring AI executes the Java method, and the model uses its result to answer. In 2.0, ChatClient's `ToolCallingAdvisor` manages this loop.

> For a large tool catalog, add the tool-search starter and enable:

```properties
spring.ai.chat.client.tool-search-advisor.enabled=true
```

> Tool search discovers relevant tools on demand, reducing the tool definitions included in each model request.

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

> Both requests use the same conversation ID, so the follow-up can recall `A123`. This uses the 2.0 memory API; the preview section introduces session memory.

## 4. Modular RAG: answer from a policy document

> First, extract the PDF text, split it into chunks, and store their embeddings:

```java
var reader = new PagePdfDocumentReader(
    new ClassPathResource("return-policy.pdf"));
var splitter = TokenTextSplitter.builder().withChunkSize(800).build();
vectorStore.add(splitter.apply(reader.get()));
```

> Next, retrieve matching chunks and add them to the question before generation:

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

> If the PDF specifies a 30-day deadline, the retrieved context gives the model evidence for answering the 45-day question. The similarity threshold controls which matches qualify and should be tuned for the data.

> Optional query rewriting—add this to the advisor builder:

```java
.queryTransformers(RewriteQueryTransformer.builder()
    .chatClientBuilder(builder.build().mutate())
    .build())
```

> Optional filtered lookup:

```java
List<Document> matches = vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("return deadline")
        .filterExpression("country == 'DK'")
        .topK(3).build());
```

> Query rewriting turns an informal question into a clearer search query. The metadata filter limits results to documents tagged `country=DK`.

## 5. MCP: expose the order lookup to another application

> Server component:

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

> Server configuration:

```properties
spring.ai.mcp.server.protocol=STREAMABLE
```

> Client connection:

```properties
spring.ai.mcp.client.streamable-http.connections.orders.url=http://localhost:8080
```

> Client request, using the configured MCP tool callback provider:

```java
String answer = client.prompt()
    .user("When will order A123 arrive?")
    .toolCallbacks(mcpToolCallbackProvider.getToolCallbacks())
    .call().content();
```

> The request travels from ChatClient through the MCP client to the order server. The server executes the lookup and returns its result. Client and server run as separate applications; endpoint security is configured separately.

## 6. Jev: classify and evaluate

```properties
spring.ai.typesafe.api-key=${TYPESAFE_API_KEY}
```

> Classify a support ticket using the injected `TypeSafeClient`:

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

> Jev returns typed judgments. This routing policy selects a department when confidence reaches `0.80` and otherwise requests human review. The threshold is an application choice.

> Judge an answer against supplied policy:

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

> The answer claims 60 days while the supplied policy says 30, so the judgment should fail and provide feedback. Jev evaluates supplied content; the chat model generates the answer.

> [More Jev examples: retries, guardrails, RAG, and tool selection](docs/spring-ai-2x-reference.md#34-typesafe-jev-structured-judgment-with-java-examples).

## 7. SkillsJars: load instructions from a dependency

> The browser skill dependency from Setup supplies instructions under `META-INF/skills`.

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

> The JAR supplies skill metadata and instructions that load on demand. Here, the model describes a browser workflow. Executing it requires browser tools and a runtime.

> [Custom skill definition, packaging, and extraction](docs/spring-ai-2x-reference.md#35-skillsjars-package-agent-instructions-as-java-dependencies).

## 8. Spring AI 2.1 preview

> Use **2.1.0-RC1** dependencies for these snippets.

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

> Session memory stores conversation events and builds a smaller view for the model. Compaction keeps recent complete turns while retaining the original event log.

### Tool confirmation

```java
@Tool(description = "Cancel an order", requiresConfirmation = true)
public String cancelOrder(String orderId) {
    return orderService.cancel(orderId);
}
```

> A request to cancel an order pauses for a decision before execution. The application collects approval, executes allowed calls, and resumes the conversation. The annotation marks the requirement; the application provides the UI and resumption logic.

### Responses API

```properties
spring.ai.openai.chat.api=responses
spring.ai.openai.responses.model=gpt-5-mini
spring.ai.openai.responses.reasoning-effort=medium
```

> These properties select the Responses integration while keeping the same ChatClient API.

### Chunk overlap

```java
var splitter = TokenTextSplitter.builder()
    .withChunkSize(800)
    .withChunkOverlap(100)
    .build();
```

> Each chunk repeats up to 100 tokens from the preceding chunk, preserving context near boundaries. Overlap counts toward the 800-token chunk size.

### Existing vectors

```java
vectorStore.upsert(List.of(new EmbeddedDocument(document, embedding)));

List<Document> matches = vectorStore.similaritySearch(
    queryEmbedding, SearchRequest.builder().topK(3).build());
```

> `upsert` stores a document with its existing embedding. The search accepts an existing query vector. Both operations require a supporting store and compatible embeddings.

## Additional capabilities

| Feature | Behavior |
|---|---|
| Semantic cache | Equivalent questions can reuse a cached answer when similarity meets the configured threshold |
| Observability | `response.getMetadata().getUsage()` exposes usage; configured tracing records model and tool operations |
| Message parts, preview | `response.getResult().getOutput().getParts()` exposes ordered content parts |
| Audio/image | Supported models transcribe recordings, stream speech, or generate images |

## What's next

> **2.1 GA target: 19 November 2026**, subject to change. The announced focus is stabilizing session memory and message parts. Legacy chat-memory APIs are deprecated in the preview for planned removal in 3.0.

## Links

- [Spring AI learning resources](https://spring.io/projects/spring-ai#learn)
- [2.0 release overview](https://spring.io/blog/2026/06/12/spring-ai-2-0-0-GA-available-now/)
- [2.1 preview and next steps](https://spring.io/blog/2026/10/23/spring-ai-2-1-0-RC1-available-now/)
- [TypeSafe structured judgment](https://spring.io/blog/2026/09/21/spring-ai-typesafe-structured-judgment/)
- [SkillsJars](https://www.skillsjars.com/docs)
- [Detailed examples and complete references](docs/spring-ai-2x-reference.md)
