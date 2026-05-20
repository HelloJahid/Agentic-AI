# LLM Prompting Systems

LLM prompting systems are the structured methods and patterns used to communicate with large language models so that they produce reliable, useful, and predictable outputs. While the underlying model remains fixed once trained, almost everything that determines whether an LLM is useful in a real application comes down to how it is instructed, what context it is given, and how its outputs are managed across one or many steps. Prompting has therefore matured from the simple act of asking a question into a layered discipline that spans single-prompt design, multi-step workflows, validation, and iterative refinement.

This document walks through that landscape in a deliberate progression. The opening sections establish what LLMs are and how they are accessed in practice through system and user prompts. The middle sections cover the core single-prompt techniques, namely role-based prompting, Chain of Thought reasoning, ReAct for tool-using agents, and the meta-skill of prompt instruction refinement. The closing sections address workflow-level patterns, including prompt chaining, gate checks for validation, and feedback loops for iterative self-improvement. Read in order, the sections move from understanding what an LLM can do, to controlling how it behaves, to building reliable multi-step systems on top of it.

---

## 1. Foundation Concepts

Every prompting technique discussed in this document rests on two ideas, namely what an LLM is and what it means to prompt one. This first section establishes those foundations so that the techniques that follow have a clear conceptual base.

### Large Language Models

A large language model (LLM) is a deep learning model trained on vast quantities of text data, enabling it to understand and generate human language across a wide range of tasks including summarisation, translation, question answering, and code generation. These models learn statistical patterns and relationships between words and concepts at scale, allowing them to produce contextually coherent and semantically rich outputs.

**Example.** Given the partial sentence "The capital of Australia is", an LLM completes it with "Canberra" because it has learned from training data that this word sequence is statistically the most likely continuation. The model is not retrieving a fact from a database; it is generating the next most probable token based on patterns observed during training.

### LLM Prompting

LLM prompting refers to the practice of crafting input instructions or queries known as prompts, which guide the model toward a desired response. Effective prompting involves being clear and specific about the task, providing relevant context, specifying the desired output format, and sometimes including examples of expected behaviour. These elements help the model produce more accurate, relevant, and useful results without modifying the underlying model itself.

**Example.** A weak prompt such as "tell me about cars" produces a vague and unfocused response. A stronger prompt such as "Write a 100-word summary explaining the difference between battery-electric and hydrogen fuel-cell vehicles, written for a high school audience" produces a focused, useful answer. The second prompt is clear, specific, includes the audience as context, and specifies the desired length.

---

## 2. Types of LLM Deployment

Having established what an LLM is and how prompting works in principle, the next question is how these models are accessed in practice. The way a prompt is structured depends on the deployment context, and this section introduces the two most common forms, namely the general LLM and the API-based LLM. The system and user prompt distinction introduced here also sets the stage for every prompting technique that follows.

### General LLM

A general LLM is first trained in an unsupervised manner to predict the next token or word in a sequence. The same model is then retrained on semi-supervised data so that it learns to generate user-friendly responses aligned with human preferences and instructions.

**Example.** In the unsupervised stage, the model sees billions of sentences such as "Once upon a ___" and learns that "time" is a likely completion. In the semi-supervised stage, when a user asks "How do I bake bread?", the model produces a structured, helpful recipe rather than simply continuing the sentence with the next statistically likely words. The second stage teaches the model how to behave like a helpful assistant rather than a text completer.

### API-Based LLM

When working with API-based LLMs such as those provided by Anthropic (Claude) or OpenAI (ChatGPT), two types of instructions are typically supplied.

The **system prompt** functions as an overarching directive that defines the model's persona, behaviour, and operational constraints for the entire interaction. The **user prompt** specifies what the user wants on each individual turn of the conversation. The system prompt establishes persistent context across the session, while the user prompt drives the specific task requested at any given moment.

**Example.**

```
System prompt: "You are a friendly Python tutor who explains concepts
using simple analogies and provides short code examples."

User prompt: "Explain what a list comprehension is."
```

The system prompt persists across every message in the conversation, ensuring the model maintains the tutor persona. The user prompt changes from turn to turn as the user asks new questions.

---

## 3. Role-Based Prompting

The system prompt introduced in the previous section is the natural location for defining how the model should behave, and the most common way to shape that behaviour is to assign the model a role or persona. Role-based prompting is therefore the first concrete prompting technique that builds directly on the system prompt mechanism.

Role-based prompting is a technique used to guide an LLM by assigning it a specific identity or persona to inhabit. This approach helps produce outputs that are specialised and consistent with the traits of the defined role. Instructing the model to act as a particular expert, character, or function shapes the tone, vocabulary, and reasoning style of its responses, leading to more focused and contextually appropriate outputs.

**Example.** The same question "Explain blockchain" produces very different responses depending on the assigned role.

