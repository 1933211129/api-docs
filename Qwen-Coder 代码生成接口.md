# 二、 Qwen-Coder 代码生成接口

## 1. 概览

- **API 名称**: Qwen-Coder 代码生成接口 (Qwen-Coder-30B)
- **接口状态**: `[稳定]`
- **接口描述**: 提供基于 Qwen-Coder-30B 大模型的代码生成与对话能力。它支持两种独立的调用模式：
  1. **单轮代码生成**: 用于一次性的代码生成任务，直接返回完整结果。
  2. **流式多轮对话**: 用于需要上下文记忆的连续对话，实时返回生成内容。

## 2. 接口 1: 单轮代码生成

此接口适用于独立的、无上下文关联的代码生成请求。

### 2.1. 请求

- **Endpoint (端点)**
  - `POST http://10.3.35.21:8001/v1/code/generate`
- **请求参数**
  - 请求体为 `application/x-www-form-urlencoded`，包含以下字段：

| 参数名   | 类型   | 是否必填 | 描述                                                |
| -------- | ------ | -------- | --------------------------------------------------- |
| `prompt` | string | 是       | 具体的代码生成指令，例如 "用Python写一个冒泡排序"。 |

### 2.2. 响应 

- **成功响应**

  - **HTTP 状态码**: `200 OK`

  - **响应体 (JSON 示例)**:

    ```shell
    {
      "status": "success",
      "result": "def bubble_sort(arr):\n    n = len(arr)\n    for i in range(n):\n        for j in range(0, n-i-1):\n            if arr[j] > arr[j+1]:\n                arr[j], arr[j+1] = arr[j+1], arr[j]"
    }
    ```

- **失败响应**

  - **HTTP 状态码**: `500 Internal Server Error`

  - **响应体 (JSON 示例)**:

    ```
    {
      "detail": "服务器内部错误: [具体的错误信息]"
    }
    ```

### 2.3. Python 调用示例

```python
import requests

SERVER_URL = "[http://10.3.35.21:8001](http://10.3.35.21:8001)"
API_ENDPOINT = f"{SERVER_URL}/v1/code/generate"

PROMPT = "用 Python 写一个最基础版的冒泡排序算法，只写函数体即可，不必写测试代码和示例"

def call_single_generation():
    """调用单轮代码生成接口"""
    print("="*60)
    print(f"🚀 正在向单轮接口发送请求...")
    print(f"📝 指令: {PROMPT}")
    print("="*60)

    try:
        # 构造请求数据
        data = {'prompt': PROMPT}
        # Coder模型推理较慢，设置长超时
        response = requests.post(API_ENDPOINT, data=data, timeout=360)

        if response.status_code == 200:
            result = response.json()
            print("✅ 请求成功！模型返回的完整代码如下：")
            print("-" * 60)
            print(result.get('result'))
            print("-" * 60)
        else:
            print(f"❌ 请求失败！")
            print(f"  - 状态码: {response.status_code}")
            print(f"  - 错误信息: {response.text}")

    except requests.exceptions.RequestException as e:
        print(f"❌ 网络请求异常！请检查服务器地址或网络连接。")
        print(f"  - 错误详情: {e}")

if __name__ == "__main__":
    call_single_generation()
```

## 3. 接口 2: 流式多轮代码对话

此接口适用于需要连续对话、保持上下文记忆的场景，并以流式方式实时返回结果。

### 3.1. 请求

- **Endpoint (端点)**
  - `POST http://10.3.35.21:8001/v1/code/chat_stream`
- **请求参数 **
  - 请求体为 `application/x-www-form-urlencoded`，包含以下字段：

| 参数名            | 类型   | 是否必填 | 描述                                                         |
| ----------------- | ------ | -------- | ------------------------------------------------------------ |
| `prompt`          | string | 是       | 当前轮次的对话内容。                                         |
| `conversation_id` | string | 否       | 会话ID。**首次请求时无需提供**，服务器会自动创建并返回。**后续请求必须携带此ID**以维持上下文。 |

### 3.2. 响应

- **HTTP 状态码**: `200 OK`
- **响应格式**: `text/event-stream`
  - **首次请求**: 响应流的开头会包含会话ID，格式为 `conversation_id:[UUID]\n---\n`，紧接着是模型生成的内内容。客户端需要解析并保存此ID用于后续对话。
  - **后续请求**: 响应流只包含模型生成的内容。

### 3.3. Python 调用示例 

