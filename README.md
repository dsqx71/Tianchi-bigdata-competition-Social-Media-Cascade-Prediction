##Social Media Cascade Prediction ：S1赛季报告

-----
@[作者：董煦，s1赛季：Rank12]

###[比赛介绍](http://tianchi.aliyun.com/competition/introduction.htm?spm=5176.100068.5678.1.fOqq0e&raceId=5)
###预处理
	1. 删掉同时满足：发出时间相同 & 相似度很高 & 同一作者（不用直接判定而是增加一些能够表现出是无效微博的特征就行）
    2. 对连续变量做放缩（有助于提升线性模型的能力）
    3. 用正则表达式把符号特征提取出来
    4. 英文变小写，并删掉中英文的停用词 
    5. 为了让变量连续，把1月改为13月 
###基础特征
	uid,pid,time,share,comment,zan raw_corpus,cleaned&segment,链接，'/@,@,#,【，《，['
###用户特征
    1. 用户总的点赞、分享、评论数量的统计量 
    2. 有效微博数量 +  无效的微博数量 + 总微博数量 + 出现在训练集中的数量
    3. 微博的长度的统计值
    5. 发微博的平均周期（若只有一条微博，则设为7个月）
    6. 周一到周日发出有效微博的数量/频率/频率的方差
    7. 用户微博的平均主题分布
    8. 用户出现在训练集的次数
###缺失值处理
	1.许多std的缺失值暂时 -1 来替代（因为-1能够体现出这条样本是没有std的）
	2.用户没有出现在训练集中，所以有关share，comment，zan的统计量都设为0（两类测试集分开预测,所以这里设定的值不重要，因为在这一类训练集中不使用上述特征）
###微博特征
    1. 原始文本长度 + 清理后的文本长度
    2. 特殊字符： [r'http[0-9a-zA-Z?:=._@%/\-#&\+|]+' ,r'//@',   r'@' ,  r'#' ,  r'【' ,r'《' ,r'\[' ]
    3. 提取 tf-idf 
    4. 人工抽取有区分度的词
    5. 离最近一次有效微博的时间间隔（若没有，则设为7个月）
    6. 最近n次有效微博的转发/评论/点赞 数量（若没有，则设为0，n待定）
    7. 星期几
    8. 与最近n次有效微博的文本相似度
    9. 与其他用户的微博相似度
        
###模型类特征
    1. LDA主题
    2. 领袖识别：设定一个阈值，在训练集中高于它的设定为意见领袖，把训练集拆分为小的训练集和验证集
    （把阈值也作为一个可调参数，通过验证集来选，可以设定多个阈值），把预测的结果作为特征。
    3. 情感极性作为特征	
    4. 高斯混合模型：对用户聚类
    5. 协同过滤（主题-微博矩阵

### 多次预测（样本倾斜）
	1.测试集分为两类：用户在训练集出现过+没有出现，两类分开预测，前者可以多用一些特殊的特征，比如share，comment，zan的统计量等，后者不能直接出现这些特征。两类模型分开训练
	（因为两者的特征重要度不同，前者一定很偏重训练集中的share，comment，zan的统计量，
	后者因为没有在训练集中出现，更加依赖文本特征，聚类特征等，这类特征间接使用了训练集中的share，comment，zan的统计量
	分为两对分别训练，保证了训练集和测试集是一致的） 
    2.分类，预测是否为0 //因为多数人的转发量为0,搞定这些就能获得很大一部分的得分
    3.分类预测不为0的，回归再预测 //清理掉很有可能不被关注的微博，只对有可能的进行回归分析

###验证集方案（用榜单上的结果对比线下的结果，调整权重）
 - 选择一定时间以后的数据（缺点：训练集数据会相对变少）
 - 在所有样本中随机选择一部分作为5-折交叉验证（缺点：没有考虑到时间序列之间的关系）
 - 所有用户选择一些时间靠后的样本作为验证集，如果某用户只有一条，并且在后两月，则放在验证集上（缺点：有些用户只有一条微博）
 - 使用bagging方法的oob（缺点：结果太悲观，不能反映真正的误差）
 - 前几个月一定放在训练集，后几个月随机抽取一部分作为训练集，另一部分为验证集（3折）
 - 加权结合上述方法

###发现

	1 分类和回归的特征重要度几乎不同，可能说明
	2 有些数据在训练集中完全没有出现的。比如：月份，用户
	3 有许多用户没有出现在训练集中
	4 没有考虑题目的意义：预测一条文博在7天内的互动
	5 寻找影响优质内容的因素
	6 发了微博之后新的微博会置顶把之前的顶下去，所以把7天之内出现其他微博的次数作为特征
	7 微博之间的内容有连续性
	8  应当考虑转发上升的速度和加速度 
	9 用户之前没有发过任何微博作为特征
	10 7天之内的微博内容绝对有影响
    11 没有给图片信息，关注网络，平台的内容推广策略，这个些都会是强噪声
    12 预测集中有24818个用户，但是1214个用户都不在测试集中的。
    13增加comment，share，zan 高的权重


###感悟
- 每次做特征的时候，一定要进行自动化检验，比如样本数量，缺失值比例等等，用python的装饰器搞定
- 如果数据很大，通常先降采样，快速特征抽取代码。
- 抽取不同的特征时，可以放在不同的函数中，做到低耦合，然后分配进程池，做并行计算。
- 不要使用pandas做特征的管理，它比numpy的开销大，而且numba等加速工具不直接支持
- 计算力不足时，可以不做grid search，而是随意指定几个典型的值，供之后的ensemble再加工
- 多做profiling，避免浪费时间等待
- 尽量向量化，否则请考虑 numpy 下的take，pandas 下的请使用irow ，icol
- 如果要用线性模型，则dummy code 和合理的缺失值
- 可以通过残差图，不同类别对应特征的分布图来决定离群点的
- 对于离群点不用直接删掉，而是降低权重
- 对于特征选择：可以选择训练集和预测集特征分布一致的特征，可以用这种特征分布的相似性来决定bagging抽取特征的概率（kl散度来衡量）