Without a role, the model returns a generic technical description full of jargon. Assigning the role "Act as a children's book author. Explain blockchain to an eight-year-old using a story about trading marbles in a schoolyard" produces an engaging, analogy-driven explanation suitable for a young audience. Assigning the role "Act as a financial regulator presenting to a parliamentary committee" produces a formal, risk-focused explanation appropriate for policymakers.

---

## 4. Chain of Thought (CoT)

Whereas role-based prompting shapes who the model is and how it speaks, the next family of techniques shapes how the model thinks. Chain of Thought is the foundational reasoning technique and the starting point for the more advanced reasoning patterns that follow.

Chain of Thought (CoT) is a prompting strategy that encourages the LLM to generate a sequence of intermediate reasoning steps before providing a final answer. Rather than asking only for the answer, the user guides the model to think through the problem step by step. This approach is particularly effective for tasks requiring logical reasoning, mathematical calculation, or multi-stage analysis, because it makes the model's reasoning explicit and reduces the likelihood of skipped logical steps.

**Example.** Consider the question "If a delivery van leaves the depot with 24 parcels, drops off 7 at the first stop and 9 at the second stop, then picks up 4 more, how many parcels remain?"

Without CoT, the model might guess "11" or "12". With CoT, prompted by adding "Let's think step by step", the model reasons as follows.

```
Step 1. The van starts with 24 parcels.
Step 2. After the first stop, 24 minus 7 leaves 17 parcels.
Step 3. After the second stop, 17 minus 9 leaves 8 parcels.
Step 4. After picking up 4 more, 8 plus 4 gives 12 parcels.
Final answer: 12 parcels remain.
```

The explicit reasoning chain makes the answer more reliable and easier to verify.

---

## 5. ReAct Prompting

Chain of Thought is powerful for problems the model can solve using its existing knowledge, but it cannot answer questions about today's weather, current stock prices, or the contents of a private database. ReAct extends CoT by giving the model the ability to interact with the outside world, turning a pure reasoner into something closer to an agent.

ReAct is a framework that combines Chain of Thought reasoning with action-taking capabilities. The name stands for **Reason + Act**, and the technique enables LLMs to handle complex tasks that require multiple steps of interaction with external information sources using tools. Tools or external information sources are common in ReAct, allowing the LLM to obtain outside information through API calls or function calls. Whereas CoT is well suited to internal reasoning, ReAct addresses tasks that require new information or interaction with the outside world.

The core of ReAct is an iterative loop consisting of three phases.

### Thought
The model internally reasons and plans the next specific step required to progress towards the overall task goal. It analyses the situation, identifies necessary actions, and decides on the next move.

### Action
Based on its plan, the model specifies the use of an external tool such as a web search, a calculator, or an API, along with the relevant parameters. An orchestrator program then executes this tool. Tools are functions or services the agent can invoke.

### Observation
The model receives the results or feedback from the executed action, such as search results, a calculator output, or confirmation that an email has been sent. This new information feeds into the next Thought, allowing the model to refine its plan or take subsequent actions. The observation phase, sometimes referred to as input from the environment when it originates from the outside world (for example, a weather report), closes the loop and supports iterative refinement until the task is resolved.

**Example.** Consider the user task "What is the weather in Melbourne today and should I bring an umbrella?"

```
Thought 1.  I need current weather data for Melbourne. The model itself
            does not know today's forecast.
Action 1.   get_weather(location="Melbourne, AU")
Observation 1. {"temperature": 18, "rain_probability": 80,
                "conditions": "afternoon showers expected"}

Thought 2.  Rain probability is 80 percent, so an umbrella is sensible.
            I now have enough information to answer.
Final Answer. The forecast in Melbourne today shows 18 degrees with
              an 80 percent chance of afternoon showers. Yes, you
              should bring an umbrella.
```

The loop continues until the model decides it has enough information to respond.

---

## 6. Prompt Instruction Refinement

All the techniques discussed so far, including role-based prompting, CoT, and ReAct, depend on the quality of the prompt that drives them. Prompt instruction refinement is therefore a meta-skill that improves the effectiveness of every technique already described, by tuning the prompt itself rather than introducing a new mechanism.

Prompt instruction refinement is the practice of evaluating and adjusting prompt instructions to produce more precise LLM outputs. While LLMs excel at understanding and generating human-like language, their outputs do not always match expectations, and this is where the art and science of tailoring prompts becomes important. Prompt engineering is essential for interacting effectively with AI systems and optimising their performance, and better prompts consistently lead to improved results.

Modifying each component of a prompt affects the LLM's output in distinct ways.

### Role
Role changes the persona, tone, style, and perspective adopted by the model. Instructing the model to act as a "helpful assistant" differs significantly from instructing it to act as a "sceptical historian".

