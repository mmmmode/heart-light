## 解决Android 7.0分享文件的FileUriExposedException问题.md

因为android 7.0以上Google对文件分享做了限制，分享文件会报FileUriExposedException 。所以需要配置来取得文件分享的权限。

### 第一步：配置manifest

***

```
        <provider
			//这里是provider的名字
            android:name="android.support.v4.content.FileProvider"
            //设置权限名字，这里我用包名加provider的方式
            android:authorities="com.uestc.mode.modetest.provider"
            //这里必须用false，否则会报错
            android:exported="false" 
            //能够请求一个临时的权限
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                //还有一个待配置的资源路径
                android:resource="@xml/file_paths" />
        </provider>

```
从FileProvider源码可知：其中exported 和 grantURIPermissions必须分别为false 和 true

```
   public void attachInfo(@NonNull Context context, @NonNull ProviderInfo info) {
        super.attachInfo(context, info);
        if (info.exported) {
            throw new SecurityException("Provider must not be exported");
        } else if (!info.grantUriPermissions) {
            throw new SecurityException("Provider must grant uri permissions");
        } else {
            this.mStrategy = getPathStrategy(context, info.authority);
        }
    }

```
在遇到问题的时候，读源码是一个好的解决办法

### 第二步：配置file_paths.xml

***

在res的xml下面配置file_paths.xml

![](https://github.com/mmmmode/heart-light/blob/master/img/android/%E8%A7%A3%E5%86%B3Android%207.0%E4%BB%A5%E4%B8%8A%E6%97%A0%E6%B3%95%E5%88%86%E4%BA%AB%E6%96%87%E4%BB%B6%E7%9A%84%E9%97%AE%E9%A2%98/1.png)

**file.xml**

```
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-path path="." name="files_root" />
</paths>
```
这里我设置了external-path，代表从存储卡中取路径。
因为我的文件存储目录是：getExternalFilesDir

```
 File file;
        File dir = new File(MainActivity.this.getExternalFilesDir(null).getPath());
        file = new File(dir, currentKeyword + ".xls");
        if (!dir.exists()) {
            dir.mkdirs();
        }       
```
完整的路径映射可以读FileProvider的源码，从源码可知：

```
if ("root-path".equals(tag)) {
    target = DEVICE_ROOT;
} else if ("files-path".equals(tag)) {
    target = context.getFilesDir();
} else if ("cache-path".equals(tag)) {
    target = context.getCacheDir();
} else if ("external-path".equals(tag)) {
    target = Environment.getExternalStorageDirectory();
} else {

```
>root-path 对应 根目录DEVICE_ROOT

>files-path 对应 getFilesDir();

>cache-path 对应 getCacheDir();

>external-path 对应Environment.getExternalStorageDirectory()

如果path设为.  代表访问目标目录下的所有资源。	
如果设置 image/,可以理解为 xxx/image/ 目录

### 第三步：Intent请求

```
    // 調用系統方法分享文件
    public void shareFile(Context context, File file) {
        if (null != file && file.exists()) {
            Intent share = new Intent(Intent.ACTION_SEND);
            share.setPackage("com.tencent.mobileqq");
            Uri uri;
            //分版本 24以上使用新方法，24以上用老方法
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
            	//这里的第二个参数必须和manifest的authorities一致
                uri = FileProvider.getUriForFile(this, "com.uestc.mode.modetest.provider", file);
                // 给目标应用一个临时授权
                share.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
            } else {
                uri = Uri.fromFile(file);
            }

            share.putExtra(Intent.EXTRA_STREAM, uri);
            share.setType("*/*");//此处可发送多种文件
            share.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            share.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
            context.startActivity(Intent.createChooser(share, "分享文件"));
        } else {
            Toast.makeText(MainActivity.this,"文件不存在",Toast.LENGTH_SHORT).show();
        }
    }
```

### 第四步：加文件读写动态权限 很关键

但是所有这些步骤都做完了之后，分享文件还是有错误，是为啥呢，当时在manifest中也配置了应用的读写权限，后面想了想觉得，既然Google都在高版本加了文件分享的限制，读写上面会不会也有限制。果然 在应用页面发现读写权限并不会默认给予，所以需要加入动态权限。

```
 ActivityCompat.requestPermissions(this,
                new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE}, 0);

        ActivityCompat.requestPermissions(this,
                new String[]{Manifest.permission.READ_EXTERNAL_STORAGE}, 0);
                
```
加入之后，就能分享了。






        

