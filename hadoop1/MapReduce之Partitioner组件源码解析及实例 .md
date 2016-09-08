####简述
Partitioner组建可以让**Map**和**Key**进行分区，从而可以根据不同的key来分发到不同的reduce中去处理；

你可以自定义key的一个分发规则，如数据文件包含不同的大学，而输出的要求是每个大学输出一个文件；

Partitioner组建提供了一个默认的HashPartitioner。

    package org.apache.hadoop.mapreduce.lib.partition;
    public class HashPartitioner<K, V> extends Partitioner<K, V> {

      /** Use {@link Object#hashCode()} to partition. */
      public int getPartition(K key, V value,
                              int numReduceTasks) {
        return (key.hashCode() & Integer.MAX_VALUE) % numReduceTasks;
      }
    }
####自定义Partitioner
1、继承抽象类Partitioner，实现自定义的getPartition（）方法；

2、通过job.setPartitionerClass（..）来设置自定义的Partitioner；

Partitioner类

    package org.apache.hadoop.mapreduce;
    public abstract class Partitioner<KEY, VALUE> {

      /** 
       * Get the partition number for a given key (hence record) given the total 
       * number of partitions i.e. number of reduce-tasks for the job.
       *   
       * <p>Typically a hash function on a all or a subset of the key.</p>
       *
       * @param key the key to be partioned.
       * @param value the entry value.
       * @param numPartitions the total number of     partitions.
       * @return the partition number for the <code>key</  code>.
       */
      public abstract int getPartition(KEY key, VALUE value, int numPartitions);

    }

####Partitioner应用场景及实例
需求：分别统计每种商品的周销售情况

address1的周销售清单（input1）：

    shoes 20
    hat 10
    stockings 30
    clothes 40
address2的周销售清单（input2）：

    shoes 15
    hat 1
    stockings 90
    clothes 80
汇总结果（output）：

    shoes 35
    hat 11
    stockings 120
    clothes 120
    package MyPartitioner;

     import java.io.IOException;
    import java.net.URI;

    import org.apache.hadoop.conf.Configuration;
    import org.apache.hadoop.fs.FileSystem;
    import org.apache.hadoop.fs.Path;
    import org.apache.hadoop.io.IntWritable;
    import org.apache.hadoop.io.LongWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Job;
    import org.apache.hadoop.mapreduce.Mapper;
    import org.apache.hadoop.mapreduce.Partitioner;
    import org.apache.hadoop.mapreduce.Reducer;
    import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
    import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;





    public class MyPartitioner {
        private final static String INPUT_PATH = "hdfs://liguodong:8020/input";
        private final static String OUTPUT_PATH = "hdfs://liguodong:8020/output";

        public static class MyMapper extends Mapper<LongWritable, Text, Text, IntWritable>{
        private Text word = new Text();
        private IntWritable one = new IntWritable();

        @Override
        protected void map(LongWritable key, Text value, Context context)
                throws IOException, InterruptedException {

                String[] str = value.toString().split("\\s+");

                word.set(str[0]);
                one.set(Integer.parseInt(str[1]));
                context.write(word, one);
            }
        }


        public static class MyReducer extends Reducer<Text, IntWritable,Text, IntWritable>{
            private IntWritable result = new IntWritable();

            @Override
            protected void reduce(Text key, Iterable<IntWritable> values,
                    Context context)
                    throws IOException, InterruptedException {
 
                int sum = 0;
                for (IntWritable val : values) {
                    sum+=val.get();
                }
                result.set(sum);
                context.write(key,result);
            }   
        }

        public static class DefPartitioner extends Partitioner<Text,IntWritable>{

            @Override
            public int getPartition(Text key, IntWritable value, int numPartitions) {
                if(key.toString().equals("shoes")){
                    return 0;

                }else if(key.toString().equals("hat")){
                    return 1;

                }else if(key.toString().equals("stockings")){
                    return 2;

                }else{
                    return 3;
                }
            }

        }



        public static void main(String[] args) throws Exception {
            //1、配置  
            Configuration conf = new Configuration();
            final FileSystem fileSystem = FileSystem.get(new URI(INPUT_PATH),conf);
            if(fileSystem.exists(new Path(OUTPUT_PATH)))
            {
                fileSystem.delete(new Path(OUTPUT_PATH),true);
            }
            Job job = Job.getInstance(conf, "define partitioner"); 

            //2、打包运行必须执行的方法
            job.setJarByClass(MyPartitioner.class);

            //3、输入路径
            FileInputFormat.addInputPath(job, new Path(INPUT_PATH));  
            //4、Map
            job.setMapperClass(MyMapper.class);

            //5、Combiner
            //job.setCombinerClass(MyReducer.class);
            job.setPartitionerClass(DefPartitioner.class);

            //6、Reducer
            job.setReducerClass(MyReducer.class);
            job.setNumReduceTasks(4);//reduce个数默认是1

            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(IntWritable.class);

            //7、 输出路径
            FileOutputFormat.setOutputPath(job, new Path   (OUTPUT_PATH));

            //8、提交作业
            System.exit(job.waitForCompletion(true) ? 0 : 1);
        }  
    }

####

    [root@liguodong file]# hdfs dfs -mkdir /input
    上传文件
    [root@liguodong file]# hdfs dfs -put input1 /input/
    [root@liguodong file]# hdfs dfs -put input2 /input/
    [root@liguodong file]# hdfs dfs -ls  /input/
    Found 2 items
     -rw-r--r--   1 root supergroup         52 2015-06-14 10:22 /input/input1
     -rw-r--r--   1 root supergroup         50 2015-06-14 10:22 /input/input2
    
    打成jar包，然后执行。
    [root@liguodong file]# jar tf partitioner.jar
    META-INF/MANIFEST.MF
    MyPartitioner/MyPartitioner$DefPartitioner.class
    MyPartitioner/MyPartitioner$MyMapper.class
    MyPartitioner/MyPartitioner$MyReducer.class
    MyPartitioner/MyPartitioner.class
    
    [root@liguodong file]# yarn jar partitioner.jar com.MyPartitioner
	注释:com.MyPartitioner是eclipse项目路径和类名
    
    输出结果
    [root@liguodong file]# hdfs dfs -ls /output/
    Found 5 items
     -rw-r--r--   1 root supergroup          0 2015-06-14 11:08 /output/_SUCCESS
     -rw-r--r--   1 root supergroup          9 2015-06-14 11:08 /output/part-r-00000
     -rw-r--r--   1 root supergroup          7 2015-06-14 11:08 /output/part-r-00001
     -rw-r--r--   1 root supergroup          0 2015-06-14 11:08 /output/part-r-00002
     -rw-r--r--   1 root supergroup         26 2015-06-14 11:08 /output/part-r-00003
    [root@liguodong file]# hdfs dfs -cat /output/part-r-00000
    shoes   35
    [root@liguodong file]# hdfs dfs -cat /output/part-r-00001
    hat     11
    [root@liguodong file]# hdfs dfs -cat /output/part-r-00002
    stockings       120
    [root@liguodong file]# hdfs dfs -cat /output/part-r-00003
    clothes 120