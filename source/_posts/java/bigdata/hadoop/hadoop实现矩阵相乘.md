---
title: hadoop实现矩阵相乘
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: hadoop
---
# hadoop 实现矩阵相乘

我们大学里学过矩阵相乘，如下，当两个矩阵A，B，A的行等于B的列时可以相乘。然后乘积是A的行乘以B的列得出。我们今天用hadoop来实现一下矩阵的乘法。
$$
    \left[
  \begin{matrix}
   A1 &  A2 &  A3 \\
    A4 &  A5 &  A6 \\
    A7 &  A8 &  A9
  \end{matrix}
  \right]
  X
     \left[
  \begin{matrix}
   B1 & B2 & B3 \\
   B4 & B5 & B6 \\
   B7 & B8 & B9
  \end{matrix}
  \right]
$$


计算过程是A行乘以B列，我们可以将B先转置（行列互换），然后在用A行乘以B行可以得出结果，具体步骤如下：
1.将B（下面可以理解为右边的矩阵）转置，结果输出B'
2.AxB'(B'的结果放在hdfs的文件系统缓存中)，输出结果

我们先看一下例子的两个矩阵数据
$$
    \left[
  \begin{matrix}
   1 &  2 &  -1 \\
    2 &  1 &  3 \\
    0 &  3 &  1
  \end{matrix}\tag{A}
  \right]
$$
$$
     \left[
  \begin{matrix}
   1 & 2 & 3 \\
   3 & -1 & 0 \\
   -4 & 2 & 1
  \end{matrix}\tag{B}
  \right]
$$
我们定义放在hdfs文件中数据形式如下
1 1_1,2_2,3_-1
2 1_2,2_2,3_3
3 1_0,2_3,3_1
一行的最左边是行号，右边的是数据，“1_1”这种左边是列号，右边是数据值

代码部分：
第一步：将B（下面可以理解为右边的矩阵）转置，结果输出B'
Map阶段：
将右矩阵的数据读入
```java
public class MapMatrixTranspose extends Mapper<LongWritable, Text, Text, Text> {

    private Text outKey = new Text();
    private Text outValue = new Text();

    /**
     * key:1
     * values:1_1,2_2,3_-1,4_0
     * 1    1_-1,2_1,3_4,4_3,5_2
     * 2    1_4,2_6,3_4,4_6,5_1
     * <p>
     * 说明：矩阵与矩阵相乘（左行X右列），考虑到hadoop是按行读取，所以需要先将右矩阵进行转置，变成（左行X右行）
     */
    @Override
    protected void map(LongWritable key, Text values, Context context) throws IOException, InterruptedException {
        //按行获取内容，每次读取一行（元素与元素之间以tab键分割）；
        String[] rowAndLines = values.toString().split("\t");
        //行号
        String row = rowAndLines[0];
        //每行内容
        String[] lines = rowAndLines[1].split(",");
        //循环输出内容 key:列号   value:行号_值，行号_值，行号_值，行号_值
        for (String line : lines) {
            //获取列号
            String colunm = line.split("_")[0];
            //获取每列的值
            String value = line.split("_")[1];
            outKey.set(colunm);
            outValue.set(row + "_" + value);
            //输出
            context.write(outKey, outValue);
        }
    }
}
```

Reducer阶段：

```java
public class ReduceMatrixTranspose extends Reducer<Text, Text, Text, Text> {
    private Text outKey = new Text();
    private Text outValues = new Text();

    /**
     * @param key     输入的列号
     * @param values  行号 值，行号 值
     * @param context
     * @throws IOException
     * @throws InterruptedException
     */
    @Override
    protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
        StringBuilder sb = new StringBuilder();
        for (Text text : values) {
            sb.append(text).append(",");
        }
        String result = null;
        if (sb.toString().endsWith(",")) {
            result = sb.substring(0, sb.length() - 1);
        }

        outKey.set(key);
        outValues.set(Objects.requireNonNull(result));
        context.write(outKey, outValues);
    }
}

```
Main：
```java
public class Transpose {

    /**
     * hdfs地址
     */
    private static String hdfs = "hdfs://localhost:9000";

    public static void main(String[] args) {
        int result = -1;
        // 创建conf配置
        Configuration conf = new Configuration();
        // 设置hdfs地址
        conf.set("fs.defaultFS", hdfs);
        // 创建job实例
        try {
            Job job = Job.getInstance(conf, "step1");
            //设置job主类
            job.setJarByClass(Transpose.class);
            //设置job的map类与reduce类
            job.setMapperClass(MapMatrixTranspose.class);
            job.setReducerClass(ReduceMatrixTranspose.class);
            //设置mapper输出类型
            job.setMapOutputKeyClass(Text.class);
            job.setMapOutputValueClass(Text.class);
            //设置reduce输出类型
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(Text.class);
            FileSystem fs = FileSystem.get(conf);
            //设置输入、输出路径
            FileInputFormat.addInputPath(job, new Path(args[0]));
            FileOutputFormat.setOutputPath(job, new Path(args[1]));
            if (job.waitForCompletion(true)) {
                System.out.println("matrix transpose success");
            } else {
                System.out.println("matrix transpose fail");
            }
        } catch (Exception e) {
            System.out.println("执行异常" + e.getMessage());
        }
    }
}
```
然后将代码打成jar包，将左矩阵的数据放入到hdfs中，运行hadoop命令。
```
hadoop jar matrix.jar 文件路径 输出路径
```
在查看输出结果
```java
hadoop fs -cat 输出结果的文件
```
得出结果
```java
1	3_-4,2_3,1_1
2	3_2,2_-1,1_2
3	3_1,2_0,1_3
```



