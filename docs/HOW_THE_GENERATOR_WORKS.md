### High-level overview

1. **Public API** (module methods you call from outside)  
2. **Private Generator class** (does the real work)  
3. **Parse the Markdown** → find sections and HTTP examples  
4. **Convert each `$ http` example** into a proper OpenAPI path/operation  

---

### Part 1: The Public Module Interface (`MonzoOpenAPIGenerator`)

```ruby
module MonzoOpenAPIGenerator
  class << self
    def generate(input_text)
      Generator.new(input_text).generate
    end

    def dump_yaml(input_text, io = $stdout)
      io.puts YAML.dump(generate(input_text), line_width: 120)
    end

    def dump_json(input_text, io = $stdout)
      io.puts JSON.pretty_generate(generate(input_text))
    end
  end
```

This is the **only part you ever call** from other scripts.

- `generate` → returns a big Ruby Hash (the OpenAPI document)
- `dump_yaml` / `dump_json` → convenience methods used by `bin/generate-openapi`
- All heavy work is delegated to the private `Generator` class → clean separation

---

### Part 2: The Real Engine — `Generator` class

```ruby
class Generator
  def initialize(text)
    @text = text                 # All Markdown concatenated together
  end

  def generate
    @openapi = base_structure   # Start with a valid empty OpenAPI skeleton
    extract_sections            # Find all top-level sections (# Accounts, # Pots, etc.)
    extract_http_examples       # Find every `$ http ...` block
    add_missing_endpoints       # Manually add endpoints that have no example
    @openapi                    # Return the final document
  end
```

This is the **main pipeline**. Four simple steps.

---

### Stage 1: `base_structure`

Creates a valid OpenAPI 3.1 document with:

- Correct `openapi: 3.1.0`
- Proper `info`, `servers`, `components/securitySchemes`
- Two security schemes:
  - `bearerAuth` → classic personal API
  - `openBankingAuth` → Open Banking OAuth2 flow

This ensures the final file is **100% valid** even if parsing finds nothing.

---

### Stage 2: `extract_sections`

```ruby
@text.scan(/^# (.+?)\n([\s\S]*?)(?=(?:^# |\z))/m)
```

This regex finds every top-level section like:

```markdown
# Accounts
...content...

# Balance
...content...
```

For each section it:

- Stores the title and full content
- Automatically creates an OpenAPI **tag** (used in Swagger UI sidebar)
- Uses the first few lines as tag description

Result: Your generated spec has nice grouped operations (Accounts, Pots, Authentication, etc.)

---

### Stage 3: `extract_http_examples` — The Magic

```ruby
@text.scan(/\n```shell\n\$ http(?: --form)?\s+([A-Z]+)\s+"(https?:\/\/[^"]+)"([^`]*)```/m)
```

This finds **every** code block that starts with `$ http` — exactly how Monzo docs are written.

For each match we get:
- HTTP method (`GET`, `POST`, etc.)
- Full URL (`https://api.monzo.com/balance`)
- The rest of the line (parameters, headers, etc.)

Then we call `process_http_example`.

#### Inside `process_http_example`

1. **Normalize the path**
   ```ruby
   path = full_url
          .sub(%r{https?://api\.monzo\.com}, '')
          .gsub(/\$[a-zA_Z0-9_]+/) { |m| "{#{m[1..]}}" }
   ```
   → `/balance?account_id==$account_id` becomes `/balance/{account_id}`

2. **Extract query/form parameters**
   - Lines like `account_id==acc_123` → become OpenAPI query parameters
   - Handles both `key==value` and form-encoded style

3. **Detect request body**
   - `--form` → `application/x-www-form-urlencoded`
   - JSON example after the block → `application/json` + example

4. **Find the best summary and tag**
   - Looks which section contains this URL → uses that section as tag
   - Finds nearest `## Heading` as operation summary

5. **Auto-detect security**
   - If path contains `open-banking` → use Open Banking auth
   - Otherwise → use Bearer token

6. **Add to `@openapi[:paths]`**
   - Creates proper `operationId`, `parameters`, `requestBody`, `responses`, etc.

---

### Stage 4: `add_missing_endpoints`

Some important endpoints exist in text but have **no `$ http` example**:

- `/ping/whoami`
- `/attachment/upload`, `/register`, `/deregister`

We manually add them with good summaries and tags so the spec is complete.

---

### Final Output

When you run:

```bash
bin/generate-openapi
```

It:
1. Reads all `_*.md` files from `source/includes`
2. Concatenates them
3. Calls `MonzoOpenAPIGenerator.dump_yaml` and `.generate → JSON.pretty_generate`
4. Writes both files to `build/`

You get:
- `build/openapi.yaml` → perfect for Redoc, Swagger Editor
- `build/openapi.json` → perfect for Postman, code generators

---

