---
title: java序列化和反序列化
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: basic
---
### 引语:
&nbsp;&nbsp;&nbsp;&nbsp;平时我们在运行程序的时候,创建的对象都在内存中,当程序停止或者中断了,对象也就不复存在了.如果我们能将对象保存起来,在需要使用它的时候在拿出来使用就好了,并且对象的信息要和我们保存
时的信息一致.序列化就可以解决了这样的问题.序列化当然不止一种方式,如下:

|序列类型|	是否跨语言	|优缺点|
|:-:|:-:|:-:|
|hession	|支持|	跨语言,序列化后体积小,速度较快|
|protostuff|	支持|	跨语言,序列化后体积小,速度快,但是需要Schema,可以动态生成|
|jackson	|支持	|跨语言,序列化后体积小,速度较快,且具有不确定性|
|fastjson|	支持|	跨语言支持较困难,序列化后体积小,速度较快,只支持java,c#|
|kryo	|支持	|跨语言支持较困难,序列化后体积小,速度较快|
|fst|	不支持	|跨语言支持较困难,序列化后体积小,速度较快，兼容jdk|
|jdk|不支持|序列化后体积很大,速度快|

我们今天介绍的就是java原生的Serializable序列化.
先列一下概念:
序列化:序列化是将对象的状态信息转换为可以存储或传输的形式的过程
反序列化:从存储或传输形式还原为对象

### Serializable的使用
序列化使用起来很简单只需要实现Serializable接口即可,然后序列化(序列化是将对象的状态信息转换为可以存储或传输的形式的过程)和反序列化(反之,从存储或传输形式还原为对象).
只要使用ObjectOutputStream和ObjectInputStream将对象转为二进制序列和还原为java对象.话不多说,看下代码示例:
```java
private static void testSerializable(String fileName) throws IOException {
       try (ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream(fileName))) {
           // "XXX" 的String也可以直接作为对象进行反序列化的
           objectOutputStream.writeObject("test serializable");
           SerializableData data = new SerializableData(1, "testStr");
           objectOutputStream.writeObject(data);
       }
   }

   private static void testDeserializable(String fileName) throws IOException, ClassNotFoundException {
       try (ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(fileName))) {
           String str = (String) objectInputStream.readObject();
           System.out.println("String的反序列化: " + str);
           SerializableData readData = (SerializableData) objectInputStream.readObject();
           System.out.println("反序列化的对象: " + readData.toString());
           // 输出:反序列化的对象: SerializableData(testInt=1, testStr=testStr)
       }
   }

// 使用到的类
@Data
@AllArgsConstructor
class SerializableData implements Serializable {

    private Integer testInt;

    private String testStr;
}
```
第一个方法是传入文件路径,将String和SerializableData对象序列化到fileName指定的文件中;第二个方法是反序列化将文件中的二进制还原为java对象.
这里其实比较简单没有什么大问题,稍微提一句的就是writeObject这个方法是可以直接将"写入的字符串"这种形式的对象直接序列化为二进制的.
这里还有一点就是反序列化的版本号必须和原本对象的版本号(private static final long serialVersionUID = 1L;这个因为是自己测试所以没有写默认是1L,修改后,反序列化的对象版本号不一致会报错)一致,并且jvm能找到反序列化的文件的位置,否则都会失败.

