# What is Prompt Caching? Best Practices Explained

> Source: https://apidog.com/blog/what-is-pro... (apidog blog)
> Collected: 2026-05-08
> Published: Unknown（飞书剪存时间 2025-09-10）

Large Language Models (LLMs) have revolutionized how we interact with AI, enabling complex tasks like text generation, translation, question answering, and more. However, interacting with these powerful models, especially with sophisticated prompts, can incur significant computational costs and latency. Many applications involve sending similar or partially identical prompts repeatedly. Imagine a chatbot with a fixed system prompt, a document analysis tool processing chunks with the same instructions, or an agent using consistent tool definitions. In these scenarios, the LLM repeatedly processes the same initial information (the prompt prefix), wasting computation and increasing response times.

Prompt caching emerges as a powerful optimization technique to address this inefficiency. It allows LLM providers to store the intermediate computational state associated with the initial, static portion of a prompt (the prefix). When subsequent requests use the same prefix, the model can reuse this cached state, skipping the redundant computation and processing only the new, dynamic part of the prompt (the suffix). This leads to substantial improvements in both latency and cost, making LLM applications faster and more economical.

This guide provides a comprehensive overview of prompt caching, how it works, its benefits, implementation details (focusing on Anthropic's API, which is also relevant for Claude models on AWS Bedrock), pricing considerations, limitations, and best practices.

## How Prompt Caching Works: The Mechanism

At its core, prompt caching exploits the repetitive nature of many LLM interactions. When you send a prompt to an LLM, the model processes the input tokens sequentially to generate an internal representation or state. Prompt caching intercepts this process.

1. **Cache Miss (First Request / Changed Prefix)**: When a request arrives with a prompt prefix that hasn't been seen recently or matches no existing cache entry, the LLM processes the entire prompt as usual. However, if caching is enabled for a specific prefix boundary, the system stores the internal model state corresponding to that prefix after processing it. This is often referred to as a *cache write*. A unique identifier, typically a cryptographic hash of the prefix content (including system prompts, tool definitions, and messages up to the cache point), is associated with this stored state, acting as the cache key.
2. **Cache Hit (Subsequent Request)**: If a subsequent request arrives within the cache's Time-To-Live (TTL) and its prefix exactly matches the content associated with a stored cache key, the system retrieves the saved internal state. The LLM effectively fast-forwards to the end of the prefix without reprocessing it. It then only needs to process the new part of the prompt (the suffix). This is known as a *cache read* or *cache hit*.

### The Cached Prefix

What exactly constitutes the "prefix" depends on the API and how you structure your request. Generally, it includes the static parts of your prompt that you intend to reuse across calls. For instance, using Anthropic's API structure as a reference:

- **Tools**: Definitions of available tools/functions.
- **System Prompt**: High-level instructions or context for the model.
- **Messages**: Initial user/assistant messages, such as few-shot examples or conversation history.

The order typically matters (e.g., Anthropic processes `tools`, then `system`, then `messages`). You designate where the cacheable prefix ends using specific API parameters.

### Cache Characteristics

- **Lifetime (TTL)**: Caches are not permanent. They have a TTL. For example, Anthropic's cache has a minimum 5-minute TTL, which is refreshed each time the cache entry is accessed (a cache hit). If a cache entry isn't used within its TTL, it expires and will need to be recreated on the next relevant request. Currently, most implementations offer "ephemeral" caching with relatively short lifetimes.
- **Scope and Privacy**: Caching is designed with privacy in mind. Caches are typically segregated at the organization or account level. Even if two different organizations send the exact same prompt prefix, they will not share a cache entry. Within an organization, users sending identical prefixes can potentially share a cache entry, improving efficiency for team applications. The use of cryptographic hashes for keys ensures that only requests with identical prefixes can access a specific cache.
- **Exact Matching**: Caching relies on exact matches of the prefix content. Any change, however small (even whitespace), within the designated prefix will result in a cache miss and necessitate a new cache write.

## Why Use Prompt Caching?

Implementing prompt caching offers significant advantages, primarily focused on performance and cost efficiency.

1. **Reduced Latency**: This is often the most immediate benefit. By skipping the reprocessing of potentially thousands of prefix tokens, the LLM can begin processing the relevant new information (the prompt suffix) much faster. This directly translates to quicker response times for the end-user. For applications requiring real-time interaction, like chatbots or code assistants, this speed-up is crucial. AWS Bedrock reports potential latency reductions of up to 85% for supported models.
2. **Reduced Cost**: LLM APIs typically charge based on the number of input and output tokens processed. When a cache hit occurs, you are often charged a significantly lower rate for the tokens read from the cache compared to the standard input token rate. You only pay the standard input rate for the new tokens in the suffix (and potentially a slightly higher rate for the initial cache write). Over many calls with large, static prefixes, this can lead to substantial cost savings. AWS Bedrock suggests cost reductions of up to 90% are possible.
3. **Optimized Performance for Common Use Cases**: Caching is particularly impactful for applications inherently involving prompt repetition:
   - **Retrieval-Augmented Generation (RAG) / Document Q&A**: Large documents or context snippets are often fed repeatedly as part of the prompt while only the user's question changes. Caching the document context significantly speeds up Q&A over that document.
   - **Few-Shot Prompting**: Providing multiple examples within the prompt (few-shot learning) improves model performance but increases prompt length. Caching these static examples avoids reprocessing them for every new query.
   - **Agentic Workflows**: AI agents often rely on complex system prompts, detailed instructions, and a fixed set of tool definitions. Caching these constant elements accelerates task execution, especially in multi-step processes.
   - **Chatbots / Multi-Turn Conversations**: While the conversation history grows, the initial system prompt and instructions often remain the same. Caching the system prompt, and potentially incrementally caching turns of the conversation, keeps the interaction snappy even as the context window fills.
   - **Code Assistants**: Static code context, library documentation, or boilerplate instructions can be cached, allowing the assistant to focus computation on the user's specific coding query.
4. **Seamless Integration**: Prompt caching is designed to work alongside other LLM features. For example, it integrates with AWS Bedrock features like Agents and Guardrails, allowing you to leverage complex setups without incurring the full latency penalty for repeated components.

## How to Enable Prompt Caching

The exact method for enabling prompt caching varies slightly between LLM providers and their APIs. Here, we'll focus on the implementation using Anthropic's Messages API, which is applicable to Claude models directly via Anthropic or through platforms like AWS Bedrock.

### General Principle: Structure Your Prompt

The key is to structure your API calls so that the static, reusable content appears first, followed by the dynamic content.

```
+-------------------------+--------------------------+
|      STATIC PREFIX      |     DYNAMIC SUFFIX       |
| (System Prompt, Tools,  | (New User Query, Latest  |
|  Few-Shot Examples,     |  Conversation Turn, etc.)|
|  Initial Context)       |                          |
+-------------------------+--------------------------+
            ^
            |
    Cache Breakpoint Here
```

### Anthropic API Implementation (`cache_control`)

Anthropic uses the `cache_control` parameter within the Messages API request body to enable and manage caching.

- **Enabling**: Include a `cache_control` object within one of the blocks you want to include as the end of your cacheable prefix. Currently, the only supported type is `"ephemeral"`.
- **Placement**: You can place the `cache_control` block on:
  - The system prompt message.
  - A specific message object (user or assistant turn).
  - A tool definition block (less common, but possible).
- **Cache Creation Order**: The cache prefix is constructed based on the content up to and including the block marked with `cache_control`, following the standard order: `tools` → `system` → `messages`.
- **Multiple Breakpoints**: You can define up to 4 cache breakpoints in a single request by adding `cache_control` to multiple blocks. The system will attempt to find the longest matching cached prefix based on these potential breakpoints.

**Example (Anthropic Python SDK): Caching a System Prompt**

```python
import anthropic

client = anthropic.Anthropic(api_key="YOUR_API_KEY")

# First Request (Cache Write)
response1 = client.messages.create(
    model="claude-3-5-sonnet-20240620",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            # This system prompt is the content we want to cache
            "text": "You are a helpful assistant specializing in astrophysics. Your knowledge base includes extensive details about stellar evolution, cosmology, and planetary science. Respond accurately and concisely.",
            "cache_control": {"type": "ephemeral"}  # Mark for caching
        }
    ],
    messages=[
        {
            "role": "user",
            "content": "What is the Chandrasekhar limit?"
        }
    ]
)
print("First Response:", response1.content)
print("Usage (Write):", response1.usage)
# Example: Usage(input_tokens=60, output_tokens=50,
#                cache_creation_input_tokens=60, cache_read_input_tokens=0)

# Subsequent Request (Cache Hit - within TTL, e.g., < 5 mins later)
response2 = client.messages.create(
    model="claude-3-5-sonnet-20240620",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            # EXACTLY the same system prompt as before
            "text": "You are a helpful assistant specializing in astrophysics. Your knowledge base includes extensive details about stellar evolution, cosmology, and planetary science. Respond accurately and concisely.",
            "cache_control": {"type": "ephemeral"}
        }
    ],
    messages=[
        {
            "role": "user",
            "content": "Explain the concept of dark energy."
        }
    ]
)
print("Second Response:", response2.content)
print("Usage (Hit):", response2.usage)
# Example: Usage(input_tokens=8, output_tokens=75,
#                cache_creation_input_tokens=0, cache_read_input_tokens=60)
```

In this example:

1. The first call processes the 60 tokens of the system prompt and the 8 tokens of the user message. It caches the system prompt (`cache_creation_input_tokens: 60`).
2. The second call finds a cache hit for the identical system prompt (`cache_read_input_tokens: 60`). It only needs to process the new user message's 8 tokens as standard input (`input_tokens: 8`).

**Example (Incremental Caching in Conversation)**

To cache conversation history, you can place `cache_control` on the last message you want included in the cache for the next turn.

```python
# Turn 1 (User asks, Assistant responds - Cache System + Turn 1)
response_turn1 = client.messages.create(
    model="claude-3-5-sonnet-20240620",
    max_tokens=500,
    system=[{"type": "text", "text": "Maintain a friendly persona."}],
    messages=[
        {"role": "user", "content": "Hello Claude!"},
        {"role": "assistant", "content": "Hello there! How can I help you today?",
         "cache_control": {"type": "ephemeral"}}  # Cache up to here
    ]
)

# Turn 2 (Cache hit for System + Turn 1, Cache write for Turn 2)
response_turn2 = client.messages.create(
    model="claude-3-5-sonnet-20240620",
    max_tokens=500,
    system=[{"type": "text", "text": "Maintain a friendly persona."}],
    messages=[
        {"role": "user", "content": "Hello Claude!"},        # Part of cached prefix now
        {"role": "assistant", "content": "Hello there! How can I help you today?"},
        {"role": "user", "content": "Tell me a fun fact."},  # New dynamic content
        {"role": "assistant", "content": "Did you know honey never spoils?",
         "cache_control": {"type": "ephemeral"}}  # Cache up to end of Turn 2
    ]
)
# Turn 2 usage would show cache_read_input_tokens for System+Turn1
# and cache_creation_input_tokens for the new user/assistant messages of Turn 2
```

### Tracking Cache Performance

The API response includes usage metrics that reveal how the cache was utilized:

- `input_tokens`: Number of non-cached input tokens processed (the dynamic suffix).
- `output_tokens`: Number of tokens generated in the response.
- `cache_creation_input_tokens`: Number of input tokens processed and written to the cache in this request (occurs on a cache miss).
- `cache_read_input_tokens`: Number of input tokens loaded from the cache in this request (occurs on a cache hit).

Monitoring these fields is crucial for understanding if your caching strategy is effective. High `cache_read_input_tokens` relative to `cache_creation_input_tokens` over time indicates successful caching.

### AWS Bedrock

For models like Anthropic Claude accessed via AWS Bedrock, the caching mechanism is typically enabled using the same `cache_control` parameter within the model invocation request body (`application/json` format passed to `InvokeModel` or `InvokeModelWithResponseStream`). You would structure the JSON body according to the specific model's requirements (e.g., Anthropic's Messages API format) and include the `cache_control` field as shown above. Check the specific Bedrock documentation for the model provider you are using.

