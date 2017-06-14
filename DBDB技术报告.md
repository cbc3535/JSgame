# DBDB: Dog Bed Database #
## Taavi Burns ##

*作为对策中最新的低音（有时是男高音），[Taavi](http://www.countermeasuremusic.com/)努力打破模具，有时只是忽略它的存在。通过职业生涯中的多样化工作，IBM（做C和Perl），FreshBooks（所有的东西），Points.com（做Python），现在在PagerDuty（做Scala）都是真的。除此之外 - 当他的Brompton折叠式自行车没有滑行时，你可能会发现他和他的儿子一起玩Minecraft，或者和他的妻子一起参加跑酷（或攀岩或其他冒险）。他编织大陆。*

## 介绍 ##

DBDB（狗床数据库）是一个实现一个简单的键/值数据库的Python库。它允许您将键与值相关联，并将该关联存储在磁盘上以供以后检索。

DBDB旨在保护面对计算机崩溃和错误状况的数据。它也避免了一次将所有数据保存在RAM中，以便您可以存储比RAM更多的数据。

## 记忆 ##

我记得我第一次真的陷入了一个错误。当我完成了BASIC程序的输入并运行它时，屏幕上出现了奇怪的闪烁像素，程序中止了。当我回去看代码时，程序的最后几行已经走了。

我妈妈的一个朋友知道如何编程，所以我们设置了一个电话。在与她说话的几分钟之内，我发现这个问题：程序太大了，已经侵入了视频内存。清除屏幕截断程序，并且闪光是Applesoft BASIC的存储程序状态在RAM中超出程序结束的行为的伪像。

从那时起，我关心内存分配。我学习了指针，以及如何使用malloc分配内存。我了解了我的数据结构在内存中的布局。我学会了如何改变他们非常非常小心。

几年之后，在阅读一个名为Erlang的面向过程的语言的同时，我了解到，实际上并没有将数据复制到进程之间发送消息，因为一切都是不可变的。然后，我在Clojure中发现了不可变数据结构，它真的开始沉入。

当我在2013年阅读关于CouchDB的时候，我只是微笑着点点头，认识到在变化时管理复杂数据的结构和机制。

我了解到，您可以设计围绕不可变数据构建的系统。

然后我同意写一本书章。

我认为描述CouchDB的核心数据存储概念（正如我所理解的）将是有趣的。

在尝试编写一个二叉树算法时，将树的位置变异，令我感到沮丧的是复杂的事情。边缘案件的数量，并试图推论一棵树的一部分变化如何影响他人，使我的头受伤。我不知道我将如何解释所有这一切。

记住经验教训，我仔细观察了递归算法来更新不可变二叉树，结果是比较简单。

我再次了解到，对于不改变的事情来说，更容易理解。

所以开始这个故事。

## 为什么有趣？ ##

大多数项目需要某种数据库。你真的不应该写自己的; 有许多边缘案例会咬你，即使你只是写JSON到磁盘：



- 如果您的文件系统空间不足，会发生什么？


- 如果您的笔记本电脑电池在保存时死机会怎么样


- 如果您的数据大小超过可用内存怎么办？（不太可能适用于现代台式机上的大多数应用程序，但移动设备或服务器端Web应用程序不太可能）
但是，如果你想了解一个数据库处理所有这些问题，写一个自己可以是一个好主意。



我们在这里讨论的技术和概念应适用于在遇到失败时需要合理，可预见的行为的任何问题。

说失败...

## 表征失败 ##

数据库的特征通常是它们与ACID属性的接近程度如何：原子性，一致性，隔离性和耐久性。

DBDB中的更新是原子和持久的，本章后面将介绍两个属性。DBDB不提供一致性保证，因为对存储的数据没有约束。隔离同样没有实现。

应用程序代码当然可以强加自己的一致性保证，但适当的隔离需要一个事务管理器。我们不会在这里试图 但是，您可以在[CircleDB](http://aosabook.org/en/500L/an-archaeology-inspired-database.html)一章中了解有关事务管理的更多信息。

我们还有其他系统维护问题需要考虑。在此实施中，不再收回过时的数据，因此重复更新（甚至相同的密钥）将最终消耗所有磁盘空间。（您将很快发现为什么会这样。）[PostgreSQL](https://www.postgresql.org/)称这个填充“吸尘”（这使旧行空间可用于重用），[CouchDB](http://couchdb.apache.org/)将其称为“压缩”（通过重写数据的“实时”部分进入一个新的文件，并原子地移动它在旧的）。

DBDB可以被增强以添加压缩功能，但作为读者的练习。

## DBDB架构 ##

DBDB将数据的逻辑结构（本示例中为二叉树，逻辑层）从数据的内容中分离出来，将“将其放在磁盘上的某个位置”（如何在文件中布置数据;物理层）键/值存储（键`a`到值的关联`foo`;公共API）。

许多数据库分离逻辑和物理方面，因为提供每个替代实现以获得不同的性能特征通常是有用的，例如DB2的SMS（文件系统中的文件）与DMS（原始块设备）表空间或MySQL的[替代引擎实现](http://dev.mysql.com/doc/refman/5.7/en/storage-engines.html)。


## 发现设计 ##


这本书的大部分章节都描述了一个程序是从一开始就完成的。然而，这并不是我们大多数人与我们正在开发的代码进行交互。我们最经常发现由他人编写的代码，并找出如何修改或扩展它来做某些不同的代码。

在本章中，我们假设DBDB是一个已完成的项目，并通过它来了解它是如何工作的。我们先来探讨整个项目的结构。


## 组织单位 ##


单位按照距离最终用户的距离排序; 也就是说，第一个模块是该程序的用户可能需要最了解的模块，而最后一个模块是他们应该有很少的交互。



- `tool.py` 定义了一个从终端窗口浏览数据库的命令行工具。



- `interface.py`定义了一个`DBDB`使用具体`BinaryTree`实现实现Python字典API 的class（）。这是在Python程序中使用DBDB的方式。



- `logical.py`定义逻辑层。这是一个键/值存储的抽象界面。





- -`LogicalBase`提供用于逻辑更新的API（如`get`，`set`和`commit`），并提供一个具体的子类来实现更新本身。它还管理存储锁定和取消引用内部节点。



- `ValueRef`是一个引用存储在数据库中的二进制`blob`的Python对象。间接使我们能够避免将整个数据存储一次性加载到内存中。



- `binary_tree.py` 定义了逻辑接口下的具体二进制树算法。



- `BinaryTree`提供二叉树的具体实现，具有获取，插入和删除键/值对的方法。`BinaryTree`代表一棵不变的树; 通过返回与旧共享共享结构的新树来执行更新。



- `BinaryNode` 在二叉树中实现一个节点。



- `BinaryNodeRef`是一个专门`ValueRef`知道如何序列化和反序列化的专业知识`BinaryNode`。



- `physical.py`定义物理层。本`Storage`类提供持久的，（大部分）仅追加记录存储。

这些模块从尝试给每个课程单一责任而增长。换句话说，每个班级应该只有一个理由要改变。

## 阅读价值 ##

我们将从最简单的情况开始：从数据库中读取一个值。让我们看看会发生什么，当我们试图获得与键关联的值`foo`在`example.db`：

这`main()`从模块运行的功能`dbdb.tool`：


	# dbdb/tool.py
	def main(argv):
	    if not (4 <= len(argv) <= 5):
	        usage()
	        return BAD_ARGS
	    dbname, verb, key, value = (argv[1:] + [None])[:4]
	    if verb not in {'get', 'set', 'delete'}:
	        usage()
	        return BAD_VERB
	    db = dbdb.connect(dbname)          # CONNECT
	    try:
	        if verb == 'get':
	            sys.stdout.write(db[key])  # GET VALUE
	        elif verb == 'set':
	            db[key] = value
	            db.commit()
	        else:
	            del db[key]
	            db.commit()
	    except KeyError:
	        print("Key not found", file=sys.stderr)
	        return BAD_KEY
	    return OK



该`connect()`函数打开数据库文件（可能创建它，但不会覆盖它）并返回一个实例`DBDB`：


	# dbdb/__init__.py
	def connect(dbname):
	    try:
	        f = open(dbname, 'r+b')
	    except IOError:
	        fd = os.open(dbname, os.O_RDWR | os.O_CREAT)
	        f = os.fdopen(fd, 'r+b')
	    return DBDB(f)


	# dbdb/interface.py
	class DBDB(object):
	
	    def __init__(self, f):
	        self._storage = Storage(f)
	        self._tree = BinaryTree(self._storage)


我们立即看到`DBDB`有一个实例的引用`Storage`，但它也与该参考共享`self._tree`。为什么？无法自行`self._tree`管理对存储的访问？

哪个对象“拥有”一个资源的问题在设计中通常是一个重要的问题，因为它提供了关于什么变化可能不安全的提示。让我们继续关注这个问题。

一旦我们有一个DBDB实例，key通过一个字典`lookup（db[key]）`来获取这个值，这会导致Python解释器的调用`DBDB.__getitem__()`。

	
	# dbdb/interface.py
	class DBDB(object):
	# ...
	    def __getitem__(self, key):
	        self._assert_not_closed()
	        return self._tree.get(key)
	
	    def _assert_not_closed(self):
	        if self._storage.closed:
	            raise ValueError('Database closed.')


`__getitem__()`确保数据库仍然通过调用打开`_assert_not_closed`。啊哈！在这里，我们看到至少有一个原因DBDB需要直接访问我们的`Storage`实例：因此它可以强制执行前提条件。（你同意这个设计吗？你能想出一个不同的方式，我们可以做到吗？）

DBDB然后通过调用检索与`key`内部关联的值，由以下提供：`_tree``_tree.get()``LogicalBase`


	# dbdb/logical.py
	class LogicalBase(object):
	# ...
	    def get(self, key):
	        if not self._storage.locked:
	            self._refresh_tree_ref()
	        return self._get(self._follow(self._tree_ref), key)


`get()`检查我们是否锁定了存储。我们不知道为什么在这里可能有一个锁，但是我们可以猜到它可能存在允许作者将序列化访问数据。如果存储没有锁定会怎么样？

	
	# dbdb/logical.py
	class LogicalBase(object):
	# ...
	def _refresh_tree_ref(self):
	        self._tree_ref = self.node_ref_class(
	            address=self._storage.get_root_address())


`_refresh_tree_ref `使用当前在磁盘上的内容重置树的“视图”，使我们能够执行完全最新的读取。

当我们尝试阅读时，如果存储被锁定怎么办？这意味着其他一些进程可能正在改变我们现在要阅读的数据; 我们的阅读不太可能是最新的数据的当前状态。这通常被称为“脏读”。这种模式允许许多读者访问数据，而不用担心阻塞，而不是稍微过时。

现在，我们来看看我们如何实际检索数据：


	# dbdb/binary_tree.py
	class BinaryTree(LogicalBase):
	# ...
	    def _get(self, node, key):
	        while node is not None:
	            if key < node.key:
	                node = self._follow(node.left_ref)
	            elif node.key < key:
	                node = self._follow(node.right_ref)
	            else:
	                return self._follow(node.value_ref)
	        raise KeyError


这是一个标准的二叉树搜索，以下是他们的节点。我们从阅读`BinaryTree`文档中知道`Nodes`和`NodeRef`s是有价值的对象：它们是不可变的，它们的内容永远不会改变。`Node`s创建与关联的键和值，以及左和右子项。那些协会也从不改变。`BinaryTree`当根节点被替换时，整个内容只会明显变化。这意味着在执行搜索时，我们不需要担心我们的树的内容被改变。

一旦相关的价值被发现，它被写入到`stdout`由`main()`不添加任何额外的新行，准确地保存用户的数据。

### 插入和更新 ###

现在，我们将设置键`foo`到值`bar`在`example.db`：


	$ python -m dbdb.tool example.db set foo bar

再次，这`main()`从模块运行功能`dbdb.tool`。由于我们之前已经看过这段代码，我们将重点介绍重要的部分：

	
	# dbdb/tool.py
	def main(argv):
	    ...
	    db = dbdb.connect(dbname)          # CONNECT
	    try:
	        ...
	        elif verb == 'set':
	            db[key] = value            # SET VALUE
	            db.commit()                # COMMIT
	        ...
	    except KeyError:
        ...


这一次我们设置了与`db[key] = value`哪个通话的值`DBDB.__setitem__()`。

	# dbdb/interface.py
	class DBDB(object):
	# ...
	    def __setitem__(self, key, value):
	        self._assert_not_closed()
	        return self._tree.set(key, value)

`__setitem__`确保数据库仍然打开，然后通过调用将关联从内部存储`key`到`value`内部。`_tree_tree.set()`

`_tree.set()`由...提供`LogicalBase`：



	# dbdb/logical.py
	class LogicalBase(object):
	# ...
	    def set(self, key, value):
	        if self._storage.lock():
	            self._refresh_tree_ref()
	        self._tree_ref = self._insert(
	            self._follow(self._tree_ref), key, self.value_ref_class(value))



`set() `首先检查存储锁：

	
	# dbdb/storage.py
	class Storage(object):
	    ...
	    def lock(self):
	        if not self.locked:
	            portalocker.lock(self._f, portalocker.LOCK_EX)
	            self.locked = True
	            return True
	        else:
	            return False


这里需要注意两点：



- 我们的锁由名为portalocker的第三方文件锁定库提供。


- `lock()``False`如果数据库已被锁定则返回，否则返回`True`。

回到`_tree.set()`现在，我们现在可以理解为什么它首先检查返回值`lock()`：它允许我们调用`_refresh_tree_ref`最新的根节点引用，所以我们不会丢失另一个进程自上次刷新树以来可能做出的更新磁盘。然后它使用包含插入（或更新）键/值的新树替换根树节点。

插入或更新树不会突变任何节点，因为`_insert()`返回一个新的树。新树与前一棵树共享不变的部分以节省内存和执行时间。递归地实现这一点很自然：


	# dbdb/binary_tree.py
	class BinaryTree(LogicalBase):
	# ...
	    def _insert(self, node, key, value_ref):
	        if node is None:
	            new_node = BinaryNode(
	                self.node_ref_class(), key, value_ref, self.node_ref_class(), 1)
	        elif key < node.key:
	            new_node = BinaryNode.from_node(
	                node,
	                left_ref=self._insert(
	                    self._follow(node.left_ref), key, value_ref))
	        elif node.key < key:
	            new_node = BinaryNode.from_node(
	                node,
	                right_ref=self._insert(
	                    self._follow(node.right_ref), key, value_ref))
	        else:
	            new_node = BinaryNode.from_node(node, value_ref=value_ref)
	        return self.node_ref_class(referent=new_node)


注意我们如何总是返回一个新的节点（包裹在一个`NodeRef`）。而不是更新一个节点来指向一个新的子树，我们创建一个新的节点，共享未更改的子树。这就是使这个二叉树成为不可变数据结构。

你可能已经注意到了一些奇怪的东西：我们还没有对磁盘上的任何东西做任何改变。我们所做的只是通过移动树节点来操纵我们对磁盘数据的视图。

实际上为了写这些更改磁盘，我们需要显式调用`commit()`，这是我们看到的是我们的第二部分`set`操作在`tool.py`本节的开始。

提交涉及写入内存中的所有脏状态，然后保存树的新根节点的磁盘地址。

从API开始：

	
	# dbdb/interface.py
	class DBDB(object):
	# ...
	    def commit(self):
	        self._assert_not_closed()
	        self._tree.commit()


执行`_tree.commit()`来自`LogicalBase`：
	
	# dbdb/logical.py
	class LogicalBase(object)
	# ...
	    def commit(self):
	        self._tree_ref.store(self._storage)
	        self._storage.commit_root_address(self._tree_ref.address)





所有人`NodeRef`都知道如何通过以下方式首先要求孩子们进行序列化`prepare_to_store()`：

	
	# dbdb/logical.py
	class ValueRef(object):
	# ...
	    def store(self, storage):
	        if self._referent is not None and not self._address:
	            self.prepare_to_store(storage)
	            self._address = storage.write(self.referent_to_string(self._referent))

`self._tree_ref`在这种情况下`LogicalBase`实际上是`BinaryNodeRef`（一个子类`ValueRef`），所以具体实现`prepare_to_store()`是：

	# dbdb/binary_tree.py
	class BinaryNodeRef(ValueRef):
	    def prepare_to_store(self, storage):
	        if self._referent:
	            self._referent.store_refs(storage)


`BinaryNode`问题，`_referent`，要求其裁判来存储自己：


	# dbdb/binary_tree.py
	class BinaryNode(object):
	# ...
	    def store_refs(self, storage):
	        self.value_ref.store(storage)
	        self.left_ref.store(storage)
	        self.right_ref.store(storage)


这一切都会一直下降到任何`NodeRef`具有不成文变化的内容（即否`_address`）。

现在我们回到了在堆栈`ValueRef`的`store`方法试。最后一步`store()`是序列化此节点并保存其存储地址：

	
	# dbdb/logical.py
	class ValueRef(object):
	# ...
	    def store(self, storage):
	        if self._referent is not None and not self._address:
	            self.prepare_to_store(storage)
	            self._address = storage.write(self.referent_to_string(self._referent))


在这一点上`NodeRef`，`_referent`我们保证有可用于所有自己的参考的地址，因此我们通过创建代表此节点的测试来进行序列化：




	# dbdb/binary_tree.py
	class BinaryNodeRef(ValueRef):
	# ...
	    @staticmethod
	    def referent_to_string(referent):
	        return pickle.dumps({
	            'left': referent.left_ref.address,
	            'key': referent.key,
	            'value': referent.value_ref.address,
	            'right': referent.right_ref.address,
	            'length': referent.length,
	        })


在`store()`方法中更新地址在技术上是一个突变`ValueRef`。因为它对用户可视值没有影响，我们可以认为它是不可变的。

一旦`store()`在根`_tree_ref`完成（`in LogicalBase.commit()`），我们知道所有的数据都写入磁盘。我们现在可以通过调用以下命令来提交根地址：


	# dbdb/physical.py
	class Storage(object):
	# ...
	    def commit_root_address(self, root_address):
	        self.lock()
	        self._f.flush()
	        self._seek_superblock()
	        self._write_integer(root_address)
	        self._f.flush()
	        self.unlock()


我们确保文件句柄被刷新（以便操作系统知道我们希望将所有数据保存到SSD等稳定存储器中），并写出根节点的地址。我们知道最后一次写入是原子的，因为我们将磁盘地址存储在扇区边界上。这是文件中的第一件事，所以无论扇区大小如何，这都是正确的，单扇区磁盘写入保证由磁盘硬件原子化。

因为根节点地址具有旧值或新值（从来没有一点点旧），所以其他进程可以从数据库中读取而不用锁定。一个外部过程可能会看到旧的或新的树，但从​​来不是两者的混合。这样，提交是原子的。

因为我们在写入根节点地址之前将新的数据写入磁盘并调用`fsync`syscall ，所以未提交的数据是不可达的。相反，一旦根节点地址被更新，我们知道它引用的所有数据也在磁盘上。以这种方式，承诺也是耐用的。

我们完成了


## NodeRefs如何保存内存 ##



为了避免同时将整个树结构保持在内存中，当从磁盘读取逻辑节点时，它的左和右子节点的磁盘地址（以及它的值）被加载到内存中。访问孩子及其值需要一个额外的函数调用`NodeRef.get()`来取消引用（“真正获取”）数据。

我们需要构建`NodeRef`一个地址是：

	
	+---------+
	| NodeRef |
	| ------- |
	| addr=3  |
	| get()   |
	+---------+

调用`get()`它将返回具体节点，连同该节点的引用为`NodeRef`s：


	+---------+     +---------+     +---------+
	| NodeRef |     | Node    |     | NodeRef |
	| ------- |     | ------- | +-> | ------- |
	| addr=3  |     | key=A   | |   | addr=1  |
	| get() ------> | value=B | |   +---------+
	+---------+     | left  ----+
	                | right ----+   +---------+
	                +---------+ |   | NodeRef |
	                            +-> | ------- |
	                                | addr=2  |
	                                +---------+



当树的更改未提交时，它们存在于内存中，该引用从根向下到更改的叶。更改不会保存到磁盘，所以更改的节点包含具体的键和值，没有磁盘地址。进行写入的过程可以看到未提交的更改，并且可以在发出提交之前进行更改，因为`NodeRef.get()`如果没有提交的值，则返回未提交的值; 在通过API访问时，承诺和未提交的数据之间没有区别。所有更新将以其他读者原子显示，因为在新的根节点地址写入磁盘之前，更改不可见。并发更新被磁盘上的锁定文件阻止。锁在第一次更新时获取，并在提交后释放。




## 读者练习 ##
DBDB允许许多进程一次读取同一个数据库而不阻塞; 权衡是读者有时可以检索陈旧的数据。如果我们需要能够一直读取一些数据怎么办？一个常见的用例是读取值，然后根据该值进行更新。你如何写一个方法`DBDB`来做到这一点？您需要采取什么权衡来提供此功能？

用于更新所述数据存储器，该算法可以通过替换字符串完全改变了`BinaryTree`在`interface.py`。数据存储往往使用更复杂类型的搜索树，例如B树，B +树等，以提高性能。虽然平衡的二叉树（而不是这个）需要执行这对于具有旋转盘的硬盘来说是非常昂贵的。SSD有助于延迟，但I / O的节省仍然存在。这对于具有旋转盘的硬盘来说是非常昂贵的。SSD有助于延迟，但I / O的节省仍然存在。*O(log2(n))O(log2(n))O(log32(n))O(log32(n))log2(232)=32log2(232)=32log32(232)≈6.4log32(232)≈6.4*

默认情况下，存储值将被`ValueRef`预期为字节值（直接传递到`Storage`）。二叉树节点本身只是一个子树`ValueRef`。通过[json](http://json.org/)或[msgpack](http://msgpack.org/)存储更丰富的数据是一个编写自己的数据并将其设置为`value_ref_class`。`BinaryNodeRef`是使用[pickle](https://docs.python.org/3.4/library/pickle.html)来串行数据的一个例子。

数据库压缩是另一个有趣的练习。压缩可以通过树中的中间遍历完成，随着你的移动，写出东西。如果树节点全部在一起，这可能是最好的，因为它们是遍历以查找任何数据。将尽可能多的中间节点包装到磁盘扇区中应提高读取性能，至少在压缩之后。这里有一些细微之处（例如，内存使用），如果您尝试自己实现。并记住：始终基准性能增强前后！你经常会对结果感到惊讶。


## 模式与原则 ##
测试接口，不实现。作为开发DBDB的一部分，我写了一些描述我想要使用它的测试。第一个测试针对数据库的内存版本进行运行，然后我将DBDB扩展到永久磁盘，甚至后来添加了NodeRefs的概念。大多数测试没有必要改变，这让我有信心，事情仍在继续。

尊重单一责任原则。课程最多只能有一个改变的原因。这不是DBDB的严格，但是有多种扩展途径，只需要本地化的更改。重新设计，因为我添加的功能是一种荣幸！


## 概要 ##

DBDB是一个简单的数据库，简单的保证，但事情仍然变得很复杂。我为管理这种复杂性所做的最重要的事情是用不可变数据结构实现一个表面上可变的对象。我鼓励你在下次发现自己处于一个棘手问题的中间时，考虑这种技术，似乎有比你追踪的更多的边缘案例。

----------
1. 奖励功能：您能保证压实的树结构是平衡的吗？这有助于在一段时间内保持性能。
2. 调用`fsync`文件描述符请求操作系统和硬盘驱动器（或SSD）立即写入所有缓冲的数据。操作系统和驱动器通常不会立即写入所有内容，以提高性能。