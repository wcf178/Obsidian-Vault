

# 基于Cursor的软件开发方法论探索

## 引子

### AI编程为什么最成功

1. 数据信息化程度高
2. 结构化、规范、标准、开放
3. 确定/客观
4. 反馈和验证最及时

### AI编程当前痛点

1. 完成到80分很容易，做到95分以上很难

2. 没有明确预期简单，有固定期望很难；宏观容易，细节难

   > 一、你自己也不清晰，核心诉求或实现链路
   >
   > 二、你自己清晰，但过于复杂，无法完全描述

3. 新建容易，修改难

   > 一、背景信息的干扰，错误的代码和设计会干扰
   >
   > 二、相关代码精准搜索的实现困难
   >
   > 三、全局视野差，框架抽象难（复杂项目/依赖问题）

4. 上下文复杂度对于生成质量影响非常大

   > 一、大模型层面：大模型的性能代价(KVCache)、注意力机制(注意力稀疏)
   >
   > 二、业务层面：概念模糊、模块干扰、逻辑冲突
   

### 问题总结

当前痛点可以统一抽象为三类**熵失控问题**：

| 问题类型             | 本质原因               |
| -------------------- | ---------------------- |
| 新建容易、修改困难   | 历史代码与设计噪声过大 |
| 宏观清晰、细节混乱   | 需求未被结构化固化     |
| 上下文越多，效果越差 | 模型上下文负载失控     |

### 目标和要点

- **上下文控制**
- **需求明晰和拆解**
- **企业/系统/垂类等私有知识管理**


## Cursor原理


### Cursor的上下文（Context）

- Prompt：用户的问题+上传的附件，一般不会很大
- 工具列表：静态列表，比较小
- 工具调用和返回：多轮调用，主体为文件内容，动态，无上限


### 智能体：大模型 + Agent

> Cursor本质上一个Agent平台
>
> Agent编程： 多轮工具决策和多轮工具调用

**Agent单轮执行过程**

1. 大模型决策
2. 多轮工具调用（单轮对话拆解为多轮大模型调用）
3. 大模型总结

**单轮对话示例**

```py
import openai

client = openai.OpenAI(api_key="your_api_key")
// 1. 工具列表
tools = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "读取指定路径文件的源代码内容",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "文件路径，如 'src/main.py'"}
                },
                "required": ["path"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_symbol",
            "description": "在工程中搜索特定的函数、类或变量定义",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "搜索关键词"}
                },
                "required": ["query"]
            }
        }
    }
]
// 2.模拟一个复杂的历史对话背景：
messages = [
    {"role": "system", "content": "你是一个类似 Cursor 的 AI 代码助手。你可以阅读代码并根据用户要求进行修改。"},
    
    # 第一轮：用户提出问题
    {"role": "user", "content": "请帮我修改 `utils.py` 里的 `calculate_discount` 函数，现在的逻辑没考虑会员等级。"},

    # 第二轮：模型决定调用读取文件的工具
    {
        "role": "assistant",
        "content": None,
        "tool_calls": [
            {
                "id": "call_001",
                "type": "function",
                "function": {"name": "read_file", "arguments": '{"path": "utils.py"}'}
            }
        ]
    },
    # 模拟工具返回的结果：代码内容
    {
        "role": "tool",
        "tool_call_id": "call_001",
        "content": "def calculate_discount(price):\n    return price * 0.9" 
    },

    # 第三轮：模型决定查看常量定义（假设在另一个文件或环境信息中）
    {
        "role": "assistant",
        "content": None,
        "tool_calls": [
            {
                "id": "call_002",
                "type": "function",
                "function": {"name": "get_member_config", "arguments": "{}"}
            }
        ]
    },
    # 模拟工具返回的结果：会员等级配置
    {
        "role": "tool",
        "tool_call_id": "call_002",
        "content": "{'GOLD': 0.8, 'SILVER': 0.85, 'BRONZE': 0.9}"
    },

    # 第四轮：用户补充细节（此时历史已经包含前面的代码和配置）
    {"role": "user", "content": "好的，请根据这些配置，把函数改成支持传入 member_level 参数。"}
]
# 3. 发起调用
response = client.chat.completions.create(
    model="gpt-5",
    messages=messages,
    tools=tools, # 传入工具列表
    tool_choice="auto" 
)
```


