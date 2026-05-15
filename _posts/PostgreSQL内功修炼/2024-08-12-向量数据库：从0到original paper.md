---
layout: post
title: "向量数据库相关概念"
date: 2024-08-12
categories: [PostgreSQL内功修炼]
description: "从零讲解向量数据库核心概念，涵盖embedding嵌入、RAG检索增强生成及LLM基础知识。"
---





# 向量数据库相关概念

## 亿点点历史知识

LLM的模型发展历史，[Harnessing the Power of LLMs in Practice: A Survey on ChatGPT and Beyond](https://arxiv.org/pdf/2304.13712)[^0.11]：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2270d4fa2caf45ac8315718da49cc148.png)

很多人都是从ChatGPT爆点后才逐渐了解到大模型，但在爆点的前几年大模型的发展已经开始了诸神之战。一些机构发布了许多革命性的论文，公司部分像是Google、DeepMind、OpenAI、Meta、Microsoft，学校部分像是Stanford、Berkeley、CMU、Princeton、MIT[^3]。

总共分为三个阵营： 

- Google和DeepMind阵营 — Gemini、Bard
- Microsoft和OpenAI阵营 —ChatGPT、Bing
- Meta开源社区阵营 — Llama 



近几年现有大模型产品发布时间线[A Survey of Large Language Models](https://arxiv.org/pdf/2303.18223.pdf)[^0.4]：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/848d42bc952b4cc8ab4cecc9078bad53.png)





## 生成式AI的简单概念

**AIGC（Artificial Intelligence generated Content）**：AIGC精准概念是利用AI自动生成内容的生产方式。广义的AIGC可以近似训练像人类一样具备生成创建能力的AI技术，即生成式AI，它可以基于数据和生成演算法模型，自主生成创建新的文本、图像、音乐、视频、3D交互内容等各种形式的内容和数据，以及包括开启科学新发现、创造新的和意义等。



**LLM（Large Language Model）**：LLM即大语言模型，LLM能够捕捉和处理复杂的语言模式和语义，也就是它能够理解和生成人类语言。GPT3、ChatGPT、BERT、T5、文心一言等都是典型的大型语言模型。



**NLP（Natural language processing）**：自然语言处理（NLP）是研究如何让计算机读懂人类语言，也就是将人的自然语言转换为计算机可以阅读的指令。LLM是 NLP 中的一个重要组成部分。



AIGC取得了令人瞩目的增长，有很大因素就在于自然语言处理（NLP），而推动NLP发展的最大功臣是大语言模型（LLM）。今年（2024年）AIGC在视频、声音等领域也同样发展迅速。[^0.1]



**prompt**：提示符、指令，提供给AI对应描述任务的自然语言，用于引导语言模型（如GPT-3或GPT-4）生成相应的输出[^1]。（大伙其实都知道是啥了，不多解释）



**embedding**：

嵌入是一种表示对象（如文本、图像和音频）的方法，将其表示为连续向量空间中的点，这些点在空间中的位置对机器学习算法具有语义。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a637bf03a28c4cbdb760d4723575d771.png)

根据[GloVe](https://nlp.stanford.edu/projects/glove/)英文单词和向量相关性，这有个可[直接看词汇embedding后的二维图](https://blog.echen.me/embedding-explorer/#/)。这个就是自然语言嵌入为二维向量：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6d88a1a8bdd743b4bc6396b66f800d31.png)





## RAG

RAG(Retrieval-Augmented Generation)是一个包含文档检索和大型语言模型（LLM）回答生成的两阶段过程。初始阶段利用密集嵌入来检索文档。根据具体使用场景，这种检索可以基于多种数据库格式，例如向量数据库、摘要索引、树索引、key索引[^1]。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c80ad5a4889f4a038da279a8a4ac4851.png)

[RAG原始论文](https://arxiv.org/pdf/2005.11401)[^2]是2020年5月22日发布，来自Facebook(Meta)、伦敦大学学院、纽约大学的人员提出RAG的通用微调方法 。RAG包含如下特性[^3]

- RAG模型结合了预训练的记忆以帮助语言的生成
- RAG模型生成的语言更加具体、多样且具有事实性

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/31764e1c55f447a591329119c115f726.png)



OpenAI在2023年3月23日发布的[chatgpt-retrieval-plugin](https://github.com/openai/chatgpt-retrieval-plugin)仓库推荐在RAG中使用向量数据库，从此在应用领域向量数据库跟随大模型的爆火而备受关注

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7b7aac95dce14a76bde81da9b808d0c5.png)



## 向量数据库能为AI带来什么？

向量数据库能在RAG中为大模型提供数据检索和数据长期存储的能力[^x]

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/8196a28659164fc2999cfa0cff93db6e.png)

为什么要使用RAG？任何话语都没有祖师爷OpenAI说话有力。下面一段话源自2023年3月OpenAI的发布的检索插件使用指引[^55]，用ChatGPT翻译如下：

> 开源检索插件使 ChatGPT 能够访问个人或组织的信息源（需获得许可）。用户可以通过自然语言提问或表达需求，从其数据源（如文件、笔记、电子邮件或公共文档）中获取最相关的文档片段。
>
> 作为开源且自托管的解决方案，开发者可以部署他们自己的插件版本并注册到 ChatGPT 上。该插件利用 OpenAI 的嵌入技术，并允许开发者选择一个矢量数据库（如 Milvus、Pinecone、Qdrant、Redis、Weaviate 或 Zilliz）来索引和搜索文档。信息源可以使用 webhook 与数据库同步。

总之，OpenAI推荐大伙使用向量数据库。

