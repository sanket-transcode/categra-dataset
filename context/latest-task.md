## api repo only - Only investigation & no modification

### The task:

Gemini Context Caching for Amazon Schema Prompt  
What: buildAmazonAttributeFillerPrompt includes the full Amazon product type  
schema JSON in every request. This is 2,000–4,000 static tokens sent on every  
single Amazon product sync call. Gemini's context caching API lets you cache  
a static prompt prefix and pay only ~25% of the input token price for  
subsequent calls.  
How:  
Extract the static part of the Amazon attribute filler prompt (the schema +  
instructions) into a Gemini cached content object  
Cache it once per product type per session (TTL: 1 hour minimum)  
Reference the cache handle in subsequent generateContent calls  
Files to change:  
src/modules/app/ai/ai.service.ts — aiSuggestedValue() method, add  
cachedContent parameter to generateContent  
Add a Map<productType, cacheHandle> in the service to store cached content  
handles  
Estimated savings: ~75% reduction in input tokens for every Amazon attribute  
fill call — the biggest per-call cost driver  
Reference: Gemini supports CachedContent via @google/generative-ai SDK 

### reference files

api\apps\api-main\src\core\llm\orchestrator\llm-orchestrator.service.ts
api\apps\api-main\src\core\llm\prompts\prompt-builder.service.ts

### Points to be considered

-> The amazon schema cannot be cached as proposed into the prompt because with every request the referenced schema can be different as amazon has more than 1800 product types and many marketplaces
-> the referenced files are as per the old structure, now we have the full LLM orchestrator file structure to execute the AI calls, so adapt the solution as per current file structure
-> The current LLM handling implementation is way more dynamic where for each LLM call from the provider itself things are dynamic, so this will be for gemini provider only, and another thing keep in mind, every request which is going to be resolved via gemini can still have distinct llm models to be used
-> Currently three type of operation options are there and 5 actual operations are there

### You CTA
-> Find out is there something to be used as caching for gemini that will give real benefits and if you found something then only give the implementation details