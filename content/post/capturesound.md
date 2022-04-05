+++
title = "C#-录音"
date = 2022-04-05T12:16:35+08:00
author = "hansomxue99"
keywords = ["C#"]
cover = ""
summary = "利用C#实现声卡录音的驱动代码"

+++

1. `using Microsoft.DirectX.DirectSound;` 

2. ```c#
   Capture;		//音频捕捉设备
   CaptureBuffer;	//缓冲区对象
   WaveFormat;		//录音格式
   ```

   

3. 设置录音参数

   ```c#
   private WaveFormat mWaveFormat;
   
   private WaveFormat CreateWaveFormat()
   {
       WaveFormat waveformat = new WaveFormat();
       waveformat.FormatTag = WaveFormatTag.Pcm;	//音频类型设为PCM
       waveformat.SamplesPerSecond = 11025;		//采样率典型值：11025、22050、44100Hz(为了简化程序，暂且设为定值)
       waveformat.BitsPerSample = 16;				//采样位数  
       waveformat.Channels = 2;					//声道：1单声道，2立体声  
       waveformat.BlockAlign = (short)(waveformat.Channels * (waveformat.BitsPerSample / 8));//单位采样点的字节数  
       waveformat.AverageBytesPerSecond = waveformat.BlockAlign * waveformat.SamplesPerSecond;
       return waveformat;
       // 按照以上采样规格，可知采样1秒钟的字节数为 11025*2=32000B 约为22K
   }
   
   //mWaveFormat = CreateWaveFormat();
   ```

   

4. 初始化录音设备

   ```c#
   private Capture mCapDev = null;		//音频捕捉设备初始设为NULL
   
   private bool CreateCaptureDevice()
   {
       //枚举可用的录音设备  
       CaptureDevicesCollection capturedev = new CaptureDevicesCollection();
       Guid devguid = Guid.Empty;
       if (capturedev.Count > 0)
       	devguid = capturedev[0].DriverGuid; 	//0选择默认录音设备
       else
       	//MessageBox.Show("当前没有可用于音频捕捉的设备", "系统提示");
       	return false;
   
       //利用设备GUID来建立一个捕捉设备对象  
       mCapDev = new Capture(devguid);
       return true;
   }
   
   //CreateCaptureDevice();
   ```

   

5. 创建录音缓存区

   ```c#
   private CaptureBuffer captureBuffer = null;//录音缓冲区 
   private WaveFormat mWaveFormat;
   
   private int iNotifyNum ;		//通知的个数  
   private int iNotifySize = 0;	//通知所在区域大小 
   private int iBufferSize;
   
   private void CreateCaptureBuffer( int bufferNum, double bufferSizeSeconds )
   {
       iNotifyNum = bufferNum;
       //缓冲区的描述对象
       CaptureBufferDescription bufferDesc = new CaptureBufferDescription();
       bufferDesc.Format = mWaveFormat;			//1. 设置录音数据格式  
       iNotifySize =(int)( mWaveFormat.SamplesPerSecond * bufferSizeSeconds* mWaveFormat.Channels);
       iBufferSize = iNotifyNum * iNotifySize;
       bufferDesc.BufferBytes = iBufferSize;	//2. 设置录音缓存大小
       captureBuffer = new CaptureBuffer(bufferDesc, mCapDev);	//3. 创建录音缓冲区  
   }
   
   //CreateCaptureBffer(bufferNum, bufferSizeSeconds);
   ```

   

6. 创建缓存区录制通知

   ```c#
   private Notify captureNotify = null;			//缓冲区录制通知
   private AutoResetEvent notifyevent = null;		//通知事件
   
   private int iNotifyNum ;		//通知的个数  
   private int iNotifySize = 0;	//通知所在区域大小 
   
   private void CreateNotification()
   {
       BufferPositionNotify[] bpn = new BufferPositionNotify[iNotifyNum];		//缓存位置通知点数组  
       notifyevent = new AutoResetEvent(false); 	//同步事件对象，初始化
   
       for (int i = 0; i < iNotifyNum; i++)
       {
           bpn[i].Offset = (i + 1) * iNotifySize - 1;//设置缓冲区位置，到达该位置发出通知  
           bpn[i].EventNotifyHandle = notifyevent.Handle;//设置通知事件，该事件发出通知
       }
       captureNotify = new Notify(captureBuffer); //1. 创建captureBuffer录音缓存的通知
       captureNotify.SetNotificationPositions(bpn); //2. 设置多个通知点
   }
   ```

   

7. 以上工作都是前期准备，对录音设备的初始化，以下功能由用户自定义。

   （1）设备初始化

   ```c#
   public bool InitReciever(int bufferNums, double bufferSeconds)
   {
       CreateWaveFormat();		//1.
   	bool jud = CreateCaptureDevice();	//2.
       if(!jud)
           return jud;
       CreateCaptureBuffer(bufferNums, bufferSeconds);
       CreateNotification();
       return true;
   }
   ```

   （2）设备开始录制和停止

   ```c#
   private Thread notifythread = null;		//新建一个线程
   private CaptureBuffer captureBuffer = null;//录音缓冲区 
   
   public void recStart()
   {
       if (notifythread != null)
           notifythread.Abort();
       notifythread = new Thread(RecordData); 		//录音线程
   
       captureBuffer.Start(true);            
       notifythread.Start();            
   }
   
   public void recStop()
   {
       if( notifythread !=null )
           notifythread.Abort();
       captureBuffer.Stop();
   }
   ```

   

8. 录音涉及双线程 `notifythread = new Thread(RecordData);` 

   ```c#
   private void RecordData()
   {
       while (true)
       {
           // 等待缓冲区的通知消息  
           notifyevent.WaitOne();
           // 录制数据  
           RecordCapturedData();
       }
   }
   ```

   这里再次封装了一个数据记录函数

   ```c#
   private CaptureBuffer captureBuffer = null;//录音缓冲区 
   private int iLastPos = 0;//上一次读取数据位置 
   
   private void RecordCapturedData()
   {
       byte[] capturedata = null;		//字节数组存储捕获的数据
       int readpos = 0, capturepos = 0, locksize = 0;
   
       //获取录制数据位置
       captureBuffer.GetCurrentPosition(out capturepos, out readpos);
       locksize = readpos - iLastPos;//可以安全读取的数据大小  
       if (locksize == 0) return;
       if (locksize < 0)
           locksize += iBufferSize;//循环缓冲区，指针绕回
       //读取录制好的数据
       capturedata = (byte[])captureBuffer.Read(iLastPos, typeof(byte), LockFlag.FromWriteCursor, locksize);
       //处理接收到的数据
       process?.Invoke(capturedata);
   
       iLastPos += capturedata.Length;
       iLastPos %= iBufferSize;//取模是因为缓冲区是循环的。  
       iSampleSize += capturedata.Length;
   }
   ```

   

9. 值得一提的是，这里自定义了一个回调函数

   ```c#
   public delegate void OnRecieveData(byte[] model); //回调函数原型定义
   
   public OnRecieveData process;//接收数据处理代理：提供回调函数。收到数据时被调用
   ```

   