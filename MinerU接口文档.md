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

