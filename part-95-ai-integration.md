# Part 95 | ขั้นตอนที่ 1701-1720 จาก 1000+

## AI/LLM Integration ใน Node.js

ในส่วนนี้เราจะเรียนรู้การผสาน AI และ Large Language Models เข้ากับ Node.js applications รวมถึง OpenAI API, LangChain.js, RAG system, และ embeddings

---

## ขั้นตอนที่ 1701: OpenAI API Integration

```javascript
// openai-service.js
// Production-ready OpenAI integration

const OpenAI = require('openai');

class AIService {
  constructor(config = {}) {
    this.client = new OpenAI({
      apiKey: config.apiKey || process.env.OPENAI_API_KEY,
      maxRetries: config.maxRetries || 3,
      timeout: config.timeout || 30000
    });
    
    this.defaultModel = config.model || 'gpt-4o-mini';
    this.tokenTracker = new TokenUsageTracker();
  }

  async chat(messages, options = {}) {
    const startTime = Date.now();
    
    try {
      const response = await this.client.chat.completions.create({
        model: options.model || this.defaultModel,
        messages,
        temperature: options.temperature ?? 0.7,
        max_tokens: options.maxTokens || 1000,
        stream: false,
        ...options.functions && {
          tools: options.functions.map(fn => ({ type: 'function', function: fn })),
          tool_choice: 'auto'
        }
      });
      
      const usage = response.usage;
      this.tokenTracker.track(usage.prompt_tokens, usage.completion_tokens);
      
      const latency = Date.now() - startTime;
      console.log(`AI response: ${latency}ms, tokens: ${usage.total_tokens}`);
      
      const choice = response.choices[0];
      
      // Handle function calls
      if (choice.finish_reason === 'tool_calls') {
        return {
          type: 'tool_calls',
          toolCalls: choice.message.tool_calls,
          message: choice.message
        };
      }
      
      return {
        type: 'text',
        content: choice.message.content,
        usage
      };
      
    } catch (err) {
      if (err instanceof OpenAI.APIError) {
        throw this.handleAPIError(err);
      }
      throw err;
    }
  }

  // Streaming response
  async *chatStream(messages, options = {}) {
    const stream = await this.client.chat.completions.create({
      model: options.model || this.defaultModel,
      messages,
      stream: true,
      temperature: options.temperature ?? 0.7
    });
    
    let totalTokens = 0;
    
    for await (const chunk of stream) {
      const delta = chunk.choices[0]?.delta;
      if (delta?.content) {
        yield { type: 'content', content: delta.content };
        totalTokens++;
      }
      
      if (chunk.choices[0]?.finish_reason === 'stop') {
        yield { type: 'done', totalTokens };
      }
    }
  }

  async embed(text, model = 'text-embedding-3-small') {
    const response = await this.client.embeddings.create({
      model,
      input: Array.isArray(text) ? text : [text]
    });
    
    return response.data.map(item => item.embedding);
  }

  handleAPIError(err) {
    const errors = {
      429: new Error('Rate limit exceeded. Please retry after a moment.'),
      503: new Error('OpenAI service is temporarily unavailable.'),
      400: new Error(`Invalid request: ${err.message}`),
      401: new Error('Invalid API key.')
    };
    return errors[err.status] || new Error(`AI API error: ${err.message}`);
  }
}

class TokenUsageTracker {
  constructor() {
    this.daily = { prompt: 0, completion: 0, date: new Date().toDateString() };
    this.total = { prompt: 0, completion: 0 };
  }

  track(promptTokens, completionTokens) {
    const today = new Date().toDateString();
    if (this.daily.date !== today) {
      this.daily = { prompt: 0, completion: 0, date: today };
    }
    
    this.daily.prompt += promptTokens;
    this.daily.completion += completionTokens;
    this.total.prompt += promptTokens;
    this.total.completion += completionTokens;
  }

  getStats() {
    return {
      daily: { ...this.daily, total: this.daily.prompt + this.daily.completion },
      total: { ...this.total, total: this.total.prompt + this.total.completion },
      estimatedCost: this.estimateCost()
    };
  }

  estimateCost() {
    // GPT-4o-mini pricing (approximate)
    const promptCost = (this.total.prompt / 1_000_000) * 0.15;
    const completionCost = (this.total.completion / 1_000_000) * 0.60;
    return `$${(promptCost + completionCost).toFixed(4)}`;
  }
}

module.exports = { AIService, TokenUsageTracker };
```

