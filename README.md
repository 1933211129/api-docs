# 一、 Qwen-VL 图像识别接口

## 1. 概览

- **API 名称**: Qwen-VL 图像识别接口 (qwen-VL-7B)
- **接口状态**: `[稳定]`
- **接口描述**: 提供基于 Qwen-VL-7B 大模型的图文对话能力。用户可以上传一张图片，并附带一个问题（Prompt），接口将返回模型对该问题的文本回答。

## 2. 请求

- **Endpoint (端点)**
  - `POST http://10.3.35.21:8000/v1/image/recognize`
- **请求头 (Headers)**
  - 该接口使用 `multipart/form-data` 格式进行文件上传，`requests` 库会自动处理，无需手动设置 `Content-Type`。
- **请求参数 (Parameters)**
  - 请求体为 `multipart/form-data`，包含以下两个字段：

| 参数名   | 类型   | 是否必填 | 描述                                                         |
| -------- | ------ | -------- | ------------------------------------------------------------ |
| `image`  | File   | 是       | 用户上传的原始图片文件。支持 `png`, `jpg`, `jpeg` 等常见格式。 |
| `prompt` | string | 是       | 用户对图片提出的问题或指令，例如 "这张图里有什么？"。        |

## 3. 响应

#### 成功响应

- **HTTP 状态码**: `200 OK`
- **响应体 (JSON 示例)**:

```shell
{
  "status": "success",
  "result": "这张图片展示了一只猫正躺在桌子上睡觉，旁边放着一个笔记本电脑。"
}
```

#### 失败响应 

- **HTTP 状态码**: `400 Bad Request`

  - **原因**: 上传的文件非图片类型。

  - **响应体 (JSON 示例)**:

    ```shell
    {
      "detail": "上传的文件必须是图片类型，但收到的是 application/octet-stream。"
    }
    ```

- **HTTP 状态码**: `500 Internal Server Error`

  - **原因**: 服务器模型加载失败、推理时发生未知错误等。

  - **响应体 (JSON 示例)**:

    ```shell
    {
      "detail": "服务器内部错误: [具体的错误信息]"
    }
    ```

## 4. Python 调用示例

```python
import requests
import os
import mimetypes

# ======================== 配置 ========================
# !!! 请修改为服务器的正确IP地址
SERVER_IP = "10.3.35.21"
# !!! 请修改为你本地测试图片的路径
IMAGE_PATH = "/path/to/your/local/image.png"
# =======================================================

API_URL = f"http://{SERVER_IP}:8000/v1/image/recognize"

def call_recognition_api(image_path: str, prompt: str):
    """
    调用图文识别API
    :param image_path: 本地图片文件的路径
    :param prompt: 向模型提出的问题
    """
    if not os.path.exists(image_path):
        print(f"❌ 错误：找不到图片文件: {image_path}")
        return

    print(f"准备向服务器 {API_URL} 发送请求...")
    print(f"图片: {image_path}")
    print(f"问题: {prompt}")

    # 1. 猜测文件的MIME类型 (例如 'image/png')
    content_type, _ = mimetypes.guess_type(image_path)
    if content_type is None:
        # 如果无法猜测，提供一个默认值
        content_type = 'application/octet-stream'
    print(f"已识别图片类型为: {content_type}")
    # ---------------------------------------------

    try:
        with open(image_path, 'rb') as image_file:
            # 格式为: (文件名, 文件对象, Content-Type)
            files = {'image': (os.path.basename(image_path), image_file, content_type)}
            data = {'prompt': prompt}

            # 注意：timeout=60，因为VL模型推理可能较慢
            response = requests.post(API_URL, files=files, data=data, timeout=60)

            if response.status_code == 200:
                result = response.json()
                print("\n--- ✅ 请求成功 ---")
                print("模型返回的识别结果:")
                print(result.get('result'))
            else:
                print(f"\n--- ❌ 请求失败 ---")
                print(f"服务器返回状态码: {response.status_code}")
                print(f"错误详情: {response.text}")

    except requests.exceptions.Timeout:
        print(f"\n--- ❌ 请求超时 ---")
        print("请求超过60秒未收到响应，服务器可能正在处理或负载较高。")
    except requests.exceptions.RequestException as e:
        print(f"\n--- ❌ 网络请求异常 ---")
        print(f"具体错误: {e}")

if __name__ == "__main__":
    # 在这里修改你的问题
    question = "这张图里有什么？请用中文描述一下。"
    call_recognition_api(IMAGE_PATH, question)
```

## 5. 注意事项

1. **超时时间**：这是一个 GPU 密集型任务，模型推理可能需要几秒到几十秒。客户端调用时，`timeout` 参数建议设置为 **60秒** 或更高，以防请求超时。
2. **图片大小**：请尽量避免上传超大分辨率（如 8K）或体积过大（如 > 20MB）的图片，以免导致服务器处理超时或内存溢出。
3. **并发限制**：不建议超过 4 个并发请求。

## 运维记录

- **接口端口**: `8000`
- **配置项地址**: `/etc/systemd/system/qwen_vl_api.service`
- **代码位置**: `/data/wangmeng/qwen_llm_api/qwen_vl/qwen_vl.py`
- **接口源代码**:

```python
# -------------------------------------------------------------------------
# Qwen3-VL (8B) 图文识别 API 服务
# -------------------------------------------------------------------------
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager
import torch
import uvicorn
from fastapi import FastAPI, File, UploadFile, Form, HTTPException
import os
import uuid
import logging
from modelscope import Qwen3VLForConditionalGeneration, AutoProcessor
from typing import Optional

# ======================== 全局配置 ========================
MODEL_PATH = "/data/LLM/Qwen/Qwen3-VL-8B-Instruct"
TEMP_IMAGE_DIR = "/data/wangmeng/qwen_vl"
API_PORT = 8000
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
# ==========================================================

model = None
processor = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global model, processor
    os.makedirs(TEMP_IMAGE_DIR, exist_ok=True)
    logging.info(f"图片临时目录已确认: {TEMP_IMAGE_DIR}")
    logging.info(f"开始加载模型: {MODEL_PATH} ...")
    model = Qwen3VLForConditionalGeneration.from_pretrained(MODEL_PATH, dtype="auto", device_map="auto")
    processor = AutoProcessor.from_pretrained(MODEL_PATH)
    logging.info("模型加载成功，API服务准备就绪！")
    yield
    model = None
    processor = None
    torch.cuda.empty_cache()
    logging.info("模型资源已释放。")

app = FastAPI(lifespan=lifespan)
# ==================== CORS配置 ====================
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
# ==========================================================

@app.post("/v1/image/recognize", summary="单轮图文识别")
async def recognize_image(
    prompt: str = Form(...),
    image: UploadFile = File(...)
):
    logging.info("--- 请求已进入处理函数 ---")
    if image is None:
        logging.error("严重错误：'image' 参数为 None！")
        raise HTTPException(status_code=500, detail="服务器未能正确解析图片文件。")

    logging.info(f"接收到的 'image' 对象类型: {type(image)}")
    logging.info(f"图片文件名: {image.filename}, 图片类型: {image.content_type}")

    if not image.content_type or not image.content_type.startswith("image/"):
        raise HTTPException(status_code=400, detail=f"上传的文件必须是图片类型，但收到的是 {image.content_type}。")

    temp_file_path = None
    try:
        # 1. 保存上传的图片
        file_ext = os.path.splitext(image.filename)[1]
        temp_file_path = os.path.join(TEMP_IMAGE_DIR, f"{uuid.uuid4()}{file_ext}")
        content = await image.read()
        with open(temp_file_path, "wb") as f:
            f.write(content)
        logging.info(f"接收到请求 -> Prompt: '{prompt}', 图片已保存至: {temp_file_path}")

        # 2. 构建 messages
        messages = [{"role": "user", "content": [{"type": "image", "image": temp_file_path}, {"type": "text", "text": prompt}]}]

        # 3. 推理
        inputs = processor.apply_chat_template(messages, tokenize=True, add_generation_prompt=True, return_dict=True, return_tensors="pt").to(model.device)
        generated_ids = model.generate(**inputs, max_new_tokens=1024)

        # 4. 解码
        generated_ids_trimmed = [out_ids[len(in_ids):] for in_ids, out_ids in zip(inputs.input_ids, generated_ids)]
        output_text = processor.batch_decode(generated_ids_trimmed, skip_special_tokens=True, clean_up_tokenization_spaces=False)
        response_text = output_text[0] if output_text else ""
        logging.info(f"模型返回结果 -> {response_text[:100]}...")

        return {"status": "success", "result": response_text}

    except Exception as e:
        logging.error(f"处理请求时发生严重错误: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail=f"服务器内部错误: {str(e)}")

    finally:
        # 7. 清理临时文件
        if temp_file_path and os.path.exists(temp_file_path):
            os.remove(temp_file_path)
            logging.info(f"清理完毕 -> 临时文件已删除: {temp_file_path}")

if __name__ == "__main__":
    logging.info(f"启动服务，监听 [http://0.0.0.0](http://0.0.0.0):{API_PORT}")
    uvicorn.run(app, host="0.0.0.0", port=API_PORT)
```

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

