# de-duplication
## definition: what are duplicate math questions?
1. exact same text
    - examples
        - A: "what is $1+1$?"
        - B: "what is $1+1$?"
    - verdict: duplicate
2. same questions; different formating
    - examples
        - A: "what is $1+1$?"
        - B: "what is $1 + 1$?"
    - verdict: duplicate
3. templates: variable/number substitution
    - examples
        - same numbers, different variables
            - A: "if $2x+5=7$, what is $x$?"
            - B: "if $2y+5=7$, what is $y$?"
        - different numbers, same/different variables
            - A: "if John has 5 apples and eats 2, how many are left?"
            - B: "if Jane has 6 oranges and eats 3, how many are left?"
            - A: "if $2x+5=7$, what is $x$?"
            - B: "if $4x+6=10$, what is $x$?"
    - verdict:
        - same numbers, different variables: duplicate
        - different numbers, same/different var: keep up to N
            - eg 2 examples of the same template
4. same concept, different narratives
    - examples
        - A: "What is the maximum area of a rectangle with a perimeter of 20?"
        - B: "Given $x+y=10$, maximize $xy$."
    - verdict: not duplicates
        - different narratives of same concept help train the model's intuition

## how to dedupe?
1. use hashing -> turn questions into hash values
    1. exact hashing: same hash = duplicates
        - tools
            - md5 (128bit): faster, higher collision possibility
            - sha-256 (256bit): slower, lower collision possibility
        - covers...
            - stage 1 duplicates: exact same texts
    2. locality-sensitive hashing (lsh): near-duplicate detection
        - tools
            - minhash: preserves jaccard similarity
        - covers...
            - stage 2 duplicates: formating difference
2. use embedding model -> turn questions into embeddings
    - model options:
        - Voyage 4
            - nDCG@10: 0.859
            - cost: $0.06 per 1M tokens
            - latency: 17ms
            - dimensions: 1024
            - comment: smartest; proprietary; recommended
        - Qwen3 Embedding 8B
            - nDCG@10: 0.818
            - cost: $0.05 per 1M tokens
            - latency: 56ms
            - dimensions: 4096
            - comment: can self-host; slower; large embeddings so requires more storage; recommended for self-hosting
        - Voyage 3.5 Lite
            - nDCG@10: 0.803
            - cost: $0.02 per 1M tokens
            - latency: 11ms
            - dimensions: 512
            - comment: extremely cheap/fast; proprietary
    - cosine similarity
        - 0.99~1.0
            - covers...
                - stage 1 duplicates: exact same texts
                - stage 2 duplicates: formating difference
            - remove duplicates
        - 0.95~0.99
            - covers...
                - stage 3.1 duplicates: same numbers, different variables
            - remove duplicates
        - 0.85~0.95
            - covers...
                - stage 3.2 duplicates: different numbers, same/different variables
            - keep up to 2~3 duplicates
        - 0~0.85
            - covers...
                - stage 4 duplicates: same concept, different narratives
                - non-duplicates
            - keep all

## dedupe future considerations
- why more complex dedupe?
    - less data to generate reasoning: cheaper/faster
    - better training data: more diverse
- how?
    - manual: human in the loop
        - if similarity 0.85~0.95, fetch all similar questions and ask human to pick the ones to keep
    - automated: create normalized questions
        - for questions with similarity 0.85~0.95, use cheap/fast llms to create a version with name and numbers as variables
        - then embed the normalized questions and deduplicate

# enriching q/a pairs with reasoning
## how to generate reasoning?
1. generate reasoning + answer
    - model options: use reasoning-optimized models
        - Gemini 3 Pro
            - artificial analysis intelligence index: 57
            - cost: $4.5 per 1M tokens
            - latency: 34.07
            - comment: smartest; slow; proprietary
        - Claude Sonnet 4.6
            - artificial analysis intelligence index: 52
            - cost: $6 per 1M tokens
            - latency: 0.85
            - comment: extremely fast; more expensive; proprietary
        - GLM-5
            - artificial analysis intelligence index: 50
            - cost: $1.55 per 1M tokens
            - latency: 1.29
            - comment: cheap/fast; self-hostable
    - input: question
    - prompt
    - output: reasoning + answer
2. compare generated answer w/ actual answer
    - model options: use fast/cheap models
        - Gemini 2.5 Flash-Lite
            - artificial analysis intelligence index: 19
            - cost: $0.17 per 1M tokens
            - latency: 0.43
            - comment: cheap/fast; proprietary
        - Nova Micro
            - artificial analysis intelligence index: 12
            - cost: $0.06 per 1M tokens
            - latency: 0.37
            - comment: extremely cheap/fast; low intelligence; proprietary
        - Claude 4.5 Haiku (reasoning)
            - artificial analysis intelligence index: 37
            - cost: $2 per 1M tokens
            - latency: 0.46
            - comment: very expensive; much higher intelligence; proprietary
    - input: generated answer, actual answer
    - prompt
    - output: same/different
        - same: step 4
