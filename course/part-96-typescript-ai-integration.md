# Part 96: TypeScript กับการ Integration กับ AI/LLM

## บทนำ

ในยุคปัจจุบัน AI และ Large Language Models (LLMs) กลายเป็นส่วนสำคัญของการพัฒนาซอฟต์แวร์ TypeScript มีข้อได้เปรียบอย่างมากในการทำงานกับ AI APIs เพราะ type system ช่วยให้เราสามารถ model ข้อมูลที่ซับซ้อนได้อย่างปลอดภัย บทนี้จะครอบคลุมการใช้งาน OpenAI SDK, LangChain.js, vector databases, RAG patterns, และการสร้าง AI agents ด้วย TypeScript

---

## 96.1 OpenAI SDK กับ TypeScript

### การติดตั้งและตั้งค่า

```bash
npm install openai
npm install -D @types/node dotenv
```

```typescript
// src/config/openai.ts
import OpenAI from 'openai';
import * as dotenv from 'dotenv';

dotenv.config();

if (!process.env.OPENAI_API_KEY) {
  throw new Error('OPENAI_API_KEY is not set in environment variables');
}

export const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  maxRetries: 3,
  timeout: 30000,
});

export default openai;
```

### Type-Safe API Calls

```typescript
// src/types/openai-types.ts
import type {
  ChatCompletion,
  ChatCompletionMessage,
  ChatCompletionMessageParam,
  ChatModel,
} from 'openai/resources/chat/completions';

// Re-export สำหรับใช้งานทั่วโปรเจกต์
export type {
  ChatCompletion,
  ChatCompletionMessage,
  ChatCompletionMessageParam,
  ChatModel,
};

// Custom types สำหรับโปรเจกต์ของเรา
export interface ConversationMessage {
  role: 'system' | 'user' | 'assistant';
  content: string;
  timestamp?: Date;
}

export interface ChatOptions {
  model?: ChatModel;
  temperature?: number;
  maxTokens?: number;
  systemPrompt?: string;
}

export interface ChatResult {
  message: string;
  usage: {
    promptTokens: number;
    completionTokens: number;
    totalTokens: number;
  };
  finishReason: string | null;
}
```

```typescript
// src/services/chat-service.ts
import openai from '../config/openai';
import type {
  ChatCompletionMessageParam,
  ChatOptions,
  ChatResult,
} from '../types/openai-types';

export class ChatService {
  private defaultModel = 'gpt-4o-mini' as const;

  async chat(
    messages: ChatCompletionMessageParam[],
    options: ChatOptions = {}
  ): Promise<ChatResult> {
    const {
      model = this.defaultModel,
      temperature = 0.7,
      maxTokens = 1000,
      systemPrompt,
    } = options;

    const allMessages: ChatCompletionMessageParam[] = systemPrompt
      ? [{ role: 'system', content: systemPrompt }, ...messages]
      : messages;

    const response = await openai.chat.completions.create({
      model,
      messages: allMessages,
      temperature,
      max_tokens: maxTokens,
    });

    const choice = response.choices[0];
    if (!choice?.message?.content) {
      throw new Error('No content in response');
    }

    return {
      message: choice.message.content,
      usage: {
        promptTokens: response.usage?.prompt_tokens ?? 0,
        completionTokens: response.usage?.completion_tokens ?? 0,
        totalTokens: response.usage?.total_tokens ?? 0,
      },
      finishReason: choice.finish_reason,
    };
  }

  async simpleChat(userMessage: string, options?: ChatOptions): Promise<string> {
    const result = await this.chat(
      [{ role: 'user', content: userMessage }],
      options
    );
    return result.message;
  }
}

// ตัวอย่างการใช้งาน
async function example() {
  const service = new ChatService();
  
  const response = await service.simpleChat(
    'อธิบาย TypeScript generics ให้ฉันเข้าใจ',
    { systemPrompt: 'คุณเป็นผู้สอน TypeScript ที่อธิบายเป็นภาษาไทย' }
  );
  
  console.log(response);
}
```

---

## 96.2 Streaming Responses

Streaming ช่วยให้ผู้ใช้เห็นผลลัพธ์ทีละส่วน แทนที่จะรอจนครบ

```typescript
// src/services/streaming-service.ts
import openai from '../config/openai';
import type { ChatCompletionMessageParam } from 'openai/resources/chat/completions';

export interface StreamChunk {
  delta: string;
  done: boolean;
  totalTokens?: number;
}

export type StreamCallback = (chunk: StreamChunk) => void;

export class StreamingChatService {
  async streamChat(
    messages: ChatCompletionMessageParam[],
    onChunk: StreamCallback,
    options: {
      model?: string;
      temperature?: number;
      systemPrompt?: string;
    } = {}
  ): Promise<string> {
    const { model = 'gpt-4o-mini', temperature = 0.7, systemPrompt } = options;

    const allMessages: ChatCompletionMessageParam[] = systemPrompt
      ? [{ role: 'system', content: systemPrompt }, ...messages]
      : messages;

    const stream = openai.chat.completions.stream({
      model,
      messages: allMessages,
      temperature,
    });

    let fullContent = '';

    for await (const chunk of stream) {
      const delta = chunk.choices[0]?.delta?.content ?? '';
      fullContent += delta;

      onChunk({
        delta,
        done: false,
      });
    }

    const finalMessage = await stream.finalMessage();
    
    onChunk({
      delta: '',
      done: true,
      totalTokens: finalMessage.usage?.total_tokens,
    });

    return fullContent;
  }

  // สำหรับ Node.js ReadableStream
  createReadableStream(
    messages: ChatCompletionMessageParam[],
    options: { model?: string; systemPrompt?: string } = {}
  ): ReadableStream<string> {
    const { model = 'gpt-4o-mini', systemPrompt } = options;

    const allMessages: ChatCompletionMessageParam[] = systemPrompt
      ? [{ role: 'system', content: systemPrompt }, ...messages]
      : messages;

    return new ReadableStream({
      async start(controller) {
        try {
          const stream = openai.chat.completions.stream({
            model,
            messages: allMessages,
          });

          for await (const chunk of stream) {
            const delta = chunk.choices[0]?.delta?.content ?? '';
            if (delta) {
              controller.enqueue(delta);
            }
          }

          controller.close();
        } catch (error) {
          controller.error(error);
        }
      },
    });
  }
}

// ตัวอย่าง Express.js endpoint สำหรับ streaming
import express from 'express';

const app = express();
const streamingService = new StreamingChatService();

app.post('/chat/stream', async (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  const { message } = req.body as { message: string };

  try {
    await streamingService.streamChat(
      [{ role: 'user', content: message }],
      (chunk) => {
        if (!chunk.done) {
          res.write(`data: ${JSON.stringify({ text: chunk.delta })}\n\n`);
        } else {
          res.write(`data: ${JSON.stringify({ done: true })}\n\n`);
          res.end();
        }
      }
    );
  } catch (error) {
    res.write(`data: ${JSON.stringify({ error: 'Stream error' })}\n\n`);
    res.end();
  }
});
```

---

## 96.3 Function Calling กับ TypeScript Types

Function calling ช่วยให้ LLM สามารถเรียกใช้ฟังก์ชันในโค้ดของเราได้