### Cursor工具列表

| **类别**   | **工具 名称**      | **功能说明**                                                 |
| ---------- | ------------------ | ------------------------------------------------------------ |
| **搜索类** | **Read File**      | 读取单个文件内容<br>（最多 250 行；最大模式下可达 750 行）。 |
|            | **List Directory** | 查看目录结构，不读取文件内容。                               |
|            | **Codebase**       | 在已索引代码库中执行语义搜索。                               |
|            | **Grep**           | 在文件中搜索精确的关键词或模式。                             |
|            | **Search Files**   | 按文件名模糊匹配搜索文件。                                   |
|            | **Web**            | 生成搜索查询并执行网络搜索。                                 |
|            | **Fetch Rules**    | 按类型和描述检索特定规则。                                   |
| **编辑类** | **Edit & Reapply** | 提出对文件的修改建议，并自动应用这些更改。                   |
|            | **Delete File**    | 自动删除文件（可在设置中关闭）。                             |
| **运行类** | **Terminal**       | 执行终端命令并查看输出。                                     |
| **集成类** | **MCP**            | MCP（Model Context Protocol）服务器<br>外部服务/工具交互。   |

### Cursor的模式
| 模式       | 适合场景                           | 工具使用策略             | 核心特点 / 行为                                              |
| ---------- | ---------------------------------- | ------------------------ | ------------------------------------------------------------ |
| **Agent ** | 任务目标明确、需要自动执行完整流程 | 依赖**全部工具**         | 可搜索、编辑、运行命令、多轮工具调用，自动完成复杂任务       |
| **Ask**    | 确认细节、明确方案、快速问答       | **仅使用搜索类工具**     | 不修改代码，不执行命令，偏向信息检索与解释                   |
| **Plan**   | 需求不清晰、需要拆解任务           | **只使用部分搜索类工具** | 1. 多轮对话澄清需求<br>2. 生成 `plan.md` 代办列表<br>3. 基于 `plan.md` 再切换 Agent 执行 |
| **Custom** | 高级用户 / 定制化工作流            | **用户自定义工具组合**   | 可按需启用工具，结合 **MCP** 接入外部系统，灵活度最高        |



## AI编程方法论

> Cursor / Agent 的能力上限，不取决于模型能力，而取决于：
> 你能否把复杂系统，压缩成“可被模型稳定理解的文档上下文”。

### 核心思维

- **分层**：抽象思维 - 复杂问题往上层抽象，简化成框架和结构
- **拆解**：将大的需求拆解成独立小的模块
- **解耦**：减少模块直接的依赖关系，将依赖标准化、协议化

### 文档即研发基准

#### 为什么选文档

人、模型、代码三者的需要一个**共同语义锚点**；
而**“文档”作为大模型的「稳定中间态表示」**是比较合适的选择。



通过「结构化文档」控制上下文复杂度，将复杂系统开发转化为
**可审计、可回滚、可推演的 AI 协同工程流程**

> 核心：大模型的出现使得文档维护变得非常简单

#### 文档的三种工程角色（非常关键）

##### （1）**上下文裁剪器**

> 文档不是增加上下文，而是**替代代码进入上下文**

- 不让模型直接吃全部代码
- 而是吃：
  - `architectures_v2.md`
  - `modules-xxx.md`
  - `master-worker-contract.md`

> 解决**“上下文复杂度对生成质量影响非常大”**问题的工程级方案。

------

##### （2）**需求冻结器**

**文档的作用是：**

- 文档驱动约束边界

- 把「阶段性共识」冻结下来
- 防止模型在修改时“顺手优化”别的模块（拆东墙补西墙）

> 文档是对大模型“自由发挥”的硬约束。

------

##### （3）**工程事实源**

- 抽象文档结构
- 统一全局框架或模块框架
- **事实来源于文档而不是代码**
- 代码只是**文档的一种实现形态**

这直接解决了：

> 新建容易，修改难
> 全局视野差，框架抽象难

#### 旧时代的工程文档

1. 需求设计
2. 产品设计说明书
3. 详细开发说明书
4. API约束文档
5. 数据字典/数据结构/数据流
6. 功能验证说明书
7. 测试报告