# 三、 MinerU接口文档

## 1. 概览

- **API 名称**: MinerU PDF 文档解析接口
- **接口状态**: `[稳定]`
- **接口描述**: 提供强大的 PDF 文档解析能力，支持将文档转换为 Markdown、JSON 等多种格式，并能提取表格、公式和图片。接口分为三种模式：上传解析并保存到服务器、上传解析直接返回 JSON、处理服务器内部已有的文件。

## 2. 接口 1: /file_parse - 单文件解析保存

此接口用于上传一个或多个 PDF 文件，服务器解析后将结果保存到指定目录。

### 2.1. 请求

- **Endpoint (端点)**
  - `POST http://10.3.35.21:8003/file_parse`
- **请求参数 **
  - 请求体为 `multipart/form-data`，包含以下字段：

| 参数名                | 类型    | 是否必填 | 默认值                       | 描述                                                         |
| --------------------- | ------- | -------- | ---------------------------- | ------------------------------------------------------------ |
| `files`               | File    | 是       | -                            | 一个或多个文件。在 cURL 或 Postman 中可以重复此字段以上传多个文件。 |
| `output_dir`          | string  | 否       | `/data/mineru/parsed_output` | 解析结果在服务器上的保存目录。                               |
| `lang_list`           | list    | 否       | `["ch"]`                     | 语言列表，例如 `["ch", "en"]`。                              |
| `backend`             | string  | 否       | `pipeline`                   | 后端解析引擎类型。                                           |
| `parse_method`        | string  | 否       | `auto`                       | 解析方法。                                                   |
| `formula_enable`      | boolean | 否       | `true`                       | 是否启用公式识别。                                           |
| `table_enable`        | boolean | 否       | `true`                       | 是否启用表格识别。                                           |
| `return_md`           | boolean | 否       | `true`                       | 是否生成 `.md` 文件。                                        |
| `return_middle_json`  | boolean | 否       | `true`                       | 是否生成中间过程的 `.json` 文件。                            |
| `return_content_list` | boolean | 否       | `true`                       | 是否生成内容列表的 `.json` 文件。                            |
| `return_images`       | boolean | 否       | `true`                       | 是否提取并保存图片。                                         |
| `response_format_zip` | boolean | 否       | `false`                      | 是否将所有输出文件打包成 ZIP 格式返回。                      |
| `start_page_id`       | integer | 否       | `0`                          | 解析的起始页码。                                             |
| `end_page_id`         | integer | 否       | `99999`                      | 解析的结束页码。                                             |

### 2.2. 响应

- **成功响应**

  - **HTTP 状态码**: `200 OK`

  - **响应体 (JSON 示例)**:

    ```shell
    {
        "message": "Successfully parsed 1 file(s).",
        "output_dir": "/Users/xiaokong/task/2025/paper/backend/uploads/papers"
    }
    ```

- **失败响应**

  - **HTTP 状态码**: `400 Bad Request` / `500 Internal Server Error`
  - **响应体**: 包含错误信息的文本或 JSON。

### 2.3. 调用示例

- **Python 请求**

  ```python
  # 请求单个文件，保存到指定路径
  # （若不写output_dir则默认保存到/data/mineru/parsed_output"）
  import requests
  import time
  import os
  import sys
  
  # --- 配置 ---
  # 服务地址
  SERVER_URL = "http://10.3.35.21:8003/file_parse"
  
  # 1. 要上传的文件路径
  PDF_FILE_PATH = "我们位于/pdfs路径下的用户上传用时间重命名之后的文档"
  # 2. 指定保存路径
  output_dir = "/Users/xiaokong/task/2025/paper/backend/uploads/papers"
  # --- 配置结束 ---
  print(f"--- 启动 /file_parse  ---")
  print(f"上传文件: {PDF_FILE_PATH}")
  # 检查文件是否存在
  if not os.path.exists(PDF_FILE_PATH):
      print(f"错误: 找不到要上传的文件! 路径: {PDF_FILE_PATH}")
      sys.exit(1)
  # 准备要提交的表单数据
  form_data = {
      'backend': 'pipeline',
      'output_dir':output_dir
    # 其他参数可以在这里配置
  }
  try:
      with open(PDF_FILE_PATH, 'rb') as f:
          files_to_upload = {
              'files': (os.path.basename(PDF_FILE_PATH), f, 'application/pdf')
          }
          print("正在发送请求到服务器...")
          # 记录开始时间
          start_time = time.time()
          # 发送 POST 请求
          response = requests.post(SERVER_URL, files=files_to_upload, data=form_data)
          # 记录结束时间
          end_time = time.time()
          total_time = end_time - start_time
      # --- 分析响应 ---
      if response.status_code == 200:
          print(f"状态: 成功 (200 OK)")
      else:
          print(f"状态: 失败 ({response.status_code})")
          print(f"错误信息: {response.text}")
  
  except requests.exceptions.ConnectionError:
      print(f"错误: 连接被拒绝。请确保 fast_api.py 正在 {SERVER_URL} 运行。")
  except Exception as e:
      print(f"发生意外错误: {e}")
  finally:
      print("-" * 30)
      print(f"后端 '/file_parse' 测试结束。")
      if 'total_time' in locals():
          print(f"总运行时长: {total_time:.2f} 秒")
      print("-" * 30)
  
  ```

- **cURL 示例**

  ```shell
  curl -X POST "http://10.3.35.21:8003/file_parse" \
    -F "files=@document1.pdf" \
    -F "output_dir=/custom/output/path" 
  ```

## 3. 接口 2: /file_parse_json - 单文件解析返回JSON

此接口用于上传单个 PDF 文件，服务器解析后不写入文件，而是直接在响应体中返回包含 Markdown 全文等信息的 JSON 对象。

### 3.1. 请求

- **Endpoint (端点)**
  - `POST http://10.3.35.21:8003/file_parse_json`
- **请求参数**
  - `file`: 单个PDF文件（必需）
  - 其他参数（如 `lang`, `backend` 等）与 `/file_parse` 接口类似，但为单数形式（例如 `lang` 而不是 `lang_list`）。

| 参数名 | 类型   | 是否必填 | 默认值 | 描述                |
| ------ | ------ | -------- | ------ | ------------------- |
| `file` | File   | 是       | -      | 单个PDF文件。       |
| `lang` | string | 否       | `ch`   | 语言，例如 `"en"`。 |

