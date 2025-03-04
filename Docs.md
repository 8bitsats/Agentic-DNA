This will guide you through creating a DNA generator for AI agents using NVIDIA's Evo 2 model, based on the provided concepts and requirements. We’ll implement this as a reusable TypeScript function that AI agents can call, and I’ll explain how to integrate it into an agent framework like Eliza, specifically with the Direct client. This solution will be in-depth, covering the model, implementation, integration, and practical considerations.

---

### Step 1: Understanding the NVIDIA Evo 2 Model and API

The NVIDIA Evo 2 model (`evo2-40b`) is a powerful 40-billion-parameter biological foundation model designed for generating DNA sequences. It’s accessible through the NVIDIA Health API at:

```
https://health.api.nvidia.com/v1/biology/arc/evo2-40b/generate
```

This API accepts POST requests with a JSON payload containing parameters such as the initial DNA sequence and the number of nucleotides to generate. Authentication is handled via an `NVCF_RUN_KEY`, typically passed in the `Authorization` header as a bearer token.

Key aspects of the API:
- **Input**: A starting DNA sequence (e.g., "ATG") and generation parameters.
- **Output**: A generated DNA sequence extending the input, potentially with metadata like token probabilities if requested.
- **Key Parameters**:
  - `sequence`: The required starting DNA sequence (minimum length ≥ 1).
  - `num_tokens`: The number of additional nucleotides to generate (default: 100).
  - `temperature`: Controls randomness in generation (default: 0.7).
  - `top_k`: Limits sampling to the top K probable tokens (default: 3).
  - `top_p`: Enables nucleus sampling (default: 1).
  - `random_seed`: Optional, for reproducible outputs.
  - `enable_sampled_probs`: Optional, includes probabilities for generated tokens (default: false).

The model is capable of generating sequences up to 1 million kilobases, making it ideal for biological simulations, research tasks, or creative applications within an AI agent’s capabilities.

---

### Step 2: Creating the Reusable DNA Generator Function

To enable AI agents to use this API, we’ll create a TypeScript function called `generateDNA`. This function will:
1. Accept a request object with API parameters.
2. Use the `NVCF_RUN_KEY` for authentication.
3. Make an asynchronous HTTP request to the NVIDIA API.
4. Return the generated DNA sequence, with robust error handling.

Here’s the detailed implementation:

```typescript
// dnaGenerator.ts
import fetch from 'node-fetch'; // Required for Node.js < 18; omit if using built-in fetch

// Request interface reflecting API parameters
export interface DNARequest {
  sequence: string;                  // Starting DNA sequence (required)
  num_tokens?: number;               // Number of nucleotides to generate (default: 100)
  temperature?: number;              // Randomness control (default: 0.7)
  top_k?: number;                    // Top K sampling (default: 3)
  top_p?: number;                    // Top P sampling (default: 1)
  random_seed?: number;              // Seed for reproducibility (optional)
  enable_logits?: boolean;           // Include logits in response (default: false)
  enable_sampled_probs?: boolean;    // Include probabilities (default: false)
  enable_elapsed_ms_per_token?: boolean; // Include timing data (default: false)
}

// Response interface based on expected API output
export interface DNAResponse {
  generated_sequence: string;        // The generated DNA sequence
  sampled_probs?: number[];          // Probabilities, if enabled
  [key: string]: any;                // Allow for additional fields
}

/**
 * Generates a DNA sequence using NVIDIA's Evo 2 model.
 * @param request - The DNA generation parameters.
 * @param runKey - The NVCF_RUN_KEY for API authentication.
 * @returns A promise resolving to the generated DNA response.
 * @throws Error if the API request fails.
 */
export async function generateDNA(request: DNARequest, runKey: string): Promise<DNAResponse> {
  const url = 'https://health.api.nvidia.com/v1/biology/arc/evo2-40b/generate';

  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${runKey}`,
    'nvcf-poll-seconds': '300', // Polling timeout for long-running requests
  };

  // Validate required fields
  if (!request.sequence || request.sequence.length < 1) {
    throw new Error('A non-empty starting sequence is required.');
  }

  const body = JSON.stringify({
    num_tokens: 100,           // Default value
    temperature: 0.7,          // Default value
    top_k: 3,                  // Default value
    top_p: 1,                  // Default value
    ...request,                // Override defaults with provided values
  });

  const response = await fetch(url, {
    method: 'POST',
    headers,
    body,
  });

  if (!response.ok) {
    const errorText = await response.text();
    throw new Error(`API request failed: ${response.status} - ${errorText}`);
  }

  const data = await response.json();
  if (!data.generated_sequence) {
    throw new Error('Invalid API response: missing generated_sequence');
  }

  return data as DNAResponse;
}
```

**Key Features**:
- **Type Safety**: Interfaces ensure parameters and responses are well-defined.
- **Defaults**: Sensible defaults are applied if optional parameters are omitted.
- **Error Handling**: Checks for network errors, invalid responses, and missing required fields.
- **Flexibility**: Additional response fields are supported via `[key: string]: any`.

**Dependencies**:
- If using Node.js < 18, install `node-fetch`: `npm install node-fetch`. For Node.js 18+, use the built-in `fetch`.

**Notes**:
- The `DNAResponse` interface assumes a basic structure (`{ "generated_sequence": "..." }`). After testing with the actual API, adjust this to match the full response schema.

---

### Step 3: Parsing User Input for Agent Use

AI agents need to interpret user requests like "Generate a DNA sequence starting with ATG, length 50." We’ll create a helper function to parse such inputs and prepare a `DNARequest`:

```typescript
// dnaGenerator.ts (continued)