3. if different, generate corrected reasoning
    - model options: same as step 1
    - input: question, previous reasoning, previous answer, actual answer
    - prompt
    - output: corrected reasoning
4. evaluate the reasoning (hallucination, missing steps, logical steps)
    - model: same as step 1
        - cross-model evaluation: use a different model to evaluate
            - eg if step 1 uses Gemini 3 Pro, use Claude Sonnet 4.6 to evaluate
    - input: question, reasoning, answer, #retry
        - #retry > 3, abandon this q/a (store for human review)
    - prompt
    - output: good/bad + if bad, why?
        - good: step 6
5. if bad, fix the reasoning
    - model: sam as step 1
    - input: question, previous reasoning, answer, why reasoning is bad
    - prompt
    - output: fixed reasoning
        - back to step 4
6. done: enriched q/a with reasoning

# end-to-end pipeline: from q/a pairs in excel -> q/a/reasoning in object storage
- compare all the tools (time/cost/quality)
1. data orchestration: schedule/manage/monitor the entire ETL pipeline
    - tools
        - data orchestration frameworks
            - dagster
                - cost: open-source (free) or cloud (paid)
                - time: fast to develop but steeper learning curve
                - quality: best-in-class data asset tracking (Software-Defined Assets)
                - comment: heavily favored in 2026 for data engineering pipelines
            - prefect
                - cost: open-source (free) or cloud (paid)
                - time: very fast to turn existing python functions into tasks (decorator-based)
                - quality: great task-based orchestration, slightly less data-aware than dagster
    - why?
        - state/checkpoint: if something breaks, the framework knows where to resume
        - retry: easy retry decorators
        - easier batch processing
        - monitoring: built-in dashboards for ETL observability
    - steps
        1. extract: data ingestion
            - input: excel
            - output: q/a pairs in db
            - tools
                - pandas/polars
                    - pandas: good enough for medium-sized data
                    - polars: use for millions of rows; better memory/thread management
                - regular db
                    - serverless
                        - cost: cheap; scales to zero
                        - eg: neon postgres, aws aurora serverless v2
                        - comment: best for this use case
                    - paas
                        - cost: expensive; always on
                        - eg: aws rds, google cloud sql
                        - comment: not recommended if only running nightly batch jobs
        2. transform: deduplicate + enrich (add reasoning)
            - input: q/a pairs in db
            - output: q/a + reasoning in db
            - tools
                - agent workflow orchestration
                    - langgraph
                        - cost: free (open source)
                        - quality: industry standard; better for complex DAG
                    - pydanticAI
                        - cost: free (open source)
                        - quality: simple; less mature for complex DAG
                - embedding/inference models: already compared
                - llm gateway
                    - litellm
                        - cost: free (self-hosted) or cheap (managed cloud)
                        - time: instantly standardizes any LLM API to OpenAI format
                        - quality: reliable, great for fallback routing and basic cost tracking
                    - portkey
                        - cost: expensive (enterprise SaaS)
                        - quality: premium UI, extreme robustness, integrated prompt management and observability
                - prompt CMS
                    - llm gateway built-in
                        - litellm: basic prompt CMS
                        - portkey: more advanced (A/B testing, etc.)
                    - standalone prompt CMS
                        - promptlayer: good UI; good for non-tech users
                        - pezzo: open-source
                - vector db
                    - serverless
                        - cost: cheap; scales to 0
                        - eg: pinecone serverless, neon pgvector
                        - comment: highly recommended for this use case
                    - paas
                        - cost: expensive; always on
                        - eg: aws rds pgvector
                - regular db: already compared
        3. load: store transformed data
            - input: q/a/r from db
            - output: jsonl in s3
            - tools
                - object storage
                    - aws s3
                        - cost: high egress fee
                        - quality: industry standard, integrations everywhere
                    - cloudflare r2
                        - cost: zero egress fees
                        - quality: S3-compatible, very popular in 2026
2. observability
    - ETL: dashboard provided by dagster/prefect
    - LLM (Traces, Cost Tracking, Eval metrics)
        - langsmith
            - cost: expensive (enterprise)
            - quality: native integration with LangGraph
        - langfuse
            - cost: free (self-hosted) or cheap (managed cloud)
            - quality: popular open-source alternative to langsmith