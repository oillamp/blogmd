---
title: react_native_android_6.0
categories: react_native
date: 2016-10-10 19:59:24
timestamp:
catAlias:
tags:
keywords:
description:
---
# react native android 6.0 动态权限申请

## android 6.0 动态权限申请

>android 在sdk>=23(6.0/M)之后，对权限问题进行了进一步的收缩，在4.x,5.x 那个年代，只需要在__AndroidManifest.xml__中声明需要的权限即可，而在M之后，TMD竟然崩溃了.


言归正传，回到6.0动态权限申请，需要三个步骤:

以下代码按照申请录音权限为例


* 权限检测

```
if(ContextCompat.checkSelfPermission(this, 
        	Manifest.permission.RECORD_AUDIO)!=PackageManager.PERMISSION_GRANTED){
 }
```

* 权限申请

这里需要注意一下，__RECORD_AUDIO_REQUEST_CODE__即__requestCode__, 是自定义的int类型数据，在第三步会用到

第二个参数是权限列表，支持申请多个。


到这里，程序运行过程中，就会弹出"要允许xxx录制音频吗?"的alert,点击允许，即为app授权成功。


* 授权回调

重写Activity的__onRequestPermissionsResult__方法，

这样，就完成了在6.0上的权限申请。

>完整代码

```
public class MainActivity extends Activity{



    public static final int RECORD_AUDIO_REQUEST_CODE = 666;


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

    }

	public void onPress() {
	        
       if(ContextCompat.checkSelfPermission(this, 
        	Manifest.permission.RECORD_AUDIO)!=PackageManager.PERMISSION_GRANTED) {
			// donot have permission 
           
           ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.RECORD_AUDIO}, RECORD_AUDIO_REQUEST_CODE);
           
         } else {
            // okay, 
            doBusiness();
         }
	}

   public voic doBusiness() {
      // do business
   
   }
   
   @Override
   public void onRequestPermissionsResult(
       int requestCode, //第二步填写的requestcode
       String[] permissions, //全县列表
       int[] grantResults // 授权结果列表
    ){
    switch (requestCode) {
    case RECORD_AUDIO_REQUEST_CODE:
         if (grantResults[0] == PackageManager.PERMISSION_GRANTED) {
             // Permission Granted
             doBusiness(true);
         } else {
             // Permission Denied
                    
         }
         break;
     default:
         // default
         break;
       }
     }

   //...

}

```



## react native android 6.0 上进行动态权限申请
 
 还是按照语音权限申请为例
 
 自己写了一个native module，作为bridge，为js提供native的功能，这个时候，遇上的权限问题。
 
 native module集成自ReactContextBaseJavaModule 可以进行上面前两步骤，但是没有回调，所以这样是不行的。
 
 在源码中ack了一下，发现一个有趣的东西:
 
 ```
 ➜  react-native ack -r 'onRequestPermissionsResult'
ReactAndroid/src/androidTest/java/com/facebook/react/testing/ReactAppTestActivity.java
252:  public void onRequestPermissionsResult(

ReactAndroid/src/main/java/com/facebook/react/modules/core/PermissionListener.java
22:   * {@link Activity#onRequestPermissionsResult}.
26:  boolean onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults);

ReactAndroid/src/main/java/com/facebook/react/modules/permissions/PermissionsModule.java
127:  public boolean onRequestPermissionsResult(

ReactAndroid/src/main/java/com/facebook/react/ReactActivity.java
199:  public void onRequestPermissionsResult(
204:        mPermissionListener.onRequestPermissionsResult(requestCode, permissions, grantResults)) {

 ```
 
显然，rn对这个问题是有考虑的，于是乎，在这里得到了答案
```
ReactAndroid/src/main/java/com/facebook/react/modules/permissions/PermissionsModule.java
```

步骤如下:

* 同样首先，进行权限检测

```
if (ContextCompat.checkSelfPermission(getCurrentActivity(), Manifest.permission.RECORD_AUDIO)
                        != PackageManager.PERMISSION_GRANTED) {
            //申请RECORD_AUDIO权限
            PermissionAwareActivity activity = getPermissionAwareActivity();
            activity.requestPermissions(new String[]{Manifest.permission.RECORD_AUDIO}, RECORD_AUDIO_REQUEST_CODE, this);
			// donot have permission 

        } else {
            doBusiness(false);
        }
```


