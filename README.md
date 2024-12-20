# Jetson Chatbot for CO2 Monitoring and Ventilation Alerts

This project implements a **Jetson-based chatbot** capable of measuring CO2 levels using a **CM1106 sensor**, determining ventilation needs, and interacting with users through a chat interface. The system integrates Gradio for the chatbot UI and OpenAI for intelligent responses.

---

## Features

- **Real-time CO2 Monitoring**:
  - Measures CO2 concentration using a CM1106 sensor via serial communication.
  - Determines ventilation needs based on CO2 levels.
  
- **AI Chatbot**:
  - Provides intelligent responses about environmental conditions and ventilation status using OpenAI's API.
  - Interactive UI powered by Gradio.

- **Integrated Functionality**:
  - Combines sensor readings and AI responses to offer real-time, actionable insights.

---

## Installation

### Prerequisites
- **Hardware**:
  - NVIDIA Jetson Nano/Xavier
  - CM1106 CO2 Sensor
  - Serial-to-USB converter (if required)
  
- **Software**:
  - Python 3.8+
  - Required Python libraries (see below)

### Required Libraries
Install the following Python packages:
```python
# 필요 패키지를 설치합니다. pyserial은 시리얼 통신을 위한 라이브러리입니다.
!pip install pyserial

# smbus는 I2C 통신을 지원하는 패키지입니다.
!pip install smbus

# openai 라이브러리를 최신 버전으로 업데이트하여 설치합니다.
!pip install -U openai
```

---

### **Setting Up OpenAI API Integration**
This section of the code sets up the environment and imports the required libraries for interacting with OpenAI's API.

```python
# 기본적인 Python 표준 라이브러리와 OpenAI API를 가져옵니다.
import os  # 환경 변수 관리
from openai import OpenAI  # OpenAI API 호출용
import json  # JSON 데이터 처리를 위한 모듈

# OpenAI API 키를 환경 변수에 저장합니다.
os.environ['OPENAI_API_KEY'] = ''  # API 키
OpenAI.api_key = os.getenv("OPENAI_API_KEY")  # 환경 변수에서 API 키를 가져옵니다.
```

- **Imports**:
  - `os`: Manages environment variables.
  - `OpenAI`: Allows communication with the OpenAI API.
  - `json`: Handles JSON data for API requests and responses.

- **OpenAI API Key Setup**:
  - The `OPENAI_API_KEY` environment variable is set to store the API key securely.
  - The `OpenAI.api_key` is initialized by retrieving the value of the environment variable.

---

### **Jetson GPIO and Additional Libraries Setup**

This section imports the necessary libraries for controlling GPIO pins, managing time, performing mathematical operations, and handling serial communication.

```python
# Jetson GPIO를 가져옵니다 (Jetson 보드에서 GPIO 제어를 위한 라이브러리).
import Jetson.GPIO as GPIO
import time  # 시간 제어를 위한 모듈
import math  # 수학 연산용 모듈
import serial  # 시리얼 통신 라이브러리
```

- **Imports**:
  - `Jetson.GPIO`: For controlling GPIO pins on Jetson boards.
  - `time`: For adding delays or measuring time intervals.
  - `math`: Provides mathematical functions for computations.
  - `serial`: Facilitates serial communication with devices like sensors.

- **Purpose**:
  - Enables interaction with hardware components via GPIO.
  - Manages timing for operations that require delays or precise measurements.
  - Handles mathematical calculations and data processing.
  - Establishes serial communication with external devices, such as the CM1106 CO2 sensor.

---

### **CO2 Measurement Function**

This function, `measure_co2`, communicates with the CM1106 CO2 sensor via serial communication to measure the CO2 concentration in ppm (parts per million).

```python
def measure_co2():
    SERIAL_PORT = "/dev/ttyUSB0"  # 실제 연결된 포트로 변경
    BAUD_RATE = 9600

    try:
        with serial.Serial(SERIAL_PORT, BAUD_RATE, timeout=10) as ser:
            ser.write(b'\x11\x01\x01\xED')  # CM1106 센서 명령어
            time.sleep(10)  # 응답 대기 시간

            # 응답 데이터 읽기
            response = ser.read_all()  # 가능한 모든 데이터 읽기
            print(f"Raw response (bytes): {response}")

            try:
                response = response.decode('ascii')  # 디코딩 시도
                print(f"Decoded response: {response}")
            except Exception as decode_error:
                print(f"Decoding error: {decode_error}")
                return "Error: Failed to decode response."

            if response.startswith("CO2:"):
                co2_value = response.split(":")[1].strip()
                return co2_value
            else:
                print("response: " + response)
                print("Error: Unexpected response format.")
                return "Error: Unexpected response format."

    except serial.SerialException as e:
        print(f"Serial connection error: {e}")
        return "Error: Serial connection issue."

    except Exception as e:
        print(f"Error during measurement: {e}")
        return f"Error: {e}"
```

