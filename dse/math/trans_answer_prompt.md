
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

下面是参考示例：

```python
import uuid
from typing import List, Optional
from app.core.entity import AnswerNode, Content, TextContent

# 假设 db Session 已由外部传入
question_set_id = "562f52dc-4a50-401d-b047-d7152f92effa"
student_id = "562f52dc-4a50-401d-b047-d7152f92effc"

# --- 辅助函数 ---


def create_text_answer(text: str) -> List[Content]:
    """创建包含单个 TextContent 的答案列表"""
    return [Content(inner=TextContent(text=text))]


def create_node(
    index: str, text: Optional[str] = None, children: List[AnswerNode] = []
) -> AnswerNode:
    """
    构建普通 AnswerNode
    - 优先构建 children (父节点)
    - 如果无 children 且有 text，则构建 answer (叶子节点)
    """
    answer_content = create_text_answer(text) if text is not None else None
    return AnswerNode(index=index, answer=answer_content, children=children or [])


def create_empty_node(index: str) -> AnswerNode:
    """构建未作答的 AnswerNode"""
    return AnswerNode(index=index, answer=[], children=[])


# --- 构建 submissions 数据 ---

submissions_data = [
    # 1
    create_node(
        index="1",
        text=r"$$ \frac{a^{12}b^{-8}}{a^{-5}b^{6}} = \frac{a^{17}}{b^{14}} $$",
    ),
    # 18 (修改为三层嵌套: 18 -> [18-a -> [18-a-i, 18-a-ii], 18-b])
    create_node(
        index="18",
        children=[
            create_node(
                index="18-a",
                children=[
                    create_node(
                        index="18-a-i",
                        text=r"""$$
QR = \sqrt{25^2+30^2-2(25)(30)\cos 95^\circ} \approx 40.7 \text{ cm}
$$""",
                    ),
                    create_node(
                        index="18-a-ii",
                        text=r"""$$
\frac{\sin \angle PQR}{25} = \frac{\sin 95^\circ}{40.7} \implies \angle PQR \approx 37.9^\circ
$$""",
                    ),
                ],
            ),
            create_node(
                index="18-b",
                text=r"""设 $R$ 在水平地面的投影为 $Z$
$RZ = 25 \sin 70^\circ \approx 23.49$
$M$ 在地面的投影为 $Y$, $MY = \frac{1}{2} RZ \approx 11.75$
$RM = \frac{1}{2} RQ \approx 20.35$
在 $\triangle PRM$ 中:
$PM = \sqrt{30^2+20.35^2-2(30)(20.35)\cos 37.9^\circ} \approx 18.67$
$$
\sin \angle MPY = \frac{11.75}{18.67} \implies \angle MPY \approx 38.99^\circ
$$
$38.99^\circ < 40^\circ$
不正确""",
            ),
        ],
    ),
    # 19 (两层: 19 -> a, b)
    create_node(
        index="19",
        children=[
            create_node(
                index="19-a",
                text=r"""$$
m_{AG} = \frac{12-112}{158-83} = \frac{-100}{75} = -\frac{4}{3}
$$
方程:
$y - 112 = -\frac{4}{3}(x-83)$
$3y - 336 = -4x + 332$
$4x + 3y - 668 = 0$""",
            ),
            create_node(
                index="19-b",
                text=r"""圆方程: $(x-83)^2 + (y-112)^2 = r^2$
过 $(23, 67) \implies r^2 = (23-83)^2 + (67-112)^2 = 3600 + 2025 = 5625$
$r=75$""",
            ),
            create_empty_node("19-c"),
            create_empty_node("19-d"),
        ],
    ),
]


```

下面是我的试卷内容：
