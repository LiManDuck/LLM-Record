# 绪论


整个的大的方向主要包括 写作，写作的风格迁移

- diffusion-llm 在这块应该有先发优势，但是似乎没有看到使用diffusion-llm 做的相关论文
- 这一领域的关键应该是，插件化，小成本，参考 diffusion中的 小样本style 




# 论文


- [想让AI更懂人类？探索大模型情感偏好的革命性方法EmoPrefer](https://zhuanlan.zhihu.com/p/1969127500455053157)

- [Implementing Long Text Style Transfer with LLMs through Dual-Layered Sentence and Paragraph Structure Extraction and Mapping](https://arxiv.org/html/2505.07888v1): 有意思的是这似乎是两个独立作者完成的事情
- [CAT-LLM: Style-enhanced Large Language Models with Text Style Definition for Chinese Article-style Transfer](https://arxiv.org/html/2401.05707v2): 
  
      这个感觉不错，偏向工程性的方案，[repo](https://github.com/TaoZhen1110/CAT-LLM) ，依靠word 和 sentence级别的style分析，具体的是对每个风格的 动词，名词，等等词性的分析，对句子级别是长度等分析，最终形成各个风格的prompt, 在风格转换的时候，使用prompt 对句子 先使用prompt转为styless的句子，再嵌入之前的风格prompt，生成 


- [Distilling Text Style Transfer With Self-Explanation From LLMs](https://arxiv.org/html/2403.01106v2): 感觉确实工作内容比较不复杂， 就是使用LLM进行不同风格的提示词生成 数据 再进行蒸馏
- [Unsupervised Text Style Transfer via LLMs and Attention Masking with
Multi-way Interaction](https://arxiv.org/pdf/2402.13647): 看起来像是一个组装拼接工作





- [**Anthropic最新研究Persona vector人格向量**](https://zhuanlan.zhihu.com/p/1935098120171161009)： [github地址](https://github.com/safety-research/persona_vectors) , 也可以看看这篇解析的[知乎回答](https://www.zhihu.com/question/597170207/answer/1943292325989164618)



- [Persona Features Control Emergent Misalignment]()： OpenAI的人格特征的向量控制

# 方法


- 1.  prompt 多步推理： 

   使用 向量 聚类等等，形成 句子模板 和段落模板；




- 2. 风格模板训练： 
LLM 生成多个风格的句子库，使用蒸馏 等等方式训练 



- 3. 风格反演+插值



# 方法


- 1.  prompt 多步推理： 

   使用 向量 聚类等等，形成 句子模板 和段落模板；




- 2. 风格模板训练： 
LLM 生成多个风格的句子库，使用蒸馏 等等方式训练 



- 3. 风格反演+插值



-
