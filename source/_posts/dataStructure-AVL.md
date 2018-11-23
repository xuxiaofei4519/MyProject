---
title: 手写平衡二叉树(AVL)
date: 2018-11-23 17:12:09
copyright: true
tags:
 - 数据结构
categories:
 - 数据结构
---


{% cq %} 
最近看到纯洁的微笑在微信公众号分享了一篇平衡二叉树（AVL）的博客，正好之前想要学习这一块，因此就一遍学习一遍手写平衡二叉树，在注释中加入自己的理解，学习过程中发现这篇博客实现的二叉树有问题，查阅相关资料，最终手写成功。
编程之道：光靠脑子编程只能是别人的编程，只有自己手动实现的才叫自己的编程。
{% endcq %}
<!-- more -->

* * *

>以下代码纯手动实现，如有问题，及时留言。

#### 手动实现源码
```java
/**
 * 平衡二叉树 AVL实现
 */
public class AVLBalanceTree {

    //根节点
    private AVLNode root;
    //节点数
    private Integer size;

    public AVLBalanceTree() {
    }

    public AVLBalanceTree(AVLNode root) {
        this.root = root;
    }


    /**
     * 节点类
     */
    @Data
    private static class AVLNode{
        //节点值
        private Integer value;
        //节点高度
        private Integer height;
        //左子节点
        private AVLNode leftNode;
        //右子节点
        private AVLNode rightNode;

        public AVLNode(Integer value) {
            //新节点的高度默认是0
            new AVLNode(value,0,null,null);
        }

        public AVLNode(Integer value, Integer height, AVLNode leftNode, AVLNode rightNode) {
            this.value = value;
            this.height = height;
            this.leftNode = leftNode;
            this.rightNode = rightNode;
        }

    }

    /**
     * 插入操作
     * @param value  插入值为Integer类型
     */
    public void insert(Integer value) throws Exception {
        if (null == this.root){
            initRoot(value);
            size++;
            return;
        }
        if (!constains(value))
            throw new Exception("该值已经存在");


    }

    /**
     * 初始化根节点
     */
    private void initRoot(Integer value){
        this.root = new AVLNode(value);
        System.out.println("root:" + this.root.getValue());
    }


    /**
     * 查询是否包含该val
     * @param val
     * @return boolean值
     */
    private boolean constains(Integer val){
        AVLNode currentNode = root;
        if (null == currentNode)
            return false;

        while (null != currentNode){
            if (val > currentNode.getValue()){
                currentNode = currentNode.getRightNode();
            } else if (val < currentNode.getValue()){
                currentNode = currentNode.getLeftNode();
            } else {
                return true;
            }
        }

        return false;
    }

    /**
     * 插入操作
     */
    private AVLNode insert(AVLNode parent,Integer val){
        if (parent == null){
            return createSimpleNode(val);
        }

        if (val < parent.getValue()){
            parent.setLeftNode(insert(parent.getLeftNode(), val));

            //在每一个递归层级上都要进行节点的平衡判断
            // 左子节点添加节点，左子节点高度减去右子节点高度来判断是否平衡。仔细思考不可能出现负值
            if (height(parent.getLeftNode()) - height(parent.getRightNode()) > 1){
                //使用parent的左节点与val进行比较，从而判断val被放在了parent左节点的左边还是右边，根据位置不同进行不同的旋转。
                Integer compareVal = parent.getLeftNode().getValue();


                if (val < compareVal){ // val相当于第三个节点并且比parent的左节点(第二个节点)小的话就会进行左左旋转
                    leftLeftRotate(parent);
                }else { // val相当于第三个节点并且比parent左节点(第二个节点)大的话就会进行左右旋转
                    leftRightRotate(parent);
                }
            }
        }
        if (val > parent.getValue()){
            parent.setRightNode(insert(parent.getRightNode(), val));
            // 右子节点添加节点，右子节点高度减去左子节点高度来判断是否平衡。仔细思考不可能出现负值
            if (height(parent.getRightNode()) - height(parent.getLeftNode()) > 1){
                //使用parent的左节点与val进行比较，从而判断val被放在了parent左节点的左边还是右边，根据位置不同进行不同的旋转。
                Integer compareVal = parent.getRightNode().getValue();


                if (val > compareVal){  // val相当于第三个节点并且比parent的左节点(第二个节点)小的话就会进行左左旋转
                    rightRightRotate(parent);
                }else { // val相当于第三个节点并且比parent左节点(第二个节点)大的话就会进行左右旋转
                    rightLeftRotate(parent);
                }
            }
        }

        /**
         * 插入成功以后都要递归将parent的节点高度加1
         */
        parent.setHeight(maxHeight(parent.getLeftNode(), parent.getRightNode()) + 1);
        return parent;
    }

    public AVLNode remove(AVLNode parent, Integer val){
        if (val < parent.getValue()){
            //递归左节点
            parent.setLeftNode(remove(parent.getLeftNode(), val));
            //重新计算当前节点高度
            parent.setHeight(maxHeight(parent.getLeftNode(), parent.getRightNode()) + 1);
            //只有左节点的高度低于右节点的高度的时候，删除左节点会导致不平衡。左节点比右节点最多高1，因此不可能出现负值。
            if (height(parent.getRightNode()) - height(parent.getLeftNode()) > 1){
                //进入到该if条件说明删除左节点后，右节点高度变高。parent节点失衡

                //获取parent的右节点
                AVLNode tempNode = parent.getRightNode();
                if (height(tempNode.getLeftNode()) > height(tempNode.getRightNode())){
                    rightLeftRotate(parent);
                }else {
                    rightRightRotate(parent);
                }
            }
            //因为是删除左节点，调整右节点因此高度不用++
        }else if (val > parent.getValue()){
            //遍历右子树
            parent.setRightNode(remove(parent.getRightNode(), val));
            //重新计算当前节点高度
            parent.setHeight(maxHeight(parent.getLeftNode(), parent.getRightNode()) + 1);
            //右节点
            if (height(parent.getLeftNode()) - height(parent.getRightNode()) > 1){
                //进入到该if条件说明删除右节点后，左节点高度变高。parent节点失衡

                //获取parent的左节点
                AVLNode tempNode = parent.getLeftNode();
                if (height(tempNode.getRightNode()) > height(tempNode.getLeftNode())){
                    leftRightRotate(parent);
                }else {
                    leftLeftRotate(parent);
                }
            }
            //因为是删除右节点，调整左节点因此高度不用++
        }else {
            //找到该节点，匹配成功

            //如果该节点的左右子节点都不为空
            if (parent.getLeftNode() != null && parent.getRightNode() != null){
                // 选择替代节点的条件：
                // 1.小于parent(即将被删除)节点，大于parent(即将被删除)左树节点中任意节点.
                // 2.大于parent(即将被删除)节点，小于parent(即将被删除)右树节点中任意节点。
                // 满足上述任意条件即可吗，那么就是选择删除左树和右树节点的问题了。为了平衡，左树，右树哪个高度最高就从中选择替代节点。
                // 步骤：
                //    先查找满足条件的节点赋值给替代节点保留，然后删除左树或右树中的该节点，设置替代节点的左右节点，重新计算左右节点高度，
                //    返回给外部(上一层级)递归层级，外部递归层级中会将该重新赋值为替代节点。



                if (height(parent.getLeftNode()) > height(parent.getRightNode())){
                    //要删除节点即parent，如果它的左节点的高度>右节点高度，那就选出左树节点最大的节点作为parent节点的替代节点，
                    //该替代节点(maxNode)一定小于要被删除的节点(parent)。正好满足替代条件

                    //获取替代节点并保留
                    AVLNode maxValue = getMaxNode(parent.getLeftNode());
                    //递归删除左树中的替代节点
                    parent.setLeftNode(remove(parent.getLeftNode(), maxValue.getValue()));
                    //设置替代节点的左右节点为parent的左右节点
                    maxValue.setLeftNode(parent.getLeftNode());
                    maxValue.setRightNode(parent.getRightNode());
                    //重新计算替代节点的高度
                    maxValue.setHeight(maxHeight(parent.getLeftNode(), parent.getRightNode()) + 1);
                    //返回给外部递归层级
                    return maxValue;
                }else {
                    //要删除节点即parent，如果它的左节点的高度<=右节点高度，那就选出右树节点最小的节点作为parent节点的替代节点，
                    //该替代节点(minNode)一定大于要被删除的节点(parent)。正好满足替代条件

                    //获取替代节点并保留
                    AVLNode minValue = getMinNode(parent.getRightNode());
                    //递归删除右树中的替代节点
                    parent.setRightNode(remove(parent.getRightNode(), minValue.getValue()));
                    //设置替代节点的左右节点为parent的左右节点
                    minValue.setLeftNode(parent.getLeftNode());
                    minValue.setRightNode(parent.getRightNode());
                    //重新计算替代节点的高度
                    minValue.setHeight(maxHeight(parent.getLeftNode(), parent.getRightNode()) + 1);
                    //返回给外部递归层级
                    return minValue;
                }
            }else {
                // 如果左右节点中有任意一个为空
                if (parent.getLeftNode() != null || parent.getRightNode() != null) {
                    parent = parent.getLeftNode() == null ? parent.getRightNode() : parent.getLeftNode();
                }else {
                    parent = null;
                }
            }
        }

        return parent;
    }

    /**
     * 获取值最大节点
     */
    private AVLNode getMaxNode(AVLNode currentNode){
        if (currentNode != null){
            currentNode = getMaxNode(currentNode.getRightNode());
        }
        return currentNode;
    }

    /**
     * 获取值最小节点
     */
    private AVLNode getMinNode(AVLNode currentNode){
        if (currentNode != null) {
            currentNode = getMinNode(currentNode.getLeftNode());
        }
        return currentNode;
    }

    private AVLNode createSimpleNode(Integer val){
        return new AVLNode(val);
    }

    /**
     * 求一个节点的高度
     * @param avlNode 节点对象
     * @return 返回高度值
     */
    private Integer height(AVLNode avlNode){
        //空节点高度默认是-1
        return avlNode == null ? -1 : avlNode.getHeight();
    }

    /**
     * 获取左右节点中高度最大的值
     * @param leftNodo 左节点
     * @param rightNode 右节点
     * @return 最大节点的高度
     */
    private Integer maxHeight(AVLNode leftNodo, AVLNode rightNode){
        return height(leftNodo) > height(rightNode) ? height(leftNodo) : height(rightNode);
    }

    /**
     * 左左旋转
     * @param avlNode 旋转之前的父节点
     * @return 旋转之后的父节点
     */
    private AVLNode leftLeftRotate(AVLNode avlNode){
        //让node节点的
        AVLNode newAvlNode = avlNode.getLeftNode();
        avlNode.setLeftNode(newAvlNode.getRightNode());
        newAvlNode.setRightNode(avlNode);

        avlNode.setHeight(maxHeight(avlNode.getLeftNode(), avlNode.getRightNode()) + 1);
        newAvlNode.setHeight(maxHeight(newAvlNode.getLeftNode(), newAvlNode.getRightNode()) + 1);
        return newAvlNode;
    }

    /**
     * 右右旋转
     * @param avlNode 旋转之前的父节点
     * @return 旋转之后的父节点
     */
    private AVLNode rightRightRotate(AVLNode avlNode){
        AVLNode newAvlNode = avlNode.getRightNode();
        avlNode.setRightNode(newAvlNode.getLeftNode());
        newAvlNode.setLeftNode(newAvlNode);

        avlNode.setHeight(maxHeight(avlNode.getLeftNode(), avlNode.getRightNode()) + 1);
        newAvlNode.setHeight(maxHeight(newAvlNode.getLeftNode(), newAvlNode.getRightNode()) + 1);
        return newAvlNode;
    }

    /**
     * 左右旋转
     * @param avlNode 旋转之前的父节点
     * @return 旋转之后的父节点
     */
    private AVLNode leftRightRotate(AVLNode avlNode){
        avlNode.setLeftNode(rightRightRotate(avlNode.getLeftNode()));
        return leftLeftRotate(avlNode);
    }

    /**
     * 右左旋转
     * @param avlNode 旋转之前的父节点
     * @return 旋转之后的父节点
     */
    private AVLNode rightLeftRotate(AVLNode avlNode){
        avlNode.setRightNode(leftLeftRotate(avlNode.getRightNode()));
        return rightRightRotate(avlNode);
    }

}

```