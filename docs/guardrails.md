# Guardrails

Guardrails run _in parallel_ to your agents, enabling you to do checks and validations of user input. For example, imagine you have an agent that uses a very smart (and hence slow/expensive) model to help with customer requests. You wouldn't want malicious users to ask the model to help them with their math homework. So, you can run a guardrail with a fast/cheap model. If the guardrail detects malicious usage, it can immediately raise an error, which stops the expensive model from running and saves you time/money.

There are two kinds of guardrails:

1. Input guardrails run on the initial user input
2. Output guardrails run on the final agent output

## Input guardrails

Input guardrails run in 3 steps:

1. First, the guardrail receives the same input passed to the agent.
2. Next, the guardrail function runs to produce a [`GuardrailFunctionOutput`][agents.guardrail.GuardrailFunctionOutput], which is then wrapped in an [`InputGuardrailResult`][agents.guardrail.InputGuardrailResult]
3. Finally, we check if [`.tripwire_triggered`][agents.guardrail.GuardrailFunctionOutput.tripwire_triggered] is true. If true, an [`InputGuardrailTripwireTriggered`][agents.exceptions.InputGuardrailTripwireTriggered] exception is raised, so you can appropriately respond to the user or handle the exception.

!!! Note

    Input guardrails are intended to run on user input, so an agent's guardrails only run if the agent is the *first* agent. You might wonder, why is the `guardrails` property on the agent instead of passed to `Runner.run`? It's because guardrails tend to be related to the actual Agent - you'd run different guardrails for different agents, so colocating the code is useful for readability.

## Output guardrails

Output guardrails run in 3 steps:

1. First, the guardrail receives the output produced by the agent.
2. Next, the guardrail function runs to produce a [`GuardrailFunctionOutput`][agents.guardrail.GuardrailFunctionOutput], which is then wrapped in an [`OutputGuardrailResult`][agents.guardrail.OutputGuardrailResult]
3. Finally, we check if [`.tripwire_triggered`][agents.guardrail.GuardrailFunctionOutput.tripwire_triggered] is true. If true, an [`OutputGuardrailTripwireTriggered`][agents.exceptions.OutputGuardrailTripwireTriggered] exception is raised, so you can appropriately respond to the user or handle the exception.

!!! Note

    Output guardrails are intended to run on the final agent output, so an agent's guardrails only run if the agent is the *last* agent. Similar to the input guardrails, we do this because guardrails tend to be related to the actual Agent - you'd run different guardrails for different agents, so colocating the code is useful for readability.

## Tripwires

If the input or output fails the guardrail, the Guardrail can signal this with a tripwire. As soon as we see a guardrail that has triggered the tripwires, we immediately raise a `{Input,Output}GuardrailTripwireTriggered` exception and halt the Agent execution.

## Implementing a guardrail

You need to provide a function that receives input, and returns a [`GuardrailFunctionOutput`][agents.guardrail.GuardrailFunctionOutput]. In this example, we'll do this by running an Agent under the hood.

```python
from pydantic import BaseModel
from agents import (
    Agent,
    GuardrailFunctionOutput,
    InputGuardrailTripwireTriggered,
    RunContextWrapper,
    Runner,
    TResponseInputItem,
    input_guardrail,
)

class MathHomeworkOutput(BaseModel):
    is_math_homework: bool
    reasoning: str

guardrail_agent = Agent( # (1)!
    name="Guardrail check",
    instructions="Check if the user is asking you to do their math homework.",
    output_type=MathHomeworkOutput,
)


@input_guardrail
async def math_guardrail( # (2)!
    ctx: RunContextWrapper[None], agent: Agent, input: str | list[TResponseInputItem]
) -> GuardrailFunctionOutput:
    result = await Runner.run(guardrail_agent, input, context=ctx.context)

    return GuardrailFunctionOutput(
        output_info=result.final_output, # (3)!
        tripwire_triggered=result.final_output.is_math_homework,
    )


agent = Agent(  # (4)!
    name="Customer support agent",
    instructions="You are a customer support agent. You help customers with their questions.",
    input_guardrails=[math_guardrail],
)

async def main():
    # This should trip the guardrail
    try:
        await Runner.run(agent, "Hello, can you help me solve for x: 2x + 3 = 11?")
        print("Guardrail didn't trip - this is unexpected")

    except InputGuardrailTripwireTriggered:
        print("Math homework guardrail tripped")
```