* 权限申请

这里需要对该native module 增加一个实现 __implements PermissionListener__ 并实现回调。

```
public boolean onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults)
```

权限申请，并对currentactivity进行类型强转

```
PermissionAwareActivity activity = (PermissionAwareActivity) getCurrentActivity();
 activity.requestPermissions(new String[]{Manifest.permission.RECORD_AUDIO}, RECORD_AUDIO_REQUEST_CODE, this);

```

* 回调
  需要注意的是，也需要给当前activity增加一个实现__implements PermissionAwareActivity__
  
  并实现方法, 该方法会在第二步函数调用之后，被执行到
 
```
 
public void requestPermissions(String[] permissions, int requestCode, PermissionListener listener){
	// 保存listener
   mPermissionListener = listener;
	// 这里进行权限申请，通普通权限申请一样
   ActivityCompat.requestPermissions(this, permissions, requestCode);
 }
 
```

到这里，会弹出"要允许xxx录制音频吗?"的alert， 点击允许，为app授权

* actvity 回调

对activity实现授权回调

```
public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
     // 使用listener回调  
      mPermissionListener.onRequestPermissionsResult(requestCode, permissions, grantResults);
}
```
 
 
 这样，就完成了在 react native android 6.0 中的动态权限申请
>完整代码

`MainActivity.java`

```
public class MainActivity extends Activity implements PermissionAwareActivity{

    private PermissionListener mPermissionListener;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

    }

   @Override
    public void requestPermissions(String[] permissions, int requestCode, PermissionListener listener) {
	    // request permissions
        mPermissionListener = listener;
        ActivityCompat.requestPermissions(this, permissions, requestCode);
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        // callback to native module 
        mPermissionListener.onRequestPermissionsResult(requestCode, permissions, grantResults);
    }
   
   //...

}

```

`test.java`

```

public class Test extends ReactContextBaseJavaModule implements PermissionListener {

    
    public static final int RECORD_AUDIO_REQUEST_CODE = 666;

    public Test(ReactApplicationContext reactContext) {
        super(reactContext);
        mReactContext = reactContext;
    }
    
    @Override
    public String getName() {
        return "Test";
    }

    @ReactMethod
    public void startRecord(
            Callback callback) {

        mCallback = callback;
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M &&
                ContextCompat.checkSelfPermission(getCurrentActivity(), Manifest.permission.RECORD_AUDIO)
                        != PackageManager.PERMISSION_GRANTED) {
            //申请RECORD_AUDIO权限
            PermissionAwareActivity activity = getPermissionAwareActivity();
            activity.requestPermissions(new String[]{Manifest.permission.RECORD_AUDIO}, RECORD_AUDIO_REQUEST_CODE, this);

        } else {
            doBeginRecord();
        }

    }
    
    public void doBeginRecord(){
      // do record business   
    
    };
    
    @Override
    public boolean onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        switch (requestCode) {
            case RECORD_AUDIO_REQUEST_CODE:
                if (grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    // Permission Granted
                    doBeginRecord(true);
                } else {
                    // Permission Denied
                }
                break;
            default:
                // default
                break;
        }
        return false;
    }

    private PermissionAwareActivity getPermissionAwareActivity() {
        Activity activity = getCurrentActivity();
        if (activity == null) {
            throw new IllegalStateException("Tried to use permissions API while not attached to an " +
                    "Activity.");
        } else if (!(activity instanceof PermissionAwareActivity)) {
            throw new IllegalStateException("Tried to use permissions API but the host Activity doesn't" +
                    " implement PermissionAwareActivity.");
        }
        return (PermissionAwareActivity) activity;
    }
    
}

```
 
 
## 一点优化

动态权限申请对于6.0一下的android os是没有作用的，所以，在权限检测是，可以对sdk 进行版本判断, 在if条件中加上以下判断即可

```
if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.M){
	// >= 6.0
} else {
 	// others
}
```