```typescript
// src/types/function-calling.ts
import type { ChatCompletionTool } from 'openai/resources/chat/completions';

// Type-safe function definitions
export interface ToolDefinition<TParams = unknown, TResult = unknown> {
  name: string;
  description: string;
  parameters: {
    type: 'object';
    properties: Record<string, {
      type: string;
      description: string;
      enum?: string[];
    }>;
    required?: string[];
  };
  execute: (params: TParams) => Promise<TResult> | TResult;
}

// Helper function สร้าง OpenAI tool format
export function createTool<TParams = unknown, TResult = unknown>(
  definition: ToolDefinition<TParams, TResult>
): ChatCompletionTool {
  return {
    type: 'function',
    function: {
      name: definition.name,
      description: definition.description,
      parameters: definition.parameters,
    },
  };
}
```

```typescript
// src/tools/weather-tool.ts
import type { ToolDefinition } from '../types/function-calling';

interface WeatherParams {
  location: string;
  unit?: 'celsius' | 'fahrenheit';
}

interface WeatherResult {
  location: string;
  temperature: number;
  unit: string;
  description: string;
  humidity: number;
  windSpeed: number;
}

export const weatherTool: ToolDefinition<WeatherParams, WeatherResult> = {
  name: 'get_weather',
  description: 'Get current weather information for a location',
  parameters: {
    type: 'object',
    properties: {
      location: {
        type: 'string',
        description: 'City name or location (e.g., "Bangkok", "New York")',
      },
      unit: {
        type: 'string',
        description: 'Temperature unit',
        enum: ['celsius', 'fahrenheit'],
      },
    },
    required: ['location'],
  },
  async execute(params: WeatherParams): Promise<WeatherResult> {
    // จริงๆ ควรเรียก weather API แต่นี่เป็น mock
    const mockData: Record<string, WeatherResult> = {
      'bangkok': {
        location: 'Bangkok, Thailand',
        temperature: params.unit === 'fahrenheit' ? 95 : 35,
        unit: params.unit ?? 'celsius',
        description: 'Hot and humid',
        humidity: 80,
        windSpeed: 10,
      },
    };

    return mockData[params.location.toLowerCase()] ?? {
      location: params.location,
      temperature: params.unit === 'fahrenheit' ? 77 : 25,
      unit: params.unit ?? 'celsius',
      description: 'Partly cloudy',
      humidity: 60,
      windSpeed: 15,
    };
  },
};
```

```typescript
// src/services/function-calling-service.ts
import openai from '../config/openai';
import { createTool } from '../types/function-calling';
import type { ToolDefinition } from '../types/function-calling';
import type {
  ChatCompletionMessageParam,
  ChatCompletionTool,
} from 'openai/resources/chat/completions';

export class FunctionCallingService {
  private tools: Map<string, ToolDefinition<unknown, unknown>>;
  private openaiTools: ChatCompletionTool[];

  constructor(toolDefinitions: ToolDefinition<unknown, unknown>[]) {
    this.tools = new Map(toolDefinitions.map((t) => [t.name, t]));
    this.openaiTools = toolDefinitions.map(createTool);
  }

  async chat(userMessage: string): Promise<string> {
    const messages: ChatCompletionMessageParam[] = [
      { role: 'user', content: userMessage },
    ];

    // Loop จนกว่าจะได้คำตอบสุดท้าย
    while (true) {
      const response = await openai.chat.completions.create({
        model: 'gpt-4o-mini',
        messages,
        tools: this.openaiTools,
        tool_choice: 'auto',
      });

      const message = response.choices[0]?.message;
      if (!message) break;

      messages.push(message);

      // ถ้า finish_reason เป็น 'stop' แสดงว่าจบแล้ว
      if (response.choices[0]?.finish_reason === 'stop') {
        return message.content ?? '';
      }

      // ถ้ามี tool_calls ให้เรียก tools
      if (message.tool_calls && message.tool_calls.length > 0) {
        for (const toolCall of message.tool_calls) {
          const tool = this.tools.get(toolCall.function.name);
          
          if (!tool) {
            throw new Error(`Unknown tool: ${toolCall.function.name}`);
          }

          try {
            const params = JSON.parse(toolCall.function.arguments) as unknown;
            const result = await tool.execute(params);

            messages.push({
              role: 'tool',
              tool_call_id: toolCall.id,
              content: JSON.stringify(result),
            });
          } catch (error) {
            messages.push({
              role: 'tool',
              tool_call_id: toolCall.id,
              content: JSON.stringify({ error: String(error) }),
            });
          }
        }
      }
    }

    return 'No response generated';
  }
}

// ตัวอย่างการใช้งาน
async function demoFunctionCalling() {
  const { weatherTool } = await import('./tools/weather-tool');
  
  const service = new FunctionCallingService([
    weatherTool as ToolDefinition<unknown, unknown>,
  ]);

  const response = await service.chat(
    'อากาศที่กรุงเทพวันนี้เป็นอย่างไรบ้าง?'
  );
  
  console.log(response);
}
```

---

## 96.4 LangChain.js กับ TypeScript

```bash
npm install langchain @langchain/openai @langchain/community
npm install langchain/core
```

```typescript
// src/langchain/basic-chain.ts
import { ChatOpenAI } from '@langchain/openai';
import { HumanMessage, SystemMessage, AIMessage } from '@langchain/core/messages';
import { ChatPromptTemplate, MessagesPlaceholder } from '@langchain/core/prompts';
import { StringOutputParser } from '@langchain/core/output_parsers';
import { RunnableSequence } from '@langchain/core/runnables';

// สร้าง Chat Model
const model = new ChatOpenAI({
  modelName: 'gpt-4o-mini',
  temperature: 0.7,
  openAIApiKey: process.env.OPENAI_API_KEY,
});

// Simple chain
const simpleChain = async () => {
  const prompt = ChatPromptTemplate.fromMessages([
    ['system', 'คุณเป็นผู้เชี่ยวชาญด้าน TypeScript ที่สอนเป็นภาษาไทย'],
    ['human', '{input}'],
  ]);

  const chain = RunnableSequence.from([
    prompt,
    model,
    new StringOutputParser(),
  ]);

  const result = await chain.invoke({
    input: 'อธิบาย interface กับ type alias ว่าต่างกันอย่างไร',
  });

  console.log(result);
  return result;
};

// Conversation chain พร้อม memory
import { BufferMemory } from 'langchain/memory';
import { ConversationChain } from 'langchain/chains';

const conversationChain = async () => {
  const memory = new BufferMemory();
  
  const chain = new ConversationChain({
    llm: model,
    memory,
  });

  const response1 = await chain.call({ input: 'สวัสดี ฉันชื่อนาย' });
  console.log('AI:', response1.response);

  const response2 = await chain.call({ input: 'ฉันชื่ออะไร?' });
  console.log('AI:', response2.response); // ควรจำได้ว่าชื่อนาย

  return { response1, response2 };
};
```

