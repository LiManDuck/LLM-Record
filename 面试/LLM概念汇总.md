
# 数据
	- 数据清洗
    - agent 数据合成
    - 多轮对话数据
    - 推理数据
    - 多步推理 数据合成
    - 拒绝采样
    - tool call 数据 : [function calling微调笔记(一)](https://zhuanlan.zhihu.com/p/4791144159)
#  模型架构
	- 注意力机制: 稀疏； BIAS ; 门控注意力机制
    - MLP层： MOE
    - 位置编码
    - 
    - MHC
    - MTP
    - 推理和非推理
    - MOE
    -
    
# AI理论
	-  

#  LLM 训练 和实践
    - PT 和CPT
    - SFT
	- 强化学习
	    - 拒绝采样
        - RL的三大采样方法
        - PPO, DPO, GRPO, GSPO
        - DPO 及其衍生算法
        - GRPO 及其衍生算法： DAPO 
        - 重要性采样
        - 过程奖励模型

    - ToolCall训练 
    - 蒸馏
    -  RL 蒸馏
    - AgentRL
    - 训练问题定位    
    - 训练框架
	    -  trl
        -  transformers
        -  deepspeed
        -  
    - MARL: multiagent rl

# LLM推理部署
	- KV CaChe
    - PD 分离
    - 批推理的对齐方式
    - 量化 a
    - pt  , 
    - MTP 

# Agent
- 多模态agent
- harness engineering , prompt -> context -> harness
- agent的产品形态： chatbot / RAG  ，-> claude code / deepresearch  -> clawbot

- Agent的设计
    
	- 记忆
    - 工具
    - 上下文管理
    - Agent的循环设计
    - 脚手架 ：skill,  hook
    - session 对话 

- Agent的之间彼此交互
	- agent 与用户 ： askuserquestion 
    - agent 与agent : subagent or multiagent：参考claude code的最新版本的功能

- 应用实践
	- prompt engineering ,  -> context engineering ->  Harness Engineering 




# RAG 
	- chunk 





# AI应用


- 意图漂移
- 上下文污染与注意力稀释
- 注意力稀释