### 3.2. 响应 

- **成功响应**

  - **HTTP 状态码**: `200 OK`

  - **响应体 (JSON 示例)**:

    ```shell
    {
      "filename": "document",
      "md_content": "# 文档标题\n\n这是解析后的markdown内容...",
      "middle_json": "{版面信息}",
      "content_list": "[内容列表]",
      "figure_dict": {
          "figure_1": "iVBORw0KGgoAAAANSUhEUgA...（base6编码，可直接解析为图片文件）",
          "figure_2": "R0lGODlhAQABAIAAAAAAAP..."
      },
      "backend": "pipeline",
      "version": "2.5.4"
    }
    ```

### 3.3. 调用示例 

- **Python 请求**

  ```python
  import requests
  import time
  import os
  import sys
  import json
  
  '''{
    "filename": "document",
    "md_content": "# 文档标题\n\n这是解析后的markdown内容...",
    "middle_json": "{...}",
    "content_list": "[...]",
    "figure_dict": {"file_name","base64"},
    "backend": "pipeline",
    "version": "2.5.4"
  }'''
  # --- 配置 ---
  # 服务地址
  SERVER_URL = "http://10.3.35.21:8003/file_parse_json"
  
  # 1. 要上传的本地文件路径
  PDF_FILE_PATH = "/Users/xiaokong/task/2025/paper_vis/vis/arxiv/1ee8ca1c-3478-4c4f-9b77-379ff4d58982/2dbbabd2678ba74fcd9b08aadae975ae.pdf"
  
  # 2. 可以把返回的JSON结果保存在本地的这个文件里，在实际使用中也可以直接当作字典变量返回
  OUTPUT_JSON_PATH = "test_parsejson.json"
  # --- 配置结束 ---
  
  print(f"--- 启动 /file_parse_json 测试 ---")
  print(f"上传文件: {PDF_FILE_PATH}")
  print(f"本地JSON输出路径: {OUTPUT_JSON_PATH}")
  
  # 检查文件是否存在
  if not os.path.exists(PDF_FILE_PATH):
      print(f"错误: 找不到要上传的文件! 路径: {PDF_FILE_PATH}")
      sys.exit(1)
  
  # 准备表单数据 (使用服务器默认值)
  form_data = {
      'backend': 'pipeline'
    # 这里可以写其他参数
  } 
  
  try:
      # 打开文件
      with open(PDF_FILE_PATH, 'rb') as f:
          # 'file' 是 @app.post 定义的字段名 (注意，不是 'files')
          files_to_upload = {
              'file': (os.path.basename(PDF_FILE_PATH), f, 'application/pdf')
          } 
          print("正在发送请求到服务器...")
          # 记录开始时间
          start_time = time.time()
          # 发送 POST 请求
          response = requests.post(SERVER_URL, files=files_to_upload, data=form_data)
          # 记录结束时间
          end_time = time.time()
          total_time = end_time - start_time
      # --- 分析响应 ---
      if response.status_code == 200:
          print(f"状态: 成功 (200 OK)")
          try:
              # 1. 获取JSON结果
              result_json = response.json() # 可以直接取出这个变量使用
              print(f"成功从服务器获取JSON结果。")
              
              # 2. 将结果写入本地文件
              with open(OUTPUT_JSON_PATH, 'w', encoding='utf-8') as f_out:
                  json.dump(result_json, f_out, ensure_ascii=False, indent=4)
              print(f"已将返回的JSON写入到: {OUTPUT_JSON_PATH}")
              
          except Exception as e:
              print(f"解析或写入JSON时出错: {e}")
      else:
          print(f"状态: 失败 ({response.status_code})")
          print(f"错误信息: {response.text}")
  
  except requests.exceptions.ConnectionError:
      print(f"错误: 连接被拒绝。请确保 fast_api.py 正在 {SERVER_URL} 运行。")
  except Exception as e:
      print(f"发生意外错误: {e}")
  
  finally:
      print("-" * 30)
      print(f"后端 '/file_parse_json' 测试结束。")
      if 'total_time' in locals():
          print(f"总运行时长: {total_time:.2f} 秒")
      print("-" * 30)
  
  ```

  **解析出实际图片**：

  ```python
  import base64
  import os
  
  def decode_and_save_images(fig_dict):
      """
      遍历 figure_dict，将干净的 Base64 编码的图片数据解码，并统一保存为 PNG 文件。
  
      文件名取自字典的键 (key)，并添加 .png 后缀。
      
      :param fig_dict: 字典，键是不带后缀的文件名，值是纯 Base64 字符串。
      """
      print("--- 开始解码和保存图片 ---")
  
      # 遍历字典
      for filename_base, base64_data in fig_dict.items():
          # 1. 设置输出文件名，统一使用 .png 后缀
          output_filename = f"{filename_base}.png"
          
          try:
              # 根据用户要求，Base64 数据是“干净”的，直接作为编码数据
              img_encoded_data = base64_data
  
              # 2. 解码 Base64 数据
              img_data = base64.b64decode(img_encoded_data)
  
              # 3. 写入文件
              with open(output_filename, "wb") as f:
                  f.write(img_data)
              
              print(f"成功保存文件: {output_filename}")
  
          except Exception as e:
              # 打印错误信息，继续处理下一个文件
              print(f"错误：保存文件 {output_filename} 时发生错误。Base64 数据可能无效。详细错误: {e}")
  
  if __name__ == "__main__":
      decode_and_save_images(figure_dict)
  
  ```

  

- **cURL 示例**

  ```shell
  curl -X POST "http://10.3.35.21:8003/file_parse_json" \
    -F "file=@document.pdf" 
  ```

## 4. 接口 3: /batch_parse - 批量解析服务器内文件

此接口不上传文件，而是处理服务器指定路径下的所有 PDF 文件，适用于大规模的、预置的解析任务。

### 4.1. 请求 

- **Endpoint (端点)**
  - `POST http://10.3.35.21:8003/batch_parse`
- **请求参数**
  - 请求体为 `application/x-www-form-urlencoded`，包含以下字段：

| 参数名       | 类型   | 是否必填 | 默认值                       | 描述                                             |
| ------------ | ------ | -------- | ---------------------------- | ------------------------------------------------ |
| `input_path` | string | 是       | -                            | **服务器上**的输入路径（可以是单个文件或目录）。 |
| `output_dir` | string | 否       | `/data/mineru/parsed_output` | 解析结果在服务器上的保存目录。                   |

### 4.2. 响应 

- **成功响应**

  - **HTTP 状态码**: `200 OK`

  - **响应体 (JSON 示例)**:

    ```shell
    {
      "message": "Batch processing completed. Processed 2 files.",
      "output_dir": "/data/mineru/parsed_output",
      "results": [
        {
          "filename": "doc1.pdf",
          "status": "success",
          "output_files": ["doc1.md", "doc1_middle.json"]
        },
        {
          "filename": "doc2.pdf",
          "status": "failed",
          "error": "Processing error message"
        }
      ],
      "backend": "pipeline",
      "version": "2.5.4"
    }
    ```

### 4.3. 调用示例