- **Parameters**:
  - `SERIAL_PORT`: Specifies the serial port connected to the CM1106 sensor (default: `/dev/ttyUSB0`).
  - `BAUD_RATE`: Sets the communication speed (default: `9600` baud).

- **Functionality**:
  1. **Serial Communication**:
     - Opens a serial connection with the specified port and baud rate.
     - Sends a command (`b'\x11\x01\x01\xED'`) to the CM1106 sensor to request CO2 data.
  2. **Response Handling**:
     - Reads all available data from the sensor.
     - Attempts to decode the response using ASCII format.
     - If the response starts with `"CO2:"`, extracts the CO2 concentration value.
  3. **Error Handling**:
     - Catches serial communication issues (`serial.SerialException`).
     - Handles unexpected response formats or decoding errors.

- **Returns**:
  - The CO2 concentration in ppm if successfully measured.
  - An error message if any issues occur during measurement.

---

### **Determine Ventilation Status Function**

This function, `determine_ventilation_status`, evaluates the need for ventilation based on the CO2 concentration provided as input.

```python
# CO2 농도를 측정하고 환기 필요 여부를 판단하는 함수
def determine_ventilation_status(co2_value):
    try:
        co2_value = int(co2_value)  # 사용자가 제공한 CO2 값을 정수로 변환
    except ValueError:
        return "올바른 이산화탄소 농도를 입력해 주세요. 예: 900ppm"

    # CO2 농도 범위에 따른 환기 판단
    if co2_value < 800:
        status = "농도가 적정 수준입니다. 환기가 필요하지 않습니다."
    elif co2_value <= 1000:
        status = "농도가 다소 높은 상태입니다. 환기를 권장합니다."
    else:
        status = "농도가 높아 공기질이 나쁩니다. 즉시 환기가 필요합니다."

    # 최종 메시지 반환
    return f"현재 이산화탄소 농도는 {co2_value}ppm입니다. {status}"
```

- **Parameters**:
  - `co2_value`: The CO2 concentration (in ppm) provided as input.

- **Functionality**:
  1. **Input Validation**:
     - Attempts to convert the provided `co2_value` to an integer.
     - Returns an error message if the input is invalid (e.g., non-numeric).
  2. **Ventilation Assessment**:
     - Determines the ventilation status based on the CO2 concentration:
       - **Below 800 ppm**: Indicates safe levels; no ventilation required.
       - **800 to 1000 ppm**: Suggests slightly elevated levels; ventilation recommended.
       - **Above 1000 ppm**: Warns of poor air quality; immediate ventilation required.
  3. **Response Generation**:
     - Constructs a message indicating the current CO2 concentration and ventilation status.

- **Returns**:
  - A message stating the CO2 concentration and whether ventilation is necessary.

---

### **Function List for OpenAI Interaction**

This section defines a list of functions that can be utilized in interactions with OpenAI to assess CO2 levels and determine ventilation requirements.

```python
# OpenAI와의 상호작용에서 CO2 상태를 판단하도록 함수 목록에 추가
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
            "name": "determine_ventilation_status",  # 함수 이름
            "description": "Determines ventilation needs based on CO2 concentration."
        }
    }
]
```

- **Purpose**:
  - Registers the functions `measure_co2` and `determine_ventilation_status` for use with OpenAI's API.
  - Provides descriptions for each function, enabling OpenAI to understand their purposes and call them appropriately.

- **Functions**:
  1. **`measure_co2`**:
     - Reads CO2 concentration from the CM1106 sensor via a serial port.
     - Returns the measured CO2 value in parts per million (ppm).
     - Description: `"Reads CO2 concentration from a CM1106 sensor connected via serial port and returns the measured value in ppm."`
  2. **`determine_ventilation_status`**:
     - Determines if ventilation is needed based on the provided CO2 concentration.
     - Description: `"Determines ventilation needs based on CO2 concentration."`

- **Structure**:
  - Each function is defined with its `name` and a brief `description` to guide OpenAI in understanding their functionality during interaction.

---

### **ask_openai Function**

The `ask_openai` function integrates OpenAI's LLM (Language Learning Model) API with the chatbot. It processes user inputs, invokes the appropriate tools or functions, and generates responses.

