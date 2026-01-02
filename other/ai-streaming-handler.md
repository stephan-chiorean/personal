---
id: ai-streaming-handler
alias: Real-time AI Streaming Response Handler
type: kit
is_base: false
version: 1
tags:
  - ai
  - streaming
  - real-time
description: Production-ready SSE streaming handler for LLM responses with token tracking, cancellation, and error recovery
---

## End State

After applying this kit, the application will have:
- Server-Sent Events (SSE) streaming endpoint with proper headers and connection management
- Unified streaming adapter supporting OpenAI, Anthropic Claude, and other LLM providers
- Token-by-token streaming with real-time token usage tracking
- Stream cancellation via AbortController with proper cleanup
- Client-side React hook for consuming SSE streams
- Streaming UI component with typing indicators and smooth rendering
- Error recovery with exponential backoff retry logic
- Rate limiting middleware for streaming endpoints
- Stream buffering and backpressure handling
- Support for structured outputs (JSON mode) with partial parsing

## Implementation Principles

- Use Server-Sent Events (SSE) for simplicity and HTTP compatibility
- Implement provider abstraction layer for multi-LLM support
- Use AbortController for cancellation, store in Map keyed by request ID
- Track tokens incrementally using tiktoken or similar libraries
- Buffer chunks for network resilience, flush every 100ms or on newline
- Handle partial JSON by maintaining state and completing on stream end
- Implement exponential backoff (1s, 2s, 4s) for retries
- Use Redis or in-memory store for rate limiting with sliding window
- Support both text and structured output modes
- Log all stream events for debugging and analytics

## File Structure

```
src/
  lib/
    ai-streaming/
      server/
        stream-handler.ts          # Core SSE stream handler
        llm-adapter.ts              # Multi-provider LLM adapter
        token-tracker.ts            # Real-time token counting
        stream-manager.ts            # Active stream registry & cancellation
        rate-limiter.ts             # Rate limiting middleware
      client/
        use-ai-stream.ts            # React hook for consuming streams
        stream-client.ts             # SSE client wrapper
        stream-buffer.ts             # Client-side buffering
      types/
        stream-types.ts              # TypeScript interfaces
      utils/
        json-parser.ts               # Partial JSON parsing
        error-handler.ts             # Error recovery logic
  components/
    ai-streaming/
      StreamingMessage.tsx          # UI component for streaming text
      TypingIndicator.tsx           # Animated typing indicator
      StreamControls.tsx            # Cancel/retry controls
  api/
    chat/
      stream/
        route.ts                    # Next.js API route (or Express handler)
```

## Core Implementation

### Server-Side Stream Handler

```typescript
// stream-handler.ts
import { Readable } from 'stream';
import { EventEmitter } from 'events';

export interface StreamConfig {
  provider: 'openai' | 'anthropic' | 'custom';
  model: string;
  temperature?: number;
  maxTokens?: number;
  stream: true;
}

export interface StreamChunk {
  type: 'token' | 'done' | 'error' | 'metadata';
  content?: string;
  tokenCount?: number;
  error?: string;
  metadata?: Record<string, any>;
}

export class StreamHandler extends EventEmitter {
  private activeStreams = new Map<string, AbortController>();
  
  async createStream(
    requestId: string,
    messages: Array<{ role: string; content: string }>,
    config: StreamConfig
  ): Promise<ReadableStream
  ) {
    const controller = new AbortController();
    this.activeStreams.set(requestId, controller);
    
    try {
      const adapter = this.getAdapter(config.provider);
      const stream = await adapter.stream(messages, config, controller.signal);
      
      return this.transformStream(stream, requestId, config);
    } catch (error) {
      this.emit('error', { requestId, error });
      throw error;
    }
  }
  
  cancelStream(requestId: string): boolean {
    const controller = this.activeStreams.get(requestId);
    if (controller) {
      controller.abort();
      this.activeStreams.delete(requestId);
      return true;
    }
    return false;
  }
  
  private transformStream(
    stream: ReadableStream,
    requestId: string,
    config: StreamConfig
  ): ReadableStream {
    // Transform provider-specific format to unified StreamChunk format
    // Handle token counting, error recovery, etc.
  }
}
```