```typescript
// src/langchain/structured-output.ts
import { ChatOpenAI } from '@langchain/openai';
import { ChatPromptTemplate } from '@langchain/core/prompts';
import { z } from 'zod';
import { StructuredOutputParser } from 'langchain/output_parsers';

// Schema สำหรับ output ที่ต้องการ
const productSchema = z.object({
  name: z.string().describe('Product name'),
  price: z.number().describe('Product price in Thai Baht'),
  category: z.enum(['Electronics', 'Clothing', 'Food', 'Other']).describe('Product category'),
  description: z.string().describe('Product description in Thai'),
  tags: z.array(z.string()).describe('Product tags'),
});

type Product = z.infer<typeof productSchema>;

const extractProductInfo = async (text: string): Promise<Product> => {
  const parser = StructuredOutputParser.fromZodSchema(productSchema);
  
  const model = new ChatOpenAI({
    modelName: 'gpt-4o-mini',
    temperature: 0,
  });

  const prompt = ChatPromptTemplate.fromMessages([
    ['system', 'Extract product information from the given text.\n{format_instructions}'],
    ['human', '{input}'],
  ]);

  const chain = prompt.pipe(model).pipe(parser);

  const result = await chain.invoke({
    input: text,
    format_instructions: parser.getFormatInstructions(),
  });

  return result;
};

// ตัวอย่างการใช้งาน
async function demo() {
  const product = await extractProductInfo(
    'iPhone 15 Pro Max สีไททาเนียม ราคา 49,900 บาท กล้อง 48MP ความจุ 256GB'
  );
  
  console.log('Product:', product);
  // { name: 'iPhone 15 Pro Max', price: 49900, category: 'Electronics', ... }
}
```

---

## 96.5 Vector Embeddings

```bash
npm install @pinecone-database/pinecone
npm install @langchain/pinecone
```

```typescript
// src/embeddings/embedding-service.ts
import OpenAI from 'openai';
import type { EmbeddingCreateParams } from 'openai/resources/embeddings';

interface EmbeddingResult {
  text: string;
  embedding: number[];
  model: string;
  tokens: number;
}

interface SimilarityResult {
  text: string;
  similarity: number;
}

export class EmbeddingService {
  private openai: OpenAI;
  private defaultModel = 'text-embedding-3-small' as const;

  constructor(private apiKey?: string) {
    this.openai = new OpenAI({ apiKey: apiKey ?? process.env.OPENAI_API_KEY });
  }

  async embed(
    text: string,
    model: EmbeddingCreateParams['model'] = this.defaultModel
  ): Promise<EmbeddingResult> {
    const response = await this.openai.embeddings.create({
      model,
      input: text,
    });

    const data = response.data[0];
    if (!data) throw new Error('No embedding data returned');

    return {
      text,
      embedding: data.embedding,
      model: response.model,
      tokens: response.usage.total_tokens,
    };
  }

  async embedBatch(
    texts: string[],
    model: EmbeddingCreateParams['model'] = this.defaultModel
  ): Promise<EmbeddingResult[]> {
    const response = await this.openai.embeddings.create({
      model,
      input: texts,
    });

    return response.data.map((item, index) => ({
      text: texts[index] ?? '',
      embedding: item.embedding,
      model: response.model,
      tokens: response.usage.total_tokens / texts.length, // approximate
    }));
  }

  // คำนวณ Cosine Similarity
  cosineSimilarity(a: number[], b: number[]): number {
    if (a.length !== b.length) {
      throw new Error('Vectors must have the same dimension');
    }

    let dotProduct = 0;
    let normA = 0;
    let normB = 0;

    for (let i = 0; i < a.length; i++) {
      dotProduct += (a[i] ?? 0) * (b[i] ?? 0);
      normA += (a[i] ?? 0) ** 2;
      normB += (b[i] ?? 0) ** 2;
    }

    return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
  }

  // ค้นหา text ที่คล้ายกันที่สุด
  async findSimilar(
    query: string,
    candidates: string[],
    topK = 5
  ): Promise<SimilarityResult[]> {
    const [queryEmbedding, ...candidateEmbeddings] = await Promise.all([
      this.embed(query),
      ...candidates.map((c) => this.embed(c)),
    ]);

    if (!queryEmbedding) throw new Error('Failed to embed query');

    const similarities = candidateEmbeddings.map((ce, index) => ({
      text: candidates[index] ?? '',
      similarity: this.cosineSimilarity(queryEmbedding.embedding, ce.embedding),
    }));

    return similarities
      .sort((a, b) => b.similarity - a.similarity)
      .slice(0, topK);
  }
}

// ตัวอย่างการใช้งาน
async function embeddingDemo() {
  const service = new EmbeddingService();

  const texts = [
    'TypeScript เป็น superset ของ JavaScript',
    'React เป็น library สำหรับสร้าง UI',
    'Node.js ใช้ V8 JavaScript engine',
    'Python เป็นภาษาที่นิยมสำหรับ Data Science',
    'TypeScript ช่วยให้โค้ดมี type safety',
  ];

  const similar = await service.findSimilar(
    'TypeScript ช่วยในการพัฒนาได้อย่างไร',
    texts,
    3
  );

  console.log('Similar texts:');
  similar.forEach((s) => {
    console.log(`- [${s.similarity.toFixed(3)}] ${s.text}`);
  });
}
```

---

## 96.6 Vector Database กับ Pinecone

```typescript
// src/vector-db/pinecone-service.ts
import { Pinecone } from '@pinecone-database/pinecone';
import type { RecordMetadata, ScoredPineconeRecord } from '@pinecone-database/pinecone';
import { EmbeddingService } from '../embeddings/embedding-service';

interface DocumentMetadata extends RecordMetadata {
  text: string;
  source: string;
  timestamp: string;
  tags: string[];
}

interface SearchResult {
  id: string;
  text: string;
  score: number;
  metadata: DocumentMetadata;
}

export class VectorDBService {
  private pinecone: Pinecone;
  private embeddingService: EmbeddingService;
  private indexName: string;

  constructor(
    private config: {
      pineconeApiKey: string;
      indexName: string;
    }
  ) {
    this.pinecone = new Pinecone({
      apiKey: config.pineconeApiKey,
    });
    this.embeddingService = new EmbeddingService();
    this.indexName = config.indexName;
  }

  async upsertDocuments(
    documents: Array<{
      id: string;
      text: string;
      source: string;
      tags?: string[];
    }>
  ): Promise<void> {
    const index = this.pinecone.index<DocumentMetadata>(this.indexName);

    const embeddings = await this.embeddingService.embedBatch(
      documents.map((d) => d.text)
    );

    const vectors = documents.map((doc, i) => ({
      id: doc.id,
      values: embeddings[i]?.embedding ?? [],
      metadata: {
        text: doc.text,
        source: doc.source,
        timestamp: new Date().toISOString(),
        tags: doc.tags ?? [],
      } as DocumentMetadata,
    }));

    // Upsert in batches of 100
    const batchSize = 100;
    for (let i = 0; i < vectors.length; i += batchSize) {
      const batch = vectors.slice(i, i + batchSize);
      await index.upsert(batch);
    }
  }

  async search(
    query: string,
    options: {
      topK?: number;
      filter?: Record<string, string | string[]>;
    } = {}
  ): Promise<SearchResult[]> {
    const { topK = 5, filter } = options;
    const index = this.pinecone.index<DocumentMetadata>(this.indexName);

    const queryEmbedding = await this.embeddingService.embed(query);

    const searchResponse = await index.query({
      vector: queryEmbedding.embedding,
      topK,
      includeMetadata: true,
      filter,
    });

    return (searchResponse.matches ?? []).map((match: ScoredPineconeRecord<DocumentMetadata>) => ({
      id: match.id,
      text: match.metadata?.text ?? '',
      score: match.score ?? 0,
      metadata: match.metadata ?? {
        text: '',
        source: '',
        timestamp: '',
        tags: [],
      },
    }));
  }

  async deleteDocuments(ids: string[]): Promise<void> {
    const index = this.pinecone.index<DocumentMetadata>(this.indexName);
    await index.deleteMany(ids);
  }
}
```