---

## ขั้นตอนที่ 1702: RAG System (Retrieval-Augmented Generation)

```javascript
// rag-system.js
// Production RAG with vector database

const { AIService } = require('./openai-service');
const { Pool } = require('pg');
const pgvector = require('pgvector/pg');

class RAGSystem {
  constructor(config) {
    this.ai = new AIService(config.ai);
    this.db = new Pool(config.database);
    this.chunkSize = config.chunkSize || 1000;
    this.chunkOverlap = config.chunkOverlap || 200;
  }

  async initialize() {
    await pgvector.registerType(this.db);
    await this.db.query(`
      CREATE TABLE IF NOT EXISTS documents (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        content TEXT NOT NULL,
        metadata JSONB DEFAULT '{}',
        embedding vector(1536),
        created_at TIMESTAMPTZ DEFAULT NOW()
      );
      CREATE INDEX IF NOT EXISTS documents_embedding_idx 
        ON documents USING ivfflat (embedding vector_cosine_ops) 
        WITH (lists = 100);
    `);
  }

  // Ingest document into vector store
  async ingestDocument(content, metadata = {}) {
    const chunks = this.splitIntoChunks(content);
    console.log(`Ingesting ${chunks.length} chunks from document`);
    
    const results = [];
    
    for (const chunk of chunks) {
      const [embedding] = await this.ai.embed(chunk);
      
      const { rows } = await this.db.query(
        `INSERT INTO documents (content, metadata, embedding) VALUES ($1, $2, $3) RETURNING id`,
        [chunk, JSON.stringify(metadata), pgvector.toSql(embedding)]
      );
      
      results.push(rows[0].id);
    }
    
    return results;
  }

  // Search similar documents
  async searchSimilar(query, options = {}) {
    const [queryEmbedding] = await this.ai.embed(query);
    const limit = options.limit || 5;
    const threshold = options.threshold || 0.7;
    
    const { rows } = await this.db.query(`
      SELECT 
        id,
        content,
        metadata,
        1 - (embedding <=> $1::vector) as similarity
      FROM documents
      WHERE 1 - (embedding <=> $1::vector) > $2
      ORDER BY similarity DESC
      LIMIT $3
    `, [pgvector.toSql(queryEmbedding), threshold, limit]);
    
    return rows;
  }

  // RAG Query - combine retrieval with generation
  async query(userQuery, options = {}) {
    // 1. Retrieve relevant documents
    const relevantDocs = await this.searchSimilar(userQuery, {
      limit: options.contextDocs || 5,
      threshold: options.threshold || 0.6
    });
    
    if (relevantDocs.length === 0) {
      // No relevant context - answer from model knowledge
      const response = await this.ai.chat([
        { role: 'user', content: userQuery }
      ]);
      return { answer: response.content, sources: [], usedContext: false };
    }
    
    // 2. Build context from retrieved docs
    const context = relevantDocs
      .map((doc, i) => `[${i + 1}] ${doc.content}`)
      .join('\n\n');
    
    // 3. Generate answer with context
    const systemPrompt = `You are a helpful assistant. Answer questions based on the provided context.
If the context doesn't contain enough information, say so.
Always cite your sources using the reference numbers [1], [2], etc.`;
    
    const response = await this.ai.chat([
      { role: 'system', content: systemPrompt },
      {
        role: 'user',
        content: `Context:\n${context}\n\nQuestion: ${userQuery}`
      }
    ], options);
    
    return {
      answer: response.content,
      sources: relevantDocs.map(doc => ({
        id: doc.id,
        similarity: doc.similarity,
        metadata: doc.metadata,
        excerpt: doc.content.substring(0, 200) + '...'
      })),
      usedContext: true
    };
  }

  // Split text into overlapping chunks
  splitIntoChunks(text) {
    const chunks = [];
    let start = 0;
    
    while (start < text.length) {
      const end = start + this.chunkSize;
      chunks.push(text.slice(start, end));
      start += this.chunkSize - this.chunkOverlap;
    }
    
    return chunks.filter(c => c.trim().length > 50);
  }
}

module.exports = { RAGSystem };
```

---

## ขั้นตอนที่ 1703: LangChain.js Integration

