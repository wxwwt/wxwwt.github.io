title:怎么重复使用inputStream



### 引语：

&nbsp;&nbsp;&nbsp;&nbsp;之前做项目的时候遇到一个问题,就是从网络中读取的图片要上传到oss,而且要对图片进行裁剪和压缩,其中上传和裁剪都要使用到图片的inputStream,
又因为inputstream不能重复读,导致裁剪是成功的,而上传是失败的.我们今天就提供两种方法来解决,inputStream不能重复读的问题.

### 问题分析:
inputStream的内部有个pos指针,当读取的时候指针会不断的移动,当移动到末尾的时候,就无法再次读取了.
我们写个简单的例子来看下:
```java
    String text = "测试inputStream内容";
    InputStream inputStream = new ByteArrayInputStream(text.getBytes());
    byte[] readArray = new byte[inputStream.available()];
    int readCount1 = inputStream.read(readArray);
    System.out.println("读取了" + readCount1 + "个字节");

    byte[] readArray2 = new byte[inputStream.available()];
    int readCount2 = inputStream.read(readArray2);
    System.out.println("读取了" + readCount2 + "个字节");
    /**
    *  执行结果是
    *  读取了23个字节
    *  读取了-1个字节
    */

```
从执行结果可以看出确实inputstream的设计是只能读取一次.
**注意: 这里稍微提一下inputStream.available()这个方法,本地的文件可以直接知道文件的大小,但是如果是网络中的数据,这个方法最好不要用,因为传输的时候不是连续的,数据的大小会读取不准**

### 问题解决:
那么我们实际项目中应该怎么解决呢?总不能就真的只使用一次inputSteam吧.我们来看解决方法:
**方法一**:使用ByteArrayOutputStream来缓存字节,然后每次读取从缓存的ByteArrayOutputStream中拿取.
很自然的想到把inputStream的缓存起来(当然不一定说是要放在ByteArrayOutputStream,其他的方式也可以,都是缓存起来的思路,实现方式有很多种,这种比较方便)
```java
       String text = "测试inputStream内容";
       InputStream rawInputStream = new ByteArrayInputStream(text.getBytes());
       ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
       byte[] buffer = new byte[1024];
       int len;
       while ((len = rawInputStream.read(buffer)) > -1) {
           outputStream.write(buffer, 0, len);
       }
       outputStream.flush();
       InputStream in1 = new ByteArrayInputStream(outputStream.toByteArray());
       InputStream in2 = new ByteArrayInputStream(outputStream.toByteArray());
       int readCount1 = in1.read(buffer);
       int readCount2 = in2.read(buffer);
       System.out.println("读取了" + readCount1 + "个字节");
       System.out.println("读取了" + readCount2 + "个字节");
       /**
       *  执行结果是
       *  读取了23个字节
       *  读取了23个字节
       *
```
这里是先将inputStream的数据读取到output中,然后要反复使用inputStream中的内容的时候,我们将output中的数据取出(很神奇的设定,output可以反复取,input只能读一次)

**方法二**:其实inputStream中有操作指针的方法,mark和reset,听名字就知道是标记和重置.在使用inputSteam前我们标记下inputStream指针的位置,读取完之后,重置,然后就可以反复使用了.我们看代码:
```java
      String text = "测试inputStream内容";
      InputStream rawInputStream = new ByteArrayInputStream(text.getBytes());
      byte[] readArray = new byte[1024];
      rawInputStream.mark(0);
      int readCount1 = rawInputStream.read(readArray);
      rawInputStream.reset();
      int readCount2 = rawInputStream.read(readArray);
      System.out.println("读取了" + readCount1 + "个字节");
      System.out.println("读取了" + readCount2 + "个字节");
```

### 总结:
1.inputStream只能读取一次,也就是说只能调用read()或者其他的带参数的read()方法一次,在下次调用读取出来是-1,做项目的时候不要忘记这一点了,可能会导致有些坑出现;
2.可以使用缓存或者mark/reset方法来重复使用inputStream,这里要注意的是如果inputStream如果内容很多,缓存不是一个好办法,因为在使用完之前会占用大量的内存(我遇到过这样的,上传很多图片然后还有缓存,导致内存不够就一直fullGC,然后cpu先爆了);
3.还有一个小点就是别忘了关闭使用完的inputStream/outputSteam.
