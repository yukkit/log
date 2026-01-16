
请根据我给你的实体类和函数签名，帮我将这套学生作业答案转成我的函数入参

这是我的函数签名：

```python
def submit_student_answers(
    db: Session,
    student_id: uuid.UUID,
    question_set_id: uuid.UUID,
    submissions: list[
        AnswerNode
    ],  # 按照题目顺序排列，这里不知道question id，通过顺序匹配
) -> None:
```

这是我的实体类：

```python
class TextContent(BaseModel):
    text: str

class MediaType(str, Enum):
    IMAGE_PNG = "image/png"
    IMAGE_JPEG = "image/jpeg"
    IMAGE_WEBP = "image/webp"
    IMAGE_GIF = "image/gif"

class DataContent(BaseModel):
    # base64 encoded string
    payload: str
    media_type: MediaType

class UriContent(BaseModel):
    uri: str
    media_type: MediaType
    properties: dict[str, Any] = {}

# 用户答案内容，可以是文本内容或者数据内容（如图片等）

class Content(BaseModel):
    inner: Union[TextContent, DataContent, UriContent]
    annotations: list[str] = []

class AnswerNode(BaseModel):
    index: str
    answer: Optional[list[Content]] = None
    children: List["AnswerNode"] = []
```

注意：

- 如果试卷答案内容中含有 图片链接 请使用 UriContent
- 每个大题对应一个 AnswerNode
- 大题下的每个小题是 child ItemNode 组成嵌套结构
- AnswerNode 的 index 按照 题目的题号来
- 小题的 AnswerNode 的 index 是 大题的题号 + '-' + 小题的题号
- 字符串请使用 r"

下面是我的试卷内容：

在我给你试卷内容之前先不要作答