---

## 96.7 RAG (Retrieval Augmented Generation) Patterns

```typescript
// src/rag/document-processor.ts
import * as fs from 'fs';
import * as path from 'path';
import * as crypto from 'crypto';

export interface Document {
  id: string;
  content: string;
  source: string;
  metadata: Record<string, string>;
}

export interface Chunk {
  id: string;
  text: string;
  source: string;
  chunkIndex: number;
  totalChunks: number;
}

export class DocumentProcessor {
  // แบ่ง document เป็น chunks
  splitIntoChunks(
    document: Document,
    options: {
      chunkSize?: number;
      overlap?: number;
    } = {}
  ): Chunk[] {
    const { chunkSize = 500, overlap = 50 } = options;
    const text = document.content;
    const chunks: Chunk[] = [];

    let start = 0;
    let chunkIndex = 0;

    while (start < text.length) {
      const end = Math.min(start + chunkSize, text.length);
      const chunkText = text.slice(start, end);

      // หยุดที่ word boundary
      let actualEnd = end;
      if (end < text.length) {
        const lastSpace = chunkText.lastIndexOf(' ');
        if (lastSpace > 0) {
          actualEnd = start + lastSpace;
        }
      }

      const finalChunk = text.slice(start, actualEnd);
      chunks.push({
        id: this.generateChunkId(document.id, chunkIndex),
        text: finalChunk,
        source: document.source,
        chunkIndex,
        totalChunks: 0, // จะ update ทีหลัง
      });

      start = actualEnd - overlap;
      chunkIndex++;
    }

    // Update totalChunks
    return chunks.map((chunk) => ({
      ...chunk,
      totalChunks: chunks.length,
    }));
  }

  private generateChunkId(docId: string, chunkIndex: number): string {
    return crypto
      .createHash('md5')
      .update(`${docId}-${chunkIndex}`)
      .digest('hex');
  }

  // โหลด text file
  loadTextFile(filePath: string): Document {
    const content = fs.readFileSync(filePath, 'utf-8');
    const fileName = path.basename(filePath);

    return {
      id: crypto.createHash('md5').update(filePath).digest('hex'),
      content,
      source: filePath,
      metadata: {
        fileName,
        fileType: path.extname(filePath),
        createdAt: new Date().toISOString(),
      },
    };
  }
}
```

```typescript
// src/rag/rag-system.ts
import { VectorDBService } from '../vector-db/pinecone-service';
import { DocumentProcessor } from './document-processor';
import openai from '../config/openai';
import type { ChatCompletionMessageParam } from 'openai/resources/chat/completions';

interface RAGConfig {
  pineconeApiKey: string;
  indexName: string;
  chunkSize?: number;
  overlap?: number;
  topK?: number;
  model?: string;
}

interface RAGResponse {
  answer: string;
  sources: string[];
  relevantChunks: string[];
}

export class RAGSystem {
  private vectorDB: VectorDBService;
  private processor: DocumentProcessor;
  private config: RAGConfig;

  constructor(config: RAGConfig) {
    this.config = config;
    this.vectorDB = new VectorDBService({
      pineconeApiKey: config.pineconeApiKey,
      indexName: config.indexName,
    });
    this.processor = new DocumentProcessor();
  }

  async addDocument(filePath: string): Promise<void> {
    const document = this.processor.loadTextFile(filePath);
    const chunks = this.processor.splitIntoChunks(document, {
      chunkSize: this.config.chunkSize,
      overlap: this.config.overlap,
    });

    await this.vectorDB.upsertDocuments(
      chunks.map((chunk) => ({
        id: chunk.id,
        text: chunk.text,
        source: chunk.source,
        tags: [`chunk-${chunk.chunkIndex}`, `doc-${document.id}`],
      }))
    );

    console.log(`Added ${chunks.length} chunks from ${filePath}`);
  }

  async query(question: string, conversationHistory: ChatCompletionMessageParam[] = []): Promise<RAGResponse> {
    // 1. ค้นหา relevant chunks
    const searchResults = await this.vectorDB.search(question, {
      topK: this.config.topK ?? 5,
    });

    // 2. สร้าง context จาก chunks
    const context = searchResults
      .map((r, i) => `[${i + 1}] ${r.text}`)
      .join('\n\n');

    const sources = [...new Set(searchResults.map((r) => r.metadata.source))];
    const relevantChunks = searchResults.map((r) => r.text);

    // 3. สร้าง prompt
    const systemPrompt = `คุณเป็น AI assistant ที่ตอบคำถามโดยอิงจากเอกสารที่กำหนดให้
ตอบคำถามเป็นภาษาไทยเสมอ และอ้างอิงจากข้อมูลใน context เท่านั้น
หากไม่มีข้อมูลเพียงพอ ให้บอกว่าไม่ทราบ

Context:
${context}`;

    const messages: ChatCompletionMessageParam[] = [
      { role: 'system', content: systemPrompt },
      ...conversationHistory,
      { role: 'user', content: question },
    ];

    // 4. สร้างคำตอบ
    const response = await openai.chat.completions.create({
      model: this.config.model ?? 'gpt-4o-mini',
      messages,
      temperature: 0.3, // ต่ำกว่าปกติเพื่อให้ตอบตรงๆ
    });

    const answer = response.choices[0]?.message?.content ?? 'ไม่สามารถตอบได้';

    return {
      answer,
      sources,
      relevantChunks,
    };
  }
}

// ตัวอย่างการใช้งาน
async function ragDemo() {
  const rag = new RAGSystem({
    pineconeApiKey: process.env.PINECONE_API_KEY ?? '',
    indexName: 'typescript-docs',
    chunkSize: 500,
    topK: 5,
  });

  // เพิ่มเอกสาร
  await rag.addDocument('./docs/typescript-handbook.txt');

  // ถามคำถาม
  const response = await rag.query('TypeScript generics คืออะไร?');
  console.log('Answer:', response.answer);
  console.log('Sources:', response.sources);
}
```

---

## 96.8 AI Agent Patterns

```typescript
// src/agents/agent-types.ts
import { z } from 'zod';

// Action types ที่ agent สามารถทำได้
export const ActionSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('search'),
    query: z.string(),
  }),
  z.object({
    type: z.literal('calculate'),
    expression: z.string(),
  }),
  z.object({
    type: z.literal('fetch_url'),
    url: z.string().url(),
  }),
  z.object({
    type: z.literal('store_memory'),
    key: z.string(),
    value: z.string(),
  }),
  z.object({
    type: z.literal('retrieve_memory'),
    key: z.string(),
  }),
  z.object({
    type: z.literal('final_answer'),
    answer: z.string(),
  }),
]);

export type Action = z.infer<typeof ActionSchema>;

export interface AgentThought {
  thought: string;
  action?: Action;
}

export interface AgentStep {
  step: number;
  thought: string;
  action?: Action;
  observation?: string;
}

export interface AgentResult {
  answer: string;
  steps: AgentStep[];
  totalSteps: number;
}
```

```typescript
// src/agents/react-agent.ts
import openai from '../config/openai';
import type { AgentStep, AgentResult, Action } from './agent-types';
import { ActionSchema } from './agent-types';
import type { ChatCompletionMessageParam } from 'openai/resources/chat/completions';

// ReAct (Reasoning + Acting) Agent
export class ReactAgent {
  private memory: Map<string, string> = new Map();
  private maxSteps: number;

  constructor(
    private config: {
      maxSteps?: number;
      model?: string;
      tools?: Record<string, (params: unknown) => Promise<string>>;
    } = {}
  ) {
    this.maxSteps = config.maxSteps ?? 10;
  }

  private getSystemPrompt(): string {
    return `คุณเป็น AI agent ที่ทำงานตาม ReAct framework (Reasoning + Acting)
    