```javascript
// langchain-agent.js
// AI Agent with tools using LangChain.js

const { ChatOpenAI } = require('@langchain/openai');
const { AgentExecutor, createOpenAIFunctionsAgent } = require('langchain/agents');
const { DynamicTool } = require('@langchain/core/tools');
const { ChatPromptTemplate, MessagesPlaceholder } = require('@langchain/core/prompts');
const { HumanMessage, AIMessage } = require('@langchain/core/messages');

class AIAgent {
  constructor(config) {
    this.llm = new ChatOpenAI({
      modelName: config.model || 'gpt-4o-mini',
      temperature: config.temperature || 0,
      openAIApiKey: config.apiKey || process.env.OPENAI_API_KEY
    });
    
    this.tools = this.createTools(config.tools || {});
    this.conversationHistory = new Map(); // session -> messages
  }

  createTools(toolConfig) {
    const tools = [];
    
    // Weather tool
    if (toolConfig.weather) {
      tools.push(new DynamicTool({
        name: 'get_weather',
        description: 'Get current weather for a city. Input should be city name.',
        func: async (city) => {
          const response = await fetch(`https://api.openweathermap.org/data/2.5/weather?q=${city}&appid=${toolConfig.weather.apiKey}&units=metric`);
          const data = await response.json();
          return JSON.stringify({
            city: data.name,
            temperature: data.main?.temp,
            description: data.weather?.[0]?.description
          });
        }
      }));
    }
    
    // Database query tool
    if (toolConfig.database) {
      const db = toolConfig.database;
      tools.push(new DynamicTool({
        name: 'query_database',
        description: 'Query the database. Input should be the query intent in natural language.',
        func: async (intent) => {
          // Simplified - would use proper query builder
          const allowedQueries = {
            'user count': 'SELECT COUNT(*) as count FROM users',
            'recent orders': 'SELECT id, total, created_at FROM orders ORDER BY created_at DESC LIMIT 5'
          };
          
          const sql = allowedQueries[intent.toLowerCase()];
          if (!sql) return 'Query not allowed';
          
          const result = await db.query(sql);
          return JSON.stringify(result.rows);
        }
      }));
    }
    
    return tools;
  }

  async createExecutor() {
    const prompt = ChatPromptTemplate.fromMessages([
      ['system', 'You are a helpful assistant. Use tools when needed to answer accurately.'],
      new MessagesPlaceholder('chat_history'),
      ['human', '{input}'],
      new MessagesPlaceholder('agent_scratchpad')
    ]);
    
    const agent = await createOpenAIFunctionsAgent({
      llm: this.llm,
      tools: this.tools,
      prompt
    });
    
    return new AgentExecutor({
      agent,
      tools: this.tools,
      verbose: process.env.NODE_ENV === 'development',
      maxIterations: 5
    });
  }

  async chat(sessionId, userMessage) {
    if (!this.executor) {
      this.executor = await this.createExecutor();
    }
    
    const history = this.conversationHistory.get(sessionId) || [];
    
    const result = await this.executor.invoke({
      input: userMessage,
      chat_history: history
    });
    
    // Update history
    history.push(new HumanMessage(userMessage));
    history.push(new AIMessage(result.output));
    
    // Keep last 20 messages
    if (history.length > 20) {
      history.splice(0, history.length - 20);
    }
    
    this.conversationHistory.set(sessionId, history);
    
    return {
      response: result.output,
      sessionId,
      messageCount: history.length
    };
  }

  clearSession(sessionId) {
    this.conversationHistory.delete(sessionId);
  }
}

// Express API for AI Chat
const express = require('express');
const app = express();
app.use(express.json());

const agent = new AIAgent({
  model: 'gpt-4o-mini',
  temperature: 0.7
});