[向量数据库凉了吗？](https://mp.weixin.qq.com/s/0eBZ4zyX6XjBQO0GqlANnw)不仅没有凉，RAG发展到今天，甚至都已经烂大街了--[RAG 技术真的“烂大街”了吗？](https://mp.weixin.qq.com/s/awIInAtPOkZz_s4jg9TO_w)。而向量数据库因具有高检索效率、数据存储可靠性等等特点，是RAG的重要一环。



## 常见的向量数据库

自从OpenAI发布RAG repo以来，出现了许多向量数据库（当然以前也有一些）。一些公司也拿到了相当可观的融资[^y]

| Company       | Headquartered in     | Funding        |
| :------------ | :------------------- | :------------- |
| Weaviate      | 🇳🇱 Amsterdam         | $68M Series B  |
| Qdrant        | 🇩🇪 Berlin            | $11M Seed      |
| Pinecone      | 🇺🇸 San Francisco     | $138M Series B |
| Milvus/Zilliz | 🇨🇳 / 🇺🇸 Redwood City | $113M Series B |
| Chroma        | 🇺🇸 San Francisco     | $20M Seed      |
| LanceDB       | 🇺🇸 San Francisco     | Venture        |
| Vespa         | 🇳🇴 / 🇺🇸 Indianapolis | Yahoo!         |
| Vald          | 🇯🇵 Tokyo             | Yahoo! Japan   |

向量数据库发布时间：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/ab0ff27930004b80b3ff58bbdd90c382.png)

[向量数据库性能对比](https://github.com/erikbern/ann-benchmarks)[^yy]：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/5d9cd531b116496cb6e5df6cfc709725.png)

专用向量数据库一般来说比插件来实现向量的传统数据库性能更好，原因大概有两点：

- 专用向量数据库针对向量做的底层存储，其性能一般都要好于无目标性的传统数据库
- 专用向量数据库一般更新（基本都是go、rust实现的），他们在代码实现层面更方便优化



但这不代表插件实现的向量数据库没有用武之地：

- 传统数据库原生就支持更多功能，不仅仅是相似度计算
- ACID，传统数据库的存储更为安全
- 在同一个数据库中更容易操作数据



向量数据功能对比：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/5a8e0cd6789b406abe5208a6e09967f0.png)

[上面对pgvector](https://github.com/pgvector/pgvector)的描述不太准确了，pgvector目前已经支持了HNSW，基于pgvector的生态[pgvectorscale](https://github.com/timescale/pgvectorscale)也支持了DiskANN。



# 数学相关概念

*数学说：“我站在山顶看你们玩”*

### **标量**

标量就是具体的数。标量没有方向，一般是相对向量来定义的。

### **向量（vector）**

在欧几里得空间中，向量包括长度和方向。如下点*A*到点*B*的向量**a**（包含两个点的信息，和方向信息）[^10]：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/888104b4e98a4a4a8fe5a04de5bcfc05.png)

### **单位向量（Unit vector）**

大小为一个单位的向量就是单位向量。单位向量等于向量除以欧几里得长度[^11]：
$$
\vec a = \frac{\mathbf a}{||\mathbf a||}
$$

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2fd36035e8d146778b4316ae2c3fba17.png)

数学中的单位向量（[Unit vector](https://en.wikipedia.org/wiki/Unit_vector)）在pgvector和OpenAI embeddings中被称为normalized vector。（注意，不要跟数学中的normal vector混淆，[normal vector](https://en.wikipedia.org/wiki/Normal_(geometry))是法向量，是另一个概念）

*为什么要使用单位向量？*

OpenAI embeddings对使用单位向量的解释[^12]:

>OpenAI embeddings are normalized to length 1, which means that:
>
>- Cosine similarity can be computed slightly faster using just a dot product
>- Cosine similarity and Euclidean distance will result in the identical rankings



### **稀疏向量（sparse vector）**

稀疏向量之所以称为稀疏向量，是因为向量中的信息分布稀疏。通常情况下，我们需要在成千上万个零中找到几个1（相关信息）。因此，这些向量可以包含许多维度，通常在数万个维度。

稀疏向量和密集向量的比较。稀疏向量包含稀疏分布的信息位，而密集向量在每个维度上都包含更多信息，信息密集。[^13]

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b075ec5d468f43418ce09a13479ae492.png)



### **欧几里得空间（Euclidean space）**

简称欧式空间，是数学中最基本的空间。现代数学中，正整数n维的空间称为欧几里得空间。

还有一些其他的空间定义，如内积空间、希尔伯特空间，它们在数学定义上有不同，但是在数据库/真实世界中不会分那么清楚，基本要知道的是内积空间、欧几里得空间、希尔伯特空间都可以包含点、向量、内积等元素，我们可以简单的称他们为“**多维空间**”。它们的区别可参考[漫谈数学中的各种空间](https://zhuanlan.zhihu.com/p/684643954)[^14]

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2faddd78735742a9a151b6d2edf4663a.png)





### **欧几里得距离(Euclidean distance)**

简称欧式距离，也就是我们一般认为的点与点的距离，即线段的长度[^15]

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/fa5f70c0868d4d8eaad0e33a1ad1a67d.png)



在二维空间中，点q与点p的欧式距离为：
$$
d(\mathbf p,\mathbf q)=\sqrt{(p_1-q_1)^2+(p_2-q_2)^2}
$$


在n维空间中，点q与点p的欧式距离为：
$$
d(\mathbf p,\mathbf q)=\sqrt{(p_1-q_1)^2+(p_2-q_2)^2+\cdots+(p_n-q_n)^2}
$$



### **曼哈顿距离（Manhattan distance OR Taxicab distance）**

$$
d(\mathbf p,\mathbf q)=  \sum_{i=1}^n |  p_i-q_i|
$$

曼哈顿距离是两个点在各个维度上的差的绝对值之和[^16]

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/d193798b2a6c4546a3b07aa22eed5ee6.png)

上图中，绿线为欧式距离，红、黄、蓝线为曼哈顿距离。



### **闵氏距离(Minkowski distance)**

$$
d(\mathbf a,\mathbf b)= \left( \sum_{i=1}^n |  a_i-b_i|^p \right)^{1/p}
$$

下图表示在闵式距离中取不同的p时，单位长度的点到原点的距离[^17]：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6d4a0669aff84fdfb1698eb110416220.png)

- 当p=1时，为曼哈顿距离，也写作“L1 distance”

- 当p=2时，为欧式距离，也写作“L2 distance”
- 当p=n时，为闵式距离，也写作“Ln distance”



### **余弦相似性（Cosine similarity）**

即两个向量的夹角的cosine值，也叫余弦值。余弦相似性只跟两个向量的夹角有关系，跟向量的长度无关[^18]

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/9bff6a982e2b4a8b95840b89d6520069.png)