คุณจะต้องตอบสนองในรูปแบบ JSON เสมอ:
{
  "thought": "ความคิดเกี่ยวกับ task นี้",
  "action": {
    "type": "action_type",
    ...params
  }
}

Actions ที่ใช้ได้:
- search: { type: "search", query: "..." }
- calculate: { type: "calculate", expression: "..." }
- fetch_url: { type: "fetch_url", url: "..." }
- store_memory: { type: "store_memory", key: "...", value: "..." }
- retrieve_memory: { type: "retrieve_memory", key: "..." }
- final_answer: { type: "final_answer", answer: "..." }

เมื่อได้คำตอบแล้วให้ใช้ final_answer`;
  }

  private async executeAction(action: Action): Promise<string> {
    switch (action.type) {
      case 'calculate': {
        try {
          // Warning: eval ไม่ควรใช้ใน production
          // ควรใช้ library เช่น mathjs แทน
          const result = eval(action.expression) as number;
          return `Result: ${result}`;
        } catch {
          return `Error: Invalid expression`;
        }
      }

      case 'store_memory': {
        this.memory.set(action.key, action.value);
        return `Stored: ${action.key} = ${action.value}`;
      }

      case 'retrieve_memory': {
        const value = this.memory.get(action.key);
        return value ? `Retrieved: ${value}` : `Not found: ${action.key}`;
      }

      case 'search': {
        // Mock search - ใน production จะเรียก search API จริง
        return `Search results for "${action.query}": [Mock results]`;
      }

      case 'fetch_url': {
        return `Fetched content from ${action.url}: [Mock content]`;
      }

      case 'final_answer': {
        return action.answer;
      }

      default: {
        return 'Unknown action';
      }
    }
  }

  async run(task: string): Promise<AgentResult> {
    const steps: AgentStep[] = [];
    const messages: ChatCompletionMessageParam[] = [
      { role: 'system', content: this.getSystemPrompt() },
      { role: 'user', content: `Task: ${task}` },
    ];

    for (let step = 1; step <= this.maxSteps; step++) {
      const response = await openai.chat.completions.create({
        model: this.config.model ?? 'gpt-4o-mini',
        messages,
        response_format: { type: 'json_object' },
        temperature: 0.1,
      });

      const content = response.choices[0]?.message?.content;
      if (!content) break;

      let thought: string;
      let action: Action | undefined;

      try {
        const parsed = JSON.parse(content) as { thought?: string; action?: unknown };
        thought = parsed.thought ?? '';
        
        if (parsed.action) {
          const actionResult = ActionSchema.safeParse(parsed.action);
          if (actionResult.success) {
            action = actionResult.data;
          }
        }
      } catch {
        thought = content;
      }

      // Execute action
      let observation: string | undefined;
      if (action) {
        observation = await this.executeAction(action);
      }

      steps.push({ step, thought, action, observation });

      // Check if done
      if (action?.type === 'final_answer') {
        return {
          answer: action.answer,
          steps,
          totalSteps: step,
        };
      }

      // Add to conversation
      messages.push({ role: 'assistant', content });
      if (observation) {
        messages.push({
          role: 'user',
          content: `Observation: ${observation}`,
        });
      }
    }

    return {
      answer: 'Agent reached maximum steps without final answer',
      steps,
      totalSteps: steps.length,
    };
  }
}

// ตัวอย่างการใช้งาน
async function agentDemo() {
  const agent = new ReactAgent({ maxSteps: 5 });

  const result = await agent.run(
    'คำนวณ (15 * 8) + (100 / 4) แล้วบอกว่าผลลัพธ์เป็นเลขคู่หรือเลขคี่'
  );

  console.log('Answer:', result.answer);
  console.log('Steps taken:', result.totalSteps);
  result.steps.forEach((step) => {
    console.log(`\nStep ${step.step}:`);
    console.log('Thought:', step.thought);
    if (step.action) console.log('Action:', step.action);
    if (step.observation) console.log('Observation:', step.observation);
  });
}
```

---

## 96.9 Prompt Templates กับ Types

```typescript
// src/prompts/prompt-template.ts
import { z } from 'zod';

// Generic prompt template สำหรับ type safety
export class TypedPromptTemplate<TInput extends z.ZodType> {
  constructor(
    private template: string,
    private schema: TInput
  ) {}

  render(input: z.infer<TInput>): string {
    // Validate input
    const validated = this.schema.parse(input);

    // Replace placeholders
    let result = this.template;
    for (const [key, value] of Object.entries(validated as Record<string, unknown>)) {
      const placeholder = `{${key}}`;
      result = result.replaceAll(placeholder, String(value));
    }

    return result;
  }

  // Type-safe render พร้อม validation
  safeRender(input: unknown): { success: true; prompt: string } | { success: false; error: string } {
    const validated = this.schema.safeParse(input);
    
    if (!validated.success) {
      return {
        success: false,
        error: validated.error.message,
      };
    }

    return {
      success: true,
      prompt: this.render(validated.data),
    };
  }
}

// สร้าง prompt templates สำหรับงานต่างๆ
const codeReviewSchema = z.object({
  code: z.string(),
  language: z.enum(['TypeScript', 'JavaScript', 'Python', 'Go']),
  focus: z.enum(['performance', 'security', 'readability', 'bugs']).optional(),
});

export const codeReviewTemplate = new TypedPromptTemplate(
  `Please review the following {language} code:

\`\`\`{language}
{code}
\`\`\`

Focus on: {focus}

Provide feedback in Thai language, including:
1. Issues found
2. Suggestions for improvement
3. Overall assessment`,
  codeReviewSchema
);

const translationSchema = z.object({
  text: z.string(),
  sourceLanguage: z.string(),
  targetLanguage: z.string(),
  style: z.enum(['formal', 'casual', 'technical']).default('formal'),
});

export const translationTemplate = new TypedPromptTemplate(
  `Translate the following {style} text from {sourceLanguage} to {targetLanguage}:

Text: {text}

Translation:`,
  translationSchema
);

// ตัวอย่างการใช้งาน
function promptDemo() {
  const codeReviewPrompt = codeReviewTemplate.render({
    code: `function add(a, b) { return a + b; }`,
    language: 'TypeScript',
    focus: 'bugs',
  });

  console.log(codeReviewPrompt);

  const safeResult = translationTemplate.safeRender({
    text: 'Hello World',
    sourceLanguage: 'English',
    targetLanguage: 'Thai',
    // style จะใช้ default value 'formal'
  });

  if (safeResult.success) {
    console.log(safeResult.prompt);
  } else {
    console.error(safeResult.error);
  }
}
```

---

## 96.10 Tool Definitions กับ Zod

```typescript
// src/tools/tool-registry.ts
import { z } from 'zod';
import type { ChatCompletionTool } from 'openai/resources/chat/completions';

type JSONSchemaType = {
  type: string;
  description?: string;
  enum?: unknown[];
  items?: JSONSchemaType;
  properties?: Record<string, JSONSchemaType>;
  required?: string[];
};