### Task
Task alters the core action and its constraints. The instruction "summarise this text" differs from "translate this text" or "write a poem under 50 words".

### Output Format
Output format dictates the structure of the response, such as bullet points versus paragraphs, or JSON versus plain text.

### Examples
Examples refine the style, structure, detail, and quality of the output by demonstrating desired patterns to the model.

### Context
Context modifies the background information and scope used by the model, such as providing a different source article to summarise.

**Example.** Refining a vague prompt by adjusting each component produces a much more useful instruction.

```
Original prompt: "Summarise this article."

Refined prompt:
  Role     -> "You are a science journalist writing for a general audience."
  Task     -> "Summarise the article in three bullet points."
  Format   -> "Each bullet point should be under 20 words."
  Examples -> "Example: 'Researchers found electric buses reduce
              urban emissions by 60 percent compared to diesel.'"
  Context  -> "Focus on practical implications rather than technical detail."
```

The refined prompt produces a focused, audience-aware, format-compliant summary, whereas the original prompt produces a generic and unpredictable output.

---

## 7. Prompt Chaining

Even a perfectly refined single prompt has limits, because some tasks are simply too complex for one model call to handle reliably. The natural next step is to decompose the task into a sequence of smaller, focused prompts, and this is precisely what prompt chaining does. Chaining marks the shift from single-prompt techniques to workflow-level patterns.

Prompt chaining, also referred to as sequential prompting or multi-step reasoning, is the practice of breaking down a complex task into a series of smaller, manageable sub-tasks (Task 1, Task 2, Task 3). Each sub-task is handled by its own dedicated prompt. Once well-defined individual prompts are available, the system programmatically calls the LLM with each prompt in succession, where the output of one call becomes the input for the next, running in code or via an orchestrator.

Prompt chaining connects the inputs and outputs of prompts programmatically and supports the construction of sophisticated, scalable workflows that handle complexity well beyond what a single prompt can manage. However, LLMs can hallucinate, produce incorrect formats, or fail to follow instructions, and an error in an early step can derail the entire process through a domino effect.

Prompt chaining therefore consists of three core components. Sequential prompting orders the sub-tasks logically by dependency. Task chaining passes outputs from one step into the next as input. Gate checks (programmatic validators placed between steps) verify the quality of intermediate outputs before they are passed downstream.

**Example.** Writing a short blog post about Sydney can be decomposed into three chained prompts.

```
Step 1 prompt -> "Generate 5 interesting topic ideas about Sydney."
                 Output: a list of 5 topics.

Step 2 prompt -> "Given these 5 topics: {output from Step 1},
                 pick the most engaging one and write a 4-point outline."
                 Output: a structured outline.

Step 3 prompt -> "Using this outline: {output from Step 2},
                 write a 500-word blog post in a casual tone."
                 Output: the final blog post.
```

Each step is simple, focused, and easier to validate than asking a single prompt to produce a polished blog post in one shot.

---

## 8. Gate Checks

Prompt chaining unlocks complex workflows but introduces a new fragility, namely that an error in one step can propagate downstream and corrupt the entire chain. Gate checks address this fragility directly, acting as the safety mechanism that makes chained workflows production-ready rather than merely possible.

Simply chaining prompts is not sufficient for building reliable systems, and intermediate validation is required. Gate checks fulfil this function. Gate checks are programmatic validations placed between steps in a prompt chain, acting as quality control points that ensure each output meets defined criteria before being passed to the next step.

When gate checks pass, the chain continues. When a check fails, the system can respond in one of three ways. It can trigger an error and halt execution. It can retry the failed step. Alternatively, it can retry the failed step while including the reason for failure in the prompt, which makes subsequent errors less likely on retry.

### Example 1. Validating LLM output in a prompt chain.

A gate check using Pydantic enforces that the LLM's output matches the expected schema for the next step in the chain.

Python developers most commonly use Pydantic for structured data validation in gate checks. Pydantic is the most popular data validation and parsing library for Python. It uses Python's standard type hints to enforce data structure, ensuring that variables, API requests, and database inputs conform to the exact shapes and data types expected by the system.

```python
from pydantic import BaseModel, Field, ValidationError
from typing import List

class BlogPost(BaseModel):
    title: str = Field(min_length=10, max_length=100)
    summary: str = Field(min_length=50, max_length=200)
    tags: List[str] = Field(min_length=3, max_length=10)

# After the LLM returns its output
try:
    validated = BlogPost.model_validate_json(llm_output)
    # Validation passed, proceed to next step
    next_step(validated)
except ValidationError as e:
    # Validation failed, retry with feedback
    retry_prompt = (
        f"Previous output failed validation with the following errors: {e}. "
        f"Please regenerate, ensuring the JSON conforms to the schema."
    )
```

The gate check confirms that the title is between 10 and 100 characters, the summary is between 50 and 200 characters, and at least three tags are present. Any failure triggers a structured retry rather than allowing malformed data to propagate downstream.

