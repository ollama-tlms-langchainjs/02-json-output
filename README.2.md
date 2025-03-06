# "Structured Output" with Ollama and LangChainJS - The LangChainJS Team is Fantastic
> My previous article is already obsolete

In my previous article [Actually Benefiting from Structured Output Support with Ollama and LangChainJS](https://k33g.hashnode.dev/actually-benefiting-from-structured-output-support-with-ollama-and-langchainjs), I explained that the `withStructuredOutput` method wasn't working when using the latest Ollama API version, and consequently, I wasn't getting exactly the expected results. However, I proposed another method to achieve this (I'll let you read the article, which aimed to generate random names for role-playing game characters).

I was lucky that [Jacob Lee](https://x.com/Hacubu), a key maintainer of @LangChain JS/TS, came across my article:

![img](/04-generate-npc/imgs/tweet.png)

So naturally, I had to test it 🥰

I updated my dependencies in the `package.json` file:
```json
  "dependencies": {
    "@langchain/ollama": "^0.1.6",
    "dotenv": "^16.4.7",
    "langchain": "^0.3.15",
    "prompts": "^2.4.2",
    "zod": "^3.24.1"
  }
```

## New Code Version

And so I modified my source code as follows:

```javascript
import { ChatOllama } from "@langchain/ollama"
import { z } from "zod"

const llm = new ChatOllama({
    model: 'qwen2.5:0.5b',
    baseUrl: "http://ollama-service:11434",
    temperature: 1.5, 
    repeatLastN: 2,
    repeatPenalty: 2.2,
})

const schema = z.object({
    name: z.string().describe("The name of the NPC"),
    kind: z.string().describe("The kind of NPC"),
})

const structuredLLM = llm.withStructuredOutput(schema, {
    method: "jsonSchema",
})

var systemInstructions = `
You are an expert for games like D&D 5th edition. 
You have freedom to be creative to get the best possible output.
`

let kind = "Dwarf"

let userContent = `Generate a random name for a ${kind}.`

var messages = [
    ["system", systemInstructions],
    ["user", userContent]
 ]

const response = await structuredLLM.invoke(messages)
console.log("Response:", response)
console.log("")
```

And everything works perfectly with the ability to get random names:

```bash
Response: { name: 'Elthos Caine', kind: 'dwarf' }
Response: { name: 'Nedryllia', kind: 'Dwarf' }
Response: { name: 'Silent Fleece', kind: 'Dwarf' }
...
```

A big thank you for this [Jacob Lee](https://x.com/Hacubu) 🎉🙏