### LLM Adapter (Multi-Provider)

```typescript
// llm-adapter.ts
export class LLMAdapter {
  async stream(
    messages: Message[],
    config: StreamConfig,
    signal: AbortSignal
  ): Promise<ReadableStream> {
    switch (config.provider) {
      case 'openai':
        return this.streamOpenAI(messages, config, signal);
      case 'anthropic':
        return this.streamAnthropic(messages, config, signal);
      default:
        throw new Error(`Unsupported provider: ${config.provider}`);
    }
  }
  
  private async streamOpenAI(
    messages: Message[],
    config: StreamConfig,
    signal: AbortSignal
  ): Promise<ReadableStream> {
    const response = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: config.model,
        messages,
        temperature: config.temperature,
        max_tokens: config.maxTokens,
        stream: true,
      }),
      signal,
    });
    
    return response.body!;
  }
  
  private async streamAnthropic(
    messages: Message[],
    config: StreamConfig,
    signal: AbortSignal
  ): Promise<ReadableStream> {
    // Similar implementation for Anthropic
  }
}
```

### Client-Side React Hook

```typescript
// use-ai-stream.ts
import { useState, useEffect, useRef, useCallback } from 'react';
import { EventSource } from 'eventsource';

export interface UseAIStreamOptions {
  endpoint: string;
  onComplete?: (fullText: string) => void;
  onError?: (error: Error) => void;
  onToken?: (token: string) => void;
}

export function useAIStream(options: UseAIStreamOptions) {
  const [streaming, setStreaming] = useState(false);
  const [text, setText] = useState('');
  const [error, setError] = useState<Error | null>(null);
  const [tokenCount, setTokenCount] = useState(0);
  const eventSourceRef = useRef<EventSource | null>(null);
  const abortControllerRef = useRef<AbortController | null>(null);
  
  const startStream = useCallback(async (
    messages: Message[],
    config: StreamConfig
  ) => {
    setStreaming(true);
    setText('');
    setError(null);
    
    const controller = new AbortController();
    abortControllerRef.current = controller;
    
    try {
      const response = await fetch(options.endpoint, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ messages, config }),
        signal: controller.signal,
      });
      
      const reader = response.body?.getReader();
      const decoder = new TextDecoder();
      
      if (!reader) throw new Error('No response body');
      
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        
        const chunk = decoder.decode(value, { stream: true });
        const lines = chunk.split('\n').filter(Boolean);
        
        for (const line of lines) {
          if (line.startsWith('data: ')) {
            const data = JSON.parse(line.slice(6));
            
            if (data.type === 'token') {
              setText(prev => prev + data.content);
              options.onToken?.(data.content);
            } else if (data.type === 'metadata') {
              setTokenCount(data.tokenCount || 0);
            } else if (data.type === 'done') {
              setStreaming(false);
              options.onComplete?.(text);
            } else if (data.type === 'error') {
              throw new Error(data.error);
            }
          }
        }
      }
    } catch (err) {
      if (err instanceof Error && err.name !== 'AbortError') {
        setError(err);
        options.onError?.(err);
      }
      setStreaming(false);
    }
  }, [options]);
  
  const cancel = useCallback(() => {
    abortControllerRef.current?.abort();
    setStreaming(false);
  }, []);
  
  useEffect(() => {
    return () => {
      cancel();
    };
  }, [cancel]);
  
  return {
    streaming,
    text,
    error,
    tokenCount,
    startStream,
    cancel,
  };
}
```

### Streaming UI Component