```python
import requests
import time

SERVER_URL = "[http://10.3.35.21:8001](http://10.3.35.21:8001)"
API_ENDPOINT = f"{SERVER_URL}/v1/code/chat_stream"

def call_streaming_chat_session():
    """演示一个完整的流式多轮对话"""
    conversation_id = None

    # --- 第一轮对话：生成Python代码 ---
    prompt1 = "用 Python 写一个最基础版的冒泡排序算法，只写函数体即可，不必写测试代码和示例"
    print("="*60)
    print(f"🚀 [第一轮] 正在向流式接口发送请求...")
    print(f"📝 你: {prompt1}")
    print("="*60)
    print("🤖 AI: ", end="", flush=True)

    try:
        data = {'prompt': prompt1}
        with requests.post(API_ENDPOINT, data=data, stream=True, timeout=360) as response:
            if response.status_code == 200:
                header_processed = False
                for chunk in response.iter_content(chunk_size=None, decode_unicode=True):
                    if not header_processed and "conversation_id:" in chunk:
                        parts = chunk.split("\n---\n", 1)
                        header = parts[0]
                        conversation_id = header.replace("conversation_id:", "").strip()
                        print(f"\n[系统提示：已建立会话, ID: {conversation_id}]\n")
                        print("🤖 AI: ", end="", flush=True)
                        if len(parts) > 1: print(parts[1], end="", flush=True)
                        header_processed = True
                        continue
                    print(chunk, end="", flush=True)
                print("\n") # 完整回答结束后换行
            else:
                print(f"❌ 请求失败！状态码: {response.status_code}, 错误: {response.text}")
                return # 如果第一轮失败，则终止
    except requests.exceptions.RequestException as e:
        print(f"❌ 网络请求异常: {e}")
        return

    time.sleep(1)

    # --- 第二轮对话：将Python代码转为Java ---
    if not conversation_id:
        print("❌ 未能获取会话ID，无法进行第二轮对话。")
        return

    prompt2 = "非常好。现在，请把上面这段 Python 代码无缝地转换成功能完全等效的 Java 代码。"
    print("="*60)
    print(f"🚀 [第二轮] 正在继续对话...")
    print(f"📝 你: {prompt2}")
    print("="*60)
    print("🤖 AI: ", end="", flush=True)

    try:
        data = {'prompt': prompt2, 'conversation_id': conversation_id}
        with requests.post(API_ENDPOINT, data=data, stream=True, timeout=360) as response:
            if response.status_code == 200:
                for chunk in response.iter_content(chunk_size=None, decode_unicode=True):
                    # 第二轮不再需要处理ID，直接打印
                    if "conversation_id:" in chunk and "---" in chunk: continue
                    print(chunk, end="", flush=True)
                print("\n")
            else:
                print(f"❌ 请求失败！状态码: {response.status_code}, 错误: {response.text}")
    except requests.exceptions.RequestException as e:
        print(f"❌ 网络请求异常: {e}")

if __name__ == "__main__":
    call_streaming_chat_session()
```

## 4. 注意事项

1. **超时时间**：Coder-30B 模型较大，推理耗时较长。客户端调用时，`timeout` 参数建议设置为 **360秒** 或更高。
2. **会话管理**：流式对话的上下文保存在服务器内存中。如果服务重启，所有会话历史将丢失。

##  运维笔记

- **接口端口**: `8001`
- **配置项地址**: `/etc/systemd/system/qwen_coder_api.service`
- **代码位置**: `/data/wangmeng/qwen_llm_api/qwen_coder/qwen_coder_api.py`
- **接口源代码**:

