# Spring AI: A Step-by-Step Walkthrough

| Version | Release Date | Status |
|---|---|---|
| 1.0.0 GA | May 20, 2025 | Released |
| 1.1.0 GA | Nov 12, 2025 | Released |
| 2.0.0 GA | Jun 12, 2026 | Released |
| 2.0.1 | Aug 21, 2026 | Released |
| **2.0.2** | **Oct 23, 2026** | Released (current) |
| 2.1.0-RC1 | Oct 23, 2026 | Released, release candidate (same day as 2.0.2) |
| 2.1.0 GA | TBD | Upcoming, date not announced |

## Walkthrough

| Step | Feature | What it adds |
|---|---|---|
| 1 | ChatClient + structured output | Text, streaming, Java objects, and response context |
| 2 | Tools + tool search | The model calls your Java methods with `@Tool`, and Spring AI runs them and feeds results back; with a large catalog, tool search sends only the relevant tool definitions |
| 3 | MCP | Expose and consume remote tools, resources, prompts, and app UIs |
| 4 | Memory | Remember an earlier turn |
| 5 | Modular RAG | Answer using your documents |
| 6 | Jev | Classify requests and judge answers |
| 7 | SkillsJars | Load reusable instructions from a dependency |
| 8 | 2.1 preview | Sessions, confirmation, Responses, and vectors |

## Setup

### Maven version management

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

> In the dependency tables, IDs are start.spring.io IDs. The Maven artifactId differs for most starters, for example `spring-ai-openai` is `org.springframework.ai:spring-ai-starter-model-openai`. Versions come from the Spring AI BOM.

## Examples

### 1. ChatClient: text, streaming, Java objects, and response context

**Dependencies**: one chat model starter (swap in any provider from the table below):

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

**Chat models**

| ID | Provider |
|---|---|
| `spring-ai-openai` | OpenAI (ChatGPT, DALL-E), Azure OpnAI |
| `spring-ai-anthropic` | Anthropic Claude |
| `spring-ai-google-genai` | Google GenAI (Gemini) |
| `spring-ai-bedrock-converse` | Amazon Bedrock Converse (tool calling, multimodal, streaming) |
| `spring-ai-mistral` | Mistral AI |
| `spring-ai-deepseek` | DeepSeek |
| `spring-ai-ollama` | Ollama (local LLMs) |

Non-chat model starters in the same group:
- `spring-ai-stabilityai`: text-to-image
- `spring-ai-elevenlabs`: text-to-speech

```java
ChatClient client = builder
    .defaultSystem("You are a store support assistant. Keep answers brief.")
    .build();

String answer = client.prompt()
    .user("Explain refunds in one sentence.")
    .call().content(); //returns the complete answer

Flux<String> stream = client.prompt()
    .user("Explain the return process.")
    .stream().content(); //returns a Stream


/*The entity API maps the response into a Java record; schema validation checks its structure. Factual and business checks remain separate.*/

record Ticket(String category, String summary) {}

Ticket ticket = client.prompt()
    .user("My card was charged twice.")
    .call()
    .entity(Ticket.class, spec -> spec
        .useProviderStructuredOutput()
        .validateSchema()); // Ticket("PAYMENT", "Duplicate card charge")
```

**ChatClientResponse** - to get response context

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