- **Python 请求**

  ```python
  import requests
  import time
  import os
  import sys
  import json
  
  # --- 配置 ---
  # 服务地址
  SERVER_URL = "http://10.3.35.21:8003/batch_parse"
  
  # 1. 你要服务器处理的【服务器本地】的文件夹路径
  INPUT_PATH_ON_SERVER = "/data/kongyb/batch/"
  # --- 配置结束 ---
  output_dir = "/data/kongyb/output/"
  print(f"--- 启动 /batch_parse 测试 ---")
  print(f"请求服务器处理路径: {INPUT_PATH_ON_SERVER}")
  
  # 准备要提交的表单数据
  # 这个接口不上传文件，只发送表单字段
  form_data = {
      'input_path': INPUT_PATH_ON_SERVER,
      'backend': 'pipeline',
      'output_dir':output_dir
  }
  try:
      print("正在发送请求到服务器...")
      # 记录开始时间
      start_time = time.time()
      # 发送 POST 请求
      response = requests.post(SERVER_URL, data=form_data)
      # 记录结束时间
      end_time = time.time()
      total_time = end_time - start_time
  
      # --- 分析响应 ---
      if response.status_code == 200:
          print(f"状态: 成功 (200 OK)")
          print("服务器返回的批量处理报告:")
      else:
          print(f"状态: 失败 ({response.status_code})")
          print(f"错误信息: {response.text}")
  except requests.exceptions.ConnectionError:
      print(f"错误: 连接被拒绝。请确保 fast_api.py 正在 {SERVER_URL} 运行。")
  except Exception as e:
      print(f"发生意外错误: {e}")
  finally:
      print("-" * 30)
      print(f"后端 '/batch_parse' 测试结束。")
      if 'total_time' in locals():
          print(f"总运行时长: {total_time:.2f} 秒")
      print("-" * 30)
  ```

- **cURL 示例**

  ```shell
  curl -X POST "http://10.3.35.21:8003/batch_parse" \
    -F "input_path=/path/to/pdf/directory" \
    -F "output_dir=/custom/output/path" 
  ```

## 5. 注意事项

1. **禁止并发请求**：此服务为重度 I/O 和 CPU/GPU 密集型任务，并发请求可能导致内存占用过高且速度更慢。**<u>请务必按顺序、单个地发送请求。</u>**
2. **长超时设置**：PDF 解析是耗时操作，特别是对于大文件。客户端在调用时，`timeout` 参数应设置为一个较大的值（例如 120秒或更高）。

## 运维记录

- **接口端口**: `8003`

- **配置项地址**: `/etc/systemd/system/mineru-api.service`

- **代码位置**: `/data/wangmeng/anaconda3/envs/vllm/lib/python3.11/site-packages/mineru/cli/fast_api.py`

