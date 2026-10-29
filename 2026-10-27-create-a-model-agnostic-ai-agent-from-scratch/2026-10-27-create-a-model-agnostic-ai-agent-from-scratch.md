```ic-metadata
{
  "name": "Create a Model Agnostic AI Agent from Scratch",
  "series": null,
  "date": "2026-10-27",
  "lastModifiedDate": "2026-10-27",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "ai", "agent"],
  "canonicalLink": ""
}
```

# Create a Model Agnostic AI Agent from Scratch

Today we will be investigating how to create a very simple AI agent that can be ram locally and which would not be tighly coupled to a specific AI Provider. Even though there is a plethora of providers, the underlying basic principles for agents are rather similar, and, in future, will probably boil to the same interface.

So what is an AI agent in actuality? While daily we read the news how this agent broke from a sandbox and that agent hacked into that, the agents that we get are disappointedly more primitive in reality. Maybe we do not get the good ones that can do that sort of feats yet. An agent could be described as a small program that given user's input communicates with an LLM model, call some functions and produces an output. In some way an agent is a facade to a pile of functions the LLM will decide how to call: your input goes into agent, then into LLM (actually not only your input, but context, tools, system prompt and a bunch of other stuff, leaving your input sometimes 5% of the whole pile of things thrown into stateless LLM), then outputs/inputs bounce between LLM and agent until a limit is reached or output satisfies your input and you get it back.

The LLM model itself does not call or invoke anything.

Here is how a simplified input for LLM could look like:

```json
[
// here goes your make no mistake system input, hidden from regular user
  { role: "system", content: "You are a helpful assistant. Use the tools when they help answer." },
  // 
  { role: "user",   content: "What time is it in Calgary? Then roll 2 dice." },
]
// + tools: [ ...the menu... ]
```

A reply from LLM could look something like this:
```json
{ role: "assistant", content: "", tool_calls: [
  { id: "call_1", type: "function", function: { name: "get_time",  arguments: "{\"timezone\":\"America/Calgary\"}" } },
  { id: "call_2", type: "function", function: { name: "roll_dice", arguments: "{\"count\":2}" } },
]}
```

Come to think of it, at this point LLM could have written its own function which an agent would eval, and nothing stops you from doing so and roll with punches.

A reply from the calls then sent back to the LLM, i.e.

```json
{ role: "tool", tool_call_id: "call_1", content: "Sunday, 2026-10-25, 05:06" },
{ role: "tool", tool_call_id: "call_2", content: "rolls=[5, 5] total=10" },
```

The next round produces the output the human consumer can actually digest and the call finishes.

The general concept how the flow goes is the same between different LLM provides, the only difference is the shame in which they would be tossing around tools list, system prompt, tool request, results and errors. Which means we could abstract it.

In this arcitle we will be taking a look how to write an abstract AI agent and a concrete implementation that works with a local LLM (we will be using ollama).

## Project Setup

The tools we will be using are just `node` and `npm`. It can be written in pythong or something else, but these more faimiar to me, so I will be using them.

Once the basic node project is created with npm, let us adjust the `package.json` in a following manner:
```json
{
  "name": "tryout_agnostic-agent",
  "version": "1.0.0",
  "description": "Learning how to build an agnostic agent",
  "exports": "./src/index.js",
  "bin": {
    "agent": "bin/agent.js"
  },
  "scripts": {
    "agent": "node bin/agent.js",
    "test": "node --test",
    "typecheck": "tsc",
    "check": "tsc && node --test"
  },
  "type": "module",
  "private": true,
  "engines": {
    "node": ">=22"
  },
  "devDependencies": {
    "@types/node": "^26.6.5",
    "typescript": "^7.0.2"
  },
  "dependencies": {
    "openai": "^7.43.0"
  }
}
```

What is notable here is `bin` which would allow using the agent elsewhere if `link` is used (though we will not be utilizing it) and `exports`, which serves as a single public entry point.

Now we are ready to install our dependencies:
```bash
npm install --save-dev typescript @types/node
```

At this point you might be wondering why typescript is here, if I did not mention it earlier. It is merely for convenience of having types, in fact, we will not have typescript files at all, we will do with `js` and `JSDoc`.

To get typescript working for our js files, we create `tsconfig.json` with the following content (there will be no compilation steps for us):

