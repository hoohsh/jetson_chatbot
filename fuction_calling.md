# How Function Calling Works in My Code

**Function Calling** in my code allows OpenAI to dynamically invoke specific Python functions during a conversation. Based on user input, the AI identifies the appropriate function to call, executes it, and integrates the results into its response. This capability enables the chatbot to interact with real-world data and provide intelligent, context-aware replies.

---

## 1. Defining the Function List

The `use_functions` list registers the functions OpenAI can call, along with descriptions of their purposes.

```python
use_functions = [
    {
        "type": "function",
        "function": {
            "name": "measure_co2",
            "description": "Reads CO2 concentration from a CM1106 sensor connected via serial port and returns the measured value in ppm."
        }
    },
    {
        "type": "function",
        "function": {
            "name": "determine_ventilation_status",
            "description": "Determines ventilation needs based on CO2 concentration."
        }
    }
]
```

---

## 2. Sending Requests to OpenAI
User messages and conversation history (messages) are passed to the ask_openai function. The functions parameter contains the use_functions list, enabling OpenAI to process tool calls.

```python
response = client.chat.completions.create(
    model=llm_model,
    messages=proc_messages,
    tools=functions,
    tool_choice="auto"
)
```

---

## 3. Parsing Tool Calls
OpenAI’s response may include tool calls indicating which function should be invoked.

```python
response_message = response.choices[0].message
tool_calls = response_message.tool_calls
```

Example Tool Call:

```json
{
    "function_name": "measure_co2",
    "arguments": {}
}
```

---

## 4. Executing Functions
Based on the tool call, the corresponding Python function is executed.

```python
available_functions = {
    "measure_co2": measure_co2,
    "determine_ventilation_status": determine_ventilation_status
}

function_to_call = available_functions[tool_call.function.name]
function_args = json.loads(tool_call.function.arguments)
function_response = function_to_call(**function_args)
```

---

## 5. Processing Results and Generating Responses
The results of the function call are appended to the conversation history and sent back to OpenAI for final response generation.

```python
proc_messages.append(
    {
        "tool_call_id": tool_call.id,
        "role": "tool",
        "name": tool_call.function.name,
        "content": function_response,
    }
)

second_response = client.chat.completions.create(
    model=llm_model,
    messages=proc_messages
)
```

---

## Execution Flow Example

1. **User Input**:  
   - "What is the current CO2 concentration?"

2. **OpenAI Tool Call**:  
   - Calls `measure_co2` to retrieve CO2 concentration (e.g., `900 ppm`).

3. **Tool Chain**:  
   - Calls `determine_ventilation_status` to evaluate the need for ventilation.

4. **Final Response**:  
   - "The current CO2 concentration is 900 ppm. Ventilation is recommended."

---

## Advantages of Function Calling

- **Dynamic Functionality**: OpenAI can directly invoke functions to integrate real-time data into responses.
- **Efficiency**: Only the necessary functions are executed based on the user query.
- **Interactive Intelligence**: Combines Python's functional capabilities with OpenAI's language model for intelligent, actionable conversations.