```python
# -------------------------------------------------------------------------
# Qwen3-Coder-30B API 服务
# 功能: 提供单轮代码生成和流式多轮代码对话能力
# 加载方式: 使用原生 Transformers 从本地路径加载
# -------------------------------------------------------------------------

import torch
import uvicorn
from fastapi import FastAPI, Form, HTTPException
from fastapi.responses import StreamingResponse
from contextlib import asynccontextmanager
import uuid
import logging
from typing import List, Dict, Optional
from threading import Thread
from transformers import AutoModelForCausalLM, AutoTokenizer, TextIteratorStreamer

# ======================== 全局配置 ========================
# 1. 模型本地绝对路径
MODEL_PATH = "/data/LLM/Qwen/qwen3-coder-30b-a3b-instruct-local/"

# 2. API 服务监听端口
API_PORT = 8001

# 3. 模型生成参数
GENERATION_CONFIG = {
    "max_new_tokens": 4096,
    "temperature": 0.7,
    "top_p": 0.8,
    "top_k": 20,
    "repetition_penalty": 1.05
}

# 4. 日志配置
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
# ==========================================================

# --- 全局变量定义 ---
model = None
tokenizer = None
conversation_histories: Dict[str, List] = {}

# --- FastAPI 应用生命周期管理 ---
@asynccontextmanager
async def lifespan(app: FastAPI):
    global model, tokenizer
    logging.info(f"开始加载模型: {MODEL_PATH} ...")
    logging.info("这将占用约 75GB 显存，请确保硬件资源充足。")

    # 使用 transformers 的加载器，它能正确处理本地路径
    tokenizer = AutoTokenizer.from_pretrained(
        MODEL_PATH,
        local_files_only=True,
        trust_remote_code=True # 对于Qwen等复杂模型，必须信任其自带的代码
    )
    model = AutoModelForCausalLM.from_pretrained(
        MODEL_PATH,
        dtype="auto",
        device_map="auto",
        local_files_only=True,
        trust_remote_code=True # 必须
    )

    logging.info("模型加载成功，Coder API服务准备就绪！")
    yield

    model = None
    tokenizer = None
    torch.cuda.empty_cache()
    logging.info("模型资源已释放，服务已关闭。")

# --- 创建 FastAPI 应用实例 ---
app = FastAPI(lifespan=lifespan)

# ======================== API 接口定义 ========================

# --- 接口 1: 单轮代码生成 (Stateless) ---
@app.post("/v1/code/generate", summary="单轮代码生成")
async def generate_code(prompt: str = Form(..., description="单次的代码生成指令")):
    logging.info(f"[单轮请求] Prompt: {prompt[:100]}...")
    try:
        messages = [{"role": "user", "content": prompt}]
        
        text = tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
        model_inputs = tokenizer([text], return_tensors="pt").to(model.device)
        generated_ids = model.generate(**model_inputs, **GENERATION_CONFIG)
        output_ids = generated_ids[0][len(model_inputs.input_ids[0]):].tolist()
        result = tokenizer.decode(output_ids, skip_special_tokens=True)
        
        logging.info(f"[单轮响应] Result: {result[:100]}...")
        return {"status": "success", "result": result}
    except Exception as e:
        logging.error(f"[单轮请求] 发生错误: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))

# --- 接口 2: 流式多轮代码对话 (Stateful) ---
@app.post("/v1/code/chat_stream", summary="[流式] 多轮代码对话")
async def stream_chat_with_coder(
    prompt: str = Form(..., description="当前轮次的对话内容"),
    conversation_id: Optional[str] = Form(None, description="会话ID，用于维持上下文")
):
    logging.info(f"[流式请求] Conv ID: {conversation_id}, Prompt: {prompt[:100]}...")

    if not conversation_id:
        conversation_id = str(uuid.uuid4())
        conversation_histories[conversation_id] = []
        logging.info(f"[流式请求] 创建新会话: {conversation_id}")

    history = conversation_histories.get(conversation_id, [])
    history.append({"role": "user", "content": prompt})

    text = tokenizer.apply_chat_template(history, tokenize=False, add_generation_prompt=True)
    model_inputs = tokenizer([text], return_tensors="pt").to(model.device)

    streamer = TextIteratorStreamer(tokenizer, skip_prompt=True, skip_special_tokens=True)

    generation_kwargs = dict(**model_inputs, streamer=streamer, **GENERATION_CONFIG)
    thread = Thread(target=model.generate, kwargs=generation_kwargs)
    thread.start()

    async def stream_generator():
        full_response = ""
        # 在流的开头，先发送 conversation_id 和一个分隔符
        yield f"conversation_id:{conversation_id}\n---\n"
        
        for new_text in streamer:
            full_response += new_text
            yield new_text
        
        # 生成结束后，将完整的回答存入历史记录
        history.append({"role": "assistant", "content": full_response})
        conversation_histories[conversation_id] = history
        logging.info(f"[流式响应] Conv ID: {conversation_id}, Full Response: {full_response[:100]}...")

    return StreamingResponse(stream_generator(), media_type="text/event-stream")

# --- 主程序入口 ---
if __name__ == "__main__":
    logging.info(f"启动服务，监听 [http://0.0.0.0](http://0.0.0.0):{API_PORT}")
    uvicorn.run(app, host="0.0.0.0", port=API_PORT)
```