```json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": true,
    "noEmit": true,
    "strict": true,
    "target": "es2023",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "types": ["node"],
    "skipLibCheck": true
  },
  "include": ["src/**/*.js", "bin/**/*.js", "test/**/*.js"]
}
```

## The Core (or Types)

Let us go on and create some folders
```bash
mkdir -p src/core src/tools src/providers bin
```

The `core` is merely for holding the abstraction and types. The `providers` will hold concrete implementations and `tools` will just contain functions the agent can utilize.

Let us start with the types for the core.

`src/core/types.js`
```js
// @ts-check
/**
 * The provider-neutral domain model every layer shares.
 */

/**
 * A function and the description, which a model would be invoking.
 * @typedef {object} Tool
 * @property {string} name
 * @property {string} description
 * @property {Record<string, unknown>} parameters
 * @property {(args: any) => string | Promise<string>} run
 */

/**
 * The requesting a tool to be run.
 * @typedef {object} ToolCall
 * @property {string} id
 * @property {string} name
 * @property {Record<string, unknown>} argument
 */

/**
 * The outcome of running one ToolCall to be sent back to the model.
 * @typedef {object} ToolResult
 * @property {string} callId
 * @property {string} content
 * @property {boolean} isError
 */

/**
 * A neutral reply from the model.
 * @typedef {object} Turn
 * @property {string} text
 * @property {ToolCall[]} toolCalls
 */

/**
 * @typedef {object} AgentOptions
 * @property {string} model
 * @property {Tool[]} tools
 * @property {string} [system]
 * @property {number} [maxTurns]
 */

/**
 * Called after each tool runs, e.g. to log progress.
 * @typedef {(call: ToolCall, result: ToolResult) => void} ToolCallListener
 */

export {};
```

These are abstractions that are applicable to any interaction of agent with the model, no matter the model implementation. Let us now proceed with the abstract agent itself. The idea behind it is that any agent will be extending it and this class itself will never be instantiated directly.

`src/core/agent.js`

```javascript
// @ts-check
/**
 * The abstract Agent: the agent loop, written once for every provider.
 */

/**
 * @typedef {import("./types.js").AgentOptions} AgentOptions
 * @typedef {import("./types.js").ToolCall} ToolCall
 * @typedef {import("./types.js").ToolCallListener} ToolCallListener
 * @typedef {import("./types.js").ToolResult} ToolResult
 * @typedef {import("./types.js").Turn} Turn
 */

/** @abstract */
export class Agent {
  /** @param {AgentOptions} options */
  constructor({ model, tools, system = "", maxTurns = 10 }) {
    if (new.target === Agent) {
      throw new TypeError("Agent is abstract; use a provider subclass.");
    }
    this.model = model;
    this.tools = new Map(tools.map((tool) => [tool.name, tool]));
    this.system = system;
    this.maxTurns = maxTurns;
    /**
     * Conversation history in the provider's own format. Only the subclass
     * reads or writes the entries.
     * @type {unknown[]}
     */
    this.messages = [];
  }

  /**
   * Append a user message to this.messages in the provider's format.
   * @abstract
   * @param {string} text
   * @returns {void}
   */
  addUserMessage(text) {
    throw new Error(`${this.constructor.name} must implement addUserMessage()`);
  }

  /**
   * Send this.messages to the model, append its reply, return it as a Turn.
   * @abstract
   * @returns {Promise<Turn>}
   */
  async callModel() {
    throw new Error(`${this.constructor.name} must implement callModel()`);
  }

  /**
   * Append the tool results to this.messages in the provider's format.
   * @abstract
   * @param {ToolResult[]} results
   * @returns {void}
   */
  addToolResults(results) {
    throw new Error(`${this.constructor.name} must implement addToolResults()`);
  }

  // --- The loop, shared by every provider ---------------------------------

  /**
   * User input => the model uses tools as needed => answer.
   * @param {string} question
   * @param {ToolCallListener} [onToolCall]
   * @returns {Promise<string>}
   */
  async run(question, onToolCall) {
    this.addUserMessage(question);
    for (let i = 0; i < this.maxTurns; i++) {
      const turn = await this.callModel();
      if (turn.toolCalls.length === 0) {
        return turn.text;
      }
      const results = await Promise.all(turn.toolCalls.map((call) => this.#execute(call)));
      turn.toolCalls.forEach((call, j) => onToolCall?.(call, results[j]));
      this.addToolResults(results);
    }
    return `(stopped after ${this.maxTurns} turns without a final answer)`;
  }

  /**
   * @param {ToolCall} call
   * @returns {Promise<ToolResult>}
   */
  async #execute(call) {
    const tool = this.tools.get(call.name);
    if (!tool) {
      return { callId: call.id, content: `Error: no tool named '${call.name}'.`, isError: true };
    }
    try {
      return { callId: call.id, content: String(await tool.run(call.arguments)), isError: false };
    } catch (e) {
      const message = e instanceof Error ? e.message : String(e);
      return { callId: call.id, content: `Error: ${message}`, isError: true };
    }
  }
}
```

