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

下面是我的标准答案：
