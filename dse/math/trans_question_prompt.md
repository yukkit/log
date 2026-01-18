请根据我给你的实体类和函数签名，帮我将这套试卷内容转成我的函数入参
这是我的函数签名：

```python
def create_question_set_from_raw(
    *,
    db: Session,
    project_id: uuid.UUID,
    class_id: Optional[uuid.UUID],
    subject_id: str,
    name: str,
    raw_questions: list[ItemNode],
    global_rubrics: Optional[Rubrics] = None,
) -> QuestionSet:
```

这是我的实体类

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

# 题目组，包含多个相关的 题目+用户答案 节点

class ItemNodeGroup(RootModel):
    root: List["ItemNode"]

# class NodeType(Enum)
# ROOT = "root"
# INTERNAL = "internal"
# TABLE = "table"
# LEAF = "leaf"

class ItemNode(BaseModel):
    """
    Represents a node in a hierarchical item structure.

    This class models a single item node that can contain child nodes or node groups,
    forming a tree-like structure. Each node has metadata, optional answer content,
    and can be indexed for retrieval.

    Attributes:
        id (str): Unique identifier for the item node.
        type (str): The type or category of the item node. root/internal/leaf/table
        title (str): The display title of the item node.
        index (str): Index value used for ordering or referencing the node.
        children (List[Union[ItemNode, ItemNodeGroup]]): List of child nodes or node groups.
            Defaults to an empty list.
        answer (Optional[list[Content]]): Optional list of content objects representing
            the user answer for this title(question). None indicates no answer is needed for this node.
        meta (Dict[str, Any]): Metadata dictionary containing additional information.
            Defaults to a dictionary with an empty 'tags' list.
    """

    id: str
    type: str
    title: str | list[Content]
    index: str
    children: List[Union["ItemNode", "ItemNodeGroup"]] = []
    answer: Optional[list[Content]] = None
    meta: Dict[str, Any] = {"tags": []}
```

注意：

- 如果试卷内容中含有 图片链接 请使用 UriContent
- ItemNode 中的 answer 不需要填
- title 中不要带 题号和分数
- 每个大题对应一个 ItemNode
- 大题下的每个小题是 child ItemNode 组成嵌套结构
- ItemNode 的 index 按照 题目的题号来
- 小题的 ItemNode 的 index 是 大题的题号 + '-' + 小题的题号
- 字符串请使用 r"

下面是参考示例：

```python
import uuid
from typing import List, Union

from app.core.entity import (
    Content,
    ItemNode,
    ItemNodeGroup,
    MediaType,
    TextContent,
    UriContent,
)


def txt(text: str) -> List[Content]:
    return [Content(inner=TextContent(text=text))]


def img_txt(
    text: str, uri: str, media_type: MediaType = MediaType.IMAGE_PNG
) -> List[Content]:
    return [
        Content(inner=TextContent(text=text)),
        Content(inner=UriContent(uri=uri, media_type=media_type)),
    ]


def node(
    index: str,
    title: Union[str, List[Content]],
    children: list[Union["ItemNode", "ItemNodeGroup"]] = [],
    node_type: str = "leaf",
) -> ItemNode:
    return ItemNode(
        id=str(uuid.uuid4()),
        type=node_type,
        title=title,
        index=index,
        children=children,
        answer=None,
    )


def group(
    children: list[ItemNode],
) -> ItemNodeGroup:
    return ItemNodeGroup(root=children)


question_set_id = "562f52dc-4a50-401d-b047-d7152f92effa"
class_id = "dse_math_2022_p1_v2"
name = "2022年香港中學文憑考試 數學 必修部分 試卷一"

# --- 构建 raw_questions 数据 ---
raw_questions = [
    # --- 甲部 (1) ---
    # Q1
    node("1", r"化簡 $\frac{(a^{3}b^{-2})^{4}}{a^{-5}b^{6}}$，並以正指數表示答案。"),
    # Q18
    node(
        "18",
        img_txt(
            r"圖2中，把三角形紙卡 $PQR$ 懸掛使得 $PQ$ 位於水平地面上。已知 $PQ=30 \text{ cm}$，$PR=25 \text{ cm}$ 及 $\angle QPR=95^{\circ}$。",
            "https://raw.githubusercontent.com/yukkit/log/main/dse/math/2022/images/p1/18.png",
        ),
        node_type="internal",
        children=[
            group(
                [
                    node(
                        "18-a",
                        "求",
                        node_type="internal",
                        children=[
                            group(
                                [
                                    node("18-a-i", "$QR$ 的長度，"),
                                    node("18-a-ii", r"$\angle PQR$。"),
                                ]
                            )
                        ],
                    ),
                    node(
                        "18-b",
                        r"設 $M$ 為 $QR$ 的中點。某工匠得知 $PR$ 與水平地面間的交角為 $70^{\circ}$。該工匠宣稱 $PM$ 與水平地面間的交角超過 $40^{\circ}$。該宣稱是否正確？試解釋你的答案。",
                    ),
                ]
            )
        ],
    ),
]

```

下面是试卷内容：
