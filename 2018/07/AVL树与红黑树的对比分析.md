前面已经分别介绍了两种平衡二叉树：[AVL树](https://subetter.com/articles/2018/06/avl-tree.html) 和 [红黑树](https://subetter.com/articles/2018/06/rb-tree.html)，先让我们来简单回顾下。

AVL树，规定其任一结点下左右子树高度不超过1。

红黑树，规定其必须满足四个性质：

1. 每个结点要么是红的，要么是黑的；
2. 根结点是黑色的；
3. 如果一个结点是红色的，则它的两个孩子都是黑色的；
4. 对于任意一个结点，其到叶子结点的每条路径上都包含相同数目的黑色结点。

对比之下，我们发现：AVL树可以说是完全平衡的平衡二叉树，因为它的硬性规定就是左右子树高度差不超过1；而红黑树，更准确地说，它应该是"几于平衡"的平衡二叉树，在最坏情况下，红黑相间的路径长度是全黑路径长度的2倍。

有趣的是，某些底层数据结构（如Linux, STL ......）选用的都是红黑树而非AVL树，这是为何？

1. 对于AVL树，在插入或删除操作后，都要利用递归的回溯，去维护从被删结点到根结点这条路径上的所有结点的平衡性，回溯的量级是需要$O(logn)$的，其中插入操作最多需要两次旋转，删除操作可能是1次、2次或2次以上。而红黑树在insert_rebalance的时候最多需要2次旋转，在erase_rebalance的时候最多也只需要3次旋转。
2. 其次，AVL树的结构相较红黑树来说更为平衡，故在插入和删除结点时更容易引起不平衡。因此在大量数据需要插入或者删除时，AVL树需要rebalance的频率会更高，相比之下，红黑树的效率会更高。

另外，读者需要注意的是，红黑树的insert_rebalance操作也可能会有$O(logn)$量级的回溯，证明如下：

当程序进入`insert_rebalance()`的`while (x != root() && x->parent->color == red)`后，它只会有如下6种运行方式：

1. Case 1；
2. Case 1 ----> Case 1 ----> Case 1 ......；
3. Case 1 ----> ...... ----> Case 2 ----> Case 3；
4. Case 1 ----> ...... ----> Case 3；
5. Case 2 ----> Case 3；
6. Case 3；

而这回溯就发生在第2，3，4种，我们就以第2种的为例，如下图，

![](https://subetter.com/images/figures/20180708_01.png)

"结点1"为新插入结点，此时属于**Case 1：当前结点的父亲是红色，叔叔存在且也是红色**。那么我们的处理策略就是：

- 将 "父亲结点" 改为黑色；
- 将 "叔叔结点" 改为黑色；
- 将 "祖父结点" 改为红色；
- 将 "祖父结点" 设为 "当前结点"，继续进行操作。

但处理完后，根据代码`while (x != root() && x->parent->color == red)`，我们发现"当前结点"又进入了Case 1。假设每次处理完后都会进入Case 1，那么这样的处理操作会直到根结点才会结束。

erase_rebalance是否也存在同样的回溯呢？事实上，它并不存在。这很好证明。

当程序进入`while (x != root() && (x == nullptr || x->color == black))`后，它只会有如下6种运行方式：

1. Case 1 ----> Case 2；
2. Case 1 ----> Case 3 ----> Case 4；
3. Case 1 ----> Case 4;
4. Case 2；
5. Case 3 ----> Case 4；
6. Case 4;

因为4~6分别是1~3的后缀，所以我们只需考虑1~3即可。

读者可以自己脑补下1~3的运行流程就会发现，`while (x != root() && (x == nullptr || x->color == black))`语句只会被用到一次，就是最初进入程序的那次，之后便不再使用。

经过如上分析，我们可以对`insert_rebalance()`和`erase_rebalance()`作些微小的优化：

```c++
void RBTree::insert_rebalance(Node * x)
{
    x->color = red;

    while (x != root() && x->parent->color == red)
    {
        if (x->parent == x->parent->parent->left)
        {
            Node * y = x->parent->parent->right;

            if (y && y->color == red)          
            {
                x->parent->color = black;
                y->color = black;
                x->parent->parent->color = red;
                x = x->parent->parent;
            }
            else
            {
                if (x == x->parent->right)      
                {
                    x = x->parent;
                    rotate_left(x);
                }

                x->parent->color = black;      
                x->parent->parent->color = red;
                rotate_right(x->parent->parent);
                break;                            // add "break;"
            }
        }
        else
        {
            Node * y = x->parent->parent->left;

            if (y && y->color == red)
            {
                x->parent->color = black;
                y->color = black;
                x->parent->parent->color = red;
                x = x->parent->parent;
            }
            else
            {
                if (x == x->parent->left)
                {
                    x = x->parent;
                    rotate_right(x);
                }

                x->parent->color = black;
                x->parent->parent->color = red;
                rotate_left(x->parent->parent);
                break;                            // add "break;"
            }
        }
    }

    root()->color = black;
}

void RBTree::erase_rebalance(Node * z)
{
    ......
    ......
    ......

    if (y->color == black)
    {
        if (x != root() && (x == nullptr || x->color == black))               // "while" to "if"
        {
            if (x == x_parent->left)
            {
                Node * w = x_parent->right;

                if (w->color == red)
                {
                    w->color = black;
                    x_parent->color = red;
                    rotate_left(x_parent);
                    w = x_parent->right;
                }

                if ((w->left == nullptr || w->left->color == black) &&
                    (w->right == nullptr || w->right->color == black))
                {
                    w->color = red;
                    x = x_parent;
                    x_parent = x_parent->parent;
                }
                else
                {
                    if (w->right == nullptr || w->right->color == black)
                    {
                        if (w->left)
                            w->left->color = black;
                        w->color = red;
                        rotate_right(w);
                        w = x_parent->right;
                    }

                    w->color = x_parent->color;
                    x_parent->color = black;
                    if (w->right)
                        w->right->color = black;
                    rotate_left(x_parent);
                                                                              // delete "break;" 
                }
            }
            else
            {
                Node * w = x_parent->left;

                if (w->color == red)
                {
                    w->color = black;
                    x_parent->color = red;
                    rotate_right(x_parent);
                    w = x_parent->left;
                }

                if ((w->right == nullptr || w->right->color == black) &&
                    (w->left == nullptr || w->left->color == black))
                {
                    w->color = red;
                    x = x_parent;
                    x_parent = x_parent->parent;
                }
                else
                {
                    if (w->left == nullptr || w->left->color == black)
                    {
                        if (w->right)
                            w->right->color = black;
                        w->color = red;
                        rotate_left(w);
                        w = x_parent->left;
                    }

                    w->color = x_parent->color;
                    x_parent->color = black;
                    if (w->left)
                        w->left->color = black;
                    rotate_right(x_parent);
                                                                              // delete "break;"
                }
            }
        }

        if (x)
            x->color = black;
    }
}
```
