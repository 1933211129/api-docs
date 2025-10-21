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

