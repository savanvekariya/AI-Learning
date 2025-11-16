# Multi-Agent SDS Generator Learning Guide

Welcome! This guide is built for a junior developer new to Python, Object-Oriented Programming (OOP), Generative AI, Large Language Models (LLMs), and multi-agent architectures. By the end you will understand:
- What this project does
- What an AI agent is
- How multiple agents cooperate
- Python OOP concepts used here
- How each part of the code fits together
- How to extend the system safely
- Suggested practice exercises

---
## 1. Big Picture: What Does This Project Do?
This application generates a **Software Design Specification (SDS)** document for a software project. Instead of manually writing each section (Introduction, System Overview, Architecture, etc.), the program uses **specialized AI agents**. Each agent focuses on one section and produces professional text using an underlying Large Language Model (LLM) accessed through **SAP AI Core**.

Think of it like having a team of virtual technical writers, each with a specialty, coordinated by a manager (the orchestrator).

---
## 2. What Is an AI Agent?
An **AI agent** here is a Python class with:
- A role (its specialty)
- A method to build a prompt (instructions for the LLM)
- A call to the LLM to generate content
- A return value containing the section title and text

Each agent inherits shared logic from a parent `BaseAgent`. This keeps common code (like talking to the LLM) in one place.

**Analogy:** If building a house:
- Architect designs structure
- Electrician handles wiring
- Plumber handles pipes

In code:
- `IntroductionAgent` writes introduction
- `ArchitectureAgent` describes system design
- `DataDesignAgent` explains data model

---
## 3. What Is a Multi-Agent System?
A **multi-agent system** coordinates several agents to solve a bigger task. Here:
1. Orchestrator creates each agent
2. Sends the same project input to all
3. Each agent generates its section independently
4. Orchestrator combines results into one SDS document

Benefits:
- Separation of concerns (each agent focuses on one responsibility)
- Easier maintenance
- Scalable (add a new section by adding a new agent)
- Better quality (specialized prompts)

---
## 4. Python OOP Basics (Step-by-Step)
If you are new to Python OOP, start here.

### 4.1 What Is a Class?
A **class** is a blueprint for creating objects. Objects have:
- Attributes (data) — e.g., `self.name`
- Methods (functions) — e.g., `generate_section()`

Example:
```python
class Dog:
    def __init__(self, name):
        self.name = name  # attribute
    def bark(self):      # method
        return f"{self.name} says woof!"
```

### 4.2 What Is Inheritance?
Inheritance lets one class reuse code from another.
```python
class BaseAgent:
    def __init__(self, name):
        self.name = name

class IntroductionAgent(BaseAgent):
    def __init__(self):
        super().__init__("Introduction Specialist")
```
`IntroductionAgent` gets everything from `BaseAgent` plus whatever it adds.

### 4.3 Why Use Inheritance Here?
We have repetitive behavior across all agents:
- Initialize model name
- Validate environment
- Call LLM API

Rather than copy/paste into every agent, `BaseAgent` centralizes the shared logic.

### 4.4 Methods vs Functions
- **Function:** standalone (`def greet(): ...`)
- **Method:** defined inside a class (`def generate_content(self): ...`)

### 4.5 `self`
Refers to the *current object instance*. It allows each object to store its own data.

### 4.6 Composition vs Inheritance
We use inheritance for shared LLM logic. We also use composition when a class **contains** other objects (e.g., `SDSDocument` contains a list of `SDSSection` objects).

---
## 5. Relevant Project Files Explained
| File | Purpose |
|------|---------|
| `main.py` | Entry point; parses arguments, runs orchestrator |
| `core/base_agent.py` | Parent class with shared LLM logic |
| `core/orchestrator.py` | Coordinates all agents; builds final SDS |
| `agents/*.py` | Each specialized section generator |
| `models/schemas.py` | Data structures (ProjectInfo, SDSSection, SDSDocument) |
| `.env` | Credentials and model configuration |
| `sample_project_input.json` | Example input for generation |