Once again, no matter on which flavour of LLM we rely on, the general principle stays the same: messages goes from user, then bounces between LLM and agent tooling until shaped into an output reply. While `messages` is typed as `unknown` it would be beneficial to make it a generic, so every concrete implementation would be setting the type appropriately when extending the class. I would leave it up to the reader to fix this minor flaw, but it does not really prevent us from proceeding.

Now, to `core` a proper module (to the extent js allows it), let us add a barrel file for exporting:

`src/core/index.js`
```javascript
// @ts-check

export { Agent } from "./agent.js";

/**
 * @typedef {import("./types.js").AgentOptions} AgentOptions
 * @typedef {import("./types.js").Tool} Tool
 * @typedef {import("./types.js").ToolCall} ToolCall
 * @typedef {import("./types.js").ToolCallListener} ToolCallListener
 * @typedef {import("./types.js").ToolResult} ToolResult
 * @typedef {import("./types.js").Turn} Turn
 */
```

We do this to prevent deep linking into the `core` folder, as a good practice. This file defines what the `core` folder lets other consumers see. Although there is nothing preventing you from deep linking if you choose to disrespect the approach.

## The Tools for an Agent

Let us create a couple of plain js functions, that will serve as tools for our future ai agents. One for checking the time and the other on for rolling the dice.

`src/tools/get-time.js`:
```javascript
// @ts-check
/** The current local time in any timezone. */

/** @typedef {import("../core/index.js").Tool} Tool */

/**
 * Return the current local time in an IANA timezone like "America/Vancouver".
 * @param {{ timezone?: string }} args
 * @returns {string}
 */
export function getTime({ timezone }) {
  try {
    return new Intl.DateTimeFormat("en-CA", {
      timeZone: timezone,
      weekday: "long",
      year: "numeric",
      month: "2-digit",
      day: "2-digit",
      hour: "2-digit",
      minute: "2-digit",
      hourCycle: "h23",
    }).format(new Date());
  } catch {
    throw new Error(`Unknown timezone '${timezone}'. Use a name like 'America/Vancouver'.`);
  }
}

/** @type {Tool} */
export const getTimeTool = {
  name: "get_time",
  description: "Get the current local date and time in a timezone.",
  parameters: {
    type: "object",
    properties: {
      timezone: {
        type: "string",
        description: "IANA timezone name, e.g. 'America/Vancouver' or 'Europe/London'.",
      },
    },
    required: ["timezone"],
  },
  run: getTime,
};
```
`src/tools/roll-dice.js`:

```javascript
// @ts-check
/** Tool: roll one or more dice. */

/** @typedef {import("../core/index.js").Tool} Tool */

/**
 * Roll `count` dice with `sides` sides each and return the results.
 * @param {{ sides?: number, count?: number }} args
 * @returns {string}
 */
export function rollDice({ sides = 6, count = 1 }) {
  if (!(sides >= 2 && sides <= 1000 && count >= 1 && count <= 100)) {
    throw new Error("sides must be 2-1000 and count must be 1-100.");
  }
  const rolls = Array.from({ length: count }, () => 1 + Math.floor(Math.random() * sides));
  return `rolls=[${rolls.join(", ")}] total=${rolls.reduce((a, b) => a + b, 0)}`;
}

/** @type {Tool} */
export const rollDiceTool = {
  name: "roll_dice",
  description: "Roll one or more dice and return each roll and the total.",
  parameters: {
    type: "object",
    properties: {
      sides: { type: "integer", description: "Sides per die (default 6)." },
      count: { type: "integer", description: "How many dice (default 1)." },
    },
  },
  run: rollDice,
};
```

