---
layout: post

title: 如何使用Java Spring Boot来调用OpenAI的ChatGPT API

data: 2023-04-12

categories: [tech]
---

让我们聊聊需要什么才能从 Java Spring Boot 微服务 Web App 调用 ChatGPT API 。
我们需要做的就是创建一个 Controller，创建一个 Service，几个 POJO，然后就完成了。最多只需要三十分钟就能大功告成。
这是代码，Rest Controller 调用 Service，Service 调用 Open AI API，返回第一个结果。

```java
// Controller 调用 Service

@RestController
@Slf4j
@RequestMapping("/api/v1")
public class ChatGPTRestController {
    private ChatGPTService chatGPTService;
    public ChatGPTRestController(ChatGPTService chatGPTService) {
        this.chatGPTService = chatGPTService;
    }
    @PostMapping("/searchChatGPT")
    public String searchChatGPT(@RequestBody SearchRequest searchRequest) {
        log.info("searchChatGPT Started query: " + searchRequest.getQuery());
        return chatGPTService.processSearch(searchRequest.getQuery());
    }
}
```

```java
// 在 Service 里面调用 OpenAI API
@Service
@Slf4j
public class ChatGPTService {
    @Value("${OPEN_AI_URL}") // 属性写在application.properties里面
    private String OPEN_AI_URL;
    @Value("${OPEN_AI_KEY}")
    private String OPEN_AI_KEY;

    public String processSearch(String query) {
        ChatGPTRequest chatGPTRequest = new ChatGPTRequest();
        chatGPTRequest.setPrompt(query);
        String url = OPEN_AI_URL;

        HttpPost post = new HttpPost(url);
        post.addHeader("Content-Type", "application/json");
        post.addHeader("Authorization", "Bearer " + OPEN_AI_KEY);

        Gson gson = new Gson();
        String body = gson.toJson(chatGPTRequest);
        log.info("body: " + body);

        try {
            final StringEntity entity = new StringEntity(body);
            post.setEntity(entity);
            try (CloseableHttpClient httpClient = HttpClients.custom().build();
                 CloseableHttpResponse response = httpClient.execute(post)) {
                String responseBody = EntityUtils.toString(response.getEntity());
                log.info("responseBody: " + responseBody);
                ChatGPTResponse chatGPTResponse = gson.fromJson(responseBody, ChatGPTResponse.class);
                return chatGPTResponse.getChoices().get(0).getText();
            } catch (Exception e) {
                return "failed";
            }
        }
        catch (Exception e) {
            return "failed";
        }
    }
}
```

开始写 POJO class， 总共四个。

```java
import lombok.Data;

@Data
public class SearchRequest {
    private String query;
}
```

```java
@Data
public class ChatGPTRequest {

    private String model = "text-davinci-003";
    private String prompt;
    private int temperature = 1;

    @SerializedName(value="max_tokens")
    private int maxTokens = 100;
}
```

```java
@Data
public class ChatGPTResponse {
    private List<ChatGptChoice> choices;
}
```

```java
@Data
public class ChatGptChoice {
    private String text;
}
```

## 工作流程

可以从 Postman 发送一个 request，对 /api/v1/searchChatGPT 进行查询

```
{
    "query": "如何学习Java?"
}
```

ChatGPTRestController 使用方法 searchChatGPT 来调用由 Spring 注入到 Constructor 中的 ChatGPTService 对象。默认情况下这是一个单例。它在查询中调用 ChatGPTService 方法 processSearch 发送，然后简单地将查询打包到 ChatGPT 所需的请求中并处理可以包含多个“选择”的响应，这仅使用第一个选择并将其返回给服务。带有 CloseableHttpClient 的 Try Block 确保关闭与 Open AI API 调用的连接。其中 URL 和 OPEN API KEY 存储在 applicaiton.properties 文件中。

实际发送给 OpenAI API 的请求如下

```
{
  "model": "text-davinci-003",
  "prompt": "?",
  "temperature": 1,
  "max_tokens": 100
}
```

而 OpenAI 的响应如下：

```
{
    "id": "cmpl-37ydIWdG",
    "object": "text_completion",
    "created": 1681406508,
    "model": "text-davinci-001",
    "choices": [
        {
            "text": "\n\n1.了解Java的基础知识：首先，学习者需要了解Java的基本知识，包括Java的历史、特性、基本概念、编译和执行过程等。\n\n2.学习Java的语言特性：其次，学习者需要学习Java的语言特性，包括变量和数据类型、运算符、流程控制、面向对象编程等。\n\n3.学习Java的核心类库：.",
            "index": 0,
            "logprobs": null,
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": 9,
        "completion_tokens": 59,
        "total_tokens": 79
    }
}
```

现在你应该学会了如何用 Spring boot 来调用 ChatGPT 的 API 了吧。