两个向量角度越小，余弦相似性越大，值域[-1,1]。cos(0)=1，cos(90)=0，cos(180)=-1。



两个向量的余弦相似性写作：
$$
cos (\theta)
$$
用向量表示：
$$
cos (\theta)=\frac{\mathbf a\cdot \mathbf b }{||\mathbf a|| \, ||\mathbf b||}= \frac{ \sum_{i=1}^n \mathbf a_i \mathbf b_i}{ \sqrt {\sum_{i=1}^n \mathbf a_i ^2} \cdot  \sqrt {\sum_{i=1}^n \mathbf b_i ^2}}
$$


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/66c36d0b08b4434ca5e88c5406a0fabf.png)





### **内积（ inner product）**

也称为点积（Dot product），可以被用来表示向量的长度和角度。内积等于向量的*欧几里得距离*乘以*夹角余弦值*。

二维空间中的内积为：
$$
\mathbf a\cdot \mathbf b=||\mathbf a|| \, ||\mathbf b||\,  cos \theta
$$
or
$$
\mathbf a\cdot \mathbf b= a_1 b_1 + a_2 b_2
$$
在n维空间中的内积为（**a**=[a1,a2,···,an],**b**=[b1,b2,···,bn]）：
$$
\mathbf a\cdot \mathbf b=\sum_{i=1}^n a_ib_i= a_1b_1 + a_2b_2 + \cdots + a_nb_n
$$




此时来看下面这个图就能看懂了。用上面的公式也可反推n维向量的距离操作符的含义。

他们分别是：欧几里得距离，余弦距离，内积[^19]

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/151c8f26130f41d3b668407f920234d1.png)

他们都可以描述两个向量的相似性。

- 欧几里得距离仅包含两个向量的距离信息
- 余弦距离：仅包含两个向量的角度信息
- 内积：既包含距离信息，也包含角度信息

当然还有更多数学模型上的向量相似性计算方法，不过要看向量数据库支不支持。



### **jaccard距离**

简而言之，交集除以并集[^20]



![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1d1b2da25b724ffe938d3eb8e3b0f24c.png)

用公式表达：
$$
J(A,B)=  \frac{|A\cap B| }{|A \cup B|}
$$


用向量表达，即计算两个相等元素个数和不相等元素个数的比[^79]

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7ee329f945b045c6b3627742b358382b.png)

### **Hamming距离**

两个相同长度的字符串或向量在各自位置上的不同数量[^22]

例子：

- "ka**rol**in" and "ka**thr**in" is 3.
- "k**a**r**ol**in" and "k**e**r**st**in" is 3.
- "k**athr**in" and "k**erst**in" is 4.
- **0000** and **1111** is 4.
- 2**17**3**8**96 and 2**23**3**7**96 is 3.



图例[^23]：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c95f9b573365451082e0995d0000f777.png)



### **Delaunay三角剖分**

Delaunay三角剖分是对平面上一组点集进行的操作，它将这些点集的凸包（包含多个点）细分为多个三角形，这些三角形的外接圆不包含任何点集中的点。这样做最大化了所有三角形中最小角的大小，并倾向于避免产生细长的三角形[^24]。

不满足 三角形的外接圆不包含任何点集中的点：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e1101a2366e34c98aa0582f9e2520a3b.png)

满足 三角形的外接圆不包含任何点集中的点：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/0ece301d99de432a9aa0ea6d040f3e50.png)



例如剖分一下点集：



![img](https://img-blog.csdnimg.cn/img_convert/589ef882cc21081d350299ecdeb8f1f5.webp?x-oss-process=image/format,png)

合理的剖分：

![img](https://img-blog.csdnimg.cn/img_convert/6663fe885724df154c0ca78d964d7bae.webp?x-oss-process=image/format,png)



Delaunay三角剖分其实并不是一种算法，它只是给出了一个“好的”三角网格的定义，它的优秀特性是空圆特性和最大化最小角特性，这两个特性避免了狭长三角形的产生，也使得Delaunay三角剖分应用广泛。



### **voronoi diagram**

Delaunay三角剖分（Delaunay triangulation）是一个在一般位置中的离散点集P的三角剖分，它对应于P的Voronoi图的对偶图。Delaunay三角形的外接圆心是Voronoi图的顶点。在二维情况下，Voronoi的顶点通过边连接，这些边可以通过Delaunay三角形的邻接关系得出：如果两个三角形在德劳内剖分中共享一条边，那么它们的外接圆心在Voronoi镶嵌中应该用一条边连接起来[^25]：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/714f4464a9b34cf99eef4bda804904aa.png)



Voronoi图的主要特性是：*从一个质心到其区域内任何点的距离都小于该点到另一个质心的距离*。
$$
R_k=\{x \in X \,|\,d(x,P_k) \le d(x,P_j) \; \mathrm{for \,all }\,j \neq k\}
$$
Rk是质心，d(x,Pk)是质心到其区域内任何点的距离，d(x,Pj)是其他质心到该区域任何点的距离。

由于d距离计算方式的不同，Voronoi图还可以有不同的样貌[^26]：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b68ccc0bfbec4f65a15631a0dece0cc9.png)









# 向量数据库的索引

## 邻近搜索

ENN（exact nearest neighbor）：指的是在给定数据集中找到与查询点最接近的点或向量。这种方法保证了最高的精确度，但随着数据集的规模增大，计算成本会急剧升高，因为它需要评估查询点与数据集中每个点之间的距离。

ANN（Approximate nearest neighbor）：为了提高效率，在牺牲一定的精确度条件下，近似的找到与查询点最近的点。这种方法通过各种算法实现，可以大大降低计算成本，特别是在处理大规模数据集时更为有效。

KNN (K-Nearest Neighbors): 这是一种常用的机器学习算法，工作原理是找到数据集中距离给定查询点最近的K个邻居



## 索引评价标准

评价一个索引的好坏总是依赖于具体的数据模型的，总体来说包含如下几点：