---
## 6. Flow of Execution
1. User runs: `python main.py --input sample_project_input.json`
2. Input JSON is loaded into Pydantic models (`ProjectInput` etc.)
3. Orchestrator initializes agents
4. For each agent:
   - Builds prompt using project data
   - Calls `generate_content()` from `BaseAgent`
   - Receives generated text
5. Orchestrator builds `SDSDocument`
6. Saves result as Markdown

Simplified flow diagram:
```
main.py → load input → SDSOrchestrator → [agents...] → collect sections → SDSDocument → .md file
```

---
## 7. Deep Dive: `BaseAgent`
Core responsibilities:
- Read environment variables
- Set model name
- Import SAP AI Core chat client
- Build standard system message
- Send messages list (`system`, `user`) to LLM
- Return text

Key excerpt (simplified):
```python
class BaseAgent:
    def __init__(self, name, role):
        self.name = name
        self.role = role
        # (Credentials validation happens earlier)
        from gen_ai_hub.proxy.native.openai import chat
        self.chat = chat

    def generate_content(self, prompt, context=None):
        messages = [
            {"role": "system", "content": f"You are {self.name}, expert in {self.role}."},
            *([{"role": "user", "content": f"Context:\n{context}"}] if context else []),
            {"role": "user", "content": prompt}
        ]
        response = self.chat.completions.create(model_name="gpt-4o-mini", messages=messages)
        return response.choices[0].message.content.strip()
```

---
## 8. Deep Dive: An Agent (Example: Introduction)
```python
class IntroductionAgent(BaseAgent):
    def __init__(self):
        super().__init__(name="Introduction Specialist", role="project introduction writing")

    def generate_section(self, project_input: dict):
        project_info = project_input.get('project_info', {})
        name = project_info.get('name', 'Unknown')
        description = project_info.get('description', 'No description')
        prompt = f"Generate an introduction for {name}. Description: {description}. Include scope, purpose, audience."
        content = self.generate_content(prompt, project_input.get('additional_context'))
        return {"title": "1. Introduction", "content": content}
```

---
## 9. Orchestrator Logic
```python
class SDSOrchestrator:
    def __init__(self):
        self.agents = [IntroductionAgent(), ArchitectureAgent(), ...]

    def generate_sds(self, project_input):
        sections = []
        input_dict = {
            'project_info': project_input.project_info.model_dump(),
            'requirements': project_input.requirements.model_dump(),
            'technology_stack': project_input.technology_stack.model_dump(),
            'additional_context': project_input.additional_context or ''
        }
        for agent in self.agents:
            try:
                data = agent.generate_section(input_dict)
                sections.append(SDSSection(title=data['title'], content=data['content']))
            except Exception as e:
                sections.append(SDSSection(title=f"Error in {agent.name}", content=str(e)))
        return SDSDocument(project_info=project_input.project_info, sections=sections, generated_at="...")
```

---
## 10. Why Pydantic Models?
Pydantic provides:
- Validation (e.g., ensures `name` is a string)
- Serialization (`model_dump()` to get plain dict)
- Clean data boundary between input and logic

Example excerpt:
```python
class ProjectInfo(BaseModel):
    name: str
    version: str = "1.0"
    description: str
    stakeholders: List[str] = []
```

---
## 11. Common Errors & How to Think About Them
| Error Message | Likely Cause | How to Fix |
|---------------|-------------|------------|
| `string indices must be integers` | Treating a string like a dict | Print type; ensure JSON parsing succeeded |
| `Missing SAP AI Core credentials` | .env not set | Fill all required env vars |
| `Empty response from SAP AI Core` | LLM returned nothing | Retry / check model name |
| `ImportError: gen_ai_hub` | Package not installed | `pip install generative-ai-hub-sdk` |

Debug tips:
- Use `print(type(var))` when unsure
- Wrap critical blocks with `try/except` and re-raise meaningful messages

