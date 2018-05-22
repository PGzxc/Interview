# 我的Android面试之路
原文出自： [我的Android面试之路][1]  

# 前言
最近在找Android开发的工作，遇到了一些面试的问题，加上朋友面试遇到的一些问题，感觉有必要在此记录一下，也分享给大家，说不定还有用呢。

# 面试题

## 1、线程同步问题：有3个线程abc，循环输出十次abc
这个问题简单的就是使用AtomicInteger来实现，具体实现方式如下：  

	private AtomicInteger sycValue = new AtomicInteger(0);
	private static final int MAX_SYC_VALUE = 3 * 10;
	public void test() 
	{
    	new Thread(new RunnableA()).start();
    	new Thread(new RunnableB()).start();
    	new Thread(new RunnableC()).start();
	}

	private class RunnableA implements Runnable 
	{
    	public void run() 
		{
        	while (sycValue.get() < MAX_SYC_VALUE) 
			{
            	if (sycValue.get() % 3 == 0) 
				{
                	System.out.println(String.format("第%d遍", sycValue.get() / 3 + 1));
                	System.out.println("A");
                	sycValue.getAndIncrement();
            	}
        	}
    	}
	}

	private class RunnableB implements Runnable 
	{
    	public void run() 
		{
        	while (sycValue.get() < MAX_SYC_VALUE) 
			{
            	if (sycValue.get() % 3 == 1) 
				{
                	System.out.println("B");
                	sycValue.getAndIncrement();
            	}
        	}
		}
	}

	private class RunnableC implements Runnable 
	{
    	public void run() 
		{
        	while (sycValue.get() < MAX_SYC_VALUE) 
			{
            	if (sycValue.get() % 3 == 2) 
				{
                	System.out.println("C");
                	System.out.println();
                	sycValue.getAndIncrement();
            	}
        	}
    	}
	}
## 2、多线程下载同一个文件
这个问题主要分三步来考虑：  

1. 获取文件总长度，通过总长度来确定开启几个线程下载，并不是线程越多就越好。   
2. 分段下载文件，确定线程数后，每个线程下载相应的数据。    
3. 将分段下载的文件拼接为一个完整的文件。