```typescript
// StreamingMessage.tsx
import { useEffect, useRef } from 'react';
import { useAIStream } from '@/lib/ai-streaming/client/use-ai-stream';

export function StreamingMessage({ 
  messages, 
  config 
}: { 
  messages: Message[];
  config: StreamConfig;
}) {
  const { streaming, text, error, tokenCount, startStream, cancel } = useAIStream({
    endpoint: '/api/chat/stream',
    onComplete: (fullText) => console.log('Complete:', fullText),
  });
  
  const textRef = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    if (textRef.current) {
      textRef.current.scrollTop = textRef.current.scrollHeight;
    }
  }, [text]);
  
  return (
    <div className="streaming-message">
      <div ref={textRef} className="message-content">
        {text}
        {streaming && <TypingIndicator />}
      </div>
      
      <div className="stream-controls">
        <div className="token-count">Tokens: {tokenCount}</div>
        {streaming && (
          <button onClick={cancel}>Cancel</button>
        )}
        {!streaming && (
          <button onClick={() => startStream(messages, config)}>Start</button>
        )}
      </div>
      
      {error && (
        <div className="error">{error.message}</div>
      )}
    </div>
  );
}
```

### API Route (Next.js Example)

```typescript
// api/chat/stream/route.ts
import { NextRequest } from 'next/server';
import { StreamHandler } from '@/lib/ai-streaming/server/stream-handler';
import { rateLimiter } from '@/lib/ai-streaming/server/rate-limiter';

const streamHandler = new StreamHandler();

export async function POST(req: NextRequest) {
  // Rate limiting
  const clientId = req.headers.get('x-client-id') || 'anonymous';
  if (!await rateLimiter.check(clientId)) {
    return new Response('Rate limit exceeded', { status: 429 });
  }
  
  const { messages, config } = await req.json();
  const requestId = crypto.randomUUID();
  
  // Create SSE response
  const stream = new ReadableStream({
    async start(controller) {
      try {
        const llmStream = await streamHandler.createStream(
          requestId,
          messages,
          config
        );
        
        const reader = llmStream.getReader();
        const encoder = new TextEncoder();
        
        while (true) {
          const { done, value } = await reader.read();
          if (done) {
            controller.enqueue(encoder.encode('data: {"type":"done"}\n\n'));
            controller.close();
            break;
          }
          
          // Transform and send chunk
          const chunk = JSON.stringify({
            type: 'token',
            content: value,
          });
          controller.enqueue(encoder.encode(`data: ${chunk}\n\n`));
        }
      } catch (error) {
        const errorChunk = JSON.stringify({
          type: 'error',
          error: error instanceof Error ? error.message : 'Unknown error',
        });
        controller.enqueue(encoder.encode(`data: ${errorChunk}\n\n`));
        controller.close();
      }
    },
    
    cancel() {
      streamHandler.cancelStream(requestId);
    },
  });
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
      'X-Accel-Buffering': 'no', // Disable nginx buffering
    },
  });
}
```

## Dependencies

```json
{
  "dependencies": {
    "tiktoken": "^1.0.0",           // Token counting
    "eventsource": "^2.0.0"          // Client-side SSE (if needed)
  },
  "devDependencies": {
    "@types/eventsource": "^1.1.0"
  }
}
```

## Environment Variables

```env
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
REDIS_URL=redis://...  # For rate limiting (optional)
```

## Usage Example

```typescript
// In your component
import { StreamingMessage } from '@/components/ai-streaming/StreamingMessage';

function ChatInterface() {
  const [messages, setMessages] = useState<Message[]>([
    { role: 'user', content: 'Hello!' }
  ]);
  
  return (
    <StreamingMessage
      messages={messages}
      config={{
        provider: 'openai',
        model: 'gpt-4',
        temperature: 0.7,
        stream: true,
      }}
    />
  );
}
```

## Verification Criteria

After generation, verify:
- ✓ SSE endpoint returns proper headers and streams tokens in real-time
- ✓ Client hook receives and displays streaming text correctly
- ✓ Stream cancellation via AbortController works and cleans up resources
- ✓ Token counting tracks usage accurately during streaming
- ✓ Error recovery retries failed streams with exponential backoff
- ✓ Rate limiting prevents abuse (test with rapid requests)
- ✓ Multiple concurrent streams work without interference
- ✓ Partial JSON parsing works for structured outputs
- ✓ Network interruptions are handled gracefully
- ✓ Stream cleanup happens on component unmount