> Explore More: [ChatClient return types](https://docs.spring.io/spring-ai/reference/api/chatclient.html#_call_return_values), [ChatClientResponse API](https://docs.spring.io/spring-ai/docs/current/api/org/springframework/ai/chat/client/ChatClientResponse.html).

### 2. Tools and tool search

**Dependencies**: none beyond the chat model starter from step 1, because `@Tool` is part of the core chat client. Tool search adds one starter:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-tool-search-advisor</artifactId>
</dependency>
```

**Tool search**

| ID | Purpose |
|---|---|
| `spring-ai-tool-search-advisor` | Dynamic tool discovery through a search index |

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

> Tool search discovers relevant tools on demand, reducing the tool definitions included in each model request. It uses `ToolSearchToolCallingAdvisor`, which indexes the tool set once per session and gives the model a built-in `toolSearchTool` to retrieve relevant tools by natural-language query.

### 3. Remote tools, resources, prompts, and apps (MCP)

**Dependencies**: the server application needs the MCP server starter; the application that calls it needs the client starter (plus a chat model). Add `mcp-security` (listed in the table below) to secure the endpoints.

```xml
<!-- MCP server application -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server</artifactId>
</dependency>

<!-- Application that consumes the MCP server -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client</artifactId>
</dependency>
```

Other dependency 

| ID | Purpose |
|---|---|
| `mcp-security` | Security for MCP server/client and OAuth2 Authorization Server (experimental) |

> Server component: a tool, two resources, an app UI, and a prompt, all exposed from another application:

```java
@Component
class OrderMcpTools {

    // Tool: the model calls it
    @McpTool(description = "Look up delivery status")
    public String deliveryStatus(
            @McpToolParam(required = true) String orderId) {
        return "Order " + orderId + " arrives Friday.";
    }

    // Resource: read-only data, fixed URI
    @McpResource(uri = "config://delivery/carriers", name = "carriers",
            description = "Supported carriers", mimeType = "application/json")
    public String carriers() {
        return "[\"GLS\", \"DHL\", \"FEDEX\"]";
    }

    // Resource template: {orderId} binds to the method parameter
    @McpResource(uri = "delivery://orders/{orderId}", name = "order",
            description = "Order details by id", mimeType = "application/json")
    public String order(String orderId) {
        return "{\"orderId\":\"" + orderId + "\",\"status\":\"IN_TRANSIT\"}";
    }

    // App UI (MCP Apps): an HTML page served as a ui:// resource
    @McpResource(uri = "ui://delivery/tracker", name = "delivery_tracker_ui",
            description = "Tracking dashboard", mimeType = "text/html;profile=mcp-app")
    public String trackerUi() {
        return "<!doctype html><html><body><h3>Delivery tracker</h3></body></html>";
    }

    // Prompt: a reusable template the client can offer to the user
    @McpPrompt(name = "delay_email", description = "Draft a delay notice to a customer")
    public GetPromptResult delayEmail(
            @McpArg(name = "customer", required = true) String customer,
            @McpArg(name = "orderId", required = true) String orderId) {
        var text = "Write a polite email to " + customer
                + " explaining that order " + orderId + " is delayed.";
        return GetPromptResult.builder(List.of(
                new PromptMessage(Role.USER, TextContent.builder(text).build())))
            .build();
    }
}
```

> The annotations come from `mcp-annotations` via `spring-ai-mcp-server`. `@McpTool`, `@McpResource`, and `@McpPrompt` accept a `metaProvider` attribute that supplies the `_meta` field of the declaration. That is the route for linking a tool to the `ui://` resource. The Spring AI MCP annotations documentation does not cover MCP Apps or `ui://`, so confirm host support and the `_meta` shape for your version before relying on it. Details: [MCP annotations](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-annotations-server.html).

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

> `toolCallbacks` carries tools only. Resources, prompts, and the app UI are handled by the MCP client or host, and are not inserted into every ChatClient call automatically.

### 4. Memory

**Dependencies**: none for the default in-memory store. For durable memory, add one repository starter, for example JDBC (other stores are in the table below):

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
</dependency>
```

**Memory (chat memory repositories)**

| ID | Backing store |
|---|---|
| `spring-ai-chat-memory-repository-in-memory` | In-memory |
| `spring-ai-chat-memory-repository-jdbc` | JDBC (any relational DB) |
| `spring-ai-chat-memory-repository-cassandra` | Cassandra |
| `spring-ai-chat-memory-repository-mongodb` | MongoDB |
| `spring-ai-chat-memory-repository-neo4j` | Neo4j |
| `spring-ai-chat-memory-repository-redis` | Redis |

> In 2.1.0-RC1, `ChatMemory`, `MessageWindowChatMemory`, `MessageChatMemoryAdvisor`, `ChatMemoryRepository`, and these repositories are deprecated for removal in 3.0.0. Session memory (see the preview below) replaces them.

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

> `chatMemory` is the `ChatMemory` bean that Spring AI auto-configures (in-memory repository and a 20-message window by default; another repository on the classpath, such as JDBC, replaces the in-memory one). The conversation ID is required on every call; a call without it throws `IllegalArgumentException`.

> Both requests use the same conversation ID, so the follow-up can recall `A123`. This uses the 2.0 memory API, which is deprecated for removal in 3.0.0; the preview section introduces session memory.

### 5. Modular RAG

**Dependencies**: a document reader and a vector store, plus an embedding model (the chat starter from step 1 usually provides one). The RAG classes themselves ship with the chat starters. Swap in other readers, stores, or embedding models from the tables below.

```xml
<!-- PDF reader (or spring-ai-tika-document-reader for Office and HTML) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pdf-document-reader</artifactId>
</dependency>

<!-- Vector store: PostgreSQL with PGvector -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>
```

There is no separate "Modular RAG" dependency on Initializr. The RAG pieces (`RetrievalAugmentationAdvisor`, query transformers, document retrievers) ship in Spring AI core and the chat client modules (`spring-ai-rag`), so any model starter brings them in.

[Modular RAG Video](https://www.youtube.com/watch?v=yQQEnXRMvUA&t=1390s)

**Document readers (ingestion / ETL)**

| ID | Purpose |
|---|---|
| `spring-ai-pdf-document-reader` | PDF text via PDFBox |
| `spring-ai-tika-document-reader` | PDF, Office and HTML via Apache Tika |
| `spring-ai-markdown-document-reader` | Markdown files |
| `spring-ai-jsoup-document-reader` | HTML via JSoup |

**Vector stores**

| ID | Store |
|---|---|
| `spring-ai-vectordb-pgvector` | PostgreSQL with PGvector |
| `spring-ai-vectordb-chroma` | Chroma |
| `spring-ai-vectordb-qdrant` | Qdrant |
| `spring-ai-vectordb-milvus` | Milvus |
| `spring-ai-vectordb-weaviate` | Weaviate |
| `spring-ai-vectordb-pinecone` | Pinecone |
| `spring-ai-vectordb-redis` | Redis Search and Query |
| `spring-ai-vectordb-elasticsearch` | Elasticsearch |
| `spring-ai-vectordb-opensearch` | OpenSearch |
| `spring-ai-vectordb-aws-opensearch` | AWS OpenSearch |
| `spring-ai-vectordb-mongodb-atlas` | MongoDB Atlas |
| `spring-ai-vectordb-neo4j` | Neo4j |
| `spring-ai-vectordb-cassandra` | Cassandra |
| `spring-ai-vectordb-couchbase` | Couchbase |
| `spring-ai-vectordb-oracle` | Oracle 23ai |
| `spring-ai-vectordb-mariadb` | MariaDB |
| `spring-ai-vectordb-azure` | Azure AI Search |
| `spring-ai-vectordb-gemfire` | GemFire |
| `spring-ai-vectordb-typesense` | Typesense |
| `spring-ai-vectordb-s3` | AWS S3 |
| `spring-ai-vectordb-bedrock-knowledgebase` | Amazon Bedrock Knowledge Base (managed RAG) |

**Embedding models**

| ID | Provider |
|---|---|
| `spring-ai-transformers` | Local ONNX embeddings |
| `spring-ai-vertexai-embeddings` | Vertex AI |
| `spring-ai-google-genai-embedding` | Google GenAI |
| `spring-ai-bedrock` | Bedrock (Cohere, Titan) |
| `spring-ai-postgresml` | PostgresML |

The chat models (`spring-ai-openai`, `spring-ai-ollama`, `spring-ai-mistral` and others) also provide embeddings.

**1. Load documents (ingestion)**

```java
var docs = new PagePdfDocumentReader("classpath:/docs/handbook.pdf").get();
var chunks = new TokenTextSplitter().apply(docs);
vectorStore.add(chunks);
```

**2. Configure retrieval**

```java
@Configuration
class RagConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder, VectorStore vectorStore) {

