---
title: Comparable与Comparator #文章页面上的显示名称，一般是中文
date: 2020-10-27 14:16:33 #文章生成时间，一般不改，当然也可以任意修改
categories: JAVA #分类
tags: [Comparable,Comparator,排序] #文章标签，可空，多标签请用格式，注意:后面有个空格
description: 详细讲解comparable及comparator的使用及用法。
---

## Comparable

[TOC]

```
import java.util.Objects;

/**
 * 实现Comparator接口的类可以方便的排序， 覆写compareTo接口
 * @author zxl
 * @date 2020/10/27 18:00
 */
public class Student implements Comparable<Student>{

    /*** 名字 */
    private String name;

    /*** 年龄 */
    private int age;

    /*** 分数 */
    private float score;

    /**
     * 排序规则：以分数降序排序，若分数一致，则根据年龄升序排序
     * @param o 比较参数
     * @return 比较结果 -1大于   0等于   1小于
     */
    @Override
    public int compareTo(Student o) {
        /*
         * this.age - o.age 升序
         * o.score - this.score 降序
         */
        return Objects.equals(o.score,this.score) ? this.age - o.age : (int) (o.score - this.score);
    }

    public Student(String name, int age, float score) {
        this.name = name;
        this.age = age;
        this.score = score;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", score=" + score +
                '}';
    }
}


public class Comparable01 {

    public static void main(String args[]){
        Student stu[] = {
                new Student("张三",20,90.0f),
                new Student("李四",22,90.0f),
                new Student("王五",20,99.0f),
                new Student("赵六",20,70.0f),
                new Student("孙七",22,100.0f)} ;
        java.util.Arrays.sort(stu) ;    // 进行排序操作
        for(int i=0;i<stu.length;i++){    // 循环输出数组中的内容
            System.out.println(stu[i]) ;
        }
    }

}

```



## Comparator

```
public class Student {
    /*** 名字 */
    private String name;

    /*** 年龄 */
    private int age;

    /*** 分数 */
    private float score;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public float getScore() {
        return score;
    }

    public void setScore(float score) {
        this.score = score;
    }

    public Student(String name, int age, float score) {
        this.name = name;
        this.age = age;
        this.score = score;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", score=" + score +
                '}';
    }

}

public class ComparatorTest {

    public static void main(String[] args) {
        Student stu[] = {
                new Student("张三",20,90.0f),
                new Student("李四",22,90.0f),
                new Student("王五",20,99.0f),
                new Student("赵六",20,70.0f),
                new Student("孙七",22,100.0f)} ;

        // 排序方法一
        java.util.Arrays.sort(stu, new Comparator<Student>() {
            @Override
            public int compare(Student o1, Student o2) {
                /*
                 * o1.param - o2.param 升序
                 * o2.param - o1.param 降序
                 */
                return Objects.equals(o1.getScore(),o2.getScore()) ? o1.getAge() - o2.getAge() : (int) (o2.getScore() - o1.getScore());
            }
        }) ;

        // 排序方法二 lambda方法
        java.util.Arrays.sort(stu, (o1, o2) -> Objects.equals(o1.getScore(),o2.getScore()) ? o1.getAge() - o2.getAge() : (int) (o2.getScore() - o1.getScore())) ;


        for(int i=0;i<stu.length;i++){    // 循环输出数组中的内容
            System.out.println(stu[i]) ;
        }
    }

    /**
     * 排序方法三：该方法需新建一个java类使用，
     * java.util.Arrays.sort(stu, new MyComparator());
     */
    class MyComparator implements Comparator<Student>{
        @Override
        public int compare(Student o1, Student o2) {
            return Objects.equals(o1.getScore(),o2.getScore()) ? o1.getAge() - o2.getAge() : (int) (o2.getScore() - o1.getScore());
        }
    }

}
```

## 两者比较

<table>
	<tr>
		<th>属性</th>
	    <th>Comparable</th>
	    <th>Comparator</th>
	</tr >
	<tr>
		<td>适用场景</td>
		<td>集合内部定义实现</td>
		<td>集合外部实现排序</td>
	</tr>
    <tr>
		<td>区别</td>
		<td>1. java.lang 包<br/>
        	2. 方法：int compareTo(Object o)
        </td>
		<td>1. java.util 包<br/>
        	2. 方法：int compare(T o1, To2)
        </td>
	</tr>
    <tr>
		<td>原理</td>
		<td colspan="2">基于红黑二叉树原理实现的</td>
	</tr>
</table>

| 返回值 | 含义 |
| ------ | ---- |
| -1     | 大于 |
| 0      | 等于 |
| 1      | 小于 |

## 参考

\- [java中compareable和comparator的区别，比较器实现的原理！](https://blog.csdn.net/wilson27/article/details/90339765)