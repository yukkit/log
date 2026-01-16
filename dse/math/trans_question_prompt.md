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

下面是答案：
