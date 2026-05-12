---
title: Add Memory
desc: The core memory ingestion interface of MemOS. Through the MemCube isolation mechanism, it supports asynchronous memory ingestion for personal memories, knowledge bases, and multi-tenant scenarios.
---

**Endpoint Path**：`POST /product/add`
**Function Description**：This is the core entry point for storing unstructured data in the system. It supports converting raw data from conversation lists, plain text, or metadata into structured memory fragments. In the open-source version, the system uses **MemCube** to provide memory isolation and dynamic organization.

## 1. Core Mechanism: MemCube and Isolation

In the open-source architecture, understanding MemCube is important for using this API effectively:

* **Isolation Unit**: MemCube is the atomic unit for memory storage. Cubes are completely independent from each other; deduplication and conflict resolution are performed only within a single Cube.
* **Flexible Mapping**:
* **Personal Mode**: Pass `user_id` as `writable_cube_ids`, which creates a private personal memory space.
* **Knowledge Base Mode**: Pass the unique identifier (QID) of the knowledge base as `writable_cube_ids`, to store the content in that knowledge base.
* **Multi-target Writing**: The interface supports writing memories to multiple Cubes simultaneously, supporting cross-domain synchronization.


## 2. Key Interface Parameters

Core parameter definitions are as follows:

| Parameter Name | Type | Required | Default Value | Description |
| :--- | :--- | :--- | :--- | :--- |
| **`user_id`** | `str` | Yes | - | User unique identifier, used for permission verification. |
| **`messages`** | `list/str`| Yes | - | Conversation messages or plain text content to store. |
| **`writable_cube_ids`** | `list[str]`| Yes | - | **Core parameter**: Specifies the list of target Cube IDs to write to. |
| **`async_mode`** | `str` | No | `async` | Processing mode: `async` (background queue processing) or `sync` (blocking the current request). |
| **`is_feedback`** | `bool` | No | `false` | If `true`, the system will automatically route to the feedback processor to perform memory correction. |
| `session_id` | `str` | No | `default` | Session identifier, used to track conversation context. |
| `custom_tags` | `list[str]`| No | - | Custom tags that can be used as filters in later searches. |
| `info` | `dict` | No | - | Extended metadata. All key-value pairs can be used for later filtering. |
| `mode` | `str` | No | - | Only applies when `async_mode='sync'`. Available options are `fast` or `fine`. |

## 3. Working Principle (Component & Handler)

When a request reaches the backend, the system's **AddHandler** dispatches core components to perform the following operations:

1. **Multimodal Parsing**: The `MemReader` component converts `messages` into internal memory objects.
2. **Feedback Routing**: If `is_feedback=True`, the Handler extracts the end of the conversation as feedback, directly corrects existing memory, without generating new memory entries.
3. **Asynchronous Dispatch**: If in `async` mode, `MemScheduler` pushes the task into the task queue, and the interface returns immediately.
4. **Internal Organization**: The system organizes and optimizes memories within the target Cube through deduplication and fusion.

## 4. Quick Start Example

It is recommended to use the `MemOSClient` SDK for standardized calls:

```python
from memos.api.client import MemOSClient

# Initialize client
client = MemOSClient(api_key="...", base_url="...")

# Scenario 1: Add memory for an individual user
client.add_message(
    user_id="sde_dev_01",
    writable_cube_ids=["user_01_private"],
    messages=[{"role": "user", "content": "我正在学习 R 语言的 ggplot2。"}],
    async_mode="async",
    custom_tags=["Programming", "R"]
)
# Scenario 2: Import content into the knowledge base and enable feedback
client.add_message(
    user_id="admin_01",
    writable_cube_ids=["kb_finance_2026"],
    messages="2026年财务审计流程已更新，请参考附件。",
    is_feedback=True, # 标记为反馈以更正旧版流程
    info={"source": "Internal_Portal"}
)
```