1. We'll use this agent in our guardrail function.
2. This is the guardrail function that receives the agent's input/context, and returns the result.
3. We can include extra information in the guardrail result.
4. This is the actual agent that defines the workflow.

Output guardrails are similar.

```python
from pydantic import BaseModel
from agents import (
    Agent,
    GuardrailFunctionOutput,
    OutputGuardrailTripwireTriggered,
    RunContextWrapper,
    Runner,
    output_guardrail,
)
class MessageOutput(BaseModel): # (1)!
    response: str

class MathOutput(BaseModel): # (2)!
    reasoning: str
    is_math: bool

guardrail_agent = Agent(
    name="Guardrail check",
    instructions="Check if the output includes any math.",
    output_type=MathOutput,
)

@output_guardrail
async def math_guardrail(  # (3)!
    ctx: RunContextWrapper, agent: Agent, output: MessageOutput
) -> GuardrailFunctionOutput:
    result = await Runner.run(guardrail_agent, output.response, context=ctx.context)

    return GuardrailFunctionOutput(
        output_info=result.final_output,
        tripwire_triggered=result.final_output.is_math,
    )

agent = Agent( # (4)!
    name="Customer support agent",
    instructions="You are a customer support agent. You help customers with their questions.",
    output_guardrails=[math_guardrail],
    output_type=MessageOutput,
)

async def main():
    # This should trip the guardrail
    try:
        await Runner.run(agent, "Hello, can you help me solve for x: 2x + 3 = 11?")
        print("Guardrail didn't trip - this is unexpected")

    except OutputGuardrailTripwireTriggered:
        print("Math output guardrail tripped")
```

1. This is the actual agent's output type.
2. This is the guardrail's output type.
3. This is the guardrail function that receives the agent's output, and returns the result.
4. This is the actual agent that defines the workflow.

## Tool Guardrails

In addition to agent-level input and output guardrails, you can also apply guardrails directly to individual tools. Tool guardrails provide fine-grained control over individual tool executions by allowing you to validate both tool inputs and outputs. This is useful for scenarios where you need to:

- Block sensitive data from being passed to specific tools (input validation)
- Prevent sensitive data from being returned by specific tools (output validation)
- Apply specific security checks to individual tools

Tool guardrails work similarly to agent-level guardrails but operate at the individual tool call level. They are defined using the [`ToolInputGuardrail`][agents.tool_guardrails.ToolInputGuardrail] and [`ToolOutputGuardrail`][agents.tool_guardrails.ToolOutputGuardrail] classes, which can be created using decorators or directly.

### Tool Input Guardrails

Tool input guardrails run before a tool is executed. They receive the tool arguments and can decide whether to allow, reject, or raise an exception. The guardrail function receives a [`ToolInputGuardrailData`][agents.tool_guardrails.ToolInputGuardrailData] object containing the context and agent information.

```python
from agents import tool_input_guardrail, ToolInputGuardrailData, ToolGuardrailFunctionOutput
import json

@tool_input_guardrail
def reject_sensitive_words(data: ToolInputGuardrailData) -> ToolGuardrailFunctionOutput:
    """Reject tool calls that contain sensitive words in arguments."""
    try:
        args = json.loads(data.context.tool_arguments) if data.context.tool_arguments else {}
    except json.JSONDecodeError:
        return ToolGuardrailFunctionOutput(output_info="Invalid JSON arguments")

    sensitive_words = ["password", "hack", "exploit", "malware", "ACME"]
    for key, value in args.items():
        value_str = str(value).lower()
        for word in sensitive_words:
            if word.lower() in value_str:
                return ToolGuardrailFunctionOutput.reject_content(
                    message=f"🚨 Tool call blocked: contains '{word}'",
                    output_info={"blocked_word": word, "argument": key},
                )

    return ToolGuardrailFunctionOutput(output_info="Input validated")
```

