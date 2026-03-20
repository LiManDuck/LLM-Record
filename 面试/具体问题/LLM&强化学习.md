
# 题目汇总


建议直接看这些

- [面经记录 LLM+RL 50问](https://zhuanlan.zhihu.com/p/1974870853121508975)
- [强化学习面试题及答案](https://zhuanlan.zhihu.com/p/1899595400442258990)

# 理论

- [SFT到RL再到On-Policy蒸馏的公式与代码实现](https://zhuanlan.zhihu.com/p/2015096490276702023)





# 通用


## 训推不一致

- [训推不一致和TIS](https://zhuanlan.zhihu.com/p/2001796330624942979)



## 怎么对一个场景，或者 一个agent进行rl

- 建议观看 openclaw RL 

# RL的训练问题





# DPO 问题


dpo主要涉及的是，
- chosen 和 rejected的值变化，
- 二者的数据构造来源影响（自己生成还是非自身生成）
- beta 的参数的设置影响
- dpo失败的可能原因


- [基于Qwen3的DPO/KTO/ORPO/Simpo经验总结](https://zhuanlan.zhihu.com/p/1907949654739513685)

- [DPO训练过程中出现chosen reward下降](https://zhuanlan.zhihu.com/p/1892320522487959674) ： 解释了不同的训练情况的问题


# GRPO 及其变种


对于GRPO的问题 ，可以主要看这个文章，以及DAPO的解法

- [GRPO训练实录：DAPO 为什么打不过原始GRPO？](https://zhuanlan.zhihu.com/p/2013380469807423705) : 不是DAPO 不好，而是是它为长序列、大搜索空间的数学推理量身定制的。工具选择是短序列、小搜索空间的任务，问题结构完全不同。

**从这句话中我们也能看出 Reason 和  工具调用 这两项任务能力是正交的**, 有个论文是做分析的然后各自训练，比将这两个能力放在一起训练要好 ，glm4.7也是分开训练的


# 其他


- 为什么有了 llm as judge还需要单独训reward model？批量下载