- **接口源代码**:

  ```python
  import uuid
  import os
  import re
  import tempfile
  import asyncio
  import uvicorn
  import click
  import zipfile
  from pathlib import Path
  import glob
  from fastapi import FastAPI, UploadFile, File, Form
  from fastapi.middleware.gzip import GZipMiddleware
  from fastapi.responses import JSONResponse, FileResponse
  from starlette.background import BackgroundTask
  from typing import List, Optional
  from loguru import logger
  from base64 import b64encode
  
  from mineru.cli.common import aio_do_parse, aio_do_parse_simple, read_fn, pdf_suffixes, image_suffixes
  from mineru.utils.cli_parser import arg_parse
  from mineru.utils.guess_suffix_or_lang import guess_suffix_by_path
  from mineru.version import __version__
  
  app = FastAPI()
  app.add_middleware(GZipMiddleware, minimum_size=1000)
  
  
  def sanitize_filename(filename: str) -> str:
      """
      格式化压缩文件的文件名
      移除路径遍历字符, 保留 Unicode 字母、数字、._- 
      禁止隐藏文件
      """
      sanitized = re.sub(r'[/\\\.]{2,}|[/\\]', '', filename)
      sanitized = re.sub(r'[^\w.-]', '_', sanitized, flags=re.UNICODE)
      if sanitized.startswith('.'):
          sanitized = '_' + sanitized[1:]
      return sanitized or 'unnamed'
  
  def cleanup_file(file_path: str) -> None:
      """清理临时 zip 文件"""
      try:
          if os.path.exists(file_path):
              os.remove(file_path)
      except Exception as e:
          logger.warning(f"fail clean file {file_path}: {e}")
  
  def encode_image(image_path: str) -> str:
      """Encode image using base64"""
      with open(image_path, "rb") as f:
          return b64encode(f.read()).decode()
  
  def get_infer_result(file_suffix_identifier: str, pdf_name: str, parse_dir: str) -> Optional[str]:
      """从结果文件中读取推理结果"""
      result_file_path = os.path.join(parse_dir, f"{pdf_name}{file_suffix_identifier}")
      if os.path.exists(result_file_path):
          with open(result_file_path, "r", encoding="utf-8") as fp:
              return fp.read()
      return None
  
  
  @app.post(path="/file_parse",)
  async def parse_pdf(
          files: List[UploadFile] = File(...),
          output_dir: Optional[str] = Form(None),
          lang_list: List[str] = Form(["ch"]),
          backend: str = Form("pipeline"),
          parse_method: str = Form("auto"),
          formula_enable: bool = Form(True),
          table_enable: bool = Form(True),
          server_url: Optional[str] = Form(None),
          return_md: bool = Form(True),
          return_middle_json: bool = Form(True),
          return_model_output: bool = Form(False),
          return_content_list: bool = Form(True),
          return_images: bool = Form(False),
          response_format_zip: bool = Form(False),
          start_page_id: int = Form(0),
          end_page_id: int = Form(99999),
  ):
  
      # 获取命令行配置参数
      config = getattr(app.state, "config", {})
  
      try:
          # 设置默认输出目录
          if output_dir is None:
              output_dir = "/data/mineru/parsed_output"
          
          # 创建唯一的输出目录
          unique_dir = os.path.join(output_dir, str(uuid.uuid4()))
          os.makedirs(unique_dir, exist_ok=True)
  
          # 处理上传的PDF文件
          pdf_file_names = []
          pdf_bytes_list = []
  
          for file in files:
              content = await file.read()
              file_path = Path(file.filename)
  
              # 创建临时文件
              temp_path = Path(unique_dir) / file_path.name
              with open(temp_path, "wb") as f:
                  f.write(content)
  
              # 如果是图像文件或PDF，使用read_fn处理
              file_suffix = guess_suffix_by_path(temp_path)
              if file_suffix in pdf_suffixes + image_suffixes:
                  try:
                      pdf_bytes = read_fn(temp_path)
                      pdf_bytes_list.append(pdf_bytes)
                      pdf_file_names.append(file_path.stem)
                      os.remove(temp_path)  # 删除临时文件
                  except Exception as e:
                      return JSONResponse(
                          status_code=400,
                          content={"error": f"Failed to load file: {str(e)}"}
                      )
              else:
                  return JSONResponse(
                      status_code=400,
                      content={"error": f"Unsupported file type: {file_suffix}"}
                  )
  
  
          # 设置语言列表，确保与文件数量一致
          actual_lang_list = lang_list
          if len(actual_lang_list) != len(pdf_file_names):
              # 如果语言列表长度不匹配，使用第一个语言或默认"ch"
              actual_lang_list = [actual_lang_list[0] if actual_lang_list else "ch"] * len(pdf_file_names)
  
          # 调用简化的异步处理函数
          await aio_do_parse_simple(
              output_dir=unique_dir,
              pdf_file_names=pdf_file_names,
              pdf_bytes_list=pdf_bytes_list,
              p_lang_list=actual_lang_list,
              backend=backend,
              parse_method=parse_method,
              formula_enable=formula_enable,
              table_enable=table_enable,
              server_url=server_url,
              f_draw_layout_bbox=False,
              f_draw_span_bbox=False,
              f_dump_md=return_md,
              f_dump_middle_json=return_middle_json,
              f_dump_model_output=return_model_output,
              f_dump_orig_pdf=False,
              f_dump_content_list=return_content_list,
              start_page_id=start_page_id,
              end_page_id=end_page_id,
              **config
          )
  
          # 根据 response_format_zip 决定返回类型
          if response_format_zip:
              zip_fd, zip_path = tempfile.mkstemp(suffix=".zip", prefix="mineru_results_")
              os.close(zip_fd) 
              with zipfile.ZipFile(zip_path, "w", compression=zipfile.ZIP_DEFLATED) as zf:
                  for pdf_name in pdf_file_names:
                      safe_pdf_name = sanitize_filename(pdf_name)
                      # 直接在unique_dir下查找文件，不再使用复杂的子目录结构
                      parse_dir = unique_dir
  
                      if not os.path.exists(parse_dir):
                          continue
  
                      # 写入文本类结果
                      if return_md:
                          path = os.path.join(parse_dir, f"{pdf_name}.md")
                          if os.path.exists(path):
                              zf.write(path, arcname=os.path.join(safe_pdf_name, f"{safe_pdf_name}.md"))
  
                      if return_middle_json:
                          path = os.path.join(parse_dir, f"{pdf_name}_middle.json")
                          if os.path.exists(path):
                              zf.write(path, arcname=os.path.join(safe_pdf_name, f"{safe_pdf_name}_middle.json"))
  
                      if return_model_output:
                          if backend.startswith("pipeline"):
                              path = os.path.join(parse_dir, f"{pdf_name}_model.json")
                          else:
                              path = os.path.join(parse_dir, f"{pdf_name}_model_output.txt")
                          if os.path.exists(path): 
                              zf.write(path, arcname=os.path.join(safe_pdf_name, os.path.basename(path)))
  
                      if return_content_list:
                          path = os.path.join(parse_dir, f"{pdf_name}_content_list.json")
                          if os.path.exists(path):
                              zf.write(path, arcname=os.path.join(safe_pdf_name, f"{safe_pdf_name}_content_list.json"))
  
                      # 写入图片
                      if return_images:
                          images_dir = os.path.join(parse_dir, "images")
                          image_paths = glob.glob(os.path.join(glob.escape(images_dir), "*.jpg"))
                          for image_path in image_paths:
                              zf.write(image_path, arcname=os.path.join(safe_pdf_name, "images", os.path.basename(image_path)))
  
              return FileResponse(
                  path=zip_path,
                  media_type="application/zip",
                  filename="results.zip",
                  background=BackgroundTask(cleanup_file, zip_path)
              )
          else:
              # 构建 JSON 结果
              result_dict = {}
              for pdf_name in pdf_file_names:
                  result_dict[pdf_name] = {}
                  data = result_dict[pdf_name]
  
                  # 直接在unique_dir下查找文件，不再使用复杂的子目录结构
                  parse_dir = unique_dir
  
                  if os.path.exists(parse_dir):
                      if return_md:
                          data["md_content"] = get_infer_result(".md", pdf_name, parse_dir)
                      if return_middle_json:
                          data["middle_json"] = get_infer_result("_middle.json", pdf_name, parse_dir)
                      if return_model_output:
                          if backend.startswith("pipeline"):
                              data["model_output"] = get_infer_result("_model.json", pdf_name, parse_dir)
                          else:
                              data["model_output"] = get_infer_result("_model_output.txt", pdf_name, parse_dir)
                      if return_content_list:
                          data["content_list"] = get_infer_result("_content_list.json", pdf_name, parse_dir)
                      if return_images:
                          images_dir = os.path.join(parse_dir, "images")
                          safe_pattern = os.path.join(glob.escape(images_dir), "*.jpg")
                          image_paths = glob.glob(safe_pattern)
                          data["images"] = {
                              os.path.basename(
                                  image_path
                              ): f"data:image/jpeg;base64,{encode_image(image_path)}"
                              for image_path in image_paths
                          }
  
              return JSONResponse(
                  status_code=200,
                  content={
                      "backend": backend,
                      "version": __version__,
                      "results": result_dict,
                      "output_path": unique_dir
                  }
              )
      except Exception as e:
          logger.exception(e)
          return JSONResponse(
              status_code=500,
              content={"error": f"Failed to process file: {str(e)}"}
          )
  
  
  @app.post(path="/file_parse_json",)
  async def parse_pdf_to_json(
          file: UploadFile = File(...),
          lang: str = Form("ch"),
          backend: str = Form("pipeline"),
          parse_method: str = Form("auto"),
          formula_enable: bool = Form(True),
          table_enable: bool = Form(True),
          server_url: Optional[str] = Form(None),
          start_page_id: int = Form(0),
          end_page_id: int = Form(99999),
  ):
      """
      上传单个PDF并解析，返回markdown全文，不写入文件
      """
      # 获取命令行配置参数
      config = getattr(app.state, "config", {})
  
      try:
          # 创建临时输出目录
          temp_output_dir = os.path.join(tempfile.gettempdir(), f"mineru_temp_{uuid.uuid4()}")
          os.makedirs(temp_output_dir, exist_ok=True)
  
          try:
              # 处理上传的PDF文件
              content = await file.read()
              file_path = Path(file.filename)
  
              # 创建临时文件
              temp_path = Path(temp_output_dir) / file_path.name
              with open(temp_path, "wb") as f:
                  f.write(content)
  
              # 检查文件类型
              file_suffix = guess_suffix_by_path(temp_path)
              if file_suffix not in pdf_suffixes + image_suffixes:
                  return JSONResponse(
                      status_code=400,
                      content={"error": f"Unsupported file type: {file_suffix}"}
                  )
  
              pdf_bytes = read_fn(temp_path)
              pdf_file_name = file_path.stem
              os.remove(temp_path)  # 删除临时文件
  
              # 调用简化的异步处理函数
              await aio_do_parse_simple(
                  output_dir=temp_output_dir,
                  pdf_file_names=[pdf_file_name],
                  pdf_bytes_list=[pdf_bytes],
                  p_lang_list=[lang],
                  backend=backend,
                  parse_method=parse_method,
                  formula_enable=formula_enable,
                  table_enable=table_enable,
                  server_url=server_url,
                  f_draw_layout_bbox=False,
                  f_draw_span_bbox=False,
                  f_dump_md=True,
                  f_dump_middle_json=True,
                  f_dump_model_output=False,
                  f_dump_orig_pdf=False,
                  f_dump_content_list=True,
                  f_dump_images=True,
                  start_page_id=start_page_id,
                  end_page_id=end_page_id,
                  **config
              )
  
              # 读取生成的markdown内容
              md_content = get_infer_result(".md", pdf_file_name, temp_output_dir)
              middle_json = get_infer_result("_middle.json", pdf_file_name, temp_output_dir)
              content_list = get_infer_result("_content_list.json", pdf_file_name, temp_output_dir)
              
              # 处理图片 - 过滤公式图片，返回字典格式
              figure_dict = {}
              images_dir = os.path.join(temp_output_dir, "images")
              if os.path.exists(images_dir):
                  # 收集常见图片格式，避免仅限于 .jpg 导致遗漏
                  image_paths = []
                  for ext in ("*.jpg", "*.jpeg", "*.png"):
                      safe_pattern = os.path.join(glob.escape(images_dir), ext)
                      image_paths.extend(glob.glob(safe_pattern))
                  for image_path in image_paths:
                      # 使用保存时的文件名（不含扩展名）作为图片标识；底层已处理过滤
                      image_id = os.path.splitext(os.path.basename(image_path))[0]
                      # 添加到字典，使用图片标识作为key，返回纯base64
                      figure_dict[image_id] = encode_image(image_path)
  
              return JSONResponse(
                  status_code=200,
                  content={
                      "filename": pdf_file_name,
                      "md_content": md_content,
                      "middle_json": middle_json,
                      "content_list": content_list,
                      "figure_dict": figure_dict,
                      "backend": backend,
                      "version": __version__
                  }
              )
  
          finally:
              # 清理临时目录
              import shutil
              if os.path.exists(temp_output_dir):
                  shutil.rmtree(temp_output_dir, ignore_errors=True)
  
      except Exception as e:
          logger.exception(e)
          return JSONResponse(
              status_code=500,
              content={"error": f"Failed to process file: {str(e)}"}
          )
  
  
  @app.post(path="/batch_parse",)
  async def batch_parse_pdfs(
          input_path: str = Form(...),
          output_dir: Optional[str] = Form(None),
          lang: str = Form("ch"),
          backend: str = Form("pipeline"),
          parse_method: str = Form("auto"),
          formula_enable: bool = Form(True),
          table_enable: bool = Form(True),
          server_url: Optional[str] = Form(None),
          start_page_id: int = Form(0),
          end_page_id: int = Form(99999),
  ):
      """
      解析指定路径下的所有PDF，按顺序处理
      """
      # 获取命令行配置参数
      config = getattr(app.state, "config", {})
  
      try:
          # 设置默认输出目录
          if output_dir is None:
              output_dir = "/data/mineru/parsed_output"
          
          # 创建输出目录
          os.makedirs(output_dir, exist_ok=True)
  
          # 检查输入路径
          input_path = Path(input_path)
          if not input_path.exists():
              return JSONResponse(
                  status_code=400,
                  content={"error": f"Input path does not exist: {input_path}"}
              )
  
          # 查找所有PDF文件
          pdf_files = []
          if input_path.is_file():
              # 单个文件
              if guess_suffix_by_path(input_path) in pdf_suffixes:
                  pdf_files = [input_path]
          else:
              # 目录，查找所有PDF文件
              for suffix in pdf_suffixes:
                  pdf_files.extend(input_path.glob(f"**/*{suffix}"))
  
          if not pdf_files:
              return JSONResponse(
                  status_code=400,
                  content={"error": "No PDF files found in the specified path"}
              )
  
          # 按文件名排序
          pdf_files.sort()
  
          # 处理每个PDF文件
          results = []
          for pdf_file in pdf_files:
              try:
                  logger.info(f"Processing: {pdf_file}")
                  
                  # 为每个文件创建独立的UUID文件夹
                  file_uuid = str(uuid.uuid4())
                  file_output_dir = os.path.join(output_dir, file_uuid)
                  os.makedirs(file_output_dir, exist_ok=True)
                  
                  # 读取PDF文件
                  pdf_bytes = read_fn(pdf_file)
                  pdf_file_name = pdf_file.stem
  
                  # 调用简化的异步处理函数
                  await aio_do_parse_simple(
                      output_dir=file_output_dir,
                      pdf_file_names=[pdf_file_name],
                      pdf_bytes_list=[pdf_bytes],
                      p_lang_list=[lang],
                      backend=backend,
                      parse_method=parse_method,
                      formula_enable=formula_enable,
                      table_enable=table_enable,
                      server_url=server_url,
                      f_draw_layout_bbox=False,
                      f_draw_span_bbox=False,
                      f_dump_md=True,
                      f_dump_middle_json=True,
                      f_dump_model_output=False,
                      f_dump_orig_pdf=False,
                      f_dump_content_list=True,
                      start_page_id=start_page_id,
                      end_page_id=end_page_id,
                      **config
                  )
  
                  results.append({
                      "filename": pdf_file_name,
                      "uuid": file_uuid,
                      "output_dir": file_output_dir,
                      "status": "success",
                      "output_files": [
                          f"{pdf_file_name}.md",
                          f"{pdf_file_name}_middle.json",
                          f"{pdf_file_name}_content_list.json"
                      ]
                  })
  
              except Exception as e:
                  logger.error(f"Failed to process {pdf_file}: {str(e)}")
                  results.append({
                      "filename": pdf_file.name,
                      "uuid": file_uuid if 'file_uuid' in locals() else None,
                      "output_dir": file_output_dir if 'file_output_dir' in locals() else None,
                      "status": "failed",
                      "error": str(e)
                  })
  
          return JSONResponse(
              status_code=200,
              content={
                  "message": f"Batch processing completed. Processed {len(pdf_files)} files.",
                  "output_dir": output_dir,
                  "results": results,
                  "backend": backend,
                  "version": __version__
              }
          )
  
      except Exception as e:
          logger.exception(e)
          return JSONResponse(
              status_code=500,
              content={"error": f"Failed to process batch: {str(e)}"}
          )
  
  
  @click.command(context_settings=dict(ignore_unknown_options=True, allow_extra_args=True))
  @click.pass_context
  @click.option('--host', default='127.0.0.1', help='Server host (default: 127.0.0.1)')
  @click.option('--port', default=8003, type=int, help='Server port (default: 8000)')
  @click.option('--reload', is_flag=True, help='Enable auto-reload (development mode)')
  def main(ctx, host, port, reload, **kwargs):
  
      kwargs.update(arg_parse(ctx))
  
      # 将配置参数存储到应用状态中
      app.state.config = kwargs
  
      """启动MinerU FastAPI服务器的命令行入口"""
      print(f"Start MinerU FastAPI Service: http://{host}:{port}")
      print("The API documentation can be accessed at the following address:")
      print(f"- Swagger UI: http://{host}:{port}/docs")
      print(f"- ReDoc: http://{host}:{port}/redoc")
  
      uvicorn.run(
          "mineru.cli.fast_api:app",
          host=host,
          port=port,
          reload=reload
      )
  
  
  if __name__ == "__main__":
      main()
  
  ```