When a tool input guardrail's behavior is set to `raise_exception`, it raises a [`ToolInputGuardrailTripwireTriggered`][agents.exceptions.ToolInputGuardrailTripwireTriggered] exception.

### Tool Output Guardrails

Tool output guardrails run after a tool is executed but before the result is returned to the agent. They can inspect the tool's output and decide how to handle it. The guardrail function receives a [`ToolOutputGuardrailData`][agents.tool_guardrails.ToolOutputGuardrailData] object that extends the input data with the tool's output.

```python
from agents import tool_output_guardrail, ToolOutputGuardrailData

@tool_output_guardrail
def block_sensitive_output(data: ToolOutputGuardrailData) -> ToolGuardrailFunctionOutput:
    """Block tool outputs that contain sensitive data."""
    output_str = str(data.output).lower()

    if "ssn" in output_str or "123-45-6789" in output_str:
        return ToolGuardrailFunctionOutput.raise_exception(
            output_info={"blocked_pattern": "SSN", "tool": data.context.tool_name},
        )

    return ToolGuardrailFunctionOutput(output_info="Output validated")
```

When a tool output guardrail's behavior is set to `raise_exception`, it raises a [`ToolOutputGuardrailTripwireTriggered`][agents.exceptions.ToolOutputGuardrailTripwireTriggered] exception.

### Applying Guardrails to Tools

You can attach guardrails to function tools by setting the `tool_input_guardrails` and `tool_output_guardrails` properties. These properties accept a list of guardrails:

```python
from agents import function_tool

@function_tool
def send_email(to: str, subject: str, body: str) -> str:
    """Send an email to the specified recipient."""
    return f"Email sent to {to} with subject '{subject}'"

# Apply guardrails to the tool
send_email.tool_input_guardrails = [reject_sensitive_words]
send_email.tool_output_guardrails = [block_sensitive_output]
```

### Guardrail Behavior Types

Tool guardrails return a [`ToolGuardrailFunctionOutput`][agents.tool_guardrails.ToolGuardrailFunctionOutput] object that can specify different behavior types:

- [`ToolGuardrailFunctionOutput.allow()`][agents.tool_guardrails.ToolGuardrailFunctionOutput.allow]: Allow normal tool execution to continue
- [`ToolGuardrailFunctionOutput.reject_content()`][agents.tool_guardrails.ToolGuardrailFunctionOutput.reject_content]: Reject the tool call/output but continue execution with a message to the model
- [`ToolGuardrailFunctionOutput.raise_exception()`][agents.tool_guardrails.ToolGuardrailFunctionOutput.raise_exception]: Halt execution by raising either a `ToolInputGuardrailTripwireTriggered` or `ToolOutputGuardrailTripwireTriggered` exception depending on the guardrail type

Additionally, you can create guardrails directly using the [`ToolInputGuardrail`][agents.tool_guardrails.ToolInputGuardrail] and [`ToolOutputGuardrail`][agents.tool_guardrails.ToolOutputGuardrail] classes instead of using decorators:

```python
from agents import ToolInputGuardrail, ToolOutputGuardrail

def my_input_guardrail_func(data: ToolInputGuardrailData) -> ToolGuardrailFunctionOutput:
    # Input guardrail logic
    return ToolGuardrailFunctionOutput.allow()

def my_output_guardrail_func(data: ToolOutputGuardrailData) -> ToolGuardrailFunctionOutput:
    # Output guardrail logic
    return ToolGuardrailFunctionOutput.allow()

my_input_guardrail = ToolInputGuardrail(guardrail_function=my_input_guardrail_func)
my_output_guardrail = ToolOutputGuardrail(guardrail_function=my_output_guardrail_func)
```