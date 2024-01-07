# OPENAI4J [![tests](https://github.com/sigpwned/openai4j/actions/workflows/tests.yml/badge.svg)](https://github.com/sigpwned/openai4j/actions/workflows/tests.yml)

OpenAI4j is an [OpenAI API](https://openai.com/blog/openai-api) client for Java 11 or later. It is generated from the [OpenAI API OpenAPI spec](https://github.com/openai/openai-openapi) using the excellent [OpenAPI Generator](https://github.com/OpenAPITools/openapi-generator), which is why this repo is (mostly) empty except for a POM and OpenAPI spec.

## Goals

* To provide an easy-to-use Java client for the OpenAI API
* To stay in sync with the OpenAI API as changes are made
* To minimize maintenance effort by generating code
* To require a minimum of dependencies

## Non-Goals

* To support the API spec as written, since it requires changes to generate valid Java code
* No dependencies

## Versioning

The versioning of this project followers the versioning of [the OpenAI OpenAPI spec](https://github.com/openai/openai-openapi), with a fourth component of format `yyyymmdd` representing the release date. While this *looks* like [SemVer](https://semver.org/), OpenAI does not seem to manage the versioning of its OpenAPI spec according to semver, so users should test before upgrading.

## Edits from Original File

The OpenAI OpenAPI spec is a poor fit for Java, as written. Therefore, this project makes some systematic changes to the OpenAPI spec before generating code. The spec used for code generation is can be found at `openapi.yml` in this repo.

### Type Adjustments

The OpenAPI spec is clearly intended for dynamically-typed languages. Overall, the OpenAPI Generator does a truly heroic job of generating sane types. However, typing like this is very hard for explicitly-typed languages to handle:

    prompt:
      description: &completions_prompt_description |
        The prompt(s) to generate completions for, encoded as a string, array of strings, array of tokens, or array of token arrays.

        Note that <|endoftext|> is the document separator that the model sees during training, so if a prompt is not specified the model will generate as if from the beginning of a new document.
      default: "<|endoftext|>"
      nullable: true
      oneOf:
        - type: string
          default: ""
          example: "This is a test."
        - type: array
          items:
            type: string
            default: ""
            example: "This is a test."
        - type: array
          minItems: 1
          items:
            type: integer
          example: "[1212, 318, 257, 1332, 13]"
        - type: array
          minItems: 1
          items:
            type: array
            minItems: 1
            items:
              type: integer
          example: "[[1212, 318, 257, 1332, 13]]"

Which type should prompt have?

* `String`
* `String[]`
* `int[]`
* `int[][]`

This is further complicated by the presence of a default value, which is a string.

As a result, this project applies the following typing changes to the original OpenAPI spec:

- For `oneOf` types with a combination of different scalar types or scalar and array types, the string type is preferred.
- For `oneOf` types witha  combination of string type and object or array types, the string type is preferred.
- For `anyOf` types with a string and an enum, the string type is preferred.

The OpenAPI Generator (surprisingly, and laudably) manages the following types well enough that they are left alone:

- `oneOf` types with a choice between multiple object types

## Example Code

### Generate a chat completion

The following code generates a chat completion using the model `gpt-3.5-turbo-1106`:

    public String generateChatResponse(ChatApi chat, String userPrompt) {
        List<ChatCompletionRequestMessage> messages = new ArrayList<>();
        if (getSystemInstructions() != null)
          messages.add(new ChatCompletionRequestMessage(new ChatCompletionRequestSystemMessage()
              .role(ChatCompletionRequestSystemMessage.RoleEnum.SYSTEM)
              .content("You are a helpful assistant.")));
        messages.add(new ChatCompletionRequestMessage(
            new ChatCompletionRequestUserMessage().role(ChatCompletionRequestUserMessage.RoleEnum.USER)
                .content(userPrompt)));
    
        CreateChatCompletionResponse response=chat.createChatCompletion(
              new CreateChatCompletionRequest().model("gpt-3.5-turbo-1106").messages(messages).n(1));
    
        CreateChatCompletionResponseChoicesInner choice = response.getChoices().get(0);
    
        return choice.getMessage().getContent();
    }