## What's the Pricing for Prompt Caching?

Prompt caching introduces a more nuanced pricing structure compared to standard token costs. While beneficial overall, it's important to understand the different token types involved:

1. **Base Input Tokens**: These are the standard input tokens that are not part of a cached prefix (i.e., the dynamic suffix in a cache hit scenario, or the entire prompt if caching isn't used or misses). They are charged at the model's standard input token rate.
2. **Cache Write Tokens** (`cache_creation_input_tokens`): When a prefix is processed for the first time (or after a cache miss) and written to the cache, the tokens associated with that prefix are often charged at a premium rate. For example, Anthropic charges 25% more than the base input token rate for cache writes. This reflects the cost of both processing and storing the cache entry.
3. **Cache Read Tokens / Cache Hit Tokens** (`cache_read_input_tokens`): When a cache hit occurs, the tokens corresponding to the prefix loaded from the cache are charged at a significantly discounted rate. For example, Anthropic charges only 10% of the base input token rate (a 90% discount) for cache reads. This reflects the computational savings.
4. **Output Tokens**: Tokens generated by the LLM in response are charged at the model's standard output token rate, regardless of whether caching was used for the input.

**Example Pricing Table (Anthropic Claude Models — illustrative rates)**

| Model             | Base Input (/MTok) | Cache Write (/MTok) (+25%) | Cache Read (/MTok) (-90%) | Output (/MTok) |
|-------------------|--------------------|----------------------------|---------------------------|----------------|
| Claude 3.5 Sonnet | $3.00              | $3.75                      | $0.30                     | $15.00         |
| Claude 3 Haiku    | $0.25              | $0.30                      | $0.03                     | $1.25          |
| Claude 3 Opus     | $15.00             | $18.75                     | $1.50                     | $75.00         |

(Note: Always refer to the official pricing pages of Anthropic and AWS Bedrock for the most current rates.)

The cost-effectiveness depends heavily on how often you get cache hits versus misses for a given prefix. If a large prefix is reused many times, the initial higher cost of the cache write is quickly offset by the substantial savings from subsequent cache reads.

## Limitations of Prompt Caching

While powerful, prompt caching has limitations and factors to consider:

**Minimum Cacheable Length**: Models often have a minimum token requirement for a prefix to be eligible for caching. Prompts shorter than this limit cannot be cached, even if marked with `cache_control`.

- Anthropic Claude 3.7/3.5 Sonnet & Opus: Minimum 1024 tokens.
- Anthropic Claude 3.5/3 Haiku: Minimum 2048 tokens.

**Cache Invalidation (Cache Busting)**: The cache is extremely sensitive to changes in the prefix. Any modification will break the cache and force a new write:

- Changing any text content within the `system`, `tools`, or `messages` blocks designated as part of the prefix.
- Changing the order or number of tool definitions.
- Changing `tool_choice` parameters (e.g., switching from auto to a specific tool).
- Adding, removing, or changing images within the prompt (for multimodal models).
- Altering other model parameters that affect initial processing might also break the cache (e.g., Anthropic's extended thinking settings can invalidate message caches).

**Cache Lifetime (TTL)**: Remember the cache is ephemeral (e.g., 5-minute minimum TTL for Anthropic, refreshed on use). Prefixes not reused within the TTL will expire. There's currently no way to manually clear or extend the cache beyond its automatic behavior.

**Concurrency**: If you send many identical requests concurrently targeting a prefix before the first request has finished and written the entry to cache, subsequent requests might result in cache misses until the first write completes. For guaranteed hits in parallel scenarios, you might need to wait for the first response before sending others.

**Supported Models**: Prompt caching is not universally available across all models or providers. Always check the documentation for the specific model you intend to use. (Currently confirmed for various Claude 3 and 3.5 models).

**Debugging**: Identifying subtle changes that cause cache misses can sometimes be tricky. Careful comparison of the exact prefix content between calls is necessary.

## Best Practices for Effective Caching

To maximize the benefits of prompt caching:

1. **Structure Prompts Intelligently**: Place the most stable, reusable information (system prompts, instructions, tool definitions, context documents, few-shot examples) at the beginning of your prompt sequence. Place dynamic, frequently changing information (user queries, latest conversation turns) after the cache breakpoint.
2. **Identify Optimal Breakpoints**: Use `cache_control` (or the equivalent mechanism) deliberately. Mark the end of the largest possible block of truly static content. If using multiple breakpoints (like Anthropic allows), consider different levels of stability in your prompt structure.
3. **Monitor Cache Usage**: Regularly check the `cache_creation_input_tokens` and `cache_read_input_tokens` in your API responses. Aim for a high ratio of reads to writes over time. If you see mostly writes, your prefixes might be changing too often, or they might be below the minimum cacheable length.
4. **Avoid Unnecessary Changes**: Be mindful that even tiny, seemingly insignificant changes to the prefix content (like adding a space or changing punctuation) will break the cache. Ensure consistency in how prefixes are generated.
5. **Consider Cost Trade-offs**: Caching is most effective for long and frequently reused prefixes. Don't cache very short prefixes, as the potential savings are minimal and might not outweigh the complexity. Avoid caching highly variable user input.
6. **Test and Iterate**: Experiment with different prompt structures and cache breakpoints to find the optimal strategy for your specific application's workload and usage patterns.

## Conclusion

Prompt caching is a vital optimization technique for anyone building applications on top of large language models. By intelligently storing and reusing the computational state of static prompt prefixes, it directly addresses the challenges of latency and cost associated with long or repetitive prompts. Understanding how caching works, its specific implementation details for your chosen LLM provider (like Anthropic's `cache_control`), the associated pricing nuances, and its limitations allows you to design more efficient, responsive, and economical AI applications. As LLM use cases grow in complexity and scale, leveraging features like prompt caching will be increasingly crucial for building performant and sustainable solutions.