// แปลง Zod schema เป็น JSON Schema สำหรับ OpenAI
function zodToJsonSchema(schema: z.ZodType): JSONSchemaType {
  if (schema instanceof z.ZodObject) {
    const shape = schema.shape as Record<string, z.ZodType>;
    const properties: Record<string, JSONSchemaType> = {};
    const required: string[] = [];

    for (const [key, value] of Object.entries(shape)) {
      properties[key] = zodToJsonSchema(value);
      if (!(value instanceof z.ZodOptional)) {
        required.push(key);
      }
    }

    return {
      type: 'object',
      properties,
      required: required.length > 0 ? required : undefined,
    };
  }

  if (schema instanceof z.ZodString) {
    return { type: 'string', description: '' };
  }

  if (schema instanceof z.ZodNumber) {
    return { type: 'number', description: '' };
  }

  if (schema instanceof z.ZodBoolean) {
    return { type: 'boolean', description: '' };
  }

  if (schema instanceof z.ZodEnum) {
    const options = schema.options as string[];
    return { type: 'string', enum: options };
  }

  if (schema instanceof z.ZodArray) {
    return {
      type: 'array',
      items: zodToJsonSchema(schema.element as z.ZodType),
    };
  }

  if (schema instanceof z.ZodOptional) {
    return zodToJsonSchema(schema.unwrap() as z.ZodType);
  }

  return { type: 'string' };
}

// Tool registry สำหรับจัดการ tools
export class ToolRegistry {
  private tools: Map<string, {
    schema: z.ZodType;
    description: string;
    execute: (params: unknown) => Promise<unknown>;
  }> = new Map();

  register<T extends z.ZodType>(
    name: string,
    description: string,
    schema: T,
    execute: (params: z.infer<T>) => Promise<unknown>
  ): this {
    this.tools.set(name, {
      schema,
      description,
      execute: execute as (params: unknown) => Promise<unknown>,
    });
    return this;
  }

  toOpenAITools(): ChatCompletionTool[] {
    return Array.from(this.tools.entries()).map(([name, tool]) => ({
      type: 'function' as const,
      function: {
        name,
        description: tool.description,
        parameters: zodToJsonSchema(tool.schema),
      },
    }));
  }

  async executeToolCall(
    name: string,
    argsJson: string
  ): Promise<unknown> {
    const tool = this.tools.get(name);
    if (!tool) throw new Error(`Tool not found: ${name}`);

    const rawArgs = JSON.parse(argsJson) as unknown;
    const args = tool.schema.parse(rawArgs);
    return await tool.execute(args);
  }
}

// ตัวอย่าง tools
export function createDefaultToolRegistry(): ToolRegistry {
  return new ToolRegistry()
    .register(
      'get_current_time',
      'Get the current date and time',
      z.object({
        timezone: z.string().optional().default('Asia/Bangkok'),
      }),
      async ({ timezone }) => {
        return new Date().toLocaleString('th-TH', { timeZone: timezone });
      }
    )
    .register(
      'calculate',
      'Perform mathematical calculations',
      z.object({
        expression: z.string().describe('Mathematical expression to evaluate'),
      }),
      async ({ expression }) => {
        // ใน production ควรใช้ library ที่ปลอดภัยกว่า
        try {
          const result = Function(`"use strict"; return (${expression})`)() as number;
          return { result, expression };
        } catch {
          return { error: 'Invalid expression' };
        }
      }
    )
    .register(
      'search_web',
      'Search the web for information',
      z.object({
        query: z.string().describe('Search query'),
        maxResults: z.number().optional().default(5),
      }),
      async ({ query, maxResults }) => {
        // Mock implementation
        return {
          query,
          results: Array.from({ length: maxResults }, (_, i) => ({
            title: `Result ${i + 1} for: ${query}`,
            url: `https://example.com/${i + 1}`,
            snippet: `Snippet ${i + 1}...`,
          })),
        };
      }
    );
}
```

---

## 96.11 Building a Typed AI Assistant

```typescript
// src/assistant/ai-assistant.ts
import openai from '../config/openai';
import { ToolRegistry, createDefaultToolRegistry } from '../tools/tool-registry';
import type { ChatCompletionMessageParam } from 'openai/resources/chat/completions';

interface AssistantConfig {
  name: string;
  systemPrompt: string;
  model?: string;
  maxHistoryLength?: number;
  tools?: ToolRegistry;
}

interface Message {
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
}

interface AssistantSession {
  sessionId: string;
  messages: Message[];
  createdAt: Date;
}

export class TypedAIAssistant {
  private sessions: Map<string, AssistantSession> = new Map();
  private config: Required<AssistantConfig>;

  constructor(config: AssistantConfig) {
    this.config = {
      model: 'gpt-4o-mini',
      maxHistoryLength: 20,
      tools: createDefaultToolRegistry(),
      ...config,
    };
  }

  createSession(): string {
    const sessionId = crypto.randomUUID();
    this.sessions.set(sessionId, {
      sessionId,
      messages: [],
      createdAt: new Date(),
    });
    return sessionId;
  }

  getSession(sessionId: string): AssistantSession | undefined {
    return this.sessions.get(sessionId);
  }

  async chat(sessionId: string, userMessage: string): Promise<string> {
    const session = this.sessions.get(sessionId);
    if (!session) throw new Error(`Session not found: ${sessionId}`);

    // เพิ่ม user message
    session.messages.push({
      role: 'user',
      content: userMessage,
      timestamp: new Date(),
    });

    // สร้าง messages สำหรับ API
    const recentMessages = session.messages.slice(-this.config.maxHistoryLength);
    const apiMessages: ChatCompletionMessageParam[] = [
      { role: 'system', content: this.config.systemPrompt },
      ...recentMessages.map((m) => ({
        role: m.role,
        content: m.content,
      })),
    ];

    // เรียก API พร้อม tools
    const tools = this.config.tools.toOpenAITools();
    
    let finalResponse = '';
    const currentMessages = [...apiMessages];

    while (true) {
      const response = await openai.chat.completions.create({
        model: this.config.model,
        messages: currentMessages,
        tools: tools.length > 0 ? tools : undefined,
        tool_choice: tools.length > 0 ? 'auto' : undefined,
      });

      const choice = response.choices[0];
      if (!choice) break;

      const message = choice.message;
      currentMessages.push(message);

      if (choice.finish_reason === 'stop') {
        finalResponse = message.content ?? '';
        break;
      }

      if (choice.finish_reason === 'tool_calls' && message.tool_calls) {
        // Execute tool calls
        for (const toolCall of message.tool_calls) {
          try {
            const result = await this.config.tools.executeToolCall(
              toolCall.function.name,
              toolCall.function.arguments
            );

            currentMessages.push({
              role: 'tool',
              tool_call_id: toolCall.id,
              content: JSON.stringify(result),
            });
          } catch (error) {
            currentMessages.push({
              role: 'tool',
              tool_call_id: toolCall.id,
              content: JSON.stringify({ error: String(error) }),
            });
          }
        }
      }
    }

    // บันทึก assistant response
    session.messages.push({
      role: 'assistant',
      content: finalResponse,
      timestamp: new Date(),
    });

    return finalResponse;
  }

  clearSession(sessionId: string): void {
    const session = this.sessions.get(sessionId);
    if (session) {
      session.messages = [];
    }
  }

  deleteSession(sessionId: string): void {
    this.sessions.delete(sessionId);
  }
}