> 知识沉淀、标准的沉淀

#### AI时代的工程文档

1. Project Rules
2. Docs （第三方/API）
3. 需求设计：背景知识、目标管理、逻辑链路、边界约束
4. 架构设计：可落地链路、规范和标准、层级结构、核心抽象关系
5. 模块设计：详细设计说明书
6. 数据流
7. 协议和规范
8. 企业知识库
9. DeepWiki / Repo Wiki

----




### AI+文档驱动编程实践


#### 研发链路

- **思维导图**：头脑风暴，复杂问题结构化，进而框架化（约束范围）

- **文档驱动**：将思路落地成方案文档，将逻辑转化为流程

- **文档审计**：方案和方案审计，调整细节和实现，**简化**

- **由上至下**：**框架** -> **代码层级结构** -> **关键类/公共类** -> **模块类/函数定义** -> **函数实现**

- **代码审计**：文档实现情况，调整代码，反向推演文档，重新调整实现

- **功能调试**：单元测试（白盒），功能验证（灰盒测试），Cursor调试

- **校验和测试**：需求验证测试（黑盒）

> 全链路AI实现，人工审核

#### 文档驱动流程

> 下述是核心流程，关键节点
> 实际按情况调整和完善

##### 一、设计文档撰写

- 脑图：头脑风暴，枚举主要的功能节点和系统整体框架
- 资料调研、AI对话、多个AI交叉验证，完善未知节点
- 将脑图转化为框架设计草稿和大纲，完善草稿和大纲
- 基于设计草稿给提供AI，完善项目框架设计
- 校验AI生成的框架设计，识别未考虑部分或AI理解或者AI设计偏差的部分，回推设计草稿
- 基于调整后的草稿，重新生成明细框架设计，重复4~6步骤，明确框架设计主体部分正确，方向正确

> 核心功能和分层结构、框架体系、关键节点数据结构、存储方案、交互方案等
> 不要假装清晰，不要依赖AI补充

##### 二、功能拆解

- Plan模式进行框架设计拆解
- 模块设计文档
- 模块协作文档
- 项目架构 + 代码组织结构
- 功能模块 + 类/函数定义

##### 三、模块研发

- 每一步都可以中断&可以回滚
- 文档始终先于代码
- 文档和代码共同演进

```
需求 / 问题
   ↓
Plan 模式 → 生成 / 修正文档 
   ↓
Ask 模式 → 文档审计 & 问题发现
   ↓
文档冻结（阶段性）
   ↓
Agent 模式 → 实现或修改代码
   ↓
代码反向校验文档
```

#### 文档撰写的目标


##### 核心目标


| 目标         | 文档的职责                       |
| ------------ | -------------------------------- |
| 上下文控制   | 限定模型只关注“当前阶段必要信息” |
| 需求明晰     | 将模糊需求冻结为可校验文档       |
| 私有知识管理 | 将隐性经验外化为长期资产         |

##### 上下文控制 —— 用「文档分层」替代代码堆叠

- 不让模型直接读复杂代码
- 强制通过：
  - 架构文档
  - 模块文档
  - 协议文档

👉 **上下文从“代码规模”降维到“设计规模”**

------

##### 需求明晰和拆解 —— 文档即「需求分解器」

- Plan 模式 ≠ 想清楚
- **写出文档 ≠ 想清楚**

👉 文档迫使你回答：

- 生命周期？
- 边界？
- 异常？
- 谁负责？

------

##### 私有知识管理 —— 项目文档即「背景知识」

- 为什么这么设计
- 哪些坑已经踩过
- 哪些方案被否决

**最佳载体：**

- 架构文档演进历史
- 模块设计调整记录


#### 可落地的四层文档体系

##### **L0：愿景与边界文档（Why / What Not）**

- 系统目标
- 不解决什么
- Phase 边界

> 防止需求膨胀、模型过度设计
> 思维导图 + `architectures_draft.md`

------

##### **L1：架构主干文档（Architecture）**

- 模块划分
- 职责边界
- 核心抽象关系

> 案例中的`architectures_v2.md`文件

👉 **这是模型最重要的上下文输入**

------

##### **L2：模块设计文档（Module Contract）**

