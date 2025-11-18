

## Summary

This paper focuses on the scenario of log data compression and highlights the inefficiency of existing binary compression or parser-based compression methods in certain contexts, such as small logs or memory-constrained environments.  In response, the paper introduces Variable-Length Substitution e (VLS) and, through a series of definitions, problem modeling, and precise solution analysis, establishes a delta compression algorithm called LogDelta. To address the high time complexity of this algorithm, the paper further proposes an approximate solution based on Q-GRAM to transform the problem from substring substitution to substring matching. Experimental results demonstrate that LogDelta can improve the compression ratio.

  

## Strength

- The concept of transformation presented in the paper is interesting.
- The overall structure of the paper is clear.

  

## Weakness

- There is a lack of solid background, problem of SOTA, and motivation in the paper.
- It is difficult to read the paper and understand its innovations.
- The evaluation is not thorough and lacks persuasiveness.

  

## Details & Comments

Thanks for contributing to FAST'26!  This work focuses on the scenario of system log data compression, proposing the use of Variable-Length Substitutions (VLS) to further improve the compression ratio of delta compression. It also introduces an approximate solution method to reduce time complexity, ultimately deploying it on IoTDB. As I understand it, traditional methods involve multiple operations for single-character replacements, whereas VLS packs more characters in a single operation, thereby reducing the number of operations and minimizing overhead. Transforming the problem from replacement lookup to substring matching allows for a more approximate method to quickly obtain results.
There are some advantages in this paper that can be further maintained and expanded upon. First, the overall structure of the paper is clear, following a logical sequence of definitions, modeling, theoretical solutions, and approximation solution optimization in an orderly manner.  Second, the transition from single-character replacement to variable-length string replacement and from substitution to substring matching is interesting. More detailed analysis can be done around this point, including extraction of the advantage, theoretical compression benefits, and the associated costs.
However, there are some critical issues. First, this paper lacks essential background, a comparison with existing state-of-the-art methods, and a clear explanation of the motivation driving this work. Starting from Chapter 2, the paper discusses the LogDelta method. Second, it is difficult to read the paper and understand its innovations. Many figures, formulas, and tables lack proper explanations or the explanations provided are unclear. There are too many proofs and too few comparisons to existing methods. It is unclear which aspects are uniquely proposed by LogDelta. Is it the VLS? Or the Q-gram matching? Third, the evaluation is not thorough and lacks persuasiveness. The dataset size, comparison methods, and the final experimental results all raise doubts. The compression ratio improvement of LogDelta over LZMA is quite limited.

The more specific details are as follows:
- "based on edit distance computation and differential encoding. By preserving the edit operations between two strings, LogDelta encodes the differences as compact information" It is difficult to discern the innovation of LogDelta from this paragraph. Doesn't this simply amount to applying differential encoding to a different scenario?
- The transition from single-character editing to multi-character editing is interesting but quite straightforward. Is this the first time it has been proposed? Why haven't previous methods adopted this approach? Does VLS potentially complicate simple problems?  The paper lacks relevant explanations and comparisons.
- "Table 1: The cost for encoding information" I don't understand what "m" refers to in this context. What does Int(m) represent?
- "The proof is given as follows." Simplify the proof of the exact solution to the problem and move the excessive proofs to the appendix. The paper should highlight the unique observations and designs introduced in this work.
- "Based on these insights, the Q-gram Matching algorithm is proposed." The same issue applies: if these are not being proposed for the first time, avoid including excessive terminology, algorithms, and proofs. The unique contribution of LogDelta should be emphasized, which is the application of Q-gram for matching identical substrings.
- A key issue is that, whether it's the introduction of VLS or the acceleration of Q-gram matching, I don't see a strong correlation between these methods and the log data scenario the paper focuses on. It seems like these methods could be applied to any string-based differential compression scenario.
- "Table 2: Statistics of experimental datasets" The selected test datasets are relatively small. Why not use larger workloads?
- In Figure 8, the log compression method performs worse than the general-purpose LZMA in both compression speed and compression ratio. Is the comparison test fair, and should there be relevant explanations provided?
- In Figure 8, is the compression speed of these methods reasonable? For example, MLC has only 0.03MB/s.
- In Figure 8, in many scenarios, such as "Zookeeper," the speed of LogDelta decreases by several times compared to LZMA, while the compression ratio improvement is quite limited. Is there an explanation for this?
- In Figure 9, the high overlap between the approximate algorithm and the exact algorithm is rather puzzling, even in the Zookeeper scenario. I would like to know under what circumstances the approximate algorithm would fail.
- "In applications, MinHash is preferred for its speed, albeit with a slight trade-off in compression efficiency." It is necessary to provide an explanation of the construction of the MinHash and Cosine objects.
- Some of the comparison methods and datasets could perhaps refer to μSlope (OSDI'24). Additionally, should there be a comparison or discussion on this?
- Perhaps this work would be better suited for a conference focused on algorithm analysis or related conferences where the comparison methods are discussed, such as ASE, ICSE, etc.