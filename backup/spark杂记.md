# spark运行框架
![运行框架](https://upload-images.jianshu.io/upload_images/20370955-0280b1215106605d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. 提交一个spark应用时候会对应生成一个driver进程
2. 注册spark任务到cluster manager 申请需要的资源
3. cluster manager 协调多个work申请需要的资源
（例如申请三个实例每个1个cpu和300m的内存就会申请到3个executor）
4. 申请的executor反向注册到driver 使driver和executor一起运行spark程序
# spark运行时
![spark运行时](https://upload-images.jianshu.io/upload_images/20370955-2424085d62c32362.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
spark job 是 appliction
![appliction](https://upload-images.jianshu.io/upload_images/20370955-58f039dce81a3279.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. 一个appliction又可以划分多个job
2. 划分依据是类似collect和save的Action算子
3. job之际划分多个stage
4. 划分依据就是groupbykey宽依赖shuffle算子
5. stage划分多个task
6. task是spark运行的最小调度单元