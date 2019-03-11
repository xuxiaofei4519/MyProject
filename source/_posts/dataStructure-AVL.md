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

>以下代码纯手动实现，根据数据结构与算法分析第三版第四章AVL树讲解实现

#### 手动实现源码
```java
package com.BalanceTree;


import com.sun.org.apache.xml.internal.resolver.readers.ExtendedXMLCatalogReader;
import lombok.Data;
import lombok.val;
import org.junit.Test;
import org.springframework.util.StringUtils;
import sun.util.locale.provider.AvailableLanguageTags;

import java.util.concurrent.atomic.AtomicInteger;

/**
 * 平衡二叉树 AVL实现
 */
public class AVLBalanceTree {

    //根节点
    private AVLNode root;
    //节点数
    private volatile AtomicInteger size = new AtomicInteger();

    private static final int ALLOW_HEIGHT = 1;

    public AVLBalanceTree() {
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
        public AVLNode leftNode;
        //右子节点
        public AVLNode rightNode;

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
            size.incrementAndGet();
            return;
        }
        if (constains(value)) {
            throw new Exception("该值已经存在");
        }
        root = insert(root, value);
    }

    /**
     * 初始化根节点
     */
    private AVLNode initRoot(Integer value){
        this.root = new AVLNode(value, 0, null,null);
        System.out.println("root:" + this.root.getValue());
        return root;
    }


    /**
     * 查询是否包含该val
     * @param val
     * @return boolean值
     */
    private boolean constains(Integer val){
        AVLNode currentNode = root;
        if (null == currentNode) {
            return false;
        }
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
     * 重新平衡
     */
    private AVLNode reBalance(AVLNode parent){
        if (parent == null){
            return parent;
        }
        //在每一个递归层级上都要进行节点的平衡判断
        // 判断parent节点左右儿子节点的高度。看看是否大于允许值
        if (height(parent.leftNode) - height(parent.rightNode) > ALLOW_HEIGHT){
            // 左孙子节点高度>=右节点高度  (有可能删除导致左右孙子节点高度相同)
            if (height(parent.leftNode.leftNode) >=  height(parent.leftNode.rightNode)){
                parent = rotateWithLeft(parent);
            }else {
                parent = leftRightRotate(parent);
            }
        }else if (height(parent.rightNode) - height(parent.leftNode) > ALLOW_HEIGHT){
            // 判断parent节点左右儿子节点的高度。看看是否大于允许值

            // 右孙子节点高度>=左节点高度  (有可能删除导致左右孙子节点高度相同)
            if (height(parent.rightNode.rightNode) >=  height(parent.rightNode.leftNode)){
                parent = rotateWithRight(parent);
            }else {
                parent = rightLeftRotate(parent);
            }
        }
        // 重新计算当前节点的高度
        parent.setHeight(Math.max(height(parent.getLeftNode()), height(parent.getRightNode())) + 1);
        return parent;
    }

    /**
     * 插入操作
     */
    private AVLNode insert(AVLNode parent,Integer val){
        if (parent == null){
            size.incrementAndGet();
            return createSimpleNode(val);
        }
        if (val < parent.getValue()){
            parent.leftNode = insert(parent.leftNode, val);
        }else if (val > parent.getValue()){
            parent.rightNode = insert(parent.rightNode, val);
        }else {
            // do nothing
        }
        return reBalance(parent);
    }

    public boolean remove(Integer value) throws Exception {
        if (this.root == null){
            return false;
        }
        if (!constains(value)) {
            throw new Exception("该值不存在");
        }

        root = remove(root, value);
        return true;
    }

    public AVLNode remove(AVLNode parent, Integer val){
        if (val == null){
            return parent;
        }
        if (val < parent.getValue()){
            parent.leftNode = remove(parent.getLeftNode(), val);
        }else if (val > parent.getValue()){
            parent.rightNode = remove(parent.getRightNode(), val);
        }else if (parent.getLeftNode() != null && parent.getRightNode() != null){
            // 因为要删除的parent节点，但是左右子节点都不为null，就取右子树最小节点的值作为parent的值
            parent.setValue(getMinNode(parent.getRightNode()).getValue());
            // 重新设置右节点
            parent.rightNode = remove(parent.getRightNode(), parent.getValue());
        }else {
            // 要删除节点有一个节点是空的，因此
            parent = (parent.getLeftNode() != null) ? parent.getLeftNode() : parent.getRightNode();
        }
        // 再平衡
        return reBalance(parent);
    }


    public void print() throws Exception {
        if (this.root != null){
            printALL(this.root);
        }
    }


    /**
     * 打印所有
     */
    public void printALL(AVLNode node) {
        if (node != null){
            printALL(node.leftNode);
            System.out.println(node.value);
            printALL(node.rightNode);
        }
    }

    /**
     * 获取值最大节点
     */
    private AVLNode getMaxNode(AVLNode currentNode){
        if (currentNode == null){
            return null;
        }else if (currentNode.rightNode == null){
            return currentNode;
        }
        return getMaxNode(currentNode);
    }

    /**
     * 获取值最小节点
     */
    private AVLNode getMinNode(AVLNode currentNode){
        if (currentNode == null){
            return null;
        }else if (currentNode.leftNode == null){
            return currentNode;
        }
        return getMinNode(currentNode);
    }

    private AVLNode createSimpleNode(Integer val){
        return new AVLNode(val, 0, null, null);
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
     * 左旋转(单旋转)
     * @param avlNode 旋转之前的父节点
     * @return 旋转之后的父节点
     */
    private AVLNode rotateWithLeft(AVLNode avlNode){
        //让node节点的
        AVLNode newAvlNode = avlNode.leftNode;
        avlNode.leftNode = newAvlNode.rightNode;
        newAvlNode.rightNode = avlNode;

        avlNode.setHeight(Math.max(height(avlNode.leftNode), height(avlNode.rightNode)) + 1);
        newAvlNode.setHeight(Math.max(height(newAvlNode.leftNode), avlNode.getHeight()) + 1);
        return newAvlNode;
    }

    /**
     * 右旋转(单旋转)
     * @param avlNode 旋转之前的父节点
     * @return 旋转之后的父节点
     */
    private AVLNode rotateWithRight(AVLNode avlNode){
        AVLNode newAvlNode = avlNode.rightNode;
        avlNode.rightNode = newAvlNode.leftNode;
        newAvlNode.leftNode = avlNode;

        avlNode.setHeight(Math.max(height(avlNode.leftNode), height(avlNode.rightNode)) + 1);
        newAvlNode.setHeight(Math.max(height(newAvlNode.rightNode), avlNode.getHeight()) + 1);
        return newAvlNode;
    }

    /**
     * 左右旋转
     * @param avlNode 旋转之前的父节点
     * @return 旋转之后的父节点
     */
    private AVLNode leftRightRotate(AVLNode avlNode){
        avlNode.rightNode = rotateWithLeft(avlNode.rightNode);
        return rotateWithRight(avlNode);
    }

    /**
     * 右左旋转
     * @param avlNode 旋转之前的父节点
     * @return 旋转之后的父节点
     */
    private AVLNode rightLeftRotate(AVLNode avlNode){
        avlNode.leftNode = rotateWithRight(avlNode.leftNode);
        return rotateWithLeft(avlNode);
    }

}

```