- 查询时间。查询的速度是非常关键的，在大模型中尤为重视。
- 查询质量。ANN查询不会总是返回最精准的结果，但查询质量也不能偏差过大，而查询质量也有很多指标，其中包括最常用的召回率。
- 内存消耗。查询索引所消耗的内存，在内存上查找明显比在磁盘上查找更快。
- 训练时间。有些查询方法需要训练才能达到较好的状态。
- 写入时间。写入向量时对索引的影响，以及包含所有维护。



其中大部分指标都比较好理解，这里主要讲解*查询质量*：

ANN查询中，不会总是返回精确的结果。搜索一个元素集时，包括：查询的范围（retrieved elements）、所有正确元素集（relevant elements）、返回的正确元素集（true postives）、返回的错误元素集（false positives）[^60]：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/4fbbe619b0ed43b592c4e053e1537297.png)



TP = True positive; FP = False positive; TN = True negative; FN = False negative

**准确率（Accuracy）**：
$$
Accuracy=\frac{TP+TN}{TP+FP+TN+FN}
$$
或写成：
$$
Accuracy=\frac{所有正确的元素}{所有元素}
$$






**精确率（Precision）**：
$$
Precision=\frac{TP}{TP+FP}
$$


或写成：
$$
Precision=\frac{检索出的正确元素}{检索出的所有元素}
$$




**召回率(Recall)**：
$$
Recall=\frac{TP}{TP+FN}
$$
或写成：
$$
Recall=\frac{检索出的正确元素}{所有正确的元素}
$$




**F-值（F-measure）**：相当于加权后的精确率和召回率
$$
Recall=2 \cdot \frac{precision \cdot recall}{precision+recall}
$$


示例：考虑一个用来识别数字照片中的狗（和相关元素）的计算机程序。在处理一张包含十只猫和十二只狗的照片时，程序识别出了八只狗。在被识别为狗的八个元素中，只有五个实际上是狗（true positives），而其他三个是猫（false positives）。有七只狗被漏识别（false negatives），七只猫被正确排除（true negatives）。那么该程序的

- 准确率=12/(10+12)（跟识别程序本身关系不大）
- 精确率=5/8（true positives/检索出的所有元素）
- 召回率=5/12（true positives/所有正确元素）
- F值=2\*[(5/18)\*(5/12)]/[(5/18)\+(5/12)]



##  locality-sensitive hashing (LSH)

LSH是一种用于通过将数据向量转换为哈希值来缩小搜索范围的方法，同时保留它们相似性的信息。

### LSH的构建

LSH有许多实现方式，这里介绍比较传统的实现。这个传统LSH的实现包含3个部分[^79]

1. Shingling：将原始文本编码为向量。

2. MinHashing：将向量转换为一种称为signature的特殊表示形式，用于比较它们之间的相似性。

3. LSH function：将signature哈希到不同的桶中。如果一对向量的signature至少有一次落入同一个桶中，它们就被认为是候选者。

#### Shingling

shingling是一种embedding方式（个人观点）。Shingling将自然语言识别成k个连续的token，重复的token将被剔除[^79]：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/20221fd8a0a449b2a394ee9db17789c2.png)

此时我们有了基于k-gram的token集，接下来就是转化为向量。

首先有一个全“0”向量，向量长度等于token集的长度。每个token所对应的向量的位置设置为1：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f3a4f1b44e904e31aa2f399228052ed1.png)

最终的结果就是，我们得到一个很长的向量，向量只有0、1元素，向量信息包含一条语句的语义。



#### MinHashing

由于向量维度实在太高，直接通过one-hot encoding出来的向量来计算近似距离的效果非常差。我们需要将稀疏向量转化为密集向量，这个过程在LSH中称为MinHashing ，转化后的向量称为MinHashing signature。

MinHashing在初学者最初认识它的时候会有点难理解，认识了以后你会发现它很简单。

> MinHashing is a hash function that permutes the components of an input vector and then returns the first index where the permutated vector component equals 1.



1. 首先打乱排序（permutation），将一个向量的元素重新排列
2. 返回排序后的向量的第一个1元素的index



例如以下例子：

u1向量(0,0,1,1,0)，第一次随机换行后，对应index是0，第二次随机换行后，对应的index是0[^80]，u1经过minhashing后的signature为(0,0)。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/24c78809d884458dadfa8ef9422e9f36.png)



实际上可以使用多个 minhash 值来近似求得向量之间的 Jaccard 相似度。实际上，使用的 minhash 值越多，近似就越准确。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f1e63f852fdd4247a31c1d56751b6602.png)

#### LSH function

即使我们将稀疏向量转化为了密集向量，但是密集向量的维度也会很高，直接检索的效率很低。

我们可以通过hash表来提升查询效率。但是要注意，如果直接用随机的hash算法容易将邻近的向量放在不同的hash桶中，我们应该找一个能将邻近的向量应该放在同一个hash桶中的hash算法，这就是LSH——局部敏感HASH算法。

> LSH mechanism builds a hash table consisting of several parts which puts a pair of signatures into the same bucket if they have at least one corresponding part.



局部敏感hash算法的理念也很简单，将signature分片，分片后的子signature各自计算hash值，子hash值碰撞的列为candidate。

以下例子比较容易理解自行阅读：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/5dc13f95660b40c2ac98240f7d08a123.png)

从极限角度思考，b=1就是不分片直接hash，完全没有发挥LSH的效果；b=signature的元素个数，每个元素一个分片，也就是每个元素一个hash值，可以做到比较精确的近似比较，但是对计算量和内存是极大的负担。



### LSH参数与错误率

一个向量的candidate向量概率直接影响召回率。candidate向量的概率如下，其中

- s代表近似度
- b代表分片数

- r代表每个分片中的行数

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/0fdf9f2f34b54f46b0de21ecb6f15a8f.png)



如果将P和s作为对比坐标，根据公式，其向量相似度和candidate概率关系如下：



![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/5be02245ffc340a293f61975c5372734.png)

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6b90e29c8c5e4d73b084d60e62d73c93.png)

分区个数b越大，candidate相似概率越小。

同时，调整b和s会影响P，P的值与FP、TN又有关系。

例如返回更多的candidates，这自然会导致更多的false postive——即返回不相似的“candidate pair”。这是修改参数 b 的不可避免的结果。

TP = True positive; FP = False positive; TN = True negative; FN = False negative

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/cb70b6a2a7f54ed7b5fdeef8394a7210.png)