- Session / Context / Node / API
- 生命周期
- 对外接口语义

>  `modules-xxx.md`
>  控制局部修改不破坏全局

------

##### **L3：交互协议 & 事实文档**

- API 定义
- 存储与状态机
- 数据流

> `master-worker-contract.md`
> 用于模型「精确实现」阶段，跨模块交互


## 案例：从零开发远程浏览器集群

### 设计阶段

#### 思维脑图

1. 从零头脑风暴，使用思维导图工具，逐步完善脑图
2. 将脑图转化为设计稿，完善设计稿
3. 基于后续框架设计，反推设计稿优化

#### 编写框架设计

```
@architecture_draft.md  这是一个browser-pool的主体功能框架设计稿，请结合这个设计稿，完成主体设计内容，不用特别设计细节，当前的核心是完善框架和功能模块，并梳理框架情况，并将完善后的内容写入到新文件architectures_v2.md中，用于后续开发工作的框架指导文件
```

> 使用模型： claude-opus-4.5
> gemini-3-pro效果一般，偏提炼

#### 针对模块完善

```
@architectures_v2.md 请结合这个文件帮忙总结一下SessionManager怎么和NodeManager进行协同的？ 如果没有相关信息，请帮忙补充下
```

> ask模式： 借助大模型输出现在设计中的已有的问题

#### 方案调整

```
@architectures_v2.md  调整这个架构设计文件，调整里面关于Session、Node、BrowserInstance、BrowserContext的构建关系；Session对应到BrowserContext，每个节点对应同一个浏览器实例，但不是唯一的BrowserContext了，也即每个节点上可能有多个Session，每个节点能够创建的Context数量由全局配置调整。

其他调整如下：
1. 确定NodeManager的作用，作为底层的设备资源管理；
2. SessionManager作为Session的全生命周期管理
3. SessionPool作为SessionManager获取Session的资源池，该池支持Session复用、idle session管理等
4. NodeManager 管理下层节点资源的管理，包括BrowserInstance的启动和Context的创建和销毁
```

#### 方案优化

```
我们现在要搭建一个browser-pool系统，用于为爬虫和自动化任务集群提供底层浏览器资源；这是一份设计底稿 @build_docs/architectures_v2.md ，请问这份设计稿目前还有哪些问题？  着重看功能或者框架体系设计上的问题
```

> gemini-3-pro
> 先询问架构设计上的问题，让他们先总结，看一下问题是否合理

```
补充说明：1. Worker节点的确是微节点，也即Pod级别，需要满足1Node=1Browser  2.拒绝策略第一期先实现直接拒绝的方案，其他方案可以预留未来实现 3.Redis 大Key问题暂不考虑，还是使用Redis作为存储方案

请结合上轮对话的修改建议 和 补充说明，优化@build_docs/architectures_v2.md  这份设计稿
```

> 补充说明：剔除不合理的部分，让大模型重写优化

```
根据上文的问题，我给出了下述指定的解决方案，请根据解决方案调整@build_docs/architectures_v2.md 设计稿；请注意只需要调整我提到的问题其他问题无需调整

- 针对问题2：采用直连的模式，需补充page数量控制的问题
- 针对问题3：添加 进程级守护 逻辑
- 针对问题4：使用Redis保存元数据，使用对象存储处理大的Value
- 针对问题5： 采用上文建议方案
- 针对问题7： 采用上文建议方案
```

> 补充解决方案：完善设计稿

```
@build_docs/architectures_v2.md  这是一份比较完整browserpool设计稿，请整体概览下全文设计，整理设计中的问题；
保证前后设计思路的统一，里面的细节部分思路一致，修正不合理的部分
```

> 大模型修改调整容易拆东墙补西墙，让它整体在过一遍优化下


### 项目框架生成

#### 基于框架设计方案生成代码框架

```
@build_docs/architectures_v2.md  这是一份爬虫集群核心服务browser-pool的设计稿；请结合这个设计稿，提炼主干内容，生成代码目录结构，已知项目主体语言为Node.js；
要求符合成熟browser-pool项目模式，创建基础结构、核心功能模块、添加占位文件，核心目标是保证项目后续开发职责清晰，提高可维护性，促进团队理解项目协作，增强可扩展性。请注意保证结构简洁清晰高效，避免具体细节实现；
```