At this point you might be scratching your head about each file containing actually two exports instead of one, i.e. why `rollDiceTool` and `rollDice`? The split of the actual function and tool allows us to separate concerns. Besides, `rollDice` and `getTime` are now just pure functions that can be covered with unit tests if we wanted (we will not be doing it as part of this tutorial).

For the tools folder we apply the same module principle and add a barrel file for export.

`src/tools/index.js`
```javascript
// @ts-check

import { getTimeTool } from "./get-time.js";
import { rollDiceTool } from "./roll-dice.js";

export { getTime, getTimeTool } from "./get-time.js";
export { rollDice, rollDiceTool } from "./roll-dice.js";

/** @type {import("../core/index.js").Tool[]} */
export const TOOLS = [getTimeTool, rollDiceTool];
```

This is our front door and `TOOLS` constant is he list the agen gets.

## First Provider, a Mere Mock

You might be wondering what even for, but it is a good thing to have for a quick test or as a proof of concept.

`src/providers/mock.js`:
```javascript
// @ts-check
/**
 * Fake it till you make it
 */

import { Agent } from "../core/index.js";

/**
 * @typedef {import("../core/index.js").AgentOptions} AgentOptions
 * @typedef {import("../core/index.js").ToolResult} ToolResult
 * @typedef {import("../core/index.js").Turn} Turn
 */

/**
 * Fake history format.
 * @typedef {{ role: "user", text: string }
 *   | { role: "assistant", turn: Turn }
 *   | { role: "tool", results: ToolResult[] }} MockMessage
 */

export class MockAgent extends Agent {
  /** @type {string | undefined} */
  static defaultModel = "mock";

  /** @param {AgentOptions} options */
  constructor(options) {
    super(options);
    /** @type {MockMessage[]} */
    this.messages = [];
  }

  /** @param {string} text */
  addUserMessage(text) {
    this.messages.push({ role: "user", text });
  }

  /** @returns {Promise<Turn>} */
  async callModel() {
    const last = this.messages.at(-1);
    /** @type {Turn} */
    const turn =
      last?.role === "tool"
        ? { text: `(mock) Tool results:\n${last.results.map((r) => `- ${r.content}`).join("\n")}`, toolCalls: [] }
        : {
            text: "",
            toolCalls: [...this.tools.keys()].map((name, i) => ({ id: `mock_${this.messages.length}_${i}`, name, arguments: {} })),
          };
    this.messages.push({ role: "assistant", turn });
    return turn;
  }

  /** @param {ToolResult[]} results */
  addToolResults(results) {
    this.messages.push({ role: "tool", results });
  }
}
```

Although this is our first provider, but we are not going to be allowing deep linking into this module, so we add another index file for exports.

`src/providers/index.js`:
```javascript
// @ts-check

export { MockAgent } from "./mock.js";
```

## The Provider Factory

Arguably the most interesting file, where the magic happens. The factory will craft our agest, specific to a given LLM. We create a provider object that will hold our current mock agent and any future concrete agents and create a factory to instantiate the agent.

`src/factory.js`:
```javascript
// @ts-check

import { MockAgent } from "./providers/index.js";

/**
 * @typedef {import("./core/index.js").Agent} Agent
 * @typedef {import("./core/index.js").Tool} Tool
 */

export const PROVIDERS = {
  mock: MockAgent,
};

/** @typedef {keyof typeof PROVIDERS} ProviderName */

/**
 * @param {string} name
 * @returns {name is ProviderName}
 */
export function isProviderName(name) {
  return Object.hasOwn(PROVIDERS, name);
}

/**
 * Build an agent for `provider`, using its default model unless `model` is given.
 * @param {ProviderName} provider
 * @param {{ tools: Tool[], model?: string, system?: string, maxTurns?: number }} options
 * @returns {Agent}
 */
export function createAgent(provider, { tools, model, system, maxTurns }) {
  if (!isProviderName(provider)) {
    throw new Error(`Unknown provider '${provider}'. Choose from: ${Object.keys(PROVIDERS).join(", ")}.`);
  }
  const AgentClass = PROVIDERS[provider];
  const chosenModel = model ?? AgentClass.defaultModel;
  if (!chosenModel) {
    throw new Error(`Provider '${provider}' has no default model; pass one with --model.`);
  }
  return new AgentClass({ model: chosenModel, tools, system, maxTurns });
}

```

Now let us add the `exports` file that we declared in our `package.json`, although you probably will never use it anyway.