# 四、 学术论文智能分析接口

## 1. 概览 

- **API 名称**: 学术论文智能分析API
- **接口状态**: `[稳定]`
- **接口描述**: 提供基于 MainScheduler 的端到端学术论文处理系统。它集成了PDF解析、元数据提取、摘要语步分析、论文脉络分析以及图表映射等核心功能，旨在为用户提供全面、结构化的论文洞察。

### 功能特性

- **PDF解析**: 自动解析PDF文件，提取文本、图表和元数据（标题、作者等）。
- **脉络分析**: 按照学术论文结构提取五大核心脉络内容。
- **摘要提取**: 智能提取摘要中的四个关键语步（背景、方法、创新点、局限）。
- **图表映射**: 将图表按所属脉络分类并建立映射关系。
- **并行处理**: 多线程并行执行，提高处理效率。
- **健康检查**: 内置健康状态检查接口，方便监控。

## 2. 核心接口: /paper_vis

此接口是系统的主要入口，接收一个PDF文件，执行完整的分析流程，并返回结构化的JSON结果。**<u>（可直接读取客户端的本地文件）</u>**

### 2.1. 请求

- **Endpoint (端点)**
  - `POST http://10.3.35.21:8004/paper_vis`
- **请求参数**
  - 请求体为 `multipart/form-data`，包含以下字段：

| 参数名 | 类型 | 是否必填 | 描述                    |
| ------ | ---- | -------- | ----------------------- |
| `file` | File | 是       | 用户上传的单个PDF文件。 |

- **备用请求方式**:
  - 除了上传文件，接口也支持通过JSON指定**服务器本地**的文件路径进行分析。
  - **Endpoint**: `POST http://10.3.35.21:8004/paper_vis`
  - **Headers**: `Content-Type: application/json`
  - **Body**: `{"pdf_path": "/path/on/server/to/paper.pdf"}`

### 2.2. 响应 

