# Actually Benefiting from Structured Output Support with Ollama and LangChainJS

A few weeks ago, I wrote an article explaining how to generate random names for role-playing game characters using an LLM. For my examples, I used Ollama's Go API (because it's simple and I really like the Go language). I also used the principle of "structured outputs" since Ollama announced [support for it last December](https://ollama.com/blog/structured-outputs).

Briefly, "structured outputs" allow you to constrain a model to provide a response in a specific format defined by a JSON schema. And it works very well.

For example, with Ollama's REST API:

```bash
curl -X POST http://localhost:11434/api/chat -H "Content-Type: application/json" -d '{
  "model": "qwen2.5:0.5b",
  "messages": [
    {"role": "system", "content": "You are an expert for games like D&D 5th edition."},
    {"role": "user", "content": "Generate a random name for a Dwarf."}
  ],
  "stream": false,
  "format": {
    "type": "object",
    "properties": {
      "name": {
        "type": "string"
      },
      "kind": {
        "type": "string"
      }
    },
    "required": [
      "name",
      "kind"
    ]
  }
}' | jq '.message.content | fromjson'
```

And I'll get:

```json
{
  "name": "Ebon-Heavenly",
  "kind": "Dwarf"
}
```

And with the Go API, it would look something like this:

```golang
schema := map[string]any{
    "type": "object",
    "properties": map[string]any{
        "name": map[string]any{
            "type": "string",
        },
        "kind": map[string]any{
            "type": "string",
        },
    },
    "required": []string{"name", "kind"},
}

jsonModel, err := json.Marshal(schema)
if err != nil {
    log.Fatalln("😡", err)
}

userContent := "Generate a random name for Dwarf"
noStream := false
req := &api.ChatRequest{
    Model:    model,
    Messages: messages,
    Options: map[string]interface{}{
        "temperature":    1.7,
        "repeat_last_n":  2,
        "repeat_penalty": 2.2,
        "top_k":          10,
        "top_p":          0.9,
    },
    Format: json.RawMessage(jsonModel),
    Stream: &noStream,
}
```

This morning, I wanted to port my example to LangChainJS...

> The blog post in question: [How to Generate Random RPG Character Names with an LLM](https://k33g.hashnode.dev/how-to-generate-random-rpg-character-names-with-an-llm)

## Using Structured Outputs with LangChainJS

So I searched through the LangChainJS documentation (which I tend to get lost in). I found a guide: [How to return structured data from a model](https://js.langchain.com/docs/how_to/structured_output) which explains that there's a `withStructuredOutput()` method implemented for `BaseChatModel` type objects. But I only found examples for: OpenAI, Anthropic, MistralAI, Groq, VertexAI.

Searching a bit more, I finally found an [example for Ollama](https://js.langchain.com/docs/integrations/chat/ollama/#withstructuredoutput), but which oddly seems to use "tools" support (formerly "function calling") to achieve its goals. I still tried to reproduce my example based on the documentation:

### Structured Output with `withStructuredOutput()`

Here's my first version:

```javascript
import { ChatOllama } from "@langchain/ollama"
import { z } from "zod"

const llm = new ChatOllama({
    model: 'qwen2.5:0.5b',
    baseUrl: "http://ollama-service:11434",
    temperature: 0.0,
    repeatLastN: 2,
    repeatPenalty: 2.2, 
    topK: 10,
    topP: 0.9,    
})

const schema = z.object({
    name: z.string().describe("The name of the NPC"),
    kind: z.string().describe("The kind of NPC"),
})

const structuredLLM = llm.withStructuredOutput(schema, {
    name: "npc_name",
})

var systemInstructions = `
You are an expert for games like D&D 5th edition. 
`
let kind = "Dwarf"

let userContent = `Generate a random name for a ${kind} (kind always equals ${kind} and always add a name). 
Ensure you use the 'npc_name' tool.`

var messages = [
    ["system", systemInstructions],
    ["user", userContent]
 ]

const response = await structuredLLM.invoke(messages, {name: "npcname"})
console.log("Response:", response)
```

And it works:

```bash
Response: { kind: 'Dwarf', name: 'Gandalf' }
```

But what bothers me a bit is that I have to give it a "tool" name and make a somewhat convoluted prompt to ensure I get the right format. Whereas theoretically, the JSON schema should be sufficient to constrain the model.

It seems that this part isn't yet implemented in LangChainJS (I'm using: `"langchain": "^0.3.15"` and `"@langchain/ollama": "^0.1.5"`), and that the workaround used is "tools". Well, okay, why not, as long as it does the job.

However, my program generates almost the same name every time. Well, that's normal, I'm using `temperature: 0.0` as a parameter for the LLM, and normally for using "tools" I don't have much choice.

Since my goal is to generate random names, I still try to increase the temperature: `temperature: 1.5`. And then, disaster strikes:

```bash
throw new Error("No tool calls found in the response.")
```

But since I plan to use LangChainJS more often, I wasn't going to let this stop me (though I was a bit disappointed).

### Real Structured Output

I took a quick look at the source code corresponding to Ollama. And I was able to verify that the `ChatOllama` class implements `ChatOllamaInput` which has a `format` property (like with Ollama's REST and Go API). So in principle, I was saved.

So I modified (and simplified) my code as follows to verify my hypothesis and used a temperature above zero:

```javascript
import { ChatOllama } from "@langchain/ollama"

let jsonSchema = {
    type: "object",
    properties: {
        name: {
            type: "string"
        },
        kind: {
            type: "string"
        },
    },
    required: ["name", "kind"]
}

const llm = new ChatOllama({
    model: 'qwen2.5:0.5b',
    baseUrl: "http://ollama-service:11434",
    temperature: 1.5,
    repeatLastN: 2,
    repeatPenalty: 2.2,
    topK: 10, 
    topP: 0.9, 
    format: jsonSchema
})

var systemInstructions = `You are an expert for games like D&D 5th edition.`

let kind = "Dwarf"

let userContent = `Generate a random name for a ${kind} (kind always equals ${kind}).`

var messages = [
    ["system", systemInstructions],
    ["user", userContent]
 ]

const response = await llm.invoke(messages)
console.log("Response:", JSON.parse(response.content))
```

And this time everything works perfectly, without errors and with more random names:

```bash
Response: { name: 'Dawnwind Dwarven Knight', kind: 'Dwarf' }
Response: { name: 'Eldritch Dwarves', kind: 'Dwarf' }
Response: { name: 'Dwarven Hunter', kind: 'Dwarf' }
```

By modifying the instructions, you can improve the generation even more. But today's topic was understanding how to use the "structured outputs" functionality with Ollama and LangChainJS.

See you soon for another blog post.