> claude-opus-4.5 大体上比较完整，但结构上还有不合理

```
@build_docs/architectures_v2.md  这是一份爬虫集群核心服务browser-pool的设计稿，我们已经基于这份设计稿生成了项目目录结构；当前项目目录结构有点问题，在项目主体目录packages目录下，将master和worker以及公共部分shared分为了三个部分；
但是worker对比master的开发工作量非常小，也无需和master共用shared，请去重shared部分，将对应的代码分别合并到master和worker里面，各取所需；
1. 可以分别根据自己的需要冗余代码，比如logger等模块
2. 请注意保持worker的轻量，不要过度设计
3. 请注意目前位于项目目录结构设计，保证调整后结构简洁清晰高效，避免具体细节实现
```

> 继续调整结构，worker的代码过于复杂，降低复杂度

```
@build_docs/architectures_v2.md  结合这份设计稿，检查当前的项目代码目录设计，是否有不合理的地方；
请综合考虑：
0. 框架体系和功能是否完善
1. 后续开发职责是否清晰，解耦是否合理
2. 是否有利于团队协作
3. 可维护性和可扩展性是否优秀
```

> 使用大模型校验审计，确认结构合理性

```
结合上文的评审的问题，给出以下修改建议：
针对问题1：去重shared相关的依赖，维持worker和master重复定义，我们期望是两个项目互相独立
针对问题2：采用方案2
针对问题4：NodeAgent 添加本地管控
针对问题7：添加配置热更新

请结合上面4个问题的修改建议，调整项目代码；请注意只需要调整结构和设计，无需具体实现，其他问题无需调整
```

> 修正上文审计中出现的不合理的部分，发现原始设计稿中scheduler不合理的地方

```
@build_docs/architectures_v2.md  这份browser-pool的设计稿，你决定关于sessionmanager的session调度器需要怎么设计？
请结合上文意见，完善 @build_docs/architectures_v2.md 文件；
1. 主要完善调度相关的内容
2. 保存简洁，请先总结内容，这是一份设计稿，只需要方案概述不需要实现
3. 请保持内容合理，上下文关联自洽，设计内容简洁大方
```

> 重新调整设计稿

```
@build_docs/architectures_v2.md  基于这份设计稿，重新设计master/schedulers模块
1. 移除独立的schedulers，所有的scheduler移到对应的功能模块下
2. session-pool无需独立的scheduler，依赖NodeManager下的NodeScheduler做资源调度
3. 转移scheduler后，请注意各个文件的依赖关系是否正确
```

> 关于schedulers部分设计有点模糊，重新调整设计稿
>
> 调整设计稿后重新调整项目目录结构和依赖设计

#### 拆解组件模块

```
@build_docs/architectures_v2.md  基于这份数据表，结合当前的项目结构，请拆解各个模块的实现方案文档，统一放到docs目录下

要求：
1. 采用自上而下的思考方式，合理的解耦各个模块
2. 模块互相独立且互相适配
3. 该文档为后续各个模块功能开发提供设计参考
4. 只需要实现master相关的内容，worker暂时忽略
5. 最后总结一个文档，说明master的功能，并提供给worker的开发提供对接说明
6. 切记：只需要写文档，文档请保证简洁高效、结构清晰
```

> gpt-5.2
>
> 生成独立模块文档，此文档结合代码结构作为参考
> 内容比较普通，但只需要保存拆解合理即可，内容不出现逻辑错误即可
>
> 核心：构建功能模块和主体框架对应的文件，为以后模块功能开发提供约束

```
接下来我们优先实现worker模块， 也即packages/Worker；请结合@build_docs/architectures_v2.md  和 相关代码组件，设计worker相关详细设计说明书请注意我们使用的是browserless/multi项目，这个项目提供了包括playwright、firefox等多种浏览器引擎；

注意事项：
1. 分模块编写，并参考@architectures_v2.md 
2. 新的文档放到 docs 目录下
3. 请尽量保持内容简洁高效，能够为后续开发提供指导
4. worker项目是browser的执行端，并会以镜像的方式扩容，请尽量保持轻量
5. 和master对接的部分，可以参考 @docs/master/master-worker-contract.md  中master设计内容
```