LSH 容易受到高维数据的影响：更多的维度需要更长的signature长度和更多的计算才能保持良好的搜索质量。在这种情况下，建议使用其他索引。

### more

还有两篇没啃完，大概看了下跟binary vector和欧几里得距离相关

https://towardsdatascience.com/similarity-search-part-6-random-projections-with-lsh-forest-f2e9b31dcc47

https://towardsdatascience.com/similarity-search-part-7-lsh-compositions-1b2ae8239aca



## HNSW索引

HNSW算法（ Hierarchical navigable small world）是基于多图层的邻近算法。HNSW目前是最流行的向量索引算法之一。

从高层次来看，HNSW基于[小世界理论](https://en.wikipedia.org/wiki/Small-world_network)。小世界理论最初源于社会心理学的[六度分隔理论](https://en.wikipedia.org/wiki/Six_degrees_of_separation)，任何两个人之间仅仅通过五层社会关系就能够被相互连接，换句话说，就是任何两个地球上的人都可以通过最多六步社会联系来相互联系上。小世界理论后来被实验和使用经验佐证而被广泛接受，并扩展到非社会关系网络中。注意小世界理论是一种现象。

总之，小世界理论是在说明“*两个事务间的联系实际上是很短的*”。HNSW所做的事情就是建立两个元素之间的联系、减少联系数。

### HNSW索引的构建

看下HNSW论文关于构建HNSW图层的算法[^65]：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/00051012b0944d7aaea6406286dcae62.png)

构建算法中有几个要素是比较重要的：

- M为新增的边connection，表示新插入的节点的新边个数。
- Mmax为节点的边的最大个数。如果一直插入的是邻近节点，那么已有的邻近节点的边有可能一直增加，查找时会浪费算力。当新增节点时发现新插入边导致已有邻近节点的边大于Mmax，那么需要做shrink connection。
- efConstruction为邻近的节点集。



构建图例[^66]：



![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6d3214a05a904a959e1161954b1f4c01.png)

**HNSW添加node的步骤（without shrink connection）**：

1. 新节点插入时，先在最上层通过*efConstruction*找到邻近节点。以找到的最邻近节点当做entry point进入到下一层，然后继续通过那层的*efConstruction*查找邻近节点。
1. 在某层（如L=2)做node插入。从*efConstruction*中找出M个节点，并连接新节点，此时新增了1个节点并添加M条边与之相连。
1. 重复step2，直到最底层layer0。



### HNSW启发式邻居构建

基本的HNSW索引结构构建还存在一个问题，如果有两个簇相隔比较远，那么根据最具备的HNSW构建算法，两个簇几乎不可能相连，因为构建HNSW的基本算法是基于最邻近的节点*efConstruction*来构建的。

[HNSW original paper](https://arxiv.org/pdf/1603.09320)除了提出最基本的构建HNSW的算法，也提出了解决孤立群集的启发式算法：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/3cb77b0aad5b4f4b82af0e659a4c4c41.png)

> Fig.2 为两个孤立的群集选择图形邻居的启发式方法。一个新元素被插入到cluster 1的边界上。该元素的所有最近邻居都属于cluster 1，因此错过了群集之间的Delaunay三角网的边缘。然而，启发式方法选择了来自群集2的元素e2，因此，如果插入的元素与来自群集1的任何其他元素相比更靠近e2，就能保持全局连通性。

**“启发式算法不仅考虑了图中节点之间最近的距离，还考虑了图上不同区域之间的连通性”**

如下图添加一个X节点，这里就应该启用启发式算法，与cluster A建立连接性，而不是简单的添加到邻近节点上：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2577ce36301842d6beffc8999d190cdb.png)



### HNSW索引寻迹

[HNSW original paper](https://arxiv.org/pdf/1603.09320)中提到的HNSW寻找KNN的方法的主体逻辑包含如下2个算法： 

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/ba06d3bb151943cbb7b48d17b212a2f9.png)

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/bb81649a8fda432bae9d212d1f436a61.png)

- 算法2看上去要稍微复杂点，实际上逻辑很简单，算法2的功能是在该层中找到对于q最邻近的节点集ef。算法2简单来说就是把candidate节点加入到ef集并比较距离，把最远的节点剔出去，这样返回的W就是该层对于q的ef。

- 算法5的功能是返回q的K个最邻近节点，它会调用算法2两次（或多次）。for中的第一行输入参数ef=1，也就是说非最底层的layer都只找最邻近的ep（entry point），最底层（lc=0）返回K个最邻近节点集W。



![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e0cae2e33c7644fbac5de89bf31430b1.png)





### HNSW复杂度

HNSW的层数为log(N)函数。

查询复杂度：在Delaunay graph中可以严格评估复杂度，平均复杂度为 O(log(N))（非Delaunay graph，如存在启发式邻近算法的graph，论文没有给出具体的复杂度公式）。

构建复杂度：HNSW是通过所有元素迭代插入构建的，平均复杂度为O(N∙log(N)) 。



### HNSW索引参数

一般向量数据的HNSW索引都有几个参数可以调整，这些参数会影响索引构建速度、召回率等。不同的数据库参数可能稍微有差异，这里以pgvector的HNSW参数为例：

索引构建参数：

• m：向量的边的最大个数，默认为16。相当于论文中Mmax。
• ef_construction：构建索引时邻近列表中的向量个数，默认64。相当于论文中ef_construction。

索引搜索参数：

• hnsw.ef_search可调整搜索时邻近列表中的向量个数（也相当于论文中ef_construction）。必须大于等于limit。

构建索引时，调整ef_construction对创建时间和召回率的影响[^19]：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/90feb5f20dba44ffb5a655cfc7e4134b.png)



提高ef_construction这个参数会提高召回率，延长创建索引时间。ef_construction=256以后，索引构建时间明显增加，但召回率提升不明显。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/34c850bb9bc84b35b11ecba8ed2a5565.png)

提高m，同样会提高召回率，延长创建索引时间。m=36以后，索引构建时间明显增加，但召回率提升不明显。

同样的，提高hnsw.ef_search会提升召回率，降低性能。



## IVFFlat索引

