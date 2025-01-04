# WebMuse: A Web-Native Research & Documentation System

## 1. System Overview

WebMuse is a **research and writing platform** that uses **web scraping** and **LLM (GPT-4) processing** to produce **comprehensive articles** or guides on complex technical topics. The system orchestrates:

1. **Web Scraping** via Playwright to gather information (documentation, tutorials, code samples).
2. **GPT-4** prompts that analyse and transform this raw data into a final structured document.

**Example Scenario**: Migrating a legacy Rails project from Heroku to AWS ECS (with or without Terraform) while ensuring all necessary AWS resources and best practices are documented.

---

## 2. Core Components

### 2.1 Web Scraping Engine (Playwright)

Responsible for **automated web searches** and **content extraction**. Using [Playwright for Ruby](https://github.com/YusukeIwaki/playwright-ruby-client) allows robust interaction with modern websites—including pages that require JavaScript rendering.

```ruby
require 'playwright'
require 'uri'

SearchResult = Struct.new(:title, :snippet, :url, keyword_init: true)
ContentResult = Struct.new(:url, :content, :error, :success, keyword_init: true)

class WebScraper
  def initialize
    @playwright = Playwright.create(ruby: true)
    @browser = @playwright.chromium.launch(
      headless: true,
      args: ['--no-sandbox', '--disable-setuid-sandbox']
    )
  end

  def search(query)
    page = @browser.new_page
    # Configure viewport for consistent rendering
    page.set_viewport_size(width: 1280, height: 800)

    # Use DuckDuckGo (no API needed)
    page.goto("https://duckduckgo.com/?q=#{URI.encode_www_form_component(query)}")
    page.wait_for_selector('.result__body')

    # Extract each result’s title, snippet, and URL
    results = page.eval_on_selector_all('.result__body', <<~JS
      elements => elements.map(el => ({
        title: el.querySelector('.result__title')?.innerText,
        snippet: el.querySelector('.result__snippet')?.innerText,
        url: el.querySelector('.result__url')?.href
      }))
    JS
    )

    results.map { |r| SearchResult.new(r) }
  end

  def scrape_content(url)
    page = @browser.new_page
    begin
      page.goto(url, timeout: 30_000)
      # Wait for dynamic content
      page.wait_for_load_state('networkidle')

      # Extract main content using common selectors
      content = page.eval_on_selector_all(
        ['article', '.content', '.post-content', 'main', '#main-content'].join(','),
        'els => els.map(el => el.innerText).join("\n")'
      )

      ContentResult.new(url: url, content: content, success: true)
    rescue => e
      ContentResult.new(url: url, error: e.message, success: false)
    end
  end

  def cleanup
    @browser.close
    @playwright.stop
  end
end
```

**Key Points**:

- **search(query)**: Automates DuckDuckGo queries. Could be modified for other engines or APIs.
- **scrape_content(url)**: Opens pages individually, waits for complete load, then aggregates textual content from typical “article-like” elements.
- **cleanup**: Frees up browser resources at the end of operations.

---

### 2.2 Research Coordinator

Orchestrates **the entire research cycle** by calling GPT-4 for query generation, scraping results, and identifying any new gaps.

```ruby
require 'openai'

class ResearchCoordinator
  def initialize(topic)
    @topic = topic
    @scraper = WebScraper.new
    @openai = OpenAI::Client.new
    @research = []
  end

  def conduct_research
    # Initial query generation
    queries = generate_search_queries

    # First round of scraping
    initial_results = queries.flat_map { |query| @scraper.search(query) }
    process_results(initial_results)

    # Generate follow-up questions/queries
    follow_ups = generate_follow_up_questions
    follow_up_results = follow_ups.flat_map { |query| @scraper.search(query) }
    process_results(follow_up_results)

    @research
  end

  private

  def generate_search_queries
    response = @openai.chat(
      parameters: {
        model: "gpt-4",
        messages: [
          {
            role: "system",
            content: "You are a research assistant planning queries for #{@topic}"
          },
          {
            role: "user",
            content: INITIAL_QUERIES_PROMPT
          }
        ],
        temperature: 0.7
      }
    )

    parse_queries(response)
  end

  # Example parser that splits lines in GPT-4 output
  def parse_queries(response)
    response['choices'].first['message']['content'].split("\n").map(&:strip)
  end

  def process_results(results)
    # Basic approach: store or refine. Could add filters, dedup, summarise, etc.
    @research.concat(results)
  end

  def generate_follow_up_questions
    # Potentially feed partial research back into GPT-4 to identify knowledge gaps
    []
  end
end

INITIAL_QUERIES_PROMPT = <<~PROMPT
  Generate search queries for researching #{@topic}.
  Consider these perspectives:
  1. Technical/Implementation details
  2. Historical context
  3. Current state/usage
  4. Key figures/organizations
  5. Challenges/controversies

  For each perspective, generate 2-3 specific search queries.
  Format each query to maximize relevant results.
PROMPT
```

**Key Points**:

- The system **relies heavily** on GPT-4 to generate relevant search queries.
- You can tailor `INITIAL_QUERIES_PROMPT` to emphasise **Terraform resource details**, optional vs. required AWS services, or other angles.
- The **research** data is simply appended to an array here; real-world usage might require advanced sorting, tagging, or summarisation.

---

### 2.3 Content Generator

Takes the curated research data and **translates** it into an **outline**, **sections**, and a **final polished article**.

```ruby
class ContentGenerator
  def initialize(research)
    @research = research
    @openai = OpenAI::Client.new
  end

  def generate_article
    outline = generate_outline
    sections = generate_sections(outline)
    polish_article(sections)
  end

  private

  def generate_outline
    response = @openai.chat(
      parameters: {
        model: "gpt-4",
        messages: [
          {
            role: "system",
            content: "You are creating a detailed outline based on research"
          },
          {
            role: "user",
            content: OUTLINE_PROMPT
          }
        ],
        temperature: 0.7
      }
    )

    response['choices'].first['message']['content']
  end

  def generate_sections(outline)
    # Potentially iterate over outline sections, prompting GPT-4 for deeper expansions
    []
  end

  def polish_article(sections)
    # Merge content, possibly re-run a final GPT-4 pass for editorial polishing
    sections.join("\n\n")
  end
end

OUTLINE_PROMPT = <<~PROMPT
  Based on this research:
  #{@research.map(&:to_s).join("\n\n")}

  Create a detailed outline that:
  1. Covers all major aspects discovered
  2. Maintains logical flow and progression
  3. Identifies areas needing additional research
  4. Suggests appropriate subsections
  5. Notes where citations will be needed

  Format the outline with:
  - Main sections (1., 2., etc.)
  - Subsections (a., b., etc.)
  - Key points to cover (•)
  - Citation needs ([Citation needed])
PROMPT
```

**Key Points**:

- The system uses **GPT-4** to **transform** raw textual data into a **structured** layout.
- **Polishing** may also perform tasks like ensuring style consistency, generating final references, or clarifying code snippets.

---

## 3. Complete Example Flow (Classic Version): AWS ECS Rails Deployment

Below is the classic demonstration of how the system might research and generate content around **deploying a Rails application on AWS ECS**.

### 3.1 Initial Query Generation

**User Prompt to GPT-4**

```
Generate specific search queries for researching AWS ECS Rails deployment.
Consider:
1. Basic setup and configuration
2. Production deployment considerations
3. Performance optimization
4. Common issues and solutions
5. Best practices and patterns
```

**GPT-4 Output**

```
Technical Setup:
- "AWS ECS Rails Dockerfile production example"
- "ECS task definition Ruby on Rails configuration"
- "Rails credentials AWS ECS setup guide"

Performance:
- "AWS ECS Rails application scaling patterns"
- "ECS Rails asset pipeline CDN configuration"
- "Rails sidekiq ECS worker configuration"

Problems/Solutions:
- "AWS ECS Rails deployment troubleshooting common issues"
- "ECS Rails application boot timeout solutions"
- "Rails asset precompilation ECS memory issues"
```

### 3.2 Initial Research Results

Imagine the system scrapes relevant docs and returns something like:

```ruby
results = [
  {
    url: "https://docs.aws.amazon.com/AmazonECS/latest/developerguide/...",
    content: "Task definitions for Rails applications require specific configuration..."
  },
  {
    url: "https://github.com/aws-samples/aws-ecs-rails-sample",
    content: "Example Dockerfile and configuration for deploying Rails 7..."
  }
  # ... more results
]
```

### 3.3 Follow-up Question Generation

**User Prompt**

```
Based on these initial findings, what critical questions remain unanswered?
Consider gaps in:
1. Technical implementation details
2. Security considerations
3. Operational concerns
4. Cost optimization
```

**GPT-4 Output**

```
Follow-up Questions:
1. How are Rails credentials managed in ECS tasks?
2. What's the optimal memory allocation for asset precompilation?
3. How are database migrations handled during deployment?
4. What monitoring solutions work best with Rails on ECS?
```

### 3.4 Outline Generation

**User Prompt**

```
Based on all research findings, create a comprehensive outline for a guide on
deploying Rails applications on AWS ECS. Include specific technical details
and cite sources where available.
```

**GPT-4 Output** (example)

```markdown
# Deploying Rails Applications on AWS ECS: A Comprehensive Guide

1. Infrastructure Setup
   a. VPC Configuration
   b. ECS Cluster Setup
   c. Load Balancer Configuration
   [Citation: AWS docs on ECS networking]

2. Application Configuration
   a. Dockerfile Setup
   • Multi-stage build process
   • Asset precompilation
   [Citation: AWS sample repo]
   b. Task Definition
   • Memory allocation
   • Environment variables
   • Logging configuration

   [Continue with full outline...]
```

### 3.5 Section Generation & Final Polish

Each **outline subsection** can be expanded with references, code snippets, or further clarifications. A final GPT-4 pass might ensure consistency, add citations, and note prerequisites.

---

## 4. Terraform-Focused Workflow & Details

Many organisations need a **Terraform**-heavy approach—especially for migrations (e.g., from Heroku to AWS ECS). The system can:

1. **Scrape Terraform Registry** for each resource (e.g., `aws_vpc`, `aws_ecs_cluster`, `aws_ecs_service`, `aws_lb`, `aws_rds_instance`, etc.).
2. Parse or summarise **required vs optional** fields, recommended modules, resource-specific gotchas.
3. Present **key decisions** (new VPC vs. existing VPC, Fargate vs. EC2, how to handle environment secrets, etc.).

**Example**:

- When generating queries, the system might ask GPT-4:
  ```
  "Terraform ECS rails docker deployment: best practices"
  "Terraform registry aws ecs module usage example"
  "Terraform VPC vs. existing VPC pros and cons for ECS"
  ```
- After scraping, the tool can compile a doc segment showing typical `.tf` snippets with commentary:

  ````ruby
  # Example code snippet from Content Generator
  ```hcl
  resource "aws_vpc" "main" {
    cidr_block = var.vpc_cidr
    # ...
  }

  module "ecs_cluster" {
    source  = "terraform-aws-modules/ecs/aws"
    version = "~> 3.0"
    # ...
  }
  ````

  _Explanation: This snippet uses an official ECS module to stand up cluster resources…_

---

## 5. Implementation Notes

### 5.1 Rate Limiting and Politeness

To avoid spamming external sites:

```ruby
class RateLimiter
  def initialize
    @last_request = {}
    @min_delay = 2  # seconds
  end

  def wait_for(domain)
    last = @last_request[domain]
    if last
      elapsed = Time.now - last
      sleep(@min_delay - elapsed) if elapsed < @min_delay
    end
    @last_request[domain] = Time.now
  end
end
```

### 5.2 Error Handling

Gracefully retry or fail:

```ruby
class ScrapingError < StandardError; end

def safe_scrape(url)
  retries = 0
  begin
    @rate_limiter.wait_for(URI.parse(url).host)
    scrape_content(url)
  rescue => e
    retries += 1
    if retries < 3
      sleep(2 ** retries)
      retry
    else
      raise ScrapingError, "Failed to scrape #{url}: #{e.message}"
    end
  end
end
```

### 5.3 Content Caching

Redis-based caching of responses for performance:

```ruby
require 'redis'
require 'digest'

class ContentCache
  def initialize(redis_url)
    @redis = Redis.new(url: redis_url)
    @ttl = 86400  # 24 hours
  end

  def get(url)
    @redis.get(cache_key(url))
  end

  def set(url, content)
    @redis.set(cache_key(url), content, ex: @ttl)
  end

  private

  def cache_key(url)
    "content:#{Digest::SHA256.hexdigest(url)}"
  end
end
```

---

## 6. Strengths & Limitations

### 6.1 Strengths

1. **Time Efficiency**: Automates repetitive searching, summarising, and drafting of technical docs.
2. **Consistency**: Maintains uniform structure (headings, code blocks, references) throughout the final output.
3. **Flexibility**: Adapts to new topics by simply changing the prompts, from ECS to Kubernetes or from AWS to GCP.
4. **Scalable**: Can be extended with additional modules for advanced HTML parsing, chunked summarisation, or custom formatting.

### 6.2 Limitations

1. **Depends on Quality of Scraped Data**: If official docs or blog posts are outdated or incomplete, the final result may mirror these gaps.
2. **LLM Constraints**: GPT-4 has token limits and may misunderstand complex or niche scenarios. Human review remains crucial.
3. **No Automatic Decision-Making**: The tool can highlight trade-offs (e.g., new vs. existing VPC) but can’t choose them for you.
4. **Recurring Costs**: OpenAI API usage can become expensive at scale; plus infrastructure for scraping and caching must be maintained.

---

## 7. Costs & Where to Use (or Not Use)

### 7.1 Costs

- **OpenAI (GPT-4) Fees**: Token-based pricing can add up quickly for large-scale or frequent usage.
- **Infrastructure**: Hosting the scraping engine, Redis cache, and any additional pipelines.
- **Maintenance**: Periodic updates for scraping scripts, library dependencies, and continuous improvement of prompts.

### 7.2 Where It Shines

- **Comprehensive Documentation**: When you need a single source of truth for a complex migration, such as Heroku → AWS ECS with Terraform.
- **Initial Research & Summaries**: Quickly collect and summarise common pitfalls and best practices from across the web.
- **Repeatable Knowledge**: If your org does many similar deployments, you can reuse the same workflows.

### 7.3 When It’s Not Ideal

- **Proprietary/Secret Docs**: If the needed docs aren’t publicly accessible or cannot be scraped, the system’s utility diminishes.
- **Highly Regulated Environments**: AI outputs often need extensive human review, which can offset the time saved.
- **Fully Automated Architecture**: It can’t replace a solutions architect who must weigh cost, compliance, and internal constraints. The system is an assistant, not a decision-maker.

---

## 8. Next Steps & Future Enhancements

1. **Enhanced Scraping**

   - Deeper JavaScript interaction for SPAs.
   - More robust content extraction (tables, images, code blocks).

2. **Deeper Terraform Integration**

   - Parse Terraform Registry docs to annotate required vs. optional arguments.
   - Merge official module usage examples (e.g., for ECS, VPC, RDS, S3, ElastiCache).
   - Possibly parse `terraform providers schema -json` to summarise resource fields.

3. **Advanced Content Processing**

   - Automatic citation extraction from documentation.
   - Intelligent code-block insertion with explanations.
   - Additional “polish” pass for style/terminology consistency.

4. **Multi-Format Export**

   - Native Markdown (with embedded `.tf` code) for dev docs.
   - PDF or DOCX generation for non-technical stakeholders.
   - JSON/YAML for programmatic references or CI/CD pipelines.

5. **User Feedback Loop**
   - Let devs/stakeholders comment on or correct doc sections.
   - Feed corrections back into the system so subsequent runs produce more accurate results.

---

## 9. Conclusion

**WebMuse** unifies **automated web research** and **AI-driven writing** into a single platform. By **scraping** official docs, community tutorials, and code samples, then **marrying** this data with GPT-4’s text-generation capabilities, it provides a powerful starting point for building comprehensive technical documentation.

- **Strength**: Saves considerable time and ensures consistency across large or frequently updated docs.
- **Limitation**: Requires human oversight, especially for architectural decisions or compliance constraints.
- **Use Case**: Ideal for tasks like creating a **thorough Heroku → AWS ECS migration guide** with Terraform, detailing each AWS resource and best practice in a structured, references-included document.

In short, WebMuse can help both **technical teams** and **technical writers** accelerate knowledge gathering, highlight critical decisions, and produce well-structured final outputs—all while acknowledging that expert input and review remain essential.

## Acknowledgement

WebMuse is inspired by Stanford's [STORM Project](https://storm.genie.stanford.edu), a groundbreaking system for LLM-powered knowledge curation and collaborative AI research.