> worker思路，重新检查

```
根据 @build_docs/architectures_v2.md  和 @docs/worker  模块设计稿，检测当前worker模块的代码框架是否合理
```



### 模块设计完善

> 不同模块可以单独处理，这里省略

```
明确context L1级别无需持久化，请调整 @build_docs/architectures_v2.md  和 @docs/master/infra-storage-config-observability.md  @docs/master/modules-context.md   context相关内容，明确相关逻辑
```

> 有问题的模块，需要把所有相关的文档都修复明确清晰

### 功能实现

#### Worker

```
@build_docs/architectures_v2.md  @docs/worker  基于框架设计稿和模块设计文档，以及worker的骨架代码，逐步实现worker的所有功能

要求：
1. 完善当前代码中的TODO以及未实现部分
2. 不需要编写测试代码等周边代码
3. 要求代码简洁，上下文清晰
4. 浏览器相关的代码可以参考
5. 底层浏览器browserless相关代码，可以参考  @packages/worker/browserless-swagger.json  swagger协议接口文档说明
```

> worker相对简单，先处理
>
> 添加依赖的文档

```
基于 @docs/worker 模块设计文档，给当前的worker的代码设计一份测试方案，并简单撰写一个测试流程的代码
```

> 代码测试

#### Master - SessionManager

```
基于 @build_docs/architectures_v2.md  browser-pool框架设计稿，以及master 的框架文件 @docs/master/architecture.md ，以及session相关的设计说明书 @docs/master/modules-session.md ，实现sessionmanager以及 session-pool相关的代码
```

> 代码实现后，在复用处理上不够清晰

```
@build_docs/architectures_v2.md  结合这份框架设计稿和代码实现，考虑acquireSession 和 releaseSession 流程,请评估目前的代码逻辑有什么问题
```

> 大模型审计代码，发现问题，原始设计文档的盲点和漏洞

```
针对问题1~4这4个问题，调整方案如下：
1. L2复用级别支持idle session复用，如果没有匹配的idle session 从context storage中重建复用，也即热复用和冷复用都支持。如果获取到了热复用，则使用热复用，否则使用冷复用
2. L1级别的idle session的复用，是包含context复用的，也即包含代理IP和userState
3. 调整代理IP的获取时机，如果复用context，则无需获取新的代理IP

请根据调整方案，优化 @build_docs/architectures_v2.md  框架设计稿，同时优化 @docs/master 中相关的模块设计文档
```

> 重新设计文档，调整复用策略方案；包括session、context/代理IP等复用

```
基于@build_docs/architectures_v2.md  设计稿 和 @docs/master  中模块设计内容，调整关于context复用的逻辑，涉及到session、context、proxyip相关的逻辑都需要调整
```

-----

```
session-pool中有warmup机制，参考设计文件 @build_docs/architectures_v2.md  @docs/master/modules-context.md  ，目前方案设计中好像没有处理warmup启动session被复用的逻辑，你决定要warmup的session怎么触发复用比较合理，请结合代码了解明细设计，给出设计方案；

请注意：
1. 无需给出代码实现
2. 给出方案和思路大纲
```

> 优化完成context模块和复用机制后，针对warmup复用调整

```
请根据Phase 1: 核心逻辑修改这个部分描述，调整 @build_docs/architectures_v2.md  @docs/master/architecture.md @docs/master/modules-session.md 设计稿

要求：
1. 不要改动代码
2. 尽量精简
```

> 调整设计稿

```
请根据 @build_docs/architectures_v2.md  @docs/master/architecture.md @docs/master/modules-session.md  里面关于session warmup机制的说明，调整相关代码；

请注意：
1.适配不同级别的复用逻辑
2.调整三个复用级别中关于域名匹配的逻辑，支持通用默认场景和通配符场景
3.通用场景使用统一的标识符
```

#### Master - API

```
基于 @build_docs/architectures_v2.md  browser-pool框架设计稿，以及master 的框架文件 architecture.md，API的设计文档modules-api.md， master-worker交互相关的api说明文件master-worker-contract.md

请结合这些设计文件，实现browser-pool所有的api服务
```

