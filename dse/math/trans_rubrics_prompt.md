请根据我给你的实体类和函数签名，帮我将这套试卷的标准答案（评分标准）转成我的函数入参
这是我的函数签名：

```python
def batch_smart_update_rubrics(
    db: Session,
    question_set_id: QuestionSetId,
    updates: dict[QuestionNo, Rubrics],
) -> dict[QuestionId, str]:
```

这是我的实体类：

```python
QuestionNo = int

class EvaluationMode(Enum):
    RULE = "rule"
    LLM = "llm"
    UNKNOWN = "unknown"

class EvaluationType(Enum):
    # scoring only
    SCORE = "score"
    # feedback only
    FEEDBACK = "feedback"
    # both scoring and feedback
    BOTH = "both"

class MatchType(Enum):
    INDEX = "index"
    TAG = "tag"

class ScoreEvaluation(BaseModel):
    score_range: tuple[int, int]
    # 参考答案
    reference_answer: Optional[str] = None
    # 选择题中每个选项对应的分值
    # 解答题中每个步骤/知识点对应的分值
    criteria: dict[str, int]  # e.g., {"A": 1, "B": 0, "C": 0}

class FeedbackEvaluation(BaseModel):
    prompt_intent: str  # e.g., "主观题批阅需检查逻辑连贯性，如果字数少于20字需扣分。"

class Evaluation(BaseModel):
    score: Optional[ScoreEvaluation] = None
    feedback: Optional[FeedbackEvaluation] = None

    @model_validator(mode="after")
    def check_not_empty(self):
        if not self.score and not self.feedback:
            raise ValueError("Evaluation must contain score or feedback")
        return self

class Rubric(BaseModel):
    id: str
    match_type: MatchType
    scope: List[str] = []
    # If match_type is INDEX, item_index is required
    # Mapped to ItemNode.index
    item_index: Optional[str] = None
    evaluation_mode: EvaluationMode = EvaluationMode.UNKNOWN
    evaluation_type: EvaluationType
    evaluation: Evaluation

class Rubrics(RootModel):
    root: list[Rubric]
```

注意：

- EvaluationMode 可以都是 llm
- EvaluationType 都是 score
- MatchType 都是 index
- 每个大题对应唯一的 QuestionId
- 大题下的所有有答案的小题 对应一个 Rubric
- 一个大题对应一个 list[Rubric]
- Rubric 的 index 是 大题的题号 + '-' + 小题的题号，如果没有小题的话就是 大题的题号
- 字符串请使用 r"
- 注意答案中的 "解" 的内容完整对应 ScoreEvaluation 中的 reference_answer
- QuestionNo 是int类似的的大题的题号

下面是参考示例：