IVFFlat索引全称为扁平压缩的倒排索引（ Inverted File with Flat Compression）（跟invert有啥关系？分不了类的索引都叫倒排？）。IVFFlat索引的核心理念基于voronoi diagram：

> Voronoi图的主要特性是：*从一个质心到其区域内任何点的距离都小于该点到另一个质心的距离*。

这个特性用公式表达：
$$
R_k=\{x \in X \,|\,d(x,P_k) \le d(x,P_j) \; \mathrm{for \,all }\,j \neq k\}
$$
Rk是质心，d(x,Pk)是质心到其区域内任何点的距离，d(x,Pj)是其他质心到该区域任何点的距离。



利用这个概念可以通过设置质心，把众多向量进行区域划分，然后利用voronio图的特性，粗略地找到邻近点。



### IVFFlat索引的构建

我们将高维空间下降到2维以便理解IVFFlat索引构建[^67]。

例如以下一堆x表示点（或者叫向量），假设我们有三个质心

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/9bbc3095f7934567a0a515ef71c7c1c0.png)

三个质心划分出3个Voronoi cell，所有点都分配到各自的Voronoi cell中

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a255c3e00b48407e946f0df49b62c8ff.png)



### IVFFlat索引寻迹

此时有一个query节点，然后计算它到所有质心的距离，然后找到最近的那个质心，这个质心所在的cell就是接下来要寻找的区域。最后在这个区域范围内，找到邻近的节点[^68]：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/0f28407a0dea42df84f1b03e3f1142a6.png)





边界问题：

上面的查询路径存在边界问题。当query靠近区域边界的时候，如果存在最近的节点在其他区域中，那么靠“仅搜索区域内邻近节点”的算法，是不会找到这个最邻近节点的。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/450eba0af56c43e8add41fdefa33a7cf.png)

边界问题本质是因为：

**voronio图只能保证某节点到其区域质心的距离小于到其他质心的距离，但不保证某节点到区域内其他节点的距离小于到其他区域节点的距离**。

这个问题可以通过设置搜索的区域数量来缓解，例如把搜索的区域数量从1增加到3：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/baa4edef667c4cbca33f5300c07d712f.png)

增加搜索区域数量一般都以参数形式设置在数据库中，例如pgvector中的ivfflat.probes。



**IVFFlat寻迹概述**：

1. 计算query节点到所有其他质心的距离，找到最近的那个
2. 根据查询cells个数的入参（例如probes），在top probes个cells中寻找邻近点



### IVFFlat索引参数

同样的，支持IVFFlat索引的向量数据库中，一般至少都有list和probe两个参数。这些参数会影响索引搜索性能、召回率。这里以Faiss的参数为例[^67]。

- nlist：构建区域的个数。增大nlist会提高搜索邻近质心的时间，但会减少区域内搜索节点的时间

- nprobe：搜索区域的个数。增大nprobe增大搜索区域个数，很明显会降低搜索性能，提升召回率



理论上来说，对于nlist，最好针对性的对向量数据的结构、数据库类型进行测试，不一定提高nlist总是会降低响应时间。对于nprobe，增大nprobe一定会降低搜索性能、提升召回率，但是nprobe太大没有意义，不符合ANN的初衷。

以下源自Pinecone对Faiss IVVFlat索引的性能测试：



![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/696c918a3a114cf9aa0e5fcc7f8d0f75.png)



## PQ乘积量化

100w个密集向量可能需要上G的内存，而真实世界中的向量远不止这个数。如果不加以管理，相似性向量搜索可能需要大量内存，然而内存RAM是有限的。向量的大小随向量的维度、向量个数而增加。

乘积量化（Product Quantization）旨在减少内存使用，还可以提升查询速度（因为计算量减少了）。PQ是一种有损压缩方法，这会导致向量检索的准确性降低，不过这在ANN需求中是行的。

PQ的算法逻辑稍微比其他算法复杂一些，强烈推荐这篇文章[Similarity Search, Part 2: Product Quantization](https://towardsdatascience.com/similarity-search-product-quantization-b2a1a6397701)[^50]

 ### PQ的构建

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1d77a56ec06845dab7e43db8e6ae8839.png)

步骤描述：

1. subvectors--将原始的高维向量切分成n个低维子向量
2. codebook--将n个*各自*所有的子向量通过k-means算法计算各自的voronoi图（或者其他算法），计算出来有n个不同的voronoi图，这些voronoi图就是codebook（这里假设每个voronoi图都有k个质心）
3. clustering--把n个子向量放进各自已聚类完成的voronoi图中计算出最近质心
4. quantized vectors--将这n个最近质心当做新的向量-量化后向量
5. reproduction values--以n个子空间的各自*最近质心的编号*为新值，合起来的新增称为PQ code



其中5.reproduction values详细描述：

根据n个子向量和每个子空间的中的k个质心，我们可以得到一个n*k的质心矩阵。取每个子向量的最近质心的编号，就是PQ code。

（btw：如果严谨一点，下图中所有元素的下角编号应该都是从1开始，而不是0）

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/580636211f084a69b2faf896d558af0f.png)



新的PQ code相当于原向量失真压缩后的新向量（reproduction value），新的距离计算可以直接计算PQ code的L2距离。



### PQ的检索

基于PQ original paper[^51]，PQ检索方式有两种：

- 对称方式：向量x，向量y之间的距离，近似通过他俩的质心q(x),q(y)的距离来代替。换言之，俩个向量的距离可通过两个向量的PQ code距离来代替
- 非对称方式：向量x，向量y之间的距离，近似通过x到质心q(y)的距离来替代。换言之，俩个向量的距离可通过查询向量原值和另一个向量的PQ code来计算



![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6ebf6e27c3b644e48ee83db5bae5c3c5.png)

很明显两个模式计算的距离准确度是不一样的：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e9136ffea3be4f3da5dfb6fa79dbf311.png)

上图表示8个子空间，256个质心，在不同模式下计算两个向量的距离准确度。可以看出非对称模式比对称模式的准确度要高。

在对比两个向量距离的时候，对称和非对称模式的距离计算模型是比较有用的。但在查找PQ近似向量的时候场景中，稍微有些差别，特别是对称模式计算失真会非常严重：

