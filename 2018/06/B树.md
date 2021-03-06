## 一：背景

B-树（或者B树，英语：B-Tree）是一种自平衡的树，这种数据结构能够让查找数据、顺序访问、插入数据及删除的动作，都在对数时间$O(logn)$内完成。与其它一般化的二叉查找树不同，它可以拥有多余2个子结点。

B-树为系统大块数据的读写操作做了优化，减少定位记录时所经历的中间过程，从而加快存取速度。这种数据结构可以用来描述外部存储，因此常被应用在数据库和文件系统的实现上。

一棵B-树具有如下性质：


1. 所有的叶子结点在同一层；
2. 每棵B-树有一个Minimum Degree，称其为t；
3. 除了根结点，其余每个结点至少包含t-1个keys，根结点可以只包含1个key；
4. 每个结点（包括根结点）最多包含2t-1个keys；
5. 一个结点的孩子指针数等于这个结点的keys数+1；
6. 每个结点的keys都按升序排列；
7. 对于每个key，其左边孩子结点的所有keys都小于它，右边孩子结点的所有keys都大于它。

如下图，是一棵Minimum Degree为3（即t=3）的B-树：

![](https://subetter.com/images/figures/20180616_01.png)

## 二：算法过程与分析

首先大致看下程序的轮廓：

```c++
struct Node
{
    bool leaf;        // 是否是叶子结点
    int n;            // keys 的数目
    int * keys;       // 保存 keys
    Node ** siblings; // 保存孩子指针
    
    Node(int t, bool leaf)
    {
        this->leaf = leaf;
        this->n = 0;
        this->keys = new int[2 * t - 1];
        this->siblings = new Node*[2 * t]{ 0 };
    }
};

class BTree
{
private:
    int t; // Minimum Degree
    Node * root;
private:
    void destroy(Node * node);
    void split_child(Node * parent, int i, Node * child);
    void insert_non_full(Node * node, int key);
    bool find_real(Node * node, int key);
    void merge(Node * node, int i);
    void erase_real(Node * node, int key);
public:
    BTree(int t);
    ~BTree();
    void insert(int key);
    bool find(int key);
    void erase(int key);
    void print();
};
```

注意，以下讲解，默认Minimum Degree = 3。

### 2.1 插入操作

和普通二叉树的插入操作一样，新的值key最后会被插入到叶子结点中。我们不妨设x就是那个要被新插入key的叶子结点。试想，若x的keys数已达上限2t-1，现在再插入一个值，势必会违反B-树的" 性质4：每个结点（包括根结点）最多包含2t-1个keys "。那该怎么做呢？

把x一分为二，具体做法是：把x的中间的那个key移到x的父亲结点里，再申请一个新结点存放x后一半的keys和孩子结点（当然，叶子结点没有孩子，所以不需操作）。

但是这么做还是有个问题。如果x的父亲结点的keys也是2t-1呢？

插入伴随着查找，从根结点开始，向右或向下移动指针查找插入位置所在。我们可以提前判断要下降的那个结点的keys数是不是已达上限2t-1，如果是就对其进行一分为二处理。这么做不仅可以避免父亲结点的keys数越过上界，同时也顺带解决了x结点数越过上界的问题，一举两得！

下面我们举个例子来具体说明插入是如何完成的。

（1）

![](https://subetter.com/images/figures/20180616_02.png)

对于一棵空树，直接插入即可。

（2）

![](https://subetter.com/images/figures/20180616_03.png)

再插入20,30,40和50，此时根结点keys数正好达到上限5（由2t - 1 = 6 - 1 = 5得）。

（3）

![](https://subetter.com/images/figures/20180616_04.png)

再插入60的话，根结点的keys就会超过上限，那么我们的做法就是先把它一分为二，具体做法是：把中间的那个key移到上面，再申请一个新的结点，把后一半t-1个keys移到新结点里。参见图四的左部分；

接着再插入60。参见图四的右部分。

（4）

![](https://subetter.com/images/figures/20180616_05.png)

再插入70和80，此时根结点最右边孩子结点的keys数已达到上限。

（5）

![](https://subetter.com/images/figures/20180616_06.png)

再插入90的话，按照之前一分为二的思想，把中间的key，即60移到上面，再申请一个新结点来存储最后t-1个keys，最后插入90。

插入操作的代码如下：

```c++
/* 一分为二 */
void BTree::split_child(Node * parent, int i, Node * child)
{
    Node * z = new Node(t, child->leaf); // 用来存储最后一半的 keys 和孩子结点（如果有的话）

    // 移动最后一半的 keys 和孩子结点到 z 中 
    for (int j = 0; j < t - 1; j++)
        z->keys[j] = child->keys[j + t];
    if (!child->leaf)
    {
        for (int j = 0; j < t; j++)
            z->siblings[j] = child->siblings[j + t];
    }

    // 因为要把中间的那个 key 移进来，所以要给它腾个位置
    for (int j = parent->n; j >= i + 1; j--)
        parent->siblings[j + 1] = parent->siblings[j];
    for (int j = parent->n - 1; j >= i; j--)
        parent->keys[j + 1] = parent->keys[j];

    // 中间的那个 key 移到父亲结点里
    parent->keys[i] = child->keys[t - 1]; 

    z->n = t - 1;
    child->n = t - 1;
    parent->siblings[i + 1] = z;
    parent->n++;
}

void BTree::insert_non_full(Node * node, int key)
{
    int i = node->n - 1;

    if (node->leaf)
    {
        while (i >= 0 && node->keys[i] > key)
        {
            node->keys[i + 1] = node->keys[i];
            i--;
        }

        node->keys[i + 1] = key;
        node->n++;
    }
    else
    {
        while (i >= 0 && node->keys[i] > key)
            i--;

        // 每次下降的时候，都要检查下降的那个结点的 keys 数是不是等于上限 2t-1，如果是就要做一分为二处理
        if (node->siblings[i + 1]->n == 2 * t - 1)
        {
            split_child(node, i + 1, node->siblings[i + 1]);
            if (node->keys[i + 1] < key)
                i++;
        }

        insert_non_full(node->siblings[i + 1], key);
    }
}

void BTree::insert(int key)
{
    // 注意：程序并未对重复插入作去重处理

    if (!root)
    {
        root = new Node(t, true);
        root->keys[0] = key;
        root->n = 1;
    }
    else
    {
        if (root->n == 2 * t - 1)
        {
            Node * s = new Node(t, false);
            s->siblings[0] = root;
            split_child(s, 0, root);

            int i = 0;
            if (s->keys[0] < key)
                i++;

            insert_non_full(s->siblings[i], key);

            root = s;
        }
        else
            insert_non_full(root, key);
    }
}
```

### 2.2 查找操作

查找很简单，直接看代码即可。

```c++
bool BTree::find_real(Node * node, int key)
{
    int i = 0;
    while (i < node->n && node->keys[i] < key)
        i++;

    if (node->keys[i] == key)
        return true;

    if (node->leaf)
        return false;

    return find_real(node->siblings[i], key);
}

bool BTree::find(int key)
{
    if (root)
        return find_real(root, key);

    return false;
}
```

### 2.3 删除操作

假设我们要删除的值是key，在当前这棵B-树中它存在且位于结点x里，

若x不是叶子结点，分三种情况，我们约定key的左边的孩子结点为left，右边的孩子结点为right，

1. left的keys数大于t-1，那就在left中找到key的"前驱"，用"前驱"替换key，转而继续删除"前驱"；
2. right的keys数大于t-1，那就在right中找到key的"后继"，用"后继"替换key，转而继续删除"后继"；
3. left和right的keys数都等于t-1，那就把left，key和right合并为一个结点，继续在这个结点中删除key。

若x是叶子结点，分两种情况，

1. x结点的keys数不等于t-1，那么直接删除即可；

2. x结点的keys数恰好等于t-1，那么我们直接删除key的话，势必会使这棵B-树违反 " 性质3：除了根结点，其余每个结点至少包含t-1个keys，根结点可以只包含1个key "。那如何解决这个问题呢？和插入一样，删除也伴随着查找，从根结点开始，向右或向下移动指针查找key的位置所在。我们可以提前判断要下降的那个结点，如果它的keys数等于t-1，我们就给它填充，使其keys数大于t-1。具体做法是：如果它的左右兄弟结点中存在keys数大于t-1的，就从那里 " 拿一个 " key；如果都等于t-1，就把左右兄弟结点和夹在兄弟中间的那个key进行合并。

好，下面以几幅图来具体说明上述的步骤和做法：

![](https://subetter.com/images/figures/20180616_01.png)

**（1）图1中，删除2。**

从根节点a开始，要下降的结点为b，发现b的keys数为t-1，而它的右兄弟结点c的keys数也只有t-1，进行合并。合并后如下图7。

![](https://subetter.com/images/figures/20180616_07.png)

接着在结点b中找，要下降的结点为d，发现d的keys为t-1，而它的右兄弟e的keys是大于t-1的，那我们就从e里拿一个key：把3放进d中，4放进b中。此时b结点keys数不变，d变成3个，e少了一个，但依旧满足大于等于t-1。删除2后，如下图8。

![](https://subetter.com/images/figures/20180616_08.png)

如果要下降的那个结点的keys数等于t-1，我们就对其进行填充（或从兄弟里拿一个，或直接合并），使其keys数大于t-1。这样的做法**很有用处**，且也**必须**这么做。

为什么"必须"呢？

请看图1，假设我们不在下降的过程中进行填充操作了，现在如果要删除结点g中的13，问题出现了，g的keys数就会小于t-1，此时违反B-树的性质3。那怎么解决这个问题呢？从兄弟结点h中拿一个？不可以，h的keys数也是t-1。合并？所谓的合并就是把g，15和h合并为一个结点，可是c的keys数也是t-1，也行不通。因此，对要下降的那个keys数等于t-1的结点进行填充是必要的。

那又如何"很有用处"了呢？

观察"删除2"后，图1到图8的变化，整棵B-树的高度降低，高度降低意味着查找效率的提高。分析发现，它之所以会降低，是因为进行了合并操作。读者可以再仔细分析下会发现，这是降低B-树高度的唯一方式：根结点一个key，左右两个孩子结点的keys数都是t-1，合并成一个具有2t-1个keys的结点。

**（2）图8中，删除4。**

4存在于结点b中，且b不是叶子结点。观察4的左右孩子结点（d和e）的keys数，发现右孩子结点e的keys数是大于t-1的，因此就在e中找到4的后继（就是5），用5替换4，接着再在e中删除5。删除后，如下图9。

![](https://subetter.com/images/figures/20180616_09.png)

**（3）图9中，删除5。**

5存在于结点b中，且b不是叶子结点。观察5的左右孩子结点（d和e）的keys数，都是恰好等于t-1，那么进行合并操作：把结点d，5和结点e合并为一个结点，再在这个结点中删除5即可。删除后，如下图10。

![](https://subetter.com/images/figures/20180616_10.png)

最后看下删除操作的代码实现：

```c++
void BTree::merge(Node * node, int i)
{
    Node * cur = node->siblings[i];
    Node * next = node->siblings[i + 1];

    // 夹在中间的那个 key 移过来
    cur->keys[t - 1] = node->keys[i];

    // 再把兄弟结点里的所有信息合并过来
    for (int j = 0; j < next->n; j++)
        cur->keys[j + t] = next->keys[j];
    if (!cur->leaf)
    {
        for (int j = 0; j <= next->n; j++)
            cur->siblings[j + t] = next->siblings[j];
    }

    // 夹在中间的那个 key 被移走，造成位置空缺，所以就全部往前挪一步
    for (int j = i + 1; j < node->n; j++)
        node->keys[j - 1] = node->keys[j];
    for (int j = i + 2; j <= node->n; j++)
        node->siblings[j - 1] = node->siblings[j];

    cur->n += next->n + 1;
    node->n--;

    delete next;
}
void BTree::erase_real(Node * node, int key)
{
	int i = 0;
	while (i < node->n && node->keys[i] < key) // 找到第一个大于等于 key 的位置
		i++;

	if (i < node->n && node->keys[i] == key) // 如果在当前结点找到 key
	{
		if (node->leaf) // 当前结点是叶子的话，直接删除
		{
			for (int j = i + 1; j < node->n; j++)
				node->keys[j - 1] = node->keys[j];

			node->n--;
		}
		else // 当前结点不是叶子的话，分为三种情况
		{
			// 如果 key 的左边那个孩子结点 keys 数大于 t-1 ，就用 key 的前驱替换 key，问题转化为删除前驱
			if (node->siblings[i]->n > t - 1)
			{
				Node * left = node->siblings[i];

				while (!left->leaf)
					left = left->siblings[left->n];

				int precursor_key = left->keys[left->n - 1];
				node->keys[i] = precursor_key;

				erase_real(node->siblings[i], precursor_key);
			}
			// 如果 key 的右边那个孩子结点 keys 数大于 t-1 ，就用 key 的后继替换 key，问题转化为删除后继
			else if (node->siblings[i + 1]->n > t - 1)
			{
				Node * right = node->siblings[i + 1];

				while (!right->leaf)
					right = right->siblings[0];

				int successor_key = right->keys[0];
				node->keys[i] = successor_key;

				erase_real(node->siblings[i + 1], successor_key);
			}
			// 如果左右两个孩子结点 keys 都等于 t-1，那就进行合并操作，再删除
			else
			{
				merge(node, i);
				erase_real(node->siblings[i], key);
			}
		}
	}
	else // 如果未在当前结点找到 key
	{
		if (node->leaf) // 是叶子的话，说明根本不存在该 key
		{
			cout << "The key " << key << " is not existed !\n";
			return;
		}

		bool flag = (i == node->n) ? true : false;

		// 每次下降的时候，都要检查下降的那个结点的 keys 数是不是等于下限 t-1，如果是就要做填充处理，分为三种情况
		if (node->siblings[i]->n == t - 1)
		{
			if (i != 0 && node->siblings[i - 1]->n > t - 1) // 左兄弟结点 keys 数大于 t-1
			{
				Node * cur = node->siblings[i];
				Node * pre = node->siblings[i - 1];

				for (int j = cur->n - 1; j >= 0; j--)
					cur->keys[j + 1] = cur->keys[j];

				if (!cur->leaf)
				{
					for (int j = cur->n; j >= 0; j--)
						cur->siblings[j + 1] = cur->siblings[j];

					cur->siblings[0] = pre->siblings[pre->n];
				}

				cur->keys[0] = node->keys[i - 1];
				node->keys[i - 1] = pre->keys[pre->n - 1];
				cur->n++;
				pre->n--;
			}
			else if (i != node->n && node->siblings[i + 1]->n > t - 1) // 右兄弟结点 keys 数大于 t-1
			{
				Node * cur = node->siblings[i];
				Node * next = node->siblings[i + 1];

				cur->keys[cur->n] = node->keys[i];
				node->keys[i] = next->keys[0];

				for (int j = 1; j < next->n; j++)
					next->keys[j - 1] = next->keys[j];

				if (!next->leaf)
				{
					for (int j = 1; j < next->n; j++)
						next->siblings[j - 1] = next->siblings[j];

					cur->siblings[cur->n + 1] = next->siblings[0];
				}

				cur->n++;
				next->n--;
			}
			else // 如果左右兄弟结点 keys 数都等于 t-1，则对它们进行合并
			{
				if (i != node->n)
					merge(node, i);
				else
					merge(node, i - 1);
			}
		}

		if (flag && i > node->n)
			erase_real(node->siblings[i - 1], key);
		else
			erase_real(node->siblings[i], key);
	}
}

void BTree::erase(int key)
{
    if (!root)
        return;

    erase_real(root, key);

    if (root->n == 0)
    {
        Node * t = root;

        if (root->leaf)
            root = nullptr;
        else
            root = root->siblings[0];

        delete t;
    }
}
```

### 2.4 打印操作

以下实现为层次遍历的代码。

```c++
void BTree::print()
{
    if (root)
    {
        queue<Node*> Q;
        Q.push(root);

        while (!Q.empty())
        {
            Node * t = Q.front();
            Q.pop();

            for (int i = 0; i < t->n; i++)
                cout << t->keys[i] << " ";
            cout << endl;

            if (!t->leaf)
            {
                for (int i = 0; i < t->n + 1; i++)
                    Q.push(t->siblings[i]);
            }
        }

        cout << endl;
    }
}
```

## 三：完整代码

```c++
#include <iostream>
#include <queue>

using namespace std;

struct Node
{
	bool leaf;        // 是否是叶子结点
	int n;            // keys 的数目
	int * keys;       // 保存 keys
	Node ** siblings; // 保存孩子指针

	Node(int t, bool leaf)
	{
		this->leaf = leaf;
		this->n = 0;
		this->keys = new int[2 * t - 1];
		this->siblings = new Node*[2 * t]{ 0 };
	}
};

class BTree
{
private:
	int t; // Minimum Degree
	Node * root;
private:
	void destroy(Node * node);
	void split_child(Node * parent, int i, Node * child);
	void insert_non_full(Node * node, int key);
	bool find_real(Node * node, int key);
	void merge(Node * node, int i);
	void erase_real(Node * node, int key);
public:
	BTree(int t);
	~BTree();
	void insert(int key);
	bool find(int key);
	void erase(int key);
	void print();
};

int main()
{
	BTree btree(3);

	// test "insert"
	btree.insert(1);
	btree.insert(3);
	btree.insert(7);
	btree.insert(10);
	btree.insert(11);
	btree.insert(13);
	btree.insert(14);
	btree.insert(15);
	btree.insert(18);
	btree.insert(16);
	btree.insert(19);
	btree.insert(24);
	btree.insert(25);
	btree.insert(26);
	btree.insert(21);
	btree.insert(4);
	btree.insert(5);
	btree.insert(20);
	btree.insert(22);
	btree.insert(2);
	btree.insert(17);
	btree.insert(12);
	btree.insert(6);
	btree.print();

	// test "find"
	cout << ((btree.find(0) == true) ? 1 : -1) << endl;
	cout << ((btree.find(1) == true) ? 1 : -1) << endl;
	cout << ((btree.find(20) == true) ? 1 : -1) << endl << endl;

	// test "erase"
	btree.erase(6);
	btree.print();
	btree.erase(21);
	btree.print();
	btree.erase(20);
	btree.print();
	btree.erase(19);
	btree.print();
	btree.erase(22);
	btree.print();
	btree.erase(0);
	btree.erase(8);

	return 0;
}

void BTree::destroy(Node * node)
{
	if (node->siblings[0])
	{
		for (int i = 0; i <= node->n; i++)
			destroy(node->siblings[i]);
	}

	delete node;
}

/* 一分为二 */
void BTree::split_child(Node * parent, int i, Node * child)
{
	Node * z = new Node(t, child->leaf); // 用来存储最后一半的 keys 和孩子结点（如果有的话）
    
	// 移动最后一半的 keys 和孩子结点到 z 中 
	for (int j = 0; j < t - 1; j++)
		z->keys[j] = child->keys[j + t];
	if (!child->leaf)
	{
		for (int j = 0; j < t; j++)
			z->siblings[j] = child->siblings[j + t];
	}

	// 因为要把中间的那个 key 移进来，所以要给它腾个位置
	for (int j = parent->n; j >= i + 1; j--)
		parent->siblings[j + 1] = parent->siblings[j];
	for (int j = parent->n - 1; j >= i; j--)
		parent->keys[j + 1] = parent->keys[j];

	// 中间的那个 key 移到父亲结点里
	parent->keys[i] = child->keys[t - 1];

	z->n = t - 1;
	child->n = t - 1;
	parent->siblings[i + 1] = z;
	parent->n++;
}

void BTree::insert_non_full(Node * node, int key)
{
	int i = node->n - 1;

	if (node->leaf)
	{
		while (i >= 0 && node->keys[i] > key)
		{
			node->keys[i + 1] = node->keys[i];
			i--;
		}

		node->keys[i + 1] = key;
		node->n++;
	}
	else
	{
		while (i >= 0 && node->keys[i] > key)
			i--;

		// 每次下降的时候，都要检查下降的那个结点的 keys 数是不是等于上限 2t-1，如果是就要做一分为二处理
		if (node->siblings[i + 1]->n == 2 * t - 1)
		{
			split_child(node, i + 1, node->siblings[i + 1]);
			if (node->keys[i + 1] < key)
				i++;
		}

		insert_non_full(node->siblings[i + 1], key);
	}
}

bool BTree::find_real(Node * node, int key)
{
	int i = 0;
	while (i < node->n && node->keys[i] < key)
		i++;

	if (node->keys[i] == key)
		return true;

	if (node->leaf)
		return false;

	return find_real(node->siblings[i], key);
}

void BTree::merge(Node * node, int i)
{
	Node * cur = node->siblings[i];
	Node * next = node->siblings[i + 1];

	// 夹在中间的那个 key 移过来
	cur->keys[t - 1] = node->keys[i];

	// 再把兄弟结点里的所有信息合并过来
	for (int j = 0; j < next->n; j++)
		cur->keys[j + t] = next->keys[j];
	if (!cur->leaf)
	{
		for (int j = 0; j <= next->n; j++)
			cur->siblings[j + t] = next->siblings[j];
	}

	// 夹在中间的那个 key 被移走，造成位置空缺，所以就全部往前挪一步
	for (int j = i + 1; j < node->n; j++)
		node->keys[j - 1] = node->keys[j];
	for (int j = i + 2; j <= node->n; j++)
		node->siblings[j - 1] = node->siblings[j];

	cur->n += next->n + 1;
	node->n--;

	delete next;
}

void BTree::erase_real(Node * node, int key)
{
	int i = 0;
	while (i < node->n && node->keys[i] < key) // 找到第一个大于等于 key 的位置
		i++;

	if (i < node->n && node->keys[i] == key) // 如果在当前结点找到 key
	{
		if (node->leaf) // 当前结点是叶子的话，直接删除
		{
			for (int j = i + 1; j < node->n; j++)
				node->keys[j - 1] = node->keys[j];

			node->n--;
		}
		else // 当前结点不是叶子的话，分为三种情况
		{
			// 如果 key 的左边那个孩子结点 keys 数大于 t-1 ，就用 key 的前驱替换 key，问题转化为删除前驱
			if (node->siblings[i]->n > t - 1)
			{
				Node * left = node->siblings[i];

				while (!left->leaf)
					left = left->siblings[left->n];

				int precursor_key = left->keys[left->n - 1];
				node->keys[i] = precursor_key;

				erase_real(node->siblings[i], precursor_key);
			}
			// 如果 key 的右边那个孩子结点 keys 数大于 t-1 ，就用 key 的后继替换 key，问题转化为删除后继
			else if (node->siblings[i + 1]->n > t - 1)
			{
				Node * right = node->siblings[i + 1];

				while (!right->leaf)
					right = right->siblings[0];

				int successor_key = right->keys[0];
				node->keys[i] = successor_key;

				erase_real(node->siblings[i + 1], successor_key);
			}
			// 如果左右两个孩子结点 keys 都等于 t-1，那就进行合并操作，再删除
			else
			{
				merge(node, i);
				erase_real(node->siblings[i], key);
			}
		}
	}
	else // 如果未在当前结点找到 key
	{
		if (node->leaf) // 是叶子的话，说明根本不存在该 key
		{
			cout << "The key " << key << " is not existed !\n";
			return;
		}

		bool flag = (i == node->n) ? true : false;

		// 每次下降的时候，都要检查下降的那个结点的 keys 数是不是等于下限 t-1，如果是就要做填充处理，分为三种情况
		if (node->siblings[i]->n == t - 1)
		{
			if (i != 0 && node->siblings[i - 1]->n > t - 1) // 左兄弟结点 keys 数大于 t-1
			{
				Node * cur = node->siblings[i];
				Node * pre = node->siblings[i - 1];

				for (int j = cur->n - 1; j >= 0; j--)
					cur->keys[j + 1] = cur->keys[j];

				if (!cur->leaf)
				{
					for (int j = cur->n; j >= 0; j--)
						cur->siblings[j + 1] = cur->siblings[j];

					cur->siblings[0] = pre->siblings[pre->n];
				}

				cur->keys[0] = node->keys[i - 1];
				node->keys[i - 1] = pre->keys[pre->n - 1];
				cur->n++;
				pre->n--;
			}
			else if (i != node->n && node->siblings[i + 1]->n > t - 1) // 右兄弟结点 keys 数大于 t-1
			{
				Node * cur = node->siblings[i];
				Node * next = node->siblings[i + 1];

				cur->keys[cur->n] = node->keys[i];
				node->keys[i] = next->keys[0];

				for (int j = 1; j < next->n; j++)
					next->keys[j - 1] = next->keys[j];

				if (!next->leaf)
				{
					for (int j = 1; j < next->n; j++)
						next->siblings[j - 1] = next->siblings[j];

					cur->siblings[cur->n + 1] = next->siblings[0];
				}

				cur->n++;
				next->n--;
			}
			else // 如果左右兄弟结点 keys 数都等于 t-1，则对它们进行合并
			{
				if (i != node->n)
					merge(node, i);
				else
					merge(node, i - 1);
			}
		}

		if (flag && i > node->n)
			erase_real(node->siblings[i - 1], key);
		else
			erase_real(node->siblings[i], key);
	}
}

BTree::BTree(int t)
{
	this->t = t;
	this->root = nullptr;
}

BTree::~BTree()
{
	if (root)
		destroy(root);
}

void BTree::insert(int key)
{
	// 注意：程序并未对重复插入作去重处理

	if (!root)
	{
		root = new Node(t, true);
		root->keys[0] = key;
		root->n = 1;
	}
	else
	{
		if (root->n == 2 * t - 1)
		{
			Node * s = new Node(t, false);
			s->siblings[0] = root;
			split_child(s, 0, root);

			int i = 0;
			if (s->keys[0] < key)
				i++;

			insert_non_full(s->siblings[i], key);

			root = s;
		}
		else
			insert_non_full(root, key);
	}
}

bool BTree::find(int key)
{
	if (root)
		return find_real(root, key);

	return false;
}

void BTree::erase(int key)
{
	if (!root)
		return;

	erase_real(root, key);

	if (root->n == 0)
	{
		Node * t = root;

		if (root->leaf)
			root = nullptr;
		else
			root = root->siblings[0];

		delete t;
	}
}

void BTree::print()
{
	if (root)
	{
		queue<Node*> Q;
		Q.push(root);

		while (!Q.empty())
		{
			Node * t = Q.front();
			Q.pop();

			for (int i = 0; i < t->n; i++)
				cout << t->keys[i] << " ";
			cout << endl;

			if (!t->leaf)
			{
				for (int i = 0; i < t->n + 1; i++)
					Q.push(t->siblings[i]);
			}
		}

		cout << endl;
	}
}
```

运行如下：

```
16
3 7 13
20 24
1 2
4 5 6
10 11 12
14 15
17 18 19
21 22
25 26

-1
1
1

16
3 7 13
20 24
1 2
4 5
10 11 12
14 15
17 18 19
21 22
25 26

13
3 7
16 19 24
1 2
4 5
10 11 12
14 15
17 18
20 22
25 26

13
3 7
16 19
1 2
4 5
10 11 12
14 15
17 18
22 24 25 26

3 7 13 16 22
1 2
4 5
10 11 12
14 15
17 18
24 25 26

3 7 13 16 24
1 2
4 5
10 11 12
14 15
17 18
25 26

The key 0 is not existed !
The key 8 is not existed !
```

## 四：参考文献

- 维基百科. [B树](https://zh.wikipedia.org/wiki/B%E6%A0%91).
- GeeksforGeeks. [B-Tree | Set 1 (Introduction)](http://www.geeksforgeeks.org/b-tree-set-1-introduction-2/).
- GeeksforGeeks. [B-Tree | Set 2 (Insert)](http://www.geeksforgeeks.org/b-tree-set-1-insert-2/).
- GeeksforGeeks. [B-Tree | Set 3 (Delete)](http://www.geeksforgeeks.org/b-tree-set-3delete/).
