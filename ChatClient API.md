Link para a página da DOC: https://docs.spring.io/spring-ai/reference/api/chatclient.html

- Suporta `Chat`, `Text to Image`, `Audio Transcription`, `Text to Speech`, e `Embedding`.

### RAG

- Permite o uso do ETL Data Engineering para uso de RAG.

# Chat Client API

- Usado para se comunicar com o modelo.
- `Prompt` é o prompt enviado para o modelo.
- Existem 2 tipos de mensagem, as enviadas pelo usuário e as enviadas pelo sistema, estas que são usadas para guiar a conversa.
- As mensagens podem conter placeholders que podem ser substituídos por instruções personalizadas a partir do input do usuário para ter respostas personalizadas.
- É possível definir parâmetros como temperatura e etc.

## Criando um ChatClient

- O objeto é `ChatClient.Builder`.
- É possível obter uma configuração automática do objeto.

### Usando um ChatClient auto configurado

- Exemplo:

```
@RestController
class MyController {

    private final ChatClient chatClient;

    public MyController(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }

    @GetMapping("/ai")
    String generation(String userInput) {
        return this.chatClient.prompt()
            .user(userInput)
            .call()
            .content();
    }
}
```

### Crie um ChatClient programaticamente

- Para desativar a configuração automática, você precisa settar a seguinte propriedade: `spring.ai.chat.client.enabled=false`.
- É bom usar essa configuração quando existe a utilização de mais de um modelo.
- Exemplo:

```
ChatModel myChatModel = ... // usually autowired

ChatClient.Builder builder = ChatClient.builder(myChatModel);

// or create a ChatClient with the default builder settings:

ChatClient chatClient = ChatClient.create(myChatModel);
```

## Respostas do ChatClient

### Retornando uma ChatResponse

- Exemplo de response:

```
public class ChatResponse implements ModelResponse<Generation> {

    private final ChatResponseMetadata chatResponseMetadata;
	private final List<Generation> generations;

	@Override
	public ChatResponseMetadata getMetadata() {...}

    @Override
	public List<Generation> getResults() {...}

    // other methods omitted
}
```

- Exemplo de retorno:

```
ChatResponse chatResponse = chatClient.prompt()
    .user("Tell me a joke")
    .call()
    .chatResponse();
```

### Retornando uma entity (DOC)

You often want to return an entity class that is mapped from the returned `String`. The `entity` method provides this functionality.

For example, given the Java record:

```java
record ActorFilms(String actor, List<String> movies) {
}
```

You can easily map the AI model’s output to this record using the `entity` method, as shown below:

```java
ActorFilms actorFilms = chatClient.prompt()
    .user("Generate the filmography for a random actor.")
    .call()
    .entity(ActorFilms.class);
```

There is also an overloaded `entity` method with the signature `entity(ParameterizedTypeReference<T> type)` that lets you specify types such as generic Lists:

```java
List<ActorFilms> actorFilms = chatClient.prompt()
    .user("Generate the filmography of 5 movies for Tom Hanks and Bill Murray.")
    .call()
    .entity(new ParameterizedTypeReference<List<ActorFilms>>() {
    });
```

### Resposta via stream

- `Stream` permite receber respostas assíncronas. 
- Exemplo de response assíncrona:

```
Flux<String> output = chatClient.prompt()
    .user("Tell me a joke")
    .stream()
    .content();
```

- You can also stream the `ChatResponse` using the method `Flux<ChatResponse> chatResponse()`. (DOC)

##### DOC


You can also stream the `ChatResponse` using the method `Flux<ChatResponse> chatResponse()`.