- **成功响应 **

  - **HTTP 状态码**: `200 OK`

  - **响应体 (JSON 示例)**:

    ```shell
    {
      "success": true,
      "metadata": {
        "title": "HYPOSPACE: EVALUATING LLM CREATIVITY AS SET-VALUED HYPOTHESIS GENERATORS UNDER UNDERDETERMINATION",
        "authors": ["作者1", "作者2", "作者3"]
      },
      "abstract": {
        "background_problem": "背景和问题描述",
        "method_approach": "方法和途径",
        "Innovation": "创新点",
        "Limitation/Future Work": "局限和未来工作"
      },
      "lanes": {
        "Context & Related Work": "相关工作和背景内容",
        "Methodology & Setup": "方法论和设置内容",
        "Results & Analysis": "结果和分析内容",
        "Conclusion": "结论内容",
        "Innovation Discovery": "学术创新机会发现"
      },
      "figure_map": {
        "Context & Related Work": [
          {
            "figure_id": "fig1",
            "caption": "图表标题",
            "content": "相关文本内容"
          }
        ],
        "Results & Analysis": [
          {
            "figure_id": "fig2",
            "caption": "结果图表",
            "content": "结果相关文本"
          }
        ]
      },
      "pdf_info": {
        "file_path": "/path/to/paper.pdf",
        "file_size": 1234567,
        "page_count": 10
      },
      "processing_info": {
        "steps_completed": ["pdf_parsing", "abstract_steps", "lane_extraction", "figure_mapping", "final_json_generation"],
        "total_time": 22.54,
        "success": true
      },
      "raw_data": {
        "md_content_length": 50000,
        "content_list_count": 150,
        "figure_dict_count": 8
      }
    
    ```

## 3. 辅助接口

用于监控服务的健康和运行状态。

### 3.1. GET /health

- **功能**: 检查服务是否健康，能否响应请求。

- **Endpoint**: `GET http://10.3.35.21:8004/health`

- **响应**:

  ```shell
  {
    "status": "healthy",
    "timestamp": "2025-01-27T10:30:00Z"
  }
  ```

### 3.2. GET /status

- **功能**: 获取服务的详细状态信息，如版本号、运行时长。

- **Endpoint**: `GET http://10.3.35.21:8004/status`

- **响应**:

  ```shell
  {
    "status": "running",
    "uptime": 3600,
    "version": "1.0.0"
  }
  ```

## 4. 调用示例