---
## 12. Extending the System (Add a New Agent)
Steps:
1. Create `agents/executive_summary_agent.py`
2. Inherit from `BaseAgent`
3. Implement `generate_section()`
4. Add agent to the list in `SDSOrchestrator.__init__`

Template:
```python
class ExecutiveSummaryAgent(BaseAgent):
    def __init__(self):
        super().__init__(name="Executive Summary Specialist", role="high-level summarization")

    def generate_section(self, project_input):
        info = project_input.get('project_info', {})
        prompt = f"Write an executive summary for {info.get('name')} focusing on goals and stakeholders."
        content = self.generate_content(prompt)
        return {"title": "Executive Summary", "content": content}
```

---
## 13. Practice Exercises
Try these to build confidence:
1. Print out each agent name before it runs.
2. Create a new agent: `RiskAssessmentAgent` listing potential technical risks.
3. Add caching so if generation fails you can retry without losing previous sections.
4. Modify prompts to be shorter or more structured (e.g., require bullet lists).
5. Add a CLI flag `--sections intro,architecture` to run only selected agents.
6. Replace Markdown output with a simple HTML template.

---
## 14. Suggested Learning Path (Roadmap)
| Stage | Focus | Outcome |
|-------|-------|---------|
| 1 | Python basics (functions, lists, dicts) | Comfort with syntax |
| 2 | OOP concepts (classes, inheritance) | Able to read project code |
| 3 | Working with APIs (requests) | Understand LLM calls |
| 4 | Prompt engineering basics | Better output quality |
| 5 | Error handling & logging | More robust tooling |
| 6 | Extension & refactor | Build your own agents |

Resources:
- Real Python (OOP tutorials)
- Pydantic docs (data modeling)
- SAP AI Core documentation (deployment & usage)
- Prompt Engineering Guide (improving prompts)

---
## 15. Glossary
| Term | Definition |
|------|------------|
| Agent | A specialized component that produces one SDS section |
| Orchestrator | Coordinator that runs all agents and assembles results |
| Prompt | Instruction text sent to an LLM |
| LLM | Large Language Model capable of generating human-like text |
| Inheritance | Mechanism where a class derives from another to reuse logic |
| Composition | Building complex types by combining simpler objects |
| Pydantic | Library for structured, validated data models |

---
## 16. Next Steps for You
Start small:
1. Open `agents/introduction_agent.py` and print the prompt before generating.
2. Add a new field to `ProjectInfo` (e.g., `industry: str`) and use it inside prompts.
3. Create a new agent for "Security Considerations".
4. Add command-line option to skip failed agents and continue.

Celebrate each win—small improvements build mastery.

---
## 17. Mental Model Summary
- **Classes** are blueprints
- **Objects** are instances with data and behavior
- **Agents** are objects that follow the same interface
- **The orchestrator** loops through agents uniformly
- **You control prompts** → Prompts control output quality
- **Data models** guard against malformed input

---
## 18. Still Confused? A Minimal Example
```python
class BaseAgent:
    def __init__(self, name):
        self.name = name
    def generate(self):
        return f"I am {self.name} and I generate text"

class SimpleAgent(BaseAgent):
    def generate(self):
        return f"{self.name} writes a custom section"

agents = [SimpleAgent("Intro"), SimpleAgent("Architecture")]
for a in agents:
    print(a.generate())
```
Focus on how the loop treats each agent the same way. This pattern scales nicely.

---
## 19. Troubleshooting Mindset
When something breaks:
1. Identify exact error message
2. Locate file + line
3. Inspect variable types involved (`type(var)`) 
4. Print intermediate values
5. Reduce to smallest reproducible code
6. Fix root cause (not just the symptom)

---
## 20. Final Encouragement
You are building with modern architectural patterns: multi-agent abstraction, layered design, separation of concerns, structured data modeling, and external AI integration. This is professional-grade thinking—keep going!

If you need a deeper dive on any section, ask specifically (e.g., "Explain inheritance again" or "Show me how to add logging").

---
**Happy Building!**