// ตัวอย่างการใช้งาน
async function assistantDemo() {
  const assistant = new TypedAIAssistant({
    name: 'TypeScript Tutor',
    systemPrompt: `คุณเป็น TypeScript tutor ที่เชี่ยวชาญ
    ตอบคำถามเป็นภาษาไทยเสมอ
    ใช้ตัวอย่างโค้ดเพื่อประกอบการอธิบาย
    เมื่อถูกถามเกี่ยวกับเวลาหรือการคำนวณ ให้ใช้ tools ที่มีให้`,
  });

  const sessionId = assistant.createSession();

  const responses = await Promise.all([
    assistant.chat(sessionId, 'สวัสดี! วันนี้วันที่เท่าไหร่?'),
  ]);

  const response2 = await assistant.chat(
    sessionId,
    'TypeScript interface กับ type alias ต่างกันอย่างไร?'
  );

  console.log('Response 1:', responses[0]);
  console.log('Response 2:', response2);
}
```

---

## 96.12 Multi-Modal AI กับ TypeScript

```typescript
// src/multimodal/vision-service.ts
import openai from '../config/openai';
import * as fs from 'fs';
import * as path from 'path';

interface ImageAnalysisResult {
  description: string;
  objects: string[];
  text: string | null;
  sentiment: 'positive' | 'negative' | 'neutral';
}

interface ImageInput {
  type: 'url' | 'base64' | 'filepath';
  value: string;
}

export class VisionService {
  async analyzeImage(
    image: ImageInput,
    question?: string
  ): Promise<string> {
    let imageUrl: string | undefined;
    let base64Data: string | undefined;

    switch (image.type) {
      case 'url':
        imageUrl = image.value;
        break;
      case 'filepath': {
        const fileContent = fs.readFileSync(image.value);
        base64Data = fileContent.toString('base64');
        const ext = path.extname(image.value).slice(1);
        base64Data = `data:image/${ext};base64,${base64Data}`;
        break;
      }
      case 'base64':
        base64Data = image.value;
        break;
    }

    const response = await openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: [
        {
          role: 'user',
          content: [
            {
              type: 'image_url',
              image_url: {
                url: imageUrl ?? base64Data ?? '',
                detail: 'auto',
              },
            },
            {
              type: 'text',
              text: question ?? 'อธิบายภาพนี้เป็นภาษาไทย',
            },
          ],
        },
      ],
      max_tokens: 1000,
    });

    return response.choices[0]?.message?.content ?? '';
  }

  async extractTextFromImage(image: ImageInput): Promise<string> {
    return this.analyzeImage(
      image,
      'Extract all text from this image. If no text found, say "ไม่พบข้อความ"'
    );
  }

  async compareImages(
    image1: ImageInput,
    image2: ImageInput
  ): Promise<string> {
    const toUrl = (img: ImageInput): string => {
      if (img.type === 'url') return img.value;
      if (img.type === 'filepath') {
        const content = fs.readFileSync(img.value);
        const base64 = content.toString('base64');
        const ext = path.extname(img.value).slice(1);
        return `data:image/${ext};base64,${base64}`;
      }
      return img.value;
    };

    const response = await openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: [
        {
          role: 'user',
          content: [
            { type: 'image_url', image_url: { url: toUrl(image1) } },
            { type: 'image_url', image_url: { url: toUrl(image2) } },
            {
              type: 'text',
              text: 'เปรียบเทียบภาพสองภาพนี้ บอกความเหมือนและความต่าง',
            },
          ],
        },
      ],
    });

    return response.choices[0]?.message?.content ?? '';
  }
}
```

---

## 96.13 Fine-tuning กับ TypeScript

```typescript
// src/fine-tuning/training-data.ts
import * as fs from 'fs';
import { z } from 'zod';

// Schema สำหรับ training data
const TrainingExampleSchema = z.object({
  messages: z.array(
    z.object({
      role: z.enum(['system', 'user', 'assistant']),
      content: z.string(),
    })
  ),
});

type TrainingExample = z.infer<typeof TrainingExampleSchema>;

export class TrainingDataManager {
  private examples: TrainingExample[] = [];

  addExample(example: TrainingExample): this {
    TrainingExampleSchema.parse(example); // validate
    this.examples.push(example);
    return this;
  }

  addConversationExample(
    systemPrompt: string,
    userMessage: string,
    assistantResponse: string
  ): this {
    return this.addExample({
      messages: [
        { role: 'system', content: systemPrompt },
        { role: 'user', content: userMessage },
        { role: 'assistant', content: assistantResponse },
      ],
    });
  }

  // Export เป็น JSONL format สำหรับ OpenAI fine-tuning
  exportToJSONL(filePath: string): void {
    const lines = this.examples.map((e) => JSON.stringify(e));
    fs.writeFileSync(filePath, lines.join('\n'), 'utf-8');
    console.log(`Exported ${this.examples.length} examples to ${filePath}`);
  }

  validate(): { valid: boolean; errors: string[] } {
    const errors: string[] = [];

    if (this.examples.length < 10) {
      errors.push('Need at least 10 examples for fine-tuning');
    }

    for (const [index, example] of this.examples.entries()) {
      const lastMessage = example.messages[example.messages.length - 1];
      if (lastMessage?.role !== 'assistant') {
        errors.push(`Example ${index + 1}: Last message must be from assistant`);
      }
    }

    return {
      valid: errors.length === 0,
      errors,
    };
  }
}

// ตัวอย่าง
const trainingManager = new TrainingDataManager();

trainingManager
  .addConversationExample(
    'คุณเป็น TypeScript expert ที่ตอบคำถามเป็นภาษาไทย',
    'TypeScript คืออะไร?',
    'TypeScript คือ programming language ที่พัฒนาโดย Microsoft เป็น superset ของ JavaScript ที่เพิ่ม static typing ทำให้โค้ดมีความปลอดภัยและ maintainable มากขึ้น'
  )
  .addConversationExample(
    'คุณเป็น TypeScript expert ที่ตอบคำถามเป็นภาษาไทย',
    'interface กับ type ต่างกันอย่างไร?',
    'Interface ใช้สำหรับ define shape ของ object และสามารถ extend ได้ง่าย ส่วน type alias ยืดหยุ่นกว่าสามารถใช้กับ union types, intersection types และ primitive types ได้ด้วย โดยทั่วไปใช้ interface สำหรับ object shapes และ type สำหรับ complex types'
  );

const validation = trainingManager.validate();
console.log('Valid:', validation.valid);
```

---

## 96.14 AI-Powered Code Analysis

```typescript
// src/code-analysis/code-analyzer.ts
import openai from '../config/openai';
import { z } from 'zod';

const CodeIssueSchema = z.object({
  severity: z.enum(['critical', 'high', 'medium', 'low', 'info']),
  category: z.enum(['bug', 'security', 'performance', 'style', 'maintainability']),
  line: z.number().optional(),
  message: z.string(),
  suggestion: z.string(),
});

const CodeAnalysisResultSchema = z.object({
  score: z.number().min(0).max(100),
  summary: z.string(),
  issues: z.array(CodeIssueSchema),
  positives: z.array(z.string()),
  refactoredCode: z.string().optional(),
});

type CodeAnalysisResult = z.infer<typeof CodeAnalysisResultSchema>;