- 对称方式的查询速度是非常快的，因为code table已经通过PQ构建过程计算并保留了，只需要先通过code table把查询向量x的PQ code算出来（计算量极小），然后通过PQ code反向获得的code table中对应的那些子code table，这个子code table里的向量都是与其距离相等的近似向量。这种方式计算量极少，直接查表即可。

- 对称方式的失真情况相对比较严重（上面2个图还不是很好，把它想象成一个cell中包含多个向量的voronoi图，就知道对称方式失真有多严重了），非对称方式可以*稍微*缓解这个问题。非对称方式先计算向量x的PQ code，然后同样通过PQ code反向获得的code table中对应的那些子code table，然后通过向量x与这个子code table中的向量进行距离计算，得到KNN。它的计算量为n*km（n=子空间数，km≈所有向量数/质心数）。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/657e111f5e8a41a08ea6149afaf066ad.png)

非对称的需要通过PQ code找到质心，在质心所在的子空间中找KNN，查询向量x与已存在向量y的距离通过向量x与y的质心来近似替代。

PQ非对称方式检索[^50]：



![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/66282bc737594a3ea58fb6779b52e7e6.png)



![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2e6392bd7c254702b665ef9ad02e3a4b.png)

PQ非对称方式检索步骤：

1. 查询向量分成为多个子向量
2. 计算子向量和质心矩阵的距离
3. 取子空间中最近的质心，当做查询向量的PQ code
4. 通过查询向量和PQ code对应的质心来计算近似距离，距离可在各自子空间单独计算然后累加



我们前面提到非对称相对对称的近似距离计算要稍微好些，在某些场景下非对称的距离跟实际距离也可能产生较大的差距：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/56d63fdb1fb44fff835263c64a8f9824.png)

以上图比较好理解。在同一cell中，最远端的vector跟最靠近质心的vector跟质心的距离可能差别较大，只计算到质心距离的partial distance无法体现这种差别。



### PQ的参数及其影响

PQ至少有两个参数对性能和内存影响是比较大的，分别是子空间数m，子空间的质心数k。

召回率：

> The product quantizer is parametrized by the number of subvectors m and the number of quantizers per subvector k ∗ , producing a code of length m × log2 k

子空间有m个，每个子空间的质心数是k^* 个，产生一个PQ code的长度（bits单位）为[^51]：
$$
code \; length \, (bits)=m \cdot \log_2 k^*
$$


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e89014ce0ea548619c928b2d81a3037f.png)

子空间m越多，召回率越高；PQ code越长，召回率也越高。PQ code越长其实就是质心数越多。注意这里的具体值是基于论文的数据集的。

内存和复杂度：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b53bea324ad5414ca08b2a9dfd0cdf7a.png)

k代表聚类质心数，D代表维度，m代表子空间数。k\*代表子空间中的质心，D\*代表子空间中的维度。

例如k=2048，D=128，m=8，复杂度如下[^52]：

| Operation |            Memory and complexity             |
| :-------- | :------------------------------------------: |
| k-means   |            kD = 2048×128 = 262144            |
| PQ        | mk*D* = (k^(1/m))×D = (2048^(1/8))×128 = 332 |

可以看到PQ在搜索时复杂度明显的下降。

##  DiskAnn&Vamana

[DiskAnn original paper](https://suhasjs.github.io/files/diskann_neurips19.pdf) 的Abstract[^56]:

> Current state-of-the-art approximate nearest neighbor search (ANNS) algorithms generate indices that must be stored in main memory for fast high-recall search. This makes them expensive and limits the size of the dataset. We present a new graph-based indexing and search system called DiskANN that can index, store, and search a billion point database on a single workstation with just 64GB RAM and an inexpensive solid-state drive (SSD).



当前（论文发布于2019年）的ANN算法为了高召回率和性能都依赖RAM。这种方式不仅昂贵而且限制了数据集的大小。DiskANN只需要64GB RAM和不算贵的SSD。

### vamana的构建

Vamana 迭代构建一个有向图，首先从一个随机图开始，其中每个节点代表向量空间中的一个数据点。起初，图是高度连接的，所有节点都彼此相连。然后使用一个目标函数对图进行优化，该目标函数旨在最大化最接近节点之间的连接性。这通过修剪大多数随机短程边，同时添加某些长程边来实现，这些长程边连接相距较远的节点（以加速图中的遍历）[^56]

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/80a0a663dab34182b0953c48b8a720e5.png)

图中有200个二维点并进行了两次迭代。第一次迭代极大的修剪了边，但也把减少路径长度的长边修剪了；当调大阿尔法放开修剪条件时，长边会被增加回来[^57]。具体算法参考论文了，大致是这个意思。

### DiskAnn的算法

论文的The DiskANN Index Design:

> The high-level idea is simple: run Vamana on a dataset P and store the resulting graph on an SSD. At search time, whenever Algorithm 1 requires the out-neighbors of a point p, we simply fetch this information from the SSD. However, note that just storing the vector data for a billion points in 100 dimensions would far exceed the RAM on a workstation! This raises two questions: how do we build a graph over a billion points, and how do we do distance comparisons between the query point and points in our candidate list at search time in Algorithm 1, if we cannot even store the vector data?

在向量集合上运行vamana并存储在SSD上。在数据集非常大时，需要面临两个问题：

1. *如何在有限的内存资源下索引如此大规模的数据集？*

**k-means和vamana叠加算法**：首先，使用 k-means 将数据分成 k 个簇，然后将每个点分配到最近的 i 个簇。通常，i 的数值为 2 就足够了。为每个簇构建基于内存的 Vamana 索引，最后将 k 个 Vamana 索引合并为一个

2. *如果原始数据不能加载到内存中，搜索时如何计算距离？*

使用压缩向量（例如PQ）并将压缩向量存储在主存中。



如果将索引数据存储在 SSD 上，必须尽量减少磁盘访问次数和磁盘读写请求，以确保搜索延迟较低；同时失真的压缩会降低召回率。因此，DiskANN论文提出了三个优化策略：