### Example 2. Gate checks in a ReAct environment with external tool failures.

ReAct agents rely heavily on external tools such as APIs and function calls, but these tools can fail in many ways. A weather API may return a rate-limit error, a malformed JSON response, a network timeout, or a payload missing expected fields. Without a gate check, the agent passes the broken observation directly into its next Thought step, where it either crashes or produces a confused reasoning chain. A gate check sits between the Action and Observation phases, validates the raw tool output against an expected schema, and triggers a controlled retry when validation fails.

```python
from pydantic import BaseModel, Field, ValidationError
from typing import Literal
import requests

# Expected schema for a successful weather API response
class WeatherResponse(BaseModel):
    location: str
    temperature: float = Field(ge=-50, le=60)
    rain_probability: int = Field(ge=0, le=100)
    conditions: str
    status: Literal["ok"] = "ok"

# ReAct Action step with embedded gate check and retry
def call_weather_api_with_gate(location: str, max_retries: int = 3):
    for attempt in range(1, max_retries + 1):
        try:
            # Action: call the external tool
            raw_response = requests.get(
                f"https://api.weather.example/v1/forecast?loc={location}",
                timeout=5
            ).json()

            # Gate Check: validate the observation against the schema
            validated = WeatherResponse.model_validate(raw_response)

            # Gate passed, return clean observation to the next Thought
            return validated

        except (ValidationError, requests.exceptions.RequestException) as e:
            # Gate failed, log and retry
            print(f"Attempt {attempt} failed: {e}")
            if attempt == max_retries:
                # Final failure, return a structured error the agent can reason about
                return {"status": "error", "reason": str(e), "attempts": attempt}
            continue
```

How this fits inside a ReAct loop is shown below.

```python
Thought 1.    I need current weather for Melbourne.
Action 1.     call_weather_api_with_gate("Melbourne, AU")

  [Inside the action]
  Attempt 1.  API returns {"error": "rate_limit_exceeded"}.
              Gate Check fails because the response does not match
              the WeatherResponse schema.
  Attempt 2.  API returns a valid payload.
              Gate Check passes.

Observation 1. WeatherResponse(location="Melbourne, AU", temperature=18,
                rain_probability=80, conditions="afternoon showers")

Thought 2.    Rain probability is 80 percent, so an umbrella is advisable.
Final Answer. Yes, bring an umbrella.
```

The gate check ensures the agent's reasoning chain only ever receives well-formed observations. Transient external failures such as rate limits, timeouts, or schema drift are absorbed by the retry mechanism rather than corrupting the wider workflow. This pattern is what makes ReAct agents reliable enough for production use, because in practice external tools fail far more often than the LLM itself does.

---

## 9. LLM Feedback Loops

Gate checks provide reactive quality control, validating outputs against fixed criteria and triggering retries when something goes wrong. Feedback loops generalise this idea into a broader and more powerful pattern, where the model itself participates in iterative evaluation and revision rather than simply being retried against a static schema. This is the most advanced workflow pattern in the document and brings the progression to its conclusion.

AI agents, with LLMs serving as their reasoning engine, can reason, plan, and act, but they are not perfect on first attempt. They sometimes make mistakes or require feedback to produce correct results. Iterative feedback loops are applied to refine LLM outputs for accuracy and quality.

Rather than relying on a single interaction, feedback loop systems allow the LLM to receive feedback, reflect, and try again, iteratively improving its response until it meets defined criteria. The principle resembles that of a student who needs feedback to revise and improve their work.

Implementing feedback loops involves moving beyond a single, one-shot interaction. An LLM's initial output may not be optimal, particularly for complex tasks. A passive LLM waits for input and responds, whereas an agent takes a goal, reasons, acts, observes results, and iterates, refining its approach based on observations. This iterative refinement is what feedback loops enable.

Feedback can originate from three sources. The model's own evaluation supports self-reflection. External validation applies automated checks, gate checks, or programmatic validators. User input introduces human-in-the-loop review and correction.

Incorporating feedback gives the LLM a mechanism to learn and correct itself within the task scope, without requiring retraining. The system effectively teaches the AI to recognise feedback, adjust its approach, and produce improved outputs over successive iterations.

**Example.** Generating a customer apology email through a feedback loop demonstrates iterative refinement.

```
Iteration 1. LLM generates draft email.
             Evaluator (a second LLM or a rule-based checker)
             reviews tone -> "Too casual for a corporate apology."

Iteration 2. LLM revises with feedback included in the prompt.
             Evaluator reviews -> "Tone is better, but missing a
             clear call-to-action."

Iteration 3. LLM revises again.
             Evaluator reviews -> "Approved. Tone and structure
             both meet criteria."

Final output. Polished email returned to the user.
```