### transient关键字
简单的使用序列化和反序列化应该没有什么问题,我们再来看看transient关键字是啥?在某些场景下,我们需要写入或者还原的数据中其实有我们不需要透露或者说不想暴露给外部的数据,如果我们将这些隐私的数据序列化,在反序列化出来,
那么这些信息就泄漏了.而transient关键字呢,就是防止这种事情的发生.
当属性被加上了transient关键字以后,序列化时不会将该属性的值给写入,所以反序列化的时候我们会发现原本写入的数据,还原出来是null.
我们写一个例子看看是否是这样呢?
```java
private static void testTransient() throws IOException, ClassNotFoundException {
       String fileName = "transientData.txt";
       try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(fileName))) {
           Account data = new Account(1, "user1", "123456");
           out.writeObject(data);
       }
       try (ObjectInputStream in = new ObjectInputStream(new FileInputStream(fileName))) {
           Account readData = (Account) in.readObject();
           System.out.println("transient关键字的对象: " + readData.toString());
           // 输出: transient关键字的对象: Account(id=1, userName=user1, idCardNumber=null)
       }
   }

// 对应的对象
@Data
@AllArgsConstructor
class Account implements Serializable {

    private Integer id;

    private String userName;

    private transient String idCardNumber;
}
```
这里我们有一个Account对象,我们不想暴露出我们的省份证号码idCardNumber,于是乎加上了transient关键字.
然后将idCardNumber已经初始化过的data对象序列化,当我们再反序列化去取得这个idCardNumber的值的时候,发现确实对象的idCardNumber是null,transient是起作用的.
如果是对基本类型数据加上transitent话,会得到对应的默认值,就好比是int的数据类型,得到的就是0.

###  Externalizable的使用
使用过自动的序列化和反序列化以后,我们又想在序列化和反序列化的时候我们能不能自己控制呢?在序列化和反序列化的时候我们能不能加点日志或者其他的操作之类的呢?
是的,阔以的.只需要轻轻一点,实现Externalizable接口即可,和Serializable使用差不多.
```java
private static void testExternalizable() throws IOException, ClassNotFoundException {
        String fileName = "testExternalizable.txt";
        try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(fileName))) {
            Account2 data = new Account2("user1", 1);
            out.writeObject(data);
        }
        try (ObjectInputStream in = new ObjectInputStream(new FileInputStream(fileName))) {
            Account2 readData = (Account2) in.readObject();
            System.out.println("Externalizable的对象: " + readData.toString());
        }
    }

// 使用到的对象
@Data
@AllArgsConstructor
class Account2 implements Externalizable {

    private Integer id;

    private String userName;

    private transient String idCardNumber;

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        System.out.println("执行了writeExternal方法");
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        System.out.println("执行了readExternal方法");
    }
}
```
如果执行了上面的代码,恭喜你,获得一个Exception的奖励.大概长这样,java.io.InvalidClassException:XXX no valid constructor,
**Externalizable在执行的时候会调用默认的无参构造函数,而且记住哦,必须是public的**,如果没有加public你会发现又奖励了一个Exception给你.
讲道理这个是比较坑的.下面我们来看看正确的用法,序列化和反序列化都是我们自己控制的:

```java
private static void testExternalizable() throws IOException, ClassNotFoundException {
       String fileName = "testExternalizable.txt";
       try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(fileName))) {
           Account3 data = new Account3("user1", 1);
           out.writeObject(data);
       }
       try (ObjectInputStream in = new ObjectInputStream(new FileInputStream(fileName))) {
           Account3 readData = (Account3) in.readObject();
           System.out.println("Externalizable的对象: " + readData.toString());
           /**
            * 输出:
            * 执行了writeExternal方法
            * 执行了readExternal方法
            * Externalizable的对象: Account3(userName=user1, id=1)
            */
       }
   }

@ToString
class Account3 implements Externalizable {

    private String userName;

    private Integer id;

    public Account3() {

    }

    public Account3(String userName, Integer id) {
        this.userName = userName;
        this.id = id;
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        System.out.println("执行了writeExternal方法");
        out.writeObject(userName);
        out.writeInt(id);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        System.out.println("执行了readExternal方法");
        userName = (String) in.readObject();
        id = in.readInt();

    }
}
```

### 总结:
1.我们介绍了jdk自带的序列化和反序列化(和其中的一些坑点);
2.知道了transient可以将隐私数据不序列化;
3.还有Externalizable可以自己来控制序列化和反序列化的进程.

### 参考资料:
1.https://docs.oracle.com/javase/8/docs/platform/serialization/spec/serial-arch.html#a6428(官网)
2.https://blog.csdn.net/do_bset_yourself/article/details/77173143(摘录了各种序列化方式的优缺点)
