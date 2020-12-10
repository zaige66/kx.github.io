---
title: java二叉树（TreeSet）实现
date: 2019-02-27 18:05:15
tags: [java]
categories: [java,jdk]
---
```
实现了两种遍历方式
1.从根节点开始找
2.从最小节点开始找（jdk采用该方式）
```
```java
import java.util.Random;

/**
 * @author kangxuan
 * @date 2019/2/27 0027 11:29.
 * @Description: treeSet
 */
public class SortTree {

    public static void main(String[] args) {
        SortTree sortTree = new SortTree();

        Random random = new Random();
        for (int i = 0; i < 10; i++) {
            int randomInt = random.nextInt(100);
            sortTree.add(randomInt);
            // 打印存入的值
            System.out.print(randomInt + ",");
        }
        System.out.println();

        // 从根节点开始寻找从小到大的节点
        sortTree.print();
        System.out.println();

        // 从最小节点开始寻找从小到大的节点
        sortTree.printFromFirstNode(sortTree.getFirstNode());
    }

    Node rootNode;

    // 打印
    public void print(){
        print(rootNode);
    }

    /**
     * 该打印方法的思路是从根节点开始找
     * @param rootNode
     */
    private void print(Node rootNode){
        boolean flag = false; // 控制只输出一次
        if (rootNode == null){
            return;
        }
        // 当该节点没下一个节点时
        if (rootNode.leftNode == null){
            if (!flag) {
                System.out.print(rootNode.data + ",");
                flag = true;
            }
        }
		// 打印左边节点
        print(rootNode.leftNode);
        if (!flag) {
            System.out.print(rootNode.data + ",");
            flag = true;
        }
        // 打印右边节点
        print(rootNode.rightNode);
        if (!flag) {
            System.out.print(rootNode.data + ",");
            flag = true;
        }
    }

    /**
     * 该打印方法的思路是从最小值节点开始找
     *  jdk中的TreeMap采用的是从最小节点开始找的方式
     *      寻找上一个没打印过的节点
     *      我：通过属性标记是否打印过
     *      jdk: 判断该节点 的父节点 的右节点 是否等于该节点，如果是，则继续往上找
     *          详见：TreeMap类中子类Entry中的successor方法
     * @param firstNode
     */
    public void printFromFirstNode(Node firstNode){
        System.out.print(firstNode.data + ",");
        firstNode.isPrint = true;

        if (firstNode.rightNode != null){
            printFromFirstNode(getFirstNode(firstNode.rightNode));
        }

        if (null != firstNode.preNode) {
            // 寻找上一个没打印过的节点
            Node next = getNext(firstNode.preNode);
            if (null != next) {
                printFromFirstNode(next);
            }
        }

    }

    /**
     * 寻找上一个没打印过的节点
     * @param preNode
     * @return
     */
    private Node getNext(Node preNode) {
        if (preNode != null)
        if (preNode.isPrint){
            return getNext(preNode.preNode);
        }
        return preNode;
    }

	// 获取最小节点
    public Node getFirstNode(){
        return getFirstNode(rootNode);
    }

    private Node getFirstNode(Node rootNode){
        if (rootNode.leftNode == null) return rootNode;
        return getFirstNode(rootNode.leftNode);
    }

	// 添加数据
    public void add(Integer data){
        // 第一次加入数据
        if (rootNode == null){
            rootNode = new Node(data);
            return;
        }
        // 添加数据
        addValue(rootNode,data);
    }

    private void addValue(Node rootNode, Integer newData) {
        Integer oldData = rootNode.data;
        if (oldData == newData) return;
        // 右节点
        if (oldData < newData){
            Node rightNode = rootNode.rightNode;
            if (rightNode == null){
                rightNode = new Node(newData);
                rightNode.preNode = rootNode;

                rootNode.rightNode = rightNode;
                return;
            }else {
                addValue(rightNode,newData);
            }
        }
        // 左节点
        if (newData < oldData){
            Node leftNode = rootNode.leftNode;
            if (leftNode == null){
                leftNode = new Node(newData);
                leftNode.preNode = rootNode;

                rootNode.leftNode = leftNode;
                return;
            }else {
                addValue(leftNode,newData);
            }
        }
    }

}
class Node {
    // 数据
    Integer data;
    // 上一个节点
    Node preNode;
    // 左节点
    Node leftNode;
    // 右节点
    Node rightNode;

    // 是否打印过
    boolean isPrint;

    public Node(Integer data) {
        this.data = data;
    }
}

```
有不对的地方还请大家指教！