`src/index.js`:
```javascript
// @ts-check

export { Agent } from "./core/index.js";
export { createAgent, isProviderName, PROVIDERS } from "./factory.js";
export { TOOLS } from "./tools/index.js";

/**
 * @typedef {import("./core/index.js").AgentOptions} AgentOptions
 * @typedef {import("./core/index.js").Tool} Tool
 * @typedef {import("./core/index.js").ToolCall} ToolCall
 * @typedef {import("./core/index.js").ToolCallListener} ToolCallListener
 * @typedef {import("./core/index.js").ToolResult} ToolResult
 * @typedef {import("./core/index.js").Turn} Turn
 * @typedef {import("./factory.js").ProviderName} ProviderName
 */
```

Now the command line tool.

`bin/agent.js`:

```javascript
#!/usr/bin/env node
// @ts-check

import { createInterface } from "node:readline/promises";
import { parseArgs } from "node:util";
import { createAgent, isProviderName, PROVIDERS, TOOLS } from "../src/index.js";

/** @typedef {import("../src/index.js").ToolCallListener} ToolCallListener */

const DEFAULT_QUESTION = "What time is it in Calgary and Vancouver? Then roll 3 dice.";
const SYSTEM_PROMPT = "You are a helpful assistant. Use the tools when they help answer.";

/** @type {ToolCallListener} */
const showToolCall = (call, result) =>
  console.log(`  [tool] ${call.name}(${JSON.stringify(call.arguments)}) -> ${result.content}`);

const { values, positionals } = parseArgs({
  allowPositionals: true,
  options: {
    provider: { type: "string", default: "mock" },
    model: { type: "string" },
    chat: { type: "boolean", default: false },
  },
});

if (!isProviderName(values.provider)) {
  console.error(`Unknown provider '${values.provider}'. Choose from: ${Object.keys(PROVIDERS).join(", ")}.`);
  process.exit(2);
}

/** @param {import("../src/index.js").ProviderName} provider */
async function main(provider) {
  const agent = createAgent(provider, { tools: TOOLS, model: values.model, system: SYSTEM_PROMPT });
  console.log(`Provider: ${provider}   Model: ${agent.model}\n`);

  if (!values.chat) {
    console.log(await agent.run(positionals.join(" ") || DEFAULT_QUESTION, showToolCall));
  } else {
    console.log("Chat mode. Type 'quit' or press Ctrl+D to stop.");
    const rl = createInterface({ input: process.stdin, output: process.stdout });
    rl.on("close", () => process.exit(0));
    while (true) {
      const question = (await rl.question("\nyou> ")).trim();
      if (["quit", "exit"].includes(question.toLowerCase())) break;
      if (question) console.log(`agent> ${await agent.run(question, showToolCall)}`);
    }
    rl.close();
  }
}

main(values.provider).catch((error) => {
  console.error(`Error: ${error instanceof Error ? error.message : error}`);
  process.exit(1);
});
```

A small caveat, if using linux, `chmod +x bin/agent.js` would allow executing `./bin/agent.js`. Do note how the file uses only export from `../src/index.js`, respecting the module boundaries.

At this point we are ready to do a first invocation of our agent, which will use the mocked class:

```bash
node bin/agent.js
```

We get a quick output (you might notice the date is in future because the virtual machine I am using for this is set to use Australian timezone, but the accuracy is not what mattes in this case).

```bash
Provider: mock   Model: mock

  [tool] get_time({}) -> Wednesday, 2026-10-28, 06:02
  [tool] roll_dice({}) -> rolls=[2] total=2
(mock) Tool results:
- Wednesday, 2026-10-28, 06:02
- rolls=[2] total=2
```

## Adding a Real Provider

Any provider we add will follow the three steps:
* get its own file in `src/providers` folder;
* get exported via `src/providers/index.js`;
* get registered in `src/factory.js` in the `PROVIDERS`.

No extra steps needed by the design of our agent project.

Now, I do not have openai keys and I am reluctact to do a tutorial for claude, cos who realistically would be reading it instead of telling claude to do it, so I will show an example for a local `ollama` model. As a pre-requisite you will need a local `ollama` model spinning somewhere in a container.

Then we need dependencies, in this case only one, which is `openai`.

```bash
npm install openai
```

Its their official client, which ships their own types, and de-facto the standard at the moment. Since `ollama` serves its models in `openai` format, this package is perfect for us for typing.