---
			

	private void test() 
	{
      	new Thread(new Runnable() 
	    {
          @Override
          public void run() 
		 {
              System.out.println("开始下载");
              download("https://a-ssl.duitang.com/uploads/item/201307/11/20130711155049_yhiWQ.jpeg", 3);
          }
      }).start();
	}
	pivate void download(String path, int threadNum) 
	{
    try {
        URL url = new URL(path);
        HttpURLConnection connection = (HttpURLConnection) url.openConnection();
        connection.setRequestMethod("GET");
        connection.setConnectTimeout(5 * 1000);
        //获取文件长度
        int length = connection.getContentLength();
        //计算每个线程下载长度
        int block = (length % threadNum) == 0 ? length / threadNum : length / threadNum + 1;
        if (connection.getResponseCode() == HttpURLConnection.HTTP_OK) {
            File file = new File(Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS),
                    path.substring(path.lastIndexOf("/") + 1));
            for (int i = 0; i < threadNum; i++)
                new DownThread(i, file, block, url).start();
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
	}
	public class DownThread extends Thread {
    /**
     * 线程序号id
     */
    private int id;
    /**
     * 目标文件
     */
    private File file;
    /**
     * 每个线程下载文件的长度
     */
    private int block;
    /**
     * 下载地址
     */
    private URL url;

    public DownThread(int id, File file, int block, URL url) {
        this.id = id;
        this.file = file;
        this.block = block;
        this.url = url;
    }

    @Override
    public void run() {
        int start = (id * block);// 当前线程开始下载处
        int end = (id + 1) * block - 1;// 当前线程结束下载处
        try {
            RandomAccessFile randomAccessFile = new RandomAccessFile(file, "rwd");
            randomAccessFile.seek(start);
            HttpURLConnection connection = (HttpURLConnection) url.openConnection();
            connection.setConnectTimeout(5 * 1000);
            connection.setRequestMethod("GET");
            // 指定网络位置从什么位置开始下载,到什么位置结束
            connection.setRequestProperty("Range", "bytes=" + start + "-" + end);
            InputStream in = connection.getInputStream();
            byte[] data = new byte[1024];
            int len = 0;
            while ((len = in.read(data)) != -1) {
                randomAccessFile.write(data, 0, len);
            }
            in.close();
            randomAccessFile.close();
            System.out.println("线程" + id + "下载完成");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
	}

## 3、网页端生成二维码，手机端扫描登录
这个问题需要服务器端的配合，这里就讲一讲大概的思路了。

1. 网页端生成二维码，同时长连接等待服务器响应。
2. 手机端扫描二维码，然后判断手机端是否登录，如果没有就先登录，然后通知服务器扫描结果。
3. 网页端收到服务器响应，显示已经扫描待手机端确认登录，同时手机端显示确认界面。
4. 手机端确认或取消登录，同时通知服务器结果，网页端收到服务器响应，同时显示相应结果。

## 4、广播的种类和不同之处
Android系统中有3种广播，话说当时只知道前面3种常用的，后来回来查资料才发现有5种。  

1. 普通广播（Normal Broadcast）
2. 系统广播（System Broadcast）
3. App应用内广播（Local Broadcast）
4. 有序广播（Ordered Broadcast）
5. 粘性广播（Sticky Broadcast）

### 普通广播（Normal Broadcast）
普通广播是完全异步的，可以在同一时刻（逻辑上）被所有广播接收者接收到，消息传递的效率比较高，广播接收者中注册时intentFilter的action与广播匹配，才会接收到此广播。发送普通广播使用的是Context.sendBroadcast()。
### 系统广播（System Broadcast）
Android中内置了多个系统广播，只要涉及到手机的基本操作（如开机、网络状态变化、拍照等等），都会发出相应的广播，每个广播都有特定的Intent - Filter（包括具体的action）。当使用系统广播时，只需要在注册广播接收者时定义相关的action即可，并不需要手动发送广播，当系统有相关操作时会自动进行系统广播。
### App应用内广播（Local Broadcast
App应用内广播可理解为一种局部广播，广播的发送者和接收者都同属于一个App。相比于全局广播（普通广播），App应用内广播优势体现在：安全性高 & 效率高。使用方式上与普通广播几乎相同，只是注册/取消注册广播接收器和发送广播时将参数的Context变成了LocalBroadcastManager的单一实例。并且只能通过LocalBroadcastManager动态注册，不能静态注册。

	//动态注册 
	LocalBroadcastManager.getInstance(this).registerReceiver(mBroadcastReceiver, intentFilter);
	//取消注册
	LocalBroadcastManager.getInstance(this).unregisterReceiver(mBroadcastReceiver);
	//发送应用内广播
	LocalBroadcastManager.getInstance(this).sendBroadcast(intent);
### 有序广播（Ordered Broadcast）
有序广播是按照接收者声明的优先级别（声明在intent-filter元素的android:priority属性中，数越大优先级别越高,取值范围:-1000到1000。也可以调用IntentFilter对象的setPriority()进行设置），被接收者依次接收广播。如：A的级别高于B,B的级别高于C,那么，广播先传给A，再传给B，最后传给C。A得到广播后，可以往广播里存入数据，当广播传给B时,B可以从广播中得到A存入的数据。

	//发送有序广播
	Context.sendOrderedBroadcast(intent)
	//终止广播
	BroadcastReceiver.abortBroadcast()
### 粘性广播（Sticky Broadcast）
粘性广播（Sticky Broadcast）
## 5、移动支付
话说现在移动支付还是用得挺常见的，正好也弄过常见的微信支付、支付宝支付和银联支付，总体来说使用起来并不会有什么难度，各家的SDK都是介绍得很详细的，这里就贴一下各家的开发文档地址。（传送门：[微信支付][2]、[支付宝支付][3]、[银联支付][4]）
## 6、死锁的四个条件
1. 互斥条件：进程对所分配到的资源不允许其他进程进行访问，若其他进程访问该资源，只能等待，直至占有该资源的进程使用完成后释放该资源
2. 请求和保持条件：进程获得一定的资源之后，又对其他资源发出请求，但是该资源可能被其他进程占有，此事请求阻塞，但又对自己获得的资源保持不放
3. 不可剥夺条件：是指进程已获得的资源，在未完成使用之前，不可被剥夺，只能在使用完后自己释放
4. 环路等待条件：是指进程发生死锁后，必然存在一个进程--资源之间的环形链

### 处理死锁的基本方法
- 预防死锁：通过设置一些限制条件，去破坏产生死锁的必要条件
- 避免死锁：在资源分配过程中，使用某种方法避免系统进入不安全的状态，从而避免发生死锁
- 检测死锁：允许死锁的发生，但是通过系统的检测之后，采取一些措施，将死锁清除掉
- 解除死锁：该方法与检测死锁配合使用

## 7、Object的公共方法
众所周知Java中所有的类都是继承于Object，所以我们编写的类默认都具有这些方法，包括有hashCode()、wait()、notify()、notifyAll()、equals()、getClass()、toString()、clone()、finalize()等。

- hashCode()方法简单地说就是返回一个integer类型的值，这个值是通过该Object的内部地址(internal address)转换过来的，这个哈希码是可以通过getClass()方法看到具体值的，显示的是十六进制的数，有时候可以通过此方法来判断对象的引用是否相等。
- wait()用于多线程中，使当前线程等待该对象的锁，当前线程必须是该对象的拥有者，也就是具有该对象的锁。
- notify用于唤醒在该对象上等待的某个线程，notifyAll()用于唤醒在该对象上等待的所有线程。
- equals()用于判断这两个引用是否指向的是同一个对象。
- getClass()用户获取该对象的运行时类的java.lang.Class 对象。
- toString()方法返回一个字符串，它的值等于：getClass().getName()+ '@' + - - ------   Integer.toHexString(hashCode())。
- clone()保护方法，实现对象的浅复制，只有实现了Cloneable接口才可以调用该方法，否则抛出CloneNotSupportedException异常。
- finalize()用于当垃圾回收器确定不存在对该对象的更多引用时，由对象的垃圾回收器调用此方法。子类重写finalize 方法，以配置系统资源或执行其他清除。

















[1]: https://www.jianshu.com/p/5ec5764cd9f1
[2]: https://pay.weixin.qq.com/wiki/doc/api/app/app.php?chapter=8_1
[3]: https://doc.open.alipay.com/docs/doc.htm?treeId=204
[4]: https://merchant.unionpay.com/join/product/detail?id=3