```python
from app.core.entity import (
    Evaluation,
    EvaluationMode,
    EvaluationType,
    MatchType,
    Rubric,
    ScoreEvaluation,
)
from app.entity import Rubrics

question_set_id = "562f52dc-4a50-401d-b047-d7152f92effa"

# -----------------------------------------------------------------------------
# 2. 直接构建 updates 变量
# -----------------------------------------------------------------------------
# 占位符 ID，实际使用时请替换为数据库中的真实 ID
updates = {
    # ==================== 第 1 题 ====================
    1: Rubrics(
        root=[
            Rubric(
                id="RUBRIC_ID_1",
                match_type=MatchType.INDEX,
                item_index="1",
                evaluation_mode=EvaluationMode.LLM,
                evaluation_type=EvaluationType.SCORE,
                evaluation=Evaluation(
                    score=ScoreEvaluation(
                        score_range=(0, 3),
                        reference_answer=r"""$$
\begin{aligned}
\frac{(a^{3}b^{-2})^{4}}{a^{-5}b^{6}} &= \frac{a^{12}b^{-8}}{a^{-5}b^{6}} \\
&= \frac{a^{12+5}}{b^{6+8}} \\
&= \frac{a^{17}}{b^{14}}
\end{aligned}
$$""",
                        criteria={
                            r"用於 $(m^h)^k = m^{hk}$ 或 $(mn)^k = m^k n^k$": 1,
                            r"用於 $\frac{x^p}{x^q} = x^{p-q}$ 或 $y^{-r} = \frac{1}{y^r}$": 1,
                            r"答案": 1,
                        },
                    )
                ),
            )
        ]
    ),
    # ==================== 第 17 题 ====================
    17: Rubrics(
        root=[
            Rubric(
                id="RUBRIC_ID_17_a",
                match_type=MatchType.INDEX,
                item_index="17-a",
                evaluation_mode=EvaluationMode.LLM,
                evaluation_type=EvaluationType.SCORE,
                evaluation=Evaluation(
                    score=ScoreEvaluation(
                        score_range=(0, 3),
                        reference_answer=r"""留意 $\alpha + \beta = -c$ 及 $\alpha\beta = -9$。
$$
\begin{aligned}
\alpha^2 + \beta^2 &= (\alpha+\beta)^2 - 2\alpha\beta \\
&= (-c)^2 - 2(-9) \\
&= c^2 + 18
\end{aligned}
$$""",
                        criteria={r"根與係數關係": 1, r"平方和公式": 1, r"答案": 1},
                    )
                ),
            ),
            Rubric(
                id="RUBRIC_ID_17_b",
                match_type=MatchType.INDEX,
                item_index="17-b",
                evaluation_mode=EvaluationMode.LLM,
                evaluation_type=EvaluationType.SCORE,
                evaluation=Evaluation(
                    score=ScoreEvaluation(
                        score_range=(0, 4),
                        reference_answer=r"""$$
\begin{aligned}
\alpha^2 + \beta^2 - c^2 &= 85 - (\alpha^2 + \beta^2) \\
c^2 + 18 - c^2 &= 85 - (c^2+18) \\
c^2 &= 49
\end{aligned}
$$
留意該數列的第 1 項及公差分別為 49 及 18。
$$
\begin{aligned}
\frac{n}{2}(2(49) + 18(n-1)) &> 2 \times 10^6 \\
9n^2 + 40n - 2 \times 10^6 &> 0
\end{aligned}
$$
$n < -473.6319808$ 或 $n > 469.1875364$
因此，$n$ 的最小值為 $470$。""",
                        criteria={
                            r"找出 $c^2$ 或首項/公差": 1,
                            r"建立不等式": 1,
                            r"解二次不等式": 1,
                            r"答案": 1,
                        },
                    )
                ),
            ),
        ]
    ),
    # ==================== 第 18 题 ====================
    18: Rubrics(
        root=[
            Rubric(
                id="RUBRIC_ID_18_a_i",
                match_type=MatchType.INDEX,
                item_index="18-a-i",
                evaluation_mode=EvaluationMode.LLM,
                evaluation_type=EvaluationType.SCORE,
                evaluation=Evaluation(
                    score=ScoreEvaluation(
                        score_range=(0, 2),
                        reference_answer=r"""$$
\begin{aligned}
QR^2 &= PQ^2 + PR^2 - 2(PQ)(PR)\cos \angle QPR \\
QR^2 &= 30^2 + 25^2 - 2(30)(25)\cos 95^{\circ} \\
QR &\approx 40.69070673 \\
QR &\approx 40.7 \text{ cm}
\end{aligned}
$$
因此，$QR$ 的長度為 $40.7 \text{ cm}$。""",
                        criteria={r"餘弦公式": 1, r"答案": 1},
                    )
                ),
            ),
            Rubric(
                id="RUBRIC_ID_18_a_ii",
                match_type=MatchType.INDEX,
                item_index="18-a-ii",
                evaluation_mode=EvaluationMode.LLM,
                evaluation_type=EvaluationType.SCORE,
                evaluation=Evaluation(
                    score=ScoreEvaluation(
                        score_range=(0, 2),
                        reference_answer=r"""$$
\begin{aligned}
\frac{\sin \angle PQR}{PR} &= \frac{\sin \angle QPR}{QR} \\
\frac{\sin \angle PQR}{25} &\approx \frac{\sin 95^{\circ}}{40.6907067} \\
\angle PQR &\approx 37.73809375^{\circ}
\end{aligned}
$$
因此，我們有 $\angle PQR \approx 37.7^{\circ}$。""",
                        criteria={r"正弦公式": 1, r"答案": 1},
                    )
                ),
            ),
            Rubric(
                id="RUBRIC_ID_18_b",
                match_type=MatchType.INDEX,
                item_index="18-b",
                evaluation_mode=EvaluationMode.LLM,
                evaluation_type=EvaluationType.SCORE,
                evaluation=Evaluation(
                    score=ScoreEvaluation(
                        score_range=(0, 3),
                        reference_answer=r"""$$
\begin{aligned}
PM^2 &= PQ^2 + QM^2 - 2(PQ)(QM)\cos \angle PQR \\
PM^2 &\approx 30^2 + (\frac{40.69070673}{2})^2 - 2(30)(\frac{40.69070673}{2})\cos 37.73809375^{\circ} \\
PM &\approx 18.6699383 \text{ cm}
\end{aligned}
$$
設 $D$ 及 $N$ 分別為 $R$ 及 $M$ 在水平地面的投影。
$$
\begin{aligned}
MN &= \frac{1}{2}RD \\
&= \frac{1}{2}PR \sin 70^{\circ} \\
&= \frac{1}{2}(25) \sin 70^{\circ} \\
&\approx 11.74615776 \text{ cm}
\end{aligned}
$$
留意 $PM$ 與水平地面的交角為 $\angle MPN$。
$$
\begin{aligned}
\sin \angle MPN &= \frac{MN}{PM} \\
\sin \angle MPN &\approx \frac{11.74615776}{18.66993831} \\
\angle MPN &\approx 38.98730493^{\circ} \\
\angle MPN &< 40^{\circ}
\end{aligned}
$$
因此，該宣稱不正確。""",
                        criteria={r"計算 $PM$": 1, r"計算交角或高度": 1, r"結論": 1},
                    )
                ),
            ),
        ]
    ),
    # ==================== 第 19 题 ====================
    19: Rubrics(
        root=[
            Rubric(
                id="RUBRIC_ID_19_a",
                match_type=MatchType.INDEX,
                item_index="19-a",
                evaluation_mode=EvaluationMode.LLM,
                evaluation_type=EvaluationType.SCORE,
                evaluation=Evaluation(
                    score=ScoreEvaluation(
                        score_range=(0, 2),
                        reference_answer=r"""$AG$ 的斜率
$$
\begin{aligned}
&= \frac{112-12}{83-158} \\
&= \frac{-4}{3}
\end{aligned}
$$
所求方程為
$$
\begin{aligned}
y - 12 &= \frac{-4}{3}(x - 158) \\
4x + 3y - 668 &= 0
\end{aligned}
$$""",
                        criteria={r"斜率": 1, r"方程": 1},
                    )
                ),
            ),
            Rubric(
                id="RUBRIC_ID_19_b",
                match_type=MatchType.INDEX,
                item_index="19-b",
                evaluation_mode=EvaluationMode.LLM,
                evaluation_type=EvaluationType.SCORE,
                evaluation=Evaluation(
                    score=ScoreEvaluation(
                        score_range=(0, 3),
                        reference_answer=r"""$C$ 的半徑為 $75$。
因 $83+75=158$，$AP$ 為鉛垂線或 $AQ$ 為鉛垂線。
所以，$P$ 的坐標或 $Q$ 的坐標為 $(158, 112)$。
留意 $\Delta AGP \cong \Delta AGQ$ 及 $AG \perp PQ$。
因此，$PQ$ 的斜率為 $\frac{3}{4}$。
$PQ$ 的方程為 $y - 112 = \frac{3}{4}(x - 158)$。
解 $y - 112 = \frac{3}{4}(x - 158)$ 及 $4x + 3y - 668 = 0$，
我們有 $x = 110$ 及 $y = 76$。
因此，$AG$ 與 $PQ$ 的交點的坐標為 $(110, 76)$。""",
                        criteria={r"求半徑": 1, r"求交點": 1, r"答案": 1},
                    )
                ),
            ),
            Rubric(
                id="RUBRIC_ID_19_c",
                match_type=MatchType.INDEX,
                item_index="19-c",
                evaluation_mode=EvaluationMode.LLM,
                evaluation_type=EvaluationType.SCORE,
                evaluation=Evaluation(
                    score=ScoreEvaluation(
                        score_range=(0, 4),
                        reference_answer=r"""設 $I$ 及 $r$ 分別為 $\Delta APQ$ 的內切圓的圓心及半徑。
留意 $AP=AQ$ 及 $I$ 位於 $AG$ 上。
$I$ 的 $x$ 坐標 $= 158 - r$
$I$ 的 $y$ 坐標 $= \frac{-4}{3}(158-r) + \frac{668}{3} = \frac{4r}{3} + 12$
所以，$I$ 的坐標為 $(158-r, \frac{4r}{3}+12)$。
$I$ 與 $AG$ 及 $PQ$ 的交點之間的距離為 $r$。
$$
\begin{aligned}
((158-r)-110)^2 + ((\frac{4r}{3}+12)-76)^2 &= r^2 \\
\frac{16}{9}r^2 - \frac{800}{3}r + 6400 &= 0 \\
r^2 - 150r + 3600 &= 0 \\
r = 30 \text{ 或 } r &= 120 \text{ (捨去)}
\end{aligned}
$$
因此，我們有 $r=30$。
由此，$I$ 的坐標為 $(128, 52)$。
故所求方程為 $(x-128)^2 + (y-52)^2 = 30^2$ (或 $x^2 + y^2 - 256x - 104y + 18188 = 0$)。""",
                        criteria={
                            r"$I$ 的坐標表達": 1,
                            r"建立方程": 1,
                            r"求出 $r$": 1,
                            r"圓方程": 1,
                        },
                    )
                ),
            ),
            Rubric(
                id="RUBRIC_ID_19_d",
                match_type=MatchType.INDEX,
                item_index="19-d",
                evaluation_mode=EvaluationMode.LLM,
                evaluation_type=EvaluationType.SCORE,
                evaluation=Evaluation(
                    score=ScoreEvaluation(
                        score_range=(0, 3),
                        reference_answer=r"""留意 $\angle APG = \angle AQG = 90^{\circ}$ 及 $\angle APG + \angle AQG = 180^{\circ}$。
所以，$APGQ$ 為圓內接四邊形，且 $AG$ 為 $\Delta APQ$ 的外接圓的直徑。
$\Delta APQ$ 的外接圓的半徑
$$
\begin{aligned}
&= \frac{1}{2}\sqrt{(83-158)^2 + (112-12)^2} \\
&= \frac{125}{2}
\end{aligned}
$$
由 (c)，$\Delta APQ$ 的內切圓的半徑為 $30$。
$\Delta APQ$ 的內切圓的面積與外接圓的面積之比
$$
\begin{aligned}
&= 30^2 : (\frac{125}{2})^2 \\
&= 144 : 625 \\
&\ne 1 : 4
\end{aligned}
$$
因此，該宣稱不同意。""",
                        criteria={r"外接圓半徑": 1, r"面積比": 1, r"結論": 1},
                    )
                ),
            ),
        ]
    ),
}

```

下面是我的标准答案：
