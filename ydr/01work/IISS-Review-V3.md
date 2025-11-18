Weak Accept

## **Summary**

Tuning storage system parameters is complex, and existing methods—such as manual tuning, traditional learning algorithms, and LLM-based approaches—often lack semantic understanding and global awareness. To address this, this paper proposes an LLM-based intent-informed storage system (IISS), which enables cross-layer and automated parameter tuning and configuration through natural language interactions. Furthermore, this paper presents the corresponding design principles, system components, and workflow. Through preliminary experiments, the authors demonstrate that LLMs can effectively perform semantic understanding, API invocation, and parameter tuning tasks.

## **Strengths**

- Excellent writing with a clear structure.
- An interesting and important topic.

## **Weaknesses**

- Relatively straightforward design with limited innovation.
- Validation experiments are not robust.

## **Details & Comments**

Thanks for contributing to HotStorage'25! This work focuses on an important topic: parameter configuration and tuning in storage systems. It leverages an interesting tool: large language models (LLMs). The paper aims to build an intent-informed storage system (IISS), which enables cross-layer and automated parameter tuning and configuration through natural language interactions. It points out a series of challenges associated with using LLMs, such as uncertainty, and proposes corresponding design principles and workflows. In the validation experiments, the paper evaluates the capabilities of LLMs in semantic understanding, API invocation, and parameter tuning tasks. 

The topic is important and interesting. The writing of the paper is clear, with a well-organized structure. However, there are some critical issues. First, the design principles and system workflow proposed in the paper are somewhat straightforward and simplistic: it essentially introduces LLM-based analysis into various stages. Second, the validation experiments are overly simplistic. Drawing conclusions from a single basic use case is insufficiently rigorous. Moreover, these experiments focus more on testing the capabilities of the LLM rather than verifying the feasibility of the proposed design. Overall, the work appears engineering-oriented, without particularly impressive innovations and compelling validation experiments.

The more specific details are as follows:

- "LLMs’ probabilistic nature poses a risk of producing unsafe configurations..." and "adhere to predefined safety constraints, ensuring system stability." — do these two statements contradict each other in this paper? Could the probabilistic behavior of LLMs potentially violate the predefined safety constraints?
- "resulting in suboptimal throughput, increased latency, and inefficient resource utilization." The motivation section lacks data support and feels somewhat superficial and conventional. The experiments in Section 4 should be moved to the motivation part, as they mainly analyze the capabilities of LLMs. 
- The new experimental section should present preliminary results that evaluate the effectiveness of the IISS design, such as comparisons with traditional learning-based approaches or non-global optimization methods to demonstrate the potential improvements. 
- The design should delve deeper into addressing the key challenges identified: how IISS will optimize for LLM uncertainty through task decomposition, and how it will tackle the difficulty of cross-layer collaborative tuning. For example, what challenges arise in cross-layer system design? If cross-layer tuning can be achieved easily by LLMs without the additional designs, could this work be viewed merely as applying LLMs to a different scenario? However, the paper devotes significant space to introducing the "Principle," "LLM-driven Opportunity," components, and workflow, which are conventional.
- I remain skeptical about the uncertainty and hallucination issues associated with applying LLMs to real-world systems. In addition, would using natural language for communication between multiple components of IISS introduce additional uncertainty? The paper should provide a deeper discussion on how to address this critical problem.
- “The LLM recommended reserving 1.2 GB/s of bandwidth for AI checkpoints while throttling streaming reads to an upper bound of 300 MB/s to avoid contention.” The demonstration of semantic understanding and intent-to-execution through only a single simple case is not convincing. Is there any relevant benchmark that could be referenced or included?
- "Our experiments use GPT-4o and o3-mini-high via OpenAI’s API." LLM inference latency and accuracy depend on the selected model. Should the paper consider discussing and evaluating different LLM models? 
- In Figure 3, what does the x-axis represent? It's difficult to understand the results in the first two subfigures. It is recommended that an explanation of the results be provided either in the caption or in the main text, clarifying how the paper’s conclusions are supported by the data.
- (optional) If the final outcome is more like an engineering-oriented agent implementation, would it be more appropriate to submit to a software engineering conference, such as FSE?