export class AICodeAnalyzer {
  async analyze(
    code: string,
    options: {
      language?: string;
      includeRefactoring?: boolean;
      focusAreas?: string[];
    } = {}
  ): Promise<CodeAnalysisResult> {
    const {
      language = 'TypeScript',
      includeRefactoring = false,
      focusAreas = ['bugs', 'security', 'performance', 'style'],
    } = options;

    const systemPrompt = `คุณเป็น code reviewer ผู้เชี่ยวชาญที่วิเคราะห์ ${language} code
ตอบในรูปแบบ JSON เสมอ โดยให้:
- score: คะแนน 0-100
- summary: สรุปภาษาไทย
- issues: รายการปัญหา
- positives: สิ่งที่ทำได้ดี
${includeRefactoring ? '- refactoredCode: โค้ดที่ปรับปรุงแล้ว' : ''}

Focus on: ${focusAreas.join(', ')}`;

    const response = await openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: [
        { role: 'system', content: systemPrompt },
        {
          role: 'user',
          content: `Analyze this ${language} code:\n\`\`\`${language}\n${code}\n\`\`\``,
        },
      ],
      response_format: { type: 'json_object' },
      temperature: 0.1,
    });

    const content = response.choices[0]?.message?.content;
    if (!content) throw new Error('No analysis returned');

    const parsed = JSON.parse(content) as unknown;
    return CodeAnalysisResultSchema.parse(parsed);
  }

  async suggestTests(
    code: string,
    framework: 'jest' | 'vitest' = 'jest'
  ): Promise<string> {
    const response = await openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: [
        {
          role: 'system',
          content: `คุณเป็น test engineer ที่เขียน ${framework} tests สำหรับ TypeScript`,
        },
        {
          role: 'user',
          content: `เขียน unit tests สำหรับ code นี้:\n\`\`\`typescript\n${code}\n\`\`\`\n\nเขียน tests ที่ครอบคลุม edge cases และ happy paths`,
        },
      ],
      temperature: 0.3,
    });

    return response.choices[0]?.message?.content ?? '';
  }
}

// ตัวอย่างการใช้งาน
async function codeAnalysisDemo() {
  const analyzer = new AICodeAnalyzer();

  const sampleCode = `
function calculateDiscount(price: number, discountPercent: number) {
  const discount = price * discountPercent / 100;
  return price - discount;
}

function processOrder(items: any[]) {
  let total = 0;
  for (let i = 0; i <= items.length; i++) {
    total += items[i].price;
  }
  return total;
}
  `;

  const result = await analyzer.analyze(sampleCode, {
    includeRefactoring: true,
    focusAreas: ['bugs', 'style'],
  });

  console.log(`Score: ${result.score}/100`);
  console.log('Summary:', result.summary);
  console.log('\nIssues:');
  result.issues.forEach((issue) => {
    console.log(`  [${issue.severity}] ${issue.message}`);
    console.log(`    Suggestion: ${issue.suggestion}`);
  });
}
```

---

## 96.15 Batch Processing กับ AI

```typescript
// src/batch/batch-processor.ts
import openai from '../config/openai';

interface BatchItem<T> {
  id: string;
  input: T;
}

interface BatchResult<T, R> {
  id: string;
  input: T;
  output?: R;
  error?: string;
  duration: number;
}

export class AIBatchProcessor<TInput, TOutput> {
  constructor(
    private processItem: (input: TInput) => Promise<TOutput>,
    private options: {
      concurrency?: number;
      retries?: number;
      retryDelay?: number;
    } = {}
  ) {}

  async process(items: BatchItem<TInput>[]): Promise<BatchResult<TInput, TOutput>[]> {
    const {
      concurrency = 5,
      retries = 3,
      retryDelay = 1000,
    } = this.options;

    const results: BatchResult<TInput, TOutput>[] = [];
    
    // Process in batches of `concurrency`
    for (let i = 0; i < items.length; i += concurrency) {
      const batch = items.slice(i, i + concurrency);
      
      const batchResults = await Promise.all(
        batch.map((item) => this.processWithRetry(item, retries, retryDelay))
      );
      
      results.push(...batchResults);
      console.log(`Processed ${results.length}/${items.length} items`);
    }

    return results;
  }

  private async processWithRetry(
    item: BatchItem<TInput>,
    retries: number,
    retryDelay: number
  ): Promise<BatchResult<TInput, TOutput>> {
    const startTime = Date.now();
    
    for (let attempt = 0; attempt <= retries; attempt++) {
      try {
        const output = await this.processItem(item.input);
        return {
          id: item.id,
          input: item.input,
          output,
          duration: Date.now() - startTime,
        };
      } catch (error) {
        if (attempt === retries) {
          return {
            id: item.id,
            input: item.input,
            error: String(error),
            duration: Date.now() - startTime,
          };
        }
        
        // Wait before retry
        await new Promise((resolve) => setTimeout(resolve, retryDelay * (attempt + 1)));
      }
    }

    // Should never reach here but TypeScript needs it
    throw new Error('Unexpected end of retry loop');
  }
}

// ตัวอย่างการใช้งาน: แปลภาษาหลายข้อความ
async function batchTranslationDemo() {
  const translateText = async (text: string): Promise<string> => {
    const response = await openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: [
        {
          role: 'user',
          content: `Translate to Thai: "${text}"`,
        },
      ],
      max_tokens: 200,
    });
    return response.choices[0]?.message?.content ?? '';
  };

  const processor = new AIBatchProcessor<string, string>(translateText, {
    concurrency: 5,
    retries: 2,
  });

  const items = [
    { id: '1', input: 'Hello, World!' },
    { id: '2', input: 'TypeScript is awesome' },
    { id: '3', input: 'AI and programming go hand in hand' },
    { id: '4', input: 'Open source software changes the world' },
  ];

  const results = await processor.process(items);

  results.forEach((r) => {
    if (r.output) {
      console.log(`${r.id}: "${r.input}" → "${r.output}"`);
    } else {
      console.log(`${r.id}: Error - ${r.error}`);
    }
  });
}
```

---

## บทสรุป Part 96

ในบทนี้เราได้เรียนรู้:

1. **OpenAI SDK** - การใช้งาน API แบบ type-safe ด้วย TypeScript
2. **Streaming** - การรับ response แบบ real-time
3. **Function Calling** - การให้ LLM เรียกฟังก์ชันในโค้ด
4. **LangChain.js** - Framework สำหรับสร้าง LLM applications
5. **Vector Embeddings** - การแปลงข้อความเป็น vectors
6. **Vector Database** - การเก็บและค้นหา vectors ด้วย Pinecone
7. **RAG Pattern** - การเพิ่ม knowledge ให้ LLM ด้วยเอกสาร
8. **AI Agents** - การสร้าง agents ที่ทำงานอัตโนมัติ
9. **Prompt Templates** - การจัดการ prompts แบบ type-safe
10. **Tool Registry** - การจัดการ tools สำหรับ function calling
11. **Typed AI Assistant** - การสร้าง AI assistant ครบวงจร

TypeScript เหมาะมากสำหรับการพัฒนา AI applications เพราะ type system ช่วยจัดการกับ complex data structures ที่ LLMs ส่งกลับมาได้อย่างปลอดภัย

---

## แบบฝึกหัด

1. สร้าง AI-powered TypeScript linter ที่ตรวจจับ anti-patterns
2. สร้าง RAG system สำหรับเอกสาร TypeScript documentation
3. สร้าง multi-step agent ที่สามารถเขียนและ test โค้ดได้เอง
4. สร้าง AI code review tool ที่ integrate กับ GitHub PR
5. สร้าง AI-powered TypeScript type generator จาก JSON schema