Onward with our concrete `ollama` agent (which will be using default model called `local-agent` in my case, you need to replace it with whatever model you are running, if you want a default one)

`src/providers/ollama.js`:
```javascript
// @ts-check

import OpenAI from "openai";
import { Agent } from "../index.js";

/**
 * @typedef {import("../core/index.js").AgentOptions} AgentOptions
 * @typedef {import("../core/index.js").ToolResult} ToolResult
 * @typedef {import("../core/index.js").Turn} Turn
 */

export class OllamaAgent extends Agent 
{
  /**
   * No default
   * @type {string | undefined}
   */
  static defaultModel = "local-agent";

  /** @param {AgentOptions} options */
  constructor(options) {
    super(options);
    this.client = new OpenAI({ baseURL: this.baseURL(), apiKey: this.apiKey() });
    // OpenAI's tool format wraps the same JSON Schema in { type: "function", ... }.
    /** @type {OpenAI.Chat.ChatCompletionTool[]} */
    this.toolSpecs = options.tools.map((t) => ({
      type: "function",
      function: { name: t.name, description: t.description, parameters: t.parameters },
    }));
    /** @type {OpenAI.Chat.ChatCompletionMessageParam[]} */
    this.messages = [];
    // OpenAI puts the system prompt in the message list
    if (this.system) this.messages.push({ role: "system", content: this.system });
  }

  /**
   * Where to send requests. Left undefined would mean OpenAI's own servers.
   * @returns {string | undefined}
   */
  baseURL() {
        return process.env.OLLAMA_BASE_URL ?? "http://localhost:11434/v1";
  }

  /** @returns {string} */
  apiKey() {
        return "ollama"; // Although Ollama ignores the key, the client requires one

  }

  /** @param {string} text */
  addUserMessage(text) {
    this.messages.push({ role: "user", content: text });
  }

  /** @returns {Promise<Turn>} */
  async callModel() {
    const response = await this.client.chat.completions.create({
      model: this.model,
      tools: this.toolSpecs,
      messages: this.messages,
    });
    const message = response.choices[0].message;
    const calls = (message.tool_calls ?? []).filter((c) => c.type === "function");

    this.messages.push({
      role: "assistant",
      content: message.content ?? "",
      ...(calls.length > 0 && { tool_calls: calls }),
    });

    return {
      text: message.content ?? "",
      toolCalls: calls.map((c) => ({ id: c.id, name: c.function.name, arguments: parseArgs(c.function.arguments) })),
    };
  }

  /** @param {ToolResult[]} results */
  addToolResults(results) {
    for (const r of results) {
      this.messages.push({ role: "tool", tool_call_id: r.callId, content: r.content });
    }
  }
}


/**
 * Small local models might produce empty or broken JSON. Pass {} to the tool in that case, which yields a readable error back to the model.
 * @param {string} raw
 * @returns {Record<string, unknown>}
 */
function parseArgs(raw) {
  try {
    const args = JSON.parse(raw || "{}");
    return args && typeof args === "object" && !Array.isArray(args) ? args : {};
  } catch {
    return {};
  }
}
```

Do note how `messages` are a pile doomed to bloat in memory, realistically you would be clearing them, but that is out of scope for us for now.

What is left is exporting our new agent through the index file and adding it to the `PROVIDERS`.

`src/providers/index.js`:

```javascript
// @ts-check

export { MockAgent } from "./mock.js";
export { OllamaAgent } from "./ollama.js";
```

`src/factory.js`:

```javascript
// the only place for change
export const PROVIDERS = {
  mock: MockAgent,
  ollama: OllamaAgent,
};
```

And that is basically it, we have added a new concrete agent implementation, which can be invoked directly, i.e.

```bash
node bin/agent.js --provider ollama "What time is it in Vancouver? Then roll 2 dice."
```

And we get a reply (how fast is dependent on your machine)
```bash
Provider: ollama   Model: local-agent

  [tool] get_time({"timezone":"America/Vancouver"}) -> Tuesday, 2026-10-27, 14:04
  [tool] roll_dice({"count":2}) -> rolls=[5, 1] total=6
The current time in Vancouver is Tuesday, October 27, 2026, at 14:04.

I rolled 2 dice, and the results are a 5 and a 1, for a total of 6.
```

Fairly easy, is it not?

As usually, links to repo below

https://github.com/Bwca/tryout_agnostic-agent