```python
def ask_openai(llm_model, messages, user_message, functions = ''):
    client = OpenAI()
    proc_messages = messages

    if user_message != '':
        proc_messages.append({"role": "user", "content": user_message})

    if functions == '':
        response = client.chat.completions.create(model=llm_model, messages=proc_messages, temperature = 1.0)
    else:
        response = client.chat.completions.create(model=llm_model, messages=proc_messages, tools=functions, tool_choice="auto") # 이전 코드와 바뀐 부분

    response_message = response.choices[0].message
    tool_calls = response_message.tool_calls

    if tool_calls:
        # Step 3: call the function
        # Note: the JSON response may not always be valid; be sure to handle errors

        available_functions = {
            "measure_co2": measure_co2,
            "determine_ventilation_status": determine_ventilation_status
        }

        messages.append(response_message)  # extend conversation with assistant's reply

        # Step 4: send the info for each function call and function response to GPT
        for tool_call in tool_calls:
            function_name = tool_call.function.name
            function_to_call = available_functions[function_name]
            function_args = json.loads(tool_call.function.arguments)


            
            print(function_args)
           

            if function_name == "measure_co2":
                co2_result= function_to_call(**function_args)
                
                ventilation_result = available_functions["determine_ventilation_status"](co2_result)

                proc_messages.append(
                    {
                        "tool_call_id": tool_call.id,
                        "role": "tool",
                        "name": "determine_ventilation_status",
                        "content": ventilation_result,
                    }
                )
                
            else:
                
                if 'user_prompt' in function_args:
                    function_response = function_to_call(function_args.get('user_prompt'))
                else:
                    function_response = function_to_call(**function_args)
    
    
                proc_messages.append(
                    {
                        "tool_call_id": tool_call.id,
                        "role": "tool",
                        "name": function_name,
                        "content": function_response,
                    }
                )  # extend conversation with function response

                
        second_response = client.chat.completions.create(
            model=llm_model,
            messages=messages,
        )  # get a new response from GPT where it can see the function response

        assistant_message = second_response.choices[0].message.content
    else:
        assistant_message = response_message.content

    text = assistant_message.replace('\n', ' ').replace(' .', '.').strip()


    proc_messages.append({"role": "assistant", "content": assistant_message})

    return proc_messages, text
```

- **Parameters**:
  1. **`llm_model`**: Specifies the model name (e.g., `gpt-4`, `gpt-4o-mini`) used for generating responses.
  2. **`messages`**: Maintains the conversation history as a list of dictionaries containing roles (`user`, `assistant`, etc.) and messages.
  3. **`user_message`**: The current message input from the user.
  4. **`functions`** (optional): A list of functions available for the chatbot to use.

---

- **Functionality**:
  1. **Message Processing**:
     - Appends the user's message to the conversation history (`proc_messages`).
  2. **OpenAI Interaction**:
     - Sends the conversation history and available functions (if provided) to OpenAI.
     - Receives a response containing generated content or tool calls.
  3. **Tool Execution**:
     - If the response includes a tool call, invokes the corresponding Python function (e.g., `measure_co2` or `determine_ventilation_status`) using the arguments provided.
     - Handles the results and appends them back to the conversation.
  4. **Final Response**:
     - After processing tool calls, sends the updated conversation back to OpenAI for generating a final response.

---

- **Error Handling**:
  - Handles JSON parsing errors and invalid tool calls gracefully.
  - Ensures smooth execution even if tool responses are not valid.

- **Returns**:
  - Updated conversation history (`proc_messages`) including assistant responses.
  - The assistant's final message as plain text.

---

### **Gradio Chatbot Interface**

This section sets up a chatbot interface using **Gradio**, which allows for user interaction with the chatbot in real-time.

```python
import gradio as gr
import random


messages = []

def process(user_message, chat_history):

    # def ask_openai(llm_model, messages, user_message, functions = ''):
    proc_messages, ai_message = ask_openai("gpt-4o-mini", messages, user_message, functions= use_functions)

    chat_history.append((user_message, ai_message))
    return "", chat_history

with gr.Blocks() as demo:
    chatbot = gr.Chatbot(label="채팅창")
    user_textbox = gr.Textbox(label="입력")
    user_textbox.submit(process, [user_textbox, chatbot], [user_textbox, chatbot])

demo.launch(share=True, debug=True)
```

- **Imports**:
  - `gradio as gr`: The Gradio library is used to create a web-based UI for the chatbot.
  - `random`: Used for generating random outputs (if applicable).

- **Components**:
  1. **`messages`**:
     - A global list to store the conversation history between the user and the chatbot.
  2. **`process` Function**:
     - Handles user input and updates the chat history.
     - Calls the `ask_openai` function with the provided `user_message`.
     - Updates the `chat_history` with the user's message and the chatbot's response.
  3. **Gradio Interface (`demo`)**:
     - Uses `gr.Blocks` to create a modular chatbot interface.
     - Components:
       - `gr.Chatbot`: Displays the chat conversation.
       - `gr.Textbox`: Accepts user input for the chatbot.
     - Submits the user input using the `process` function.

- **Execution**:
  - The `demo.launch` function starts the Gradio application.
  - Parameters:
    - `share=True`: Enables sharing the application via a public link.
    - `debug=True`: Provides detailed logs for debugging purposes.

---
### **How Function Calling Works in My Code**

**Function Calling** in my code allows OpenAI to dynamically invoke specific Python functions during a conversation. Based on user input, the AI identifies the appropriate function to call, executes it, and integrates the results into its response. This capability enables the chatbot to interact with real-world data and provide intelligent, context-aware replies.

For detailed explanation, [click here](#).


---

### **Result Video**
[Watch the chatbot in action as it monitors CO2 levels and determines ventilation needs!](https://youtu.be/-8HzmXPytvM)