/**
 * Parses a user message to extract DNA generation parameters.
 * @param text - The user’s input text.
 * @returns A DNARequest object or null if parsing fails.
 */
export function parseDNARequest(text: string): DNARequest | null {
  // Match format: "starting with [sequence], length [number]"
  const match = text.match(/starting with (\w+), length (\d+)/i);
  if (!match) return null;

  const sequence = match[1].toUpperCase(); // e.g., "ATG"
  const totalLength = parseInt(match[2]);  // e.g., 50
  const num_tokens = totalLength - sequence.length;

  // Validate
  if (num_tokens <= 0) return null;
  if (!/^[ACGT]+$/i.test(sequence)) return null; // Ensure valid DNA bases

  return {
    sequence,
    num_tokens,
  };
}
```

**How It Works**:
- Extracts the starting sequence and total desired length.
- Calculates `num_tokens` as the number of nucleotides to generate beyond the starting sequence.
- Validates that the sequence contains only A, C, G, T (case-insensitive).

**Example**:
- Input: "starting with ATG, length 50"
- Output: `{ sequence: "ATG", num_tokens: 47 }`

---

### Step 4: Integrating with the Eliza Framework (Direct Client)

The Eliza framework allows AI agents to process messages and perform actions. The Direct client, built with Express.js, provides an API-driven interface (e.g., `POST /:agentId/message`). We’ll integrate the DNA generator as a custom action that agents can trigger based on user input.

#### Defining the Action

In Eliza, actions are objects with a `validate` function (to check applicability) and a `handler` function (to execute the action). Here’s the DNA generation action:

```typescript
// actions.ts
import { Action, IAgentRuntime, Memory } from '@elizaos/core'; // Hypothetical Eliza imports
import { generateDNA, DNARequest, parseDNARequest } from './dnaGenerator';

export const dnaGenerationAction: Action = {
  name: 'GENERATE_DNA',
  similes: ['CREATE_DNA', 'MAKE_DNA_SEQUENCE'], // Alternative triggers
  description: 'Generates a DNA sequence using NVIDIA’s Evo 2 model',
  suppressInitialMessage: true, // Prevent default response; handler manages output
  validate: async (runtime: IAgentRuntime, message: Memory): Promise<boolean> => {
    const text = message.content.text.toLowerCase();
    return (
      text.includes('generate dna') ||
      text.includes('create dna sequence') ||
      !!parseDNARequest(message.content.text)
    );
  },
  handler: async (runtime: IAgentRuntime, message: Memory): Promise<boolean> => {
    // Retrieve the API key from agent settings
    const runKey = runtime.character.settings.secrets?.NVCF_RUN_KEY;
    if (!runKey) {
      await runtime.messageManager.createMemory({
        content: { text: 'Error: NVCF_RUN_KEY is not configured.' },
        userId: message.userId,
        roomId: message.roomId,
      });
      return true;
    }

    // Parse the request
    const params = parseDNARequest(message.content.text);
    if (!params) {
      await runtime.messageManager.createMemory({
        content: { text: 'Please provide a starting sequence and desired length (e.g., "starting with ATG, length 50").' },
        userId: message.userId,
        roomId: message.roomId,
      });
      return true;
    }

    try {
      const response = await generateDNA(params, runKey);
      await runtime.messageManager.createMemory({
        content: { text: `Generated DNA sequence: ${response.generated_sequence}` },
        userId: message.userId,
        roomId: message.roomId,
      });
    } catch (error) {
      console.error('DNA generation error:', error);
      await runtime.messageManager.createMemory({
        content: { text: 'Failed to generate DNA sequence due to an error.' },
        userId: message.userId,
        roomId: message.roomId,
      });
    }

    return true; // Indicate action was handled
  },
};
```

**Key Components**:
- **Validation**: Checks if the message indicates a DNA generation request using keywords or the parser.
- **Handler**: Parses the input, calls `generateDNA`, and stores the result as a new memory in the conversation.
- **Memory Management**: Uses Eliza’s `messageManager` to persist responses, ensuring they’re accessible to the client.

#### Registering the Action

Add the action to the agent’s runtime during setup:

```typescript
// runtimeSetup.ts
import { AgentRuntime } from '@elizaos/core';
import { dnaGenerationAction } from './actions';

