## 模型

### 传统方式

识别(ASR) + 大模型(LLM) + 合成(TTS)

传统的三段式流水线

语音\--\>ASR\--\>文字\--\>模型-\>回复文字\--\>TTS\--\>回复语音

需要过3次调用模型, 不可避免的延迟

### Omni模型子集

(Speech to Speech LLM / Voice-Text LLM)

#### Moshi(Kyutai)

目前实时性最强, 专门对话设计

理论延迟160ms, 通常200ms左右

#### GPT-4o Realtime

延迟250ms左右

#### Qwen 2.5-Omni-7B

延迟未知, 中文强

#### Fun-Audio-Chat-8B

也是阿里的千问的, qwen fun系列, 专门做s2s的. 比omni专精

<https://huggingface.co/FunAudioLLM/Fun-Audio-Chat-8B>

#### CosyVoice-0.5B

小模型, 可以端上部署

## 工程

### RAG

#### 级联式语音 RAG

-   实时转录: ASR过程

-   文本检索: 上一步的文本处理结果作为输入

-   

#### 原生语音 RAG

**SpeechRAG框架**

-   **跨模态对齐:** 将语音embedding和文本embedding对齐到一个向量空间

-   **直接语音检索:** 直接使用语音向量检索相关文本

#### 工具

LiveKit

vLLM-OMNI

NVIDA ACE

### 优化手段

#### VAD(语音活动检测)

用于首包延迟优化