> 引入了ServiceContainer，独立各个模块routers
>
> 分session/node/context/proxy/admin API模块

```
基于一下设计文件：
1.  @build_docs/architectures_v2.md  browser-pool框架设计稿
2. master 的框架文件 architecture.md
3. API的设计文档modules-api.md
4. master-worker交互相关的api说明文件master-worker-contract.md

审计当前的api服务实现有什么问题?
```

> gemini-3-pro
>
> 没有发现明显的问题

#### Master - ContextManager

```
基于一下设计文件：
1. @build_docs/architectures_v2.md   browser-pool框架设计稿，
2. master 的框架文件 @docs/master/architecture.md 
3. context的模块设计说明书 @docs/master/modules-context.md 

请结合这些设计文件，实现browser-pool中关于context管理的功能模块，要求兼顾Context存储、上下游调用、复用管理等逻辑
```

> *ContextKeyBuilder*
>
> **隐患**：reuseContextKey的需Fetcher提供

```
根据上面的调整内容:
@packages/master/src/index.ts 完善新加的context接口说明
@docs/master/modules-context.md 调整相关设计说明
```

> 反向优化模块设计说明书，调整存储相关的逻辑

```
当前是在搭建一份browser-pool系统中的context管理模块，下面是一系列设计文件：
1. @build_docs/architectures_v2.md   browser-pool框架设计稿，
2. master 的框架文件 @docs/master/architecture.md 
3. context的模块设计说明书 @docs/master/modules-context.md 

请结合设计文稿，按照工业级解决方案标准，审计当前context模块代码
```

> 审计代码设计

```
重新审计下， @packages/master/src/services/context/context-manager.ts  @context-storage.ts 和@blob-storage.ts  这个三个context的核心代码是否有设计上不合理、逻辑瑕疵、函数和参数定义不合理、代码层级不正确的情况
```

> contextKey 和 contextId 定义不合理

```
@build_docs/architectures_v2.md  @docs/master/modules-context.md @docs/master/architecture.md 
查看三个设计文件，调整context storage相关的逻辑，热复用和冷复用的机制。
调整内容如下：
1. 明确热复用使用 SessionPool，纯内存解决方案
2. 明确冷复用，分元数据和blob数据的存储；其中元数据存储支持redis和内存，blob存储支持local,s3,minio,内存
3. 明确contextKey使用contextId作为上下游交互主键，复用场景下使用独立的storageKey
```

> 待定

```
@docs/master/architecture.md @docs/master/modules-context.md @docs/master/modules-session.md @docs/master/architecture.md 
调整上面文档中所有关于ContextKey的设计，ContextKey独立于StorageKey，有独立的方案进行生成，调整范围如下：
1. ContextKey需要返回给Fetcher侧
2. ContextKey长度适中，用于API Path参数
3. ContextKey在Master中实现和StorageKey的对应关系

请详细思考L2级别的复用逻辑，确定给出的方案合理自洽
```

> 已确定流程自洽，给出大模型建议，让它自行完善

```
基于@docs/master/modules-api.md  @docs/master/modules-session.md  @docs/master/modules-context.md  这三份设计文件，检查关于接口设计、session管理、context管理和存储的相关代码有什么偏差？ 另外在逻辑上有什么漏洞？
----
帮我修复上面所说的问题
```

> 分两个步骤：先确认设计方案和已有问题，然后再修复



```
@docs/master/architecture.md @docs/master/modules-context.md @docs/master/modules-session.md @docs/master/architecture.md  请结合者四份设计文件，检测当前关于session和context的代码，是否有设计上不合理的内容，是否逻辑上纰漏，是否有代码组织上职责不清晰的地方？
```

> 再检查一遍

```
针对问题1：releaseSession完善 saveContext=true场景下context落地的逻辑
针对问题2：统一生成逻辑，每个Context原始定义由ContextKeyBuilder产生，后续通过参数传递来获取
针对问题3：按照建议修改
针对问题6：检测所有context相关代码，所有依赖ContextStorage save方法的代码找一个优化修整方案

请结合描述的4个问题的解决思路，调整优化代码
```

> 针对修复