### 4.1. Python 请求

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
API测试客户端
测试文件上传接口 /paper_vis
"""

import requests
import json
import time
import os


def test_paper_vis_upload():
    """测试 /paper_vis 文件上传接口"""
    
    # API服务器地址
    base_url = "http://10.3.35.21:8004"
    
    # 测试PDF文件路径
    test_pdf_path = "/Users/xiaokong/Desktop/2510.15614v1.pdf"
    
    print("🧪 测试 /paper_vis 文件上传接口")
    print("=" * 60)
    
    # 检查文件是否存在
    if not os.path.exists(test_pdf_path):
        print(f"❌ 测试文件不存在: {test_pdf_path}")
        return
    
    print(f"📄 上传文件: {test_pdf_path}")
    print("⏳ 开始分析...")
    
    start_time = time.time()
    
    try:
        # 准备文件上传
        with open(test_pdf_path, 'rb') as f:
            files = {
                'file': (os.path.basename(test_pdf_path), f, 'application/pdf')
            }
            
            response = requests.post(
                f"{base_url}/paper_vis",
                files=files,
                timeout=300  # 5分钟超时
            )
        
        end_time = time.time()
        duration = end_time - start_time
        
        if response.status_code == 200:
            result = response.json()
            print("✅ 论文分析成功")
            print(f"⏱️ 总耗时: {duration:.2f}秒")
            print(f"📊 处理结果:")
            print(f"   - 成功状态: {result.get('success', False)}")
            print(f"   - 处理时间: {result.get('total_time', 0):.2f}秒")
            
            if result.get('success'):
                print(f"   - 论文标题: {result.get('metadata', {}).get('title', 'N/A')}")
                authors = result.get('metadata', {}).get('authors', [])
                print(f"   - 作者数量: {len(authors) if authors else 0}")
                lanes = result.get('lanes', {})
                print(f"   - 泳道数量: {len(lanes) if lanes else 0}")
                figure_map = result.get('figure_map', {})
                total_figures = sum(len(figures) for figures in figure_map.values()) if figure_map else 0
                print(f"   - 图表总数: {total_figures}")
                
                # 保存结果到文件
                output_file = "paper_vis_upload_result.json"
                with open(output_file, 'w', encoding='utf-8') as f:
                    json.dump(result, f, ensure_ascii=False, indent=2)
                print(f"💾 结果已保存到: {output_file}")
            else:
                print(f"❌ 分析失败: {result.get('error', '未知错误')}")
        else:
            print(f"❌ 论文分析失败: {response.status_code}")
            print(f"   错误信息: {response.text}")
            
    except requests.exceptions.Timeout:
        print("❌ 请求超时（5分钟）")
    except Exception as e:
        print(f"❌ 论文分析异常: {e}")
    
    print("\n" + "=" * 60)
    print("🏁 测试完成")


def test_server_health():
    """测试服务器健康状态"""
    base_url = "http://10.3.35.21:8004"
    
    try:
        # 直接测试 /paper_vis 接口的OPTIONS请求
        response = requests.options(f"{base_url}/paper_vis", timeout=10)
        if response.status_code in [200, 405]:  # 405表示方法不允许，但接口存在
            print("✅ 服务器运行正常")
            return True
        else:
            print(f"❌ 服务器响应异常: {response.status_code}")
            return False
    except Exception as e:
        print(f"❌ 无法连接到服务器: {e}")
        return False


if __name__ == "__main__":
    print("🚀 开始API测试")
    print("=" * 60)
    
    # 首先测试服务器健康状态
    if test_server_health():
        print("\n" + "=" * 60)
        # 测试文件上传接口
        test_paper_vis_upload()
    else:
        print("❌ 服务器不可用，请先启动API服务器")
        print("   运行命令: python api_server.py")
```

### 4.2. cURL 示例

```shell
# 方式一：上传本地文件进行分析
curl -X POST "[http://10.3.35.21:8004/paper_vis](http://10.3.35.21:8004/paper_vis)" \
  -F "file=@/path/to/your/paper.pdf"

# 方式二：分析服务器上的文件
curl -X POST "[http://10.3.35.21:8004/paper_vis](http://10.3.35.21:8004/paper_vis)" \
  -H "Content-Type: application/json" \
  -d '{"pdf_path": "/path/on/server/to/paper.pdf"}'

# 检查健康状态
curl -X GET "[http://10.3.35.21:8004/health](http://10.3.35.21:8004/health)"
```

### 4.3. JavaScript (Fetch API) 示例

```javascript
/**
 * 论文分析API客户端
 */

// API基础URL - 使用代理路径避免CORS问题
const API_BASE_URL = process.env.NODE_ENV === 'development' ? '/api' : 'http://10.3.35.21:8004'

/**
 * 上传PDF文件并进行分析
 * @param {File} pdfFile - PDF文件对象
 * @param {Function} onProgress - 进度回调函数
 * @returns {Promise<Object>} 分析结果
 */
export async function analyzePaper(pdfFile, onProgress = null) {
  try {
    console.log('🚀 开始论文分析...')
    console.log('📄 文件信息:', {
      name: pdfFile.name,
      size: pdfFile.size,
      type: pdfFile.type
    })

    // 验证文件类型
    if (pdfFile.type !== 'application/pdf') {
      throw new Error('请选择PDF文件')
    }

    // 创建FormData
    const formData = new FormData()
    formData.append('file', pdfFile)

    // 显示进度
    if (onProgress) {
      onProgress(10, '准备上传文件...')
    }

    const startTime = Date.now()

    // 发送请求到 /paper_vis 接口
    const response = await fetch(`${API_BASE_URL}/paper_vis`, {
      method: 'POST',
      body: formData,
      mode: 'cors', // 明确指定CORS模式
      headers: {
        // 不设置Content-Type，让浏览器自动设置multipart/form-data
      },
      // 注意：fetch不支持进度回调，这里我们模拟进度
    })

    const endTime = Date.now()
    const duration = (endTime - startTime) / 1000

    if (!response.ok) {
      throw new Error(`服务器错误: ${response.status} ${response.statusText}`)
    }

    const result = await response.json()

    console.log('✅ 论文分析完成')
    console.log('⏱️ 总耗时:', duration.toFixed(2), '秒')
    console.log('📊 处理结果:', result)

    // 验证返回结果
    if (!result.success) {
      throw new Error(result.error || '分析失败')
    }

    // 显示最终进度
    if (onProgress) {
      onProgress(100, '分析完成！')
    }

    return {
      success: true,
      data: result,
      duration: duration,
      metadata: {
        title: result.metadata?.title || '未知标题',
        authors: result.metadata?.authors || [],
        totalTime: result.total_time || duration,
        lanesCount: Object.keys(result.lanes || {}).length,
        figuresCount: Object.values(result.figure_map || {}).reduce((total, figures) => total + figures.length, 0)
      }
    }

  } catch (error) {
    console.error('❌ 论文分析失败:', error)
    
    // 检查是否是CORS错误
    let errorMessage = error.message || '未知错误'
    if (error.message.includes('CORS') || error.message.includes('Failed to fetch')) {
      errorMessage = '跨域请求被阻止，请检查后端服务器CORS配置或使用代理'
    }
    
    // 显示错误进度
    if (onProgress) {
      onProgress(0, `分析失败: ${errorMessage}`)
    }

    return {
      success: false,
      error: errorMessage,
      data: null
    }
  }
}

/**
 * 检查服务器健康状态
 * @returns {Promise<boolean>} 服务器是否可用
 */
export async function checkServerHealth() {
  try {
    const response = await fetch(`${API_BASE_URL}/paper_vis`, {
      method: 'OPTIONS',
      timeout: 10000
    })
    
    // 200或405都表示接口存在
    return response.status === 200 || response.status === 405
  } catch (error) {
    console.warn('服务器健康检查失败:', error)
    return false
  }
}

/**
 * 模拟进度更新（因为fetch不支持进度回调）
 * @param {Function} onProgress - 进度回调函数
 * @param {number} duration - 预计总时长（秒）
 */
export function simulateProgress(onProgress) {
  let progress = 0
  const interval = setInterval(() => {
    progress += Math.random() * 10
    if (progress >= 90) {
      progress = 90
      clearInterval(interval)
    }
    
    const statuses = [
      '正在上传文件...',
      'PDF解析中...',
      '内容抽取中...',
      '语义分析中...',
      '生成结果中...'
    ]
    
    const statusIndex = Math.floor((progress / 100) * statuses.length)
    const status = statuses[Math.min(statusIndex, statuses.length - 1)]
    
    onProgress(Math.floor(progress), status)
  }, 1000)

  return interval
}

```

## 5. 注意事项

1. **文件路径**: 使用JSON请求时，请确保 `pdf_path` 是服务器可访问的绝对路径。
2. **处理时间**: 大型或复杂的PDF论文可能需要较长的处理时间，客户端应设置足够长的超时时间（建议2分钟以上）。
3. **文件格式**: 目前仅支持标准的PDF格式文件。

## 运维笔记

- **接口端口**: `8004`

- **配置项地址**: `/etc/systemd/system/paper-vis-api.service`

- **代码位置**: `/data/kongyb/paper_vis/paper_vis/api_server.py`

- **接口源代码**:

  ```python
  #!/usr/bin/env python3
  # -*- coding: utf-8 -*-
  """
  FastAPI 服务器 - 学术论文智能分析接口
  基于 MainScheduler 的完整论文处理系统
  
  唯一接口：
  POST /paper_vis
  - 输入：上传PDF文件
  - 输出：MainScheduler的完整JSON结果
  """
  from fastapi.middleware.cors import CORSMiddleware
  from fastapi import FastAPI, HTTPException, UploadFile, File
  import uvicorn
  
  # 导入主调度器
  from MainScheduler import MainScheduler
  
  
  # 创建FastAPI应用
  app = FastAPI(
      title="学术论文智能分析API",
      description="基于AI的学术论文智能分析系统",
      version="1.0.0"
  )
  
  origins = ["*"]
  
  # 将 CORS 中间件添加到你的应用中
  app.add_middleware(
      CORSMiddleware,
      allow_origins=origins,  # 使用新的 origins 列表
      allow_credentials=True,
      allow_methods=["*"],
      allow_headers=["*"],
  )
  
  @app.post("/paper_vis")
  async def paper_vis(file: UploadFile = File(...)):
      """
      分析PDF论文 - 唯一接口
      
      输入：
      - file: 上传的PDF文件
      
      输出：
      - MainScheduler的完整JSON结果
      """
      try:
          print(f"🚀 开始处理PDF文件: {file.filename}")
          
          # 检查文件类型
          if not file.filename.lower().endswith('.pdf'):
              raise HTTPException(
                  status_code=400, 
                  detail="只支持PDF文件格式"
              )
          
          # 读取上传的文件内容
          file_content = await file.read()
          
          # 创建主调度器实例
          scheduler = MainScheduler()
          
          # 执行完整的论文分析流程（直接使用文件内容，不保存到服务器）
          result = scheduler.process_uploaded_pdf(file_content, file.filename)
          
          return result
              
      except Exception as e:
          print(f"❌ 服务器内部错误: {e}")
          raise HTTPException(
              status_code=500,
              detail=f"服务器内部错误: {str(e)}"
          )
  
  
  if __name__ == "__main__":
      print("🚀 启动学术论文智能分析API服务器...")
      print("📡 服务地址: http://10.3.35.21:8004")
      print("📊 唯一接口: POST /paper_vis")
      print("=" * 60)
      
      # 启动服务器
      uvicorn.run(
          "api_server:app",
          host="0.0.0.0",
          port=8004,
          reload=True,
          log_level="info"
      )
  ```

  # 五、 学术论文智能分析服务平台 

  ## 1. 概览

  - **平台名称**: 学术论文智能分析服务平台
  - **平台状态**: `[稳定]`
  - **平台描述**: 本平台是“学术论文智能分析API”的可视化前端界面，为用户提供了一个便捷的图形化操作窗口，可以直接上传PDF文件，并直观地查看分析结果。

  ![50a3614bcb6612ad37ba3bce27bfd143](/Users/xiaokong/Library/Containers/com.tencent.xinWeChat/Data/Documents/xwechat_files/wxid_e0erez0j9yre22_7560/temp/RWTemp/2025-10/50a3614bcb6612ad37ba3bce27bfd143.png)

  ![7f5bea41e11b370867ebba38be801295](/Users/xiaokong/Library/Containers/com.tencent.xinWeChat/Data/Documents/xwechat_files/wxid_e0erez0j9yre22_7560/temp/RWTemp/2025-10/7f5bea41e11b370867ebba38be801295.png)

  ![338bb8e07c311fcc83a19cebbf1fbd44](/Users/xiaokong/Library/Containers/com.tencent.xinWeChat/Data/Documents/xwechat_files/wxid_e0erez0j9yre22_7560/temp/RWTemp/2025-10/338bb8e07c311fcc83a19cebbf1fbd44.png)

  ## 2. 访问地址

  - **平台访问链接**:
    - `http://10.3.35.21:8080/upload`

  ## 3. 注意事项

  1. **浏览器兼容性**: 建议使用最新版本的 Chrome, Firefox, 或 Edge 浏览器以获得最佳体验。
  2. **文件上传**: 用户通过此平台上传的PDF文件，将由后端“学术论文智能分析API” (`http://10.3.35.21:8004`) 进行处理。
  3. **网络环境**: 请确保客户端能够访问服务器 `10.3.35.21` 的 `8080` 端口。

  ## 运维记录

  - **服务端口**: `8080`

  - **Web服务器配置**: `/etc/nginx/sites-enabled/paper-vis-frontend`

  - **项目代码位置**: `/data/kongyb/paper_vis/frontend/`

    

    