第二步：AxB'(B'的结果放在hdfs的文件系统缓存中)，输出结果
Map阶段：
这里从分布式缓存中读取了右矩阵right_matrix的值，这个别名是在main方法里面设置的，用法可以参考：[]()
```java
public class MapMatrixMultiply extends Mapper<LongWritable, Text, Text, Text> {

    private Text outKey = new Text();
    private Text outValue = new Text();
    private List<String> cacheList = new ArrayList<>();

    /**
     * 初始化方法
     * 会在map方法之前执行一次，且只执行一次
     * 作用：将转置的右侧矩阵放在list中
     */
    @Override
    protected void setup(Mapper<LongWritable, Text, Text, Text>.Context context) throws IOException, InterruptedException {
        super.setup(context);
        // 读取缓存文件中的内容
        try (FileReader fr = new FileReader("right_matrix");
             BufferedReader br = new BufferedReader(fr)) {
            String line;
            while ((line = br.readLine()) != null) {
                cacheList.add(line);
            }
        }
    }

    /**
     * map实现方法
     * key:行
     * values:行 tab 列_值,列_值,列_值,列_值
     */
    @Override
    protected void map(LongWritable key, Text values, Context context)
            throws IOException, InterruptedException {
        // 左矩阵
        String rowMatrix1 = values.toString().split("\t")[0];
        String[] columnValueArrayMatrix1 = values.toString().split("\t")[1].split(",");
        for (String line : cacheList) {
            // 右矩阵行数据

            String rowMatrix2 = line.split("\t")[0];
            String[] columnValueArrayMatrix2 = line.toString().split("\t")[1].split(",");

            int result = 0;
            // 遍历左矩阵每一行的每一列
            for (String columnValueMatrix1 : columnValueArrayMatrix1) {
                String columnMatrix1 = columnValueMatrix1.split("_")[0];
                String columnValue1 = columnValueMatrix1.split("_")[1];
                for (String columnValueMatrix2 : columnValueArrayMatrix2) {
                    // 判断前缀相同，进行相乘
                    if (columnValueMatrix2.startsWith(columnMatrix1 + "_")) {
                        String columnValue2 = columnValueMatrix2.split("_")[1];
                        result += Integer.valueOf(columnValue1) * Integer.valueOf(columnValue2);
                    }
                }
            }

            outKey.set(rowMatrix1);
            outValue.set(rowMatrix2 + "_" + result);
            context.write(outKey, outValue);
        }
    }
}
```

Reducer阶段：
```java
public class ReduceMatrixMultiply extends Reducer<Text, Text, Text, Text> {

    private Text outKey = new Text();
    private Text outValue = new Text();

    @Override
    protected void reduce(Text key, Iterable<Text> values, Context context)
            throws IOException, InterruptedException {
        StringBuilder sb = new StringBuilder();
        for (Text text : values) {
            sb.append(text).append(",");
        }
        String result = null;
        if (sb.toString().endsWith(",")) {
            result = sb.substring(0, sb.length() - 1);
        }

        outKey.set(key);
        outValue.set(Objects.requireNonNull(result));
        context.write(outKey, outValue);
    }
}
```
Main：
在这里设置了hdfs分布式缓存的路径，通过args[0]传入的，然后在map阶段进行了调用，#文件名，就是给它取的别名。
```java
public class Multiply {

    /**
     * hdfs地址
     */
    private static String hdfs = "hdfs://localhost:9000";


    public static void main(String[] args) {
        Configuration conf = new Configuration();

        conf.set("fs.defaultFS", hdfs);
        int result = -1;
        // 创建job实例
        try {
            Job job = Job.getInstance(conf, "matrix_multiply");
            //添加分布式缓存文件
            job.addCacheArchive(new URI(args[0] + "#right_matrix"));
            //设置job主类
            job.setJarByClass(Transpose.class);
            //设置job的map类与reduce类
            job.setMapperClass(MapMatrixMultiply.class);
            job.setReducerClass(ReduceMatrixMultiply.class);

            job.setMapOutputKeyClass(Text.class);
            job.setMapOutputValueClass(Text.class);

            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(Text.class);

            FileSystem fs = FileSystem.get(conf);
            //设置 输入、输出路径
            FileInputFormat.addInputPath(job, new Path(args[1]));
            FileOutputFormat.setOutputPath(job, new Path(args[2]));

            if (job.waitForCompletion(true)) {
                System.out.println("matrix transpose success");
            } else {
                System.out.println("matrix transpose fail");
            }
        } catch (Exception e) {
            System.out.println("执行异常" + e.getMessage());
        }
    }
}
```
运行代码
```java
hadoop jar matrix2.jar 分布式缓存地址 输入地址 输出地址
```
得到结果,大功告成：
```java
1	3_2,2_-2,1_11
2	3_9,2_9,1_-7
3	3_1,2_-1,1_5
```