In the 1.0.0 M2 we will offer a convenience method that will let you return an Java entity with the reactive `stream()` method. In the meantime, you should use the [Structured Output Converter](https://docs.spring.io/spring-ai/reference/api/structured-output-converter.html#StructuredOutputConverter) to convert the aggregated response explicity as shown below. This also demonstrates the use of parameters in the fluent API that will be discussed in more detail in a later section of the documentation.

```java
    var converter = new BeanOutputConverter<>(new ParameterizedTypeReference<List<ActorsFilms>>() {
    });

    Flux<String> flux = this.chatClient.prompt()
        .user(u -> u.text("""
                            Generate the filmography for a random actor.
                            {format}
                          """)
                .param("format", converter.getFormat()))
        .stream()
        .content();

    String content = flux.collectList().block().stream().collect(Collectors.joining());

    List<ActorFilms> actorFilms = converter.convert(content);
```

## Valores de retorno do call()

- Pode ser retornado `String content()`, `ChatResponse chatResponse()` e `entity`.

## Valores de retorno do stream()

- Pode ser retornado `Flux<String> content()` e `Flux<ChatResponse> chatResponse()`.

# Usando os defaults

## Default System Text

- Classe de config:

```
@Configuration
class Config {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder) {
        return builder.defaultSystem("You are a friendly chat bot that answers question in the voice of a Pirate")
                .build();
    }

}
```

- Chamando na REST:

```
@RestController
class AIController {

	private final ChatClient chatClient;

	AIController(ChatClient chatClient) {
		this.chatClient = chatClient;
	}

	@GetMapping("/ai/simple")
	public Map<String, String> completion(@RequestParam(value = "message", defaultValue = "Tell me a joke") String message) {
		return Map.of("completion", chatClient.prompt().user(message).call().content());
	}
}
```

- Curl:

```
❯ curl localhost:8080/ai/simple
{"generation":"Why did the pirate go to the comedy club? To hear some arrr-rated jokes! Arrr, matey!"}
```

## Default System Text com parâmetros

- Classe de config:

```
@Configuration
class Config {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder) {
        return builder.defaultSystem("You are a friendly chat bot that answers question in the voice of a {voice}")
                .build();
    }

}
```

- Chamando na REST:

```
@RestController
class AIController {
	private final ChatClient chatClient
	AIController(ChatClient chatClient) {
		this.chatClient = chatClient;
	}
	@GetMapping("/ai")
	Map<String, String> completion(@RequestParam(value = "message", defaultValue = "Tell me a joke") String message, String voice) {
		return Map.of(
				"completion",
				chatClient.prompt()
						.system(sp -> sp.param("voice", voice))
						.user(message)
						.call()
						.content());
	}
}
```

- Resposta:

```
http localhost:8080/ai voice=='Robert DeNiro'
{
    "completion": "You talkin' to me? Okay, here's a joke for ya: Why couldn't the bicycle stand up by itself? Because it was two tired! Classic, right?"
}
```

### Outros defaults (DOC)

At the `ChatClient.Builder` level, you can specify the default prompt.

- `defaultOptions(ChatOptions chatOptions)`: Pass in either portable options defined in the `ChatOptions` class or model-specific options such as those in `OpenAiChatOptions`. For more information on model-specific `ChatOptions` implementations, refer to the JavaDocs.
    
- `defaultFunction(String name, String description, java.util.function.Function<I, O> function)`: The `name` is used to refer to the function in user text. The `description` explains the function’s purpose and helps the AI model choose the correct function for an accurate response. The `function` argument is a Java function instance that the model will execute when necessary.
    
- `defaultFunctions(String…​ functionNames)`: The bean names of `java.util.Function`s defined in the application context.
    
- `defaultUser(String text)`, `defaultUser(Resource text)`, `defaultUser(Consumer<UserSpec> userSpecConsumer)`: These methods let you define the user text. The `Consumer<UserSpec>` allows you to use a lambda to specify the user text and any default parameters.
    
- `defaultAdvisors(RequestResponseAdvisor…​ advisor)`: Advisors allow modification of the data used to create the `Prompt`. The `QuestionAnswerAdvisor` implementation enables the pattern of `Retrieval Augmented Generation` by appending the prompt with context information related to the user text.
    
- `defaultAdvisors(Consumer<AdvisorSpec> advisorSpecConsumer)`: This method allows you to define a `Consumer` to configure multiple advisors using the `AdvisorSpec`. Advisors can modify the data used to create the final `Prompt`. The `Consumer<AdvisorSpec>` lets you specify a lambda to add advisors, such as `QuestionAnswerAdvisor`, which supports `Retrieval Augmented Generation` by appending the prompt with relevant context information based on the user text.
    

You can override these defaults at runtime using the corresponding methods without the `default` prefix.

- `options(ChatOptions chatOptions)`
    
- `function(String name, String description, java.util.function.Function<I, O> function)`
    
- `functions(String…​ functionNames)
    
- `user(String text)` , `user(Resource text)`, `user(Consumer<UserSpec> userSpecConsumer)`
    
- `advisors(RequestResponseAdvisor…​ advisor)`
    
- `advisors(Consumer<AdvisorSpec> advisorSpecConsumer)`

# Advisors