const runtime = new AgentRuntime({
  // Other configurations (e.g., modelProvider, database)
  actions: [dnaGenerationAction],
});
```

#### Configuring the Character

Define the agent in a character file, including the `NVCF_RUN_KEY`:

```json
// characters/DNAAgent.json
{
  "name": "DNAAgent",
  "modelProvider": "openai",
  "clients": ["direct"],
  "settings": {
    "secrets": {
      "NVCF_RUN_KEY": "your_run_key_here"
    }
  }
}
```

#### How It Works in the Direct Client

The Direct client’s message endpoint (`POST /:agentId/message`) processes incoming messages:

1. **Message Received**: User sends "Generate a DNA sequence starting with ATG, length 50."
2. **Action Validation**: The runtime checks all registered actions; `dnaGenerationAction.validate` returns `true`.
3. **Action Execution**: The `handler` runs, generates the sequence, and creates a new memory.
4. **Response Delivery**: The Direct client retrieves the latest memory and sends it back as the API response.

Example Express handler (simplified):

```typescript
// directClient.ts
import express from 'express';
import { AgentRuntime } from '@elizaos/core';

const app = express();
app.use(express.json());

const runtimes: { [agentId: string]: AgentRuntime } = { /* Initialize runtimes */ };

app.post('/:agentId/message', async (req, res) => {
  const { agentId } = req.params;
  const message = req.body; // { text: "Generate a DNA sequence..." }
  const runtime = runtimes[agentId];

  // Process actions
  await runtime.processActions(message);

  // Retrieve the latest response from memory
  const response = await runtime.messageManager.getLatestMemory(message.roomId);
  res.json(response.content);
});
```

---

### Step 5: Enhancing the Solution

#### Robust Error Handling

Add specific error cases:

```typescript
// In generateDNA
if (!response.ok) {
  const errorText = await response.text();
  if (response.status === 401) throw new Error('Invalid NVCF_RUN_KEY');
  if (response.status === 429) throw new Error('API rate limit exceeded');
  throw new Error(`API request failed: ${response.status} - ${errorText}`);
}
```

#### Advanced Parsing

Use NLP for more flexible input (e.g., "Make a 50-base DNA starting with ATG"):

```typescript
export function parseDNARequest(text: string): DNARequest | null {
  const patterns = [
    /starting with (\w+), length (\d+)/i,
    /make a (\d+)-base dna starting with (\w+)/i,
  ];
  for (const pattern of patterns) {
    const match = text.match(pattern);
    if (match) {
      const sequence = pattern.source.includes('starting with (\w+)') ? match[1] : match[2];
      const totalLength = parseInt(pattern.source.includes('length') ? match[2] : match[1]);
      const num_tokens = totalLength - sequence.length;
      if (num_tokens > 0 && /^[ACGT]+$/i.test(sequence)) {
        return { sequence: sequence.toUpperCase(), num_tokens };
      }
    }
  }
  return null;
}
```

#### Testing and Validation

- **Unit Test**:
  ```typescript
  const testRequest: DNARequest = { sequence: 'ATG', num_tokens: 47 };
  const response = await generateDNA(testRequest, 'valid_key');
  console.assert(response.generated_sequence.startsWith('ATG'), 'Sequence should extend input');
  ```

- **API Response**: After initial use, log the full response to refine `DNAResponse`.

#### Scalability

For large `num_tokens`, consider streaming or async updates:

```typescript
// Future enhancement
app.get('/:agentId/dna-status/:taskId', async (req, res) => {
  // Return progress for long-running DNA generation
});
```

---

### Complete Solution

1. **DNA Generator** (`dnaGenerator.ts`): As shown above.
2. **Action Definition** (`actions.ts`): Custom action integrated with Eliza.
3. **Runtime Setup** (`runtimeSetup.ts`): Register the action.
4. **Character Config** (`DNAAgent.json`): Include the API key.
5. **Direct Client**: Processes messages and delivers responses.

**Usage Example**:
- **Request**: `POST /DNAAgent/message` with `{ "text": "Generate a DNA sequence starting with ATG, length 50" }`
- **Response**: `{ "text": "Generated DNA sequence: ATG..." }`

This DNA generator is now a reusable, robust component within your AI agents, fully integrated with the Eliza Direct client, ready to handle biological tasks with NVIDIA’s Evo 2 model!