app.post('/chat', async (req, res) => {
  const { sessionId, message } = req.body;
  
  if (!message) {
    return res.status(400).json({ error: 'Message required' });
  }
  
  try {
    const result = await agent.chat(sessionId || 'default', message);
    res.json(result);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Streaming endpoint
app.post('/chat/stream', async (req, res) => {
  const { message } = req.body;
  
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  
  const aiService = new (require('./openai-service').AIService)();
  
  try {
    for await (const chunk of aiService.chatStream([{ role: 'user', content: message }])) {
      if (chunk.type === 'content') {
        res.write(`data: ${JSON.stringify({ content: chunk.content })}\n\n`);
      } else if (chunk.type === 'done') {
        res.write(`data: ${JSON.stringify({ done: true })}\n\n`);
        res.end();
      }
    }
  } catch (err) {
    res.write(`data: ${JSON.stringify({ error: err.message })}\n\n`);
    res.end();
  }
});

module.exports = { AIAgent };
```

---

## ขั้นตอนที่ 1704-1720: AI Function Calling และ Structured Output

```javascript
// ai-structured.js
// Structured output และ function calling

const { AIService } = require('./openai-service');
const { z } = require('zod');

class StructuredAI {
  constructor() {
    this.ai = new AIService();
  }

  // Extract structured data from text
  async extract(text, schema) {
    const schemaDescription = this.zodToDescription(schema);
    
    const response = await this.ai.chat([
      {
        role: 'system',
        content: `Extract information from text and return it as JSON matching this schema:\n${schemaDescription}\nReturn ONLY valid JSON.`
      },
      { role: 'user', content: text }
    ], { temperature: 0 });
    
    try {
      const parsed = JSON.parse(response.content);
      return schema.parse(parsed); // Validate with Zod
    } catch (err) {
      throw new Error(`Failed to extract structured data: ${err.message}`);
    }
  }

  zodToDescription(schema) {
    return JSON.stringify(schema._def, null, 2);
  }
}

// Function Calling example
class CustomerSupportAI {
  constructor(orderService, customerService) {
    this.ai = new AIService();
    this.orderService = orderService;
    this.customerService = customerService;
  }

  get functions() {
    return [
      {
        name: 'get_order_status',
        description: 'Get the status of a customer order',
        parameters: {
          type: 'object',
          properties: {
            orderId: { type: 'string', description: 'The order ID' }
          },
          required: ['orderId']
        }
      },
      {
        name: 'cancel_order',
        description: 'Cancel a customer order',
        parameters: {
          type: 'object',
          properties: {
            orderId: { type: 'string' },
            reason: { type: 'string', description: 'Cancellation reason' }
          },
          required: ['orderId', 'reason']
        }
      },
      {
        name: 'get_customer_info',
        description: 'Get customer information',
        parameters: {
          type: 'object',
          properties: {
            customerId: { type: 'string' }
          },
          required: ['customerId']
        }
      }
    ];
  }

  async executeFunctionCall(name, args) {
    switch (name) {
      case 'get_order_status':
        return this.orderService.getStatus(args.orderId);
      case 'cancel_order':
        return this.orderService.cancel(args.orderId, args.reason);
      case 'get_customer_info':
        return this.customerService.getCustomer(args.customerId);
      default:
        throw new Error(`Unknown function: ${name}`);
    }
  }

  async handleCustomerQuery(customerId, query) {
    const messages = [
      {
        role: 'system',
        content: `You are a helpful customer support agent. The current customer ID is ${customerId}. Use available functions to help customers.`
      },
      { role: 'user', content: query }
    ];

    let response = await this.ai.chat(messages, { functions: this.functions });

    // Handle function calls in a loop
    while (response.type === 'tool_calls') {
      const toolResults = [];
      
      for (const toolCall of response.toolCalls) {
        const args = JSON.parse(toolCall.function.arguments);
        const result = await this.executeFunctionCall(toolCall.function.name, args);
        
        toolResults.push({
          role: 'tool',
          tool_call_id: toolCall.id,
          content: JSON.stringify(result)
        });
      }
      
      messages.push(response.message);
      messages.push(...toolResults);
      
      response = await this.ai.chat(messages, { functions: this.functions });
    }

    return response.content;
  }
}

module.exports = { StructuredAI, CustomerSupportAI };
```

---

## แบบฝึกหัด

### Exercise 1: RAG with PDF
สร้าง RAG system ที่ ingest PDF documents และตอบคำถามเกี่ยวกับเนื้อหาใน PDF

### Exercise 2: AI Code Review
สร้าง GitHub webhook ที่ใช้ AI review pull requests และ comment แนะนำการปรับปรุง

### Exercise 3: Semantic Search
สร้าง search engine ที่ใช้ embeddings แทน keyword matching

### คำถามทบทวน
1. RAG แตกต่างจากการ fine-tune model อย่างไร?
2. Token usage tracking สำคัญอย่างไรใน production?
3. Function calling เหมาะกับ use case ใดบ้าง?

---

*ต่อไป: Part 96 - Real-time Analytics*
