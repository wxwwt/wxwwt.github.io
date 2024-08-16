---
title: java的深拷贝和浅拷贝
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: basic
---
### java的深拷贝和浅拷贝

我们知道拷贝就是生成一个新对象和原对象一模一样，但是拷贝也是分方式和程度的，我们来看一下什么是浅拷贝什么是深拷贝

### 浅拷贝
在Java中，java.lang.Object类的clone()方法用于克隆（浅拷贝，属性的指向是相同的）。
该方法创建一个对象的副本，并通过逐字段分配在其上对其进行调用并返回该对象的引用。
要实现浅拷贝需要实现Cloneable接口，该接口里面没有任何方法，它指向的是java.lang.Object类的clone()
```
protected native Object clone() throws CloneNotSupportedException;
```
它是一个native方法，由C/C++实现
我们看看例子，理解下它为啥叫浅拷贝
首先有一个字典对象类，它实现了序列化（）和克隆的接口
```java
@Data
class Dictionary implements Serializable {

   private String name;

   private List<String> words;

   @Override
   public Object clone() {
       try {
           return super.clone();
       } catch (CloneNotSupportedException e) {
           e.printStackTrace();
       }
       return null;
   }
}

```

```java
public static void shallowCopy() {
       Dictionary dictionary1 = new Dictionary();
       dictionary1.setName("汉语词典");
       dictionary1.setWords(new ArrayList<String>() {{
           add("你好");
           add("浅拷贝");
       }});
       Dictionary dictionary2 = (Dictionary) dictionary1.clone();
       System.out.println(dictionary1 == dictionary2);
       dictionary2.getWords().add("新词语");
       System.out.println("dictionary1: " + dictionary1.toString());
       System.out.println("dictionary2: " + dictionary2.toString());

       dictionary1.setName("新名字");
       System.out.println("dictionary1: " + dictionary1.toString());
       System.out.println("dictionary2: " + dictionary2.toString());
   }
```
运行结果：
```
false
dictionary1: Dictionary(name=汉语词典, words=[你好, 浅拷贝, 新词语])
dictionary2: Dictionary(name=汉语词典, words=[你好, 浅拷贝, 新词语])
dictionary1: Dictionary(name=新名字, words=[你好, 浅拷贝, 新词语])
dictionary2: Dictionary(name=汉语词典, words=[你好, 浅拷贝, 新词语])
```
从结果上我们知道dictionary1，dictionary2不是指向的同一个对象，确实创建了两个对象，但是当第二个对象属性被修改时，第一个对象也跟着变了。
验证了我们之前说的浅拷贝，两个对象的属性指向的是堆中相同的对象。

### 深拷贝
深拷贝相对于浅拷贝来说就是属性也是新的对象，我们可以将对象的每一个属性也实现cloneable接口，就可以达到深拷贝的效果。我们也可以使用序列化反序列化来实现深拷贝。
首先将Dictionary实现Serializable接口
```java
@Data
class Dictionary implements Cloneable, Serializable {

    private String name;

    private List<String> words;

    @Override
    public Object clone() {
        try {
            return super.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

```java
private static void deepCopy() throws IOException, ClassNotFoundException {
        Dictionary dictionary1 = new Dictionary();
        dictionary1.setName("汉语词典");
        dictionary1.setWords(new ArrayList<String>() {{
            add("你好");
            add("浅拷贝");
        }});

        Dictionary dictionary2 = null;

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeObject(dictionary1);

        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(byteArrayOutputStream.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(byteArrayInputStream);
        dictionary2 = (Dictionary) ois.readObject();

        // 测试方法没关闭流 实际项目记得关闭流
        System.out.println(dictionary1 == dictionary2);
        dictionary2.getWords().add("新词语");
        System.out.println("dictionary1: " + dictionary1.toString());
        System.out.println("dictionary2: " + dictionary2.toString());
    }
```
运行结果
```java
false
dictionary1: Dictionary(name=汉语词典, words=[你好, 深拷贝])
dictionary2: Dictionary(name=汉语词典, words=[你好, 深拷贝, 新词语])
```
我可以看到不管是对象还是它的属性都是独立的