1. **Beam Search**：简单来说，就是预加载邻居信息。在搜索点 p 时，如果 p 的邻居点不在内存中，则需要从磁盘加载。由于少量 SSD 随机访问操作所需的时间与一次 SSD 单扇区访问操作所需的时间大致相同，可以一次性加载 W 个未访问点的邻居信息。W 的设置不能太大也不能太小。W 设置过大会浪费计算资源和 SSD 带宽，而设置过小则会增加搜索延迟。
2. **Caching Frequently Visited Vertices**：旨在减少磁盘访问次数。在内存中缓存从起始点出发 C 个跳点范围内的所有点。C 的值最好设置在 3 到 4 之间。

3. **Implicit Re-Ranking Using Full-Precision Vectors**：因为PQ是失真压缩，基于PQ的距离算法只是邻近实际距离。为了消除这个差异，我们存储每个点到其邻近的所有距离，这个就是full-precision。至于实现原理，简单来说，也是利用了磁盘的加载效率问题。



依据论文，DiskANN的执行效率和召回率好于IVF和HNSW：



![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c21b76d77c744fa9bd6789ac67c7fb19.png)



# References

[^0.11]: [Harnessing the Power of LLMs in Practice: A Survey on ChatGPT and Beyond](https://arxiv.org/pdf/2304.13712)
[^0.1]: [一文讲清楚，AI、AGI、AIGC与AIGC、NLP、LLM，ChatGPT等概念](https://juejin.cn/post/7346233811212386345)
[^0.4]: [A Survey of Large Language Models](https://arxiv.org/pdf/2303.18223.pdf)
[^1]: https://en.wikipedia.org/wiki/Prompt_engineering
[^2]: [ RAG原始论文 ](https://arxiv.org/pdf/2005.11401)
[^3]: 劉智皓 (Chih-Hao Liu) [66個大型語言模型LLM經典論文](https://tomohiroliu22.medium.com/66%E5%80%8B%E5%A4%A7%E5%9E%8B%E8%AA%9E%E8%A8%80%E6%A8%A1%E5%9E%8Bllm%E7%B6%93%E5%85%B8%E8%AB%96%E6%96%87-0fcdab74e822)
[^x]: Jonathan Katz pgconfdev2024[Vectors: How to better support a nasty data type](https://www.pgevents.ca/events/pgconfdev2024/sessions/session/1/slides/42/pgconfdev-2024-vectors.pdf)
[^y]: [Vector databases (1): What makes each one different?](https://thedataquarry.com/posts/vector-db-1/)
[^yy]: [向量数据库性能对比](https://github.com/erikbern/ann-benchmarks)
[^10]: https://en.wikipedia.org/wiki/Vector_(mathematics_and_physics)
[^11]: https://en.wikipedia.org/wiki/Unit_vector
[^12]: OpenAI解释单位向量的使用 https://platform.openai.com/docs/guides/embeddings/frequently-asked-questions
[^13]: Pinecone Natural Language Processing for Semantic Search https://www.pinecone.io/learn/series/nlp/dense-vector-embeddings-nlp/
[^14]: 姚远 [漫谈数学中的各种空间](https://zhuanlan.zhihu.com/p/684643954)
[^15]: https://en.wikipedia.org/wiki/Euclidean_distance
[^16]: https://en.wikipedia.org/wiki/Taxicab_geometry
[^17]: https://en.wikipedia.org/wiki/Minkowski_distance
[^18]: https://en.wikipedia.org/wiki/Sine_and_cosine
[^19]: Jonathan Katz pgconfeu2023 [Vectors are the new JSON](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4592/slides/435/pgconfeu2023_vectors.pdf) 
[^20]: https://en.wikipedia.org/wiki/Jaccard_index
[^22]: https://en.wikipedia.org/wiki/Hamming_distance
[^23]: Vyacheslav Efimov [Similarity Search, Part 6: Random Projections with LSH Fores](https://towardsdatascience.com/similarity-search-part-6-random-projections-with-lsh-forest-f2e9b31dcc47) ↩
[^24]: earthwjl [Delaunay三角剖分学习笔记](https://www.jianshu.com/p/172749e6116a)
[^25]: https://en.wikipedia.org/wiki/Delaunay_triangulation
[^26]: https://en.wikipedia.org/wiki/Voronoi_diagram
[^40]: 冯若航 pigsty 非法加码 [向量数据库凉了吗？](https://mp.weixin.qq.com/s/0eBZ4zyX6XjBQO0GqlANnw)
[^50]: Vyacheslav Efimov [Similarity Search, Part 2: Product Quantization](https://towardsdatascience.com/similarity-search-product-quantization-b2a1a6397701)
[^51]: [PQ原始论文](https://inria.hal.science/file/index/docid/514462/filename/paper_hal.pdf)
[^52]: Pinecone Faiss Manual https://www.pinecone.io/learn/series/faiss/product-quantization/
[^55]: OpenAI推荐使用vector database https://openai.com/index/chatgpt-plugins/
[^56]: [DiskANN原始论文](https://suhasjs.github.io/files/diskann_neurips19.pdf)
[^57]: DiskANN, A Disk-based ANNS Solution with High Recall and High QPS on Billion-scale Dataset https://milvus.io/blog/2021-09-24-diskann.md
[^60]: https://en.wikipedia.org/wiki/Precision_and_recall
[^65]: [HNSW原始论文](https://arxiv.org/pdf/1603.09320)
[^66]: Vyacheslav Efimov [Similarity Search, Part 4: Hierarchical Navigable Small World (HNSW)](https://towardsdatascience.com/similarity-search-part-4-hierarchical-navigable-small-world-hnsw-2aad4fe87d37)
[^67]: https://www.pinecone.io/learn/series/faiss/vector-indexes/
[^68]: Vyacheslav Efimov [Similarity Search, Part 1: kNN & Inverted File Index](https://towardsdatascience.com/similarity-search-knn-inverted-file-index-7cab80cc0e79)
[^79]: Vyacheslav Efimov [Similarity Search, Part 5: Locality Sensitive Hashing (LSH)](https://towardsdatascience.com/similarity-search-part-5-locality-sensitive-hashing-lsh-76ae4b388203)
[^80]: 简书 [LSH（局部敏感哈希）算法](https://www.jianshu.com/p/d4368c8f40cb) 