        var retriever = VectorStoreDocumentRetriever.builder()
                .vectorStore(vectorStore)
                .similarityThreshold(0.50)
                .topK(4)
                .build();

        var ragAdvisor = RetrievalAugmentationAdvisor.builder()
                .documentRetriever(retriever)
                .queryAugmenter(ContextualQueryAugmenter.builder()
                        .allowEmptyContext(true) // answer even if nothing is retrieved
                        .build())
                .build();

        return builder
                .defaultAdvisors(ragAdvisor, new SimpleLoggerAdvisor())
                .build();
    }
}
```

> `allowEmptyContext` defaults to `false`, in which case the model is told not to answer when nothing is retrieved.

**3. Ask a question**

```java
@RestController
class AskController {

    private final ChatClient chatClient;

    AskController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    @GetMapping("/ask")
    String ask(@RequestParam String q) {
        return chatClient.prompt()
                .user(q)
                .call()
                .content();
    }
}
```

**Per-request advisor (instead of default)**

```java
// ragAdvisor is built as in the configuration above
String answer = chatClient.prompt()
        .advisors(ragAdvisor)
        .user(q)
        .call()
        .content();
```

**Optional modular pieces**

Add these to `RetrievalAugmentationAdvisor.builder()` for the "modular" part:

```java
.queryTransformers(
    RewriteQueryTransformer.builder()
        .chatClientBuilder(builder.build().mutate())
        .build())
.queryExpander(
    MultiQueryExpander.builder()
        .chatClientBuilder(builder.build().mutate())
        .numberOfQueries(3)
        .build())
.documentJoiner(new ConcatenationDocumentJoiner())
```

> A low model temperature (for example `0.0`) on the builder gives more deterministic query rewrites.

### 6. Jev: classify and evaluate

**Dependencies**: community libraries use explicit versions, independently of the Spring AI BOM. Jev requires Spring AI **2.0.1+**.

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
```

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

[More Jev examples: retries, guardrails, RAG, and tool selection](docs/spring-ai-2x-reference.md#34-typesafe-jev-structured-judgment-with-java-examples).

### 7. SkillsJars: load instructions from a dependency

**Dependencies** (SkillsJars docs): also explicit versions, outside the BOM.

```xml
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

> The browser skill dependency supplies instructions under `META-INF/skills`.

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

### 8. Spring AI 2.1 preview

> Use the **2.1.0-RC1** Spring AI BOM (available from Maven Central) with the Spring Boot **4.2** RC baseline (Initializr offers `4.2.0-RC1`). Keep the preview setup separate from the stable examples.

**Dependencies**: the JDBC session starter and a database driver; H2 is sufficient for a local example.

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

#### Session memory

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

> The session ID is required on every call, and a session that does not exist yet is created on first use. Compaction keeps turn boundaries intact, and the original events stay in the log.