```
请参考 @build_docs/architectures_v2.md  @docs/master/modules-context.md  设计文稿，优化关于context存储和交互相关的设计。

有以下相关修改方向：
1. contextKey作为同其他模块交互的key
2. storagekey作为Context模块内部的存储key，不对外暴露
3. 重写ContextStorage相关代码，基于设计稿完成函数定义和存储方案
4. 调整优化session、api、contextmanager等相关模块，按照新的方案进行联动调用
```



#### Master - NodeManager

```
现在在搭建一个browser-pool的项目，目前有两份设计文件，@build_docs/architectures_v2.md  @docs/master/modules-node.md  结合这两份设计文件，检测一下当前nodemanager的这两个设计文件的规划的有哪些问题

请注意：
1. 不用查看代码具体实现
2. 可以查阅 @docs/worker 文档内容进行对比分析
```

```
针对 @build_docs/architectures_v2.md  和 @modules-node.md 两个设计文件，完善一下内容：

1. Master Node管理添加handleContextDestroyedReport() 方法设计
2. NodeManager和WorkerNodes之间完整的对账逻辑（双向对账）
3. 完善同Session模块的交互设计
4. WS Token 签发逻辑
5. NodeAgentClient.extractContextState() 方法
6. 废弃contextId，统一使用runtimeContextId
7. Node模块同其他模块协作统一改成事件驱动，EventEmitter
8. 完善NodeManager同WorkerNode的NodeCommand设计，明确场景

需注意：
1. @architectures_v2.md 为整体架构设计文件，着重大的框架，请避免明细设计
2. @modules-node.md 为Master中Nodes的管理模块，这次的主要调整对象
3. @docs/master/master-worker-contract.md  为master和worker的交互定义，涉及到NodeManager相关，需配套调整
```

> 调整node模块

```
基于 @build_docs/architectures_v2.md  @docs/master/master-worker-contract.md  @docs/master/modules-context.md  基于框架设计稿和模块设计文档，重新检查master中关于node模块的代码并调整

要求：
1. 完善代码骨架和模块依赖关系
2. 不要完善Node模块的代码实现内容
3. 上游的session和context模块的内容，请根据情况调整
```

> 基于上面的调整，启动**plan模式**，Agent完成修改



#### Master - NodeAgentClient

```
基于 @build_docs/architectures_v2.md  @docs/master/modules-node.md  @docs/master/master-worker-contract.md  @docs/worker 相关文档，完善Master中NodeAgentClient的实现
```

> NodeAgentClient负责和WorkerNode进行交互
>
> **plan模式**



### 功能验证(黑盒测试)

#### 整体方案运行

```
启动Master和Worker，检测相关功能
```

> 准备redis/browserless docker实例

**问题1**

```
当前node节点因故退出后，master感知不到，请优化心跳校验机制
```

#### 针对核心模块测试

```
编写一个测试session案例代码，实现完整的session获取，session测试，页面抓取，session释放完整的流程；
要求：
1. 案例代码统一放在examples目录下
2. 案例代码使用的的环境变量，方便调试
```

**问题2**

```
这个是一个browser-pool项目，worker负责浏览器实例，worker将构建成docker实例，基于browserless/multi构建。
其中browserless都是node本地启动，NodeAgent访问默认是本地，与Master的对接也通过NodeAgent的代理管理，Master给上层Fetcher提供的浏览器ws和cdp默认也是由NodeAgent提供代理。

基于这些信息，结合文档，请完善以下内容：
1. worker docker file的定义
2. worker节点对外暴露ws的host解决方案
```

> 解决worker host问题，不用ip的解决方案

## 最后&总结

### 探索总结

1. 先想后做：先设计后开发
2. 复杂的问题简单化：分层设计、结构清楚、模块职责清晰
3. 文档和代码同样重要，文档跟随项目
4. 掌控AI，掌控你的项目，知道你在做什么

> 如果你觉得某些模块复杂，不用怀疑是你的设计有问题

### 注意事项

1. 成本控制：正确的方法论比模型更重要，成本更低
2. 效率提升：代码采纳率高、代码删除率低
3. 文档管理：合理的文档分层和组织结构，定期梳理，多而不杂

### 个人成长

- 设计/抽象思维
- 架构设计能力/知道正确的路/少走弯路
- 产品和开发边界的弱化
