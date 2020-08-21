# `APK`安装原理

[TOC]

## 一、`APK`有界面安装

`APK`有界面安装调用的是系统的安装应用 [`PackageInstaller`](https://www.androidos.net.cn/android/9.0.0_r8/xref/packages/apps/PackageInstaller/) ，以`Android 9.0`源码为例，其中 [清单文件](https://www.androidos.net.cn/android/9.0.0_r8/xref/packages/apps/PackageInstaller/AndroidManifest.xml) 部分内容如下：

```xml
<application android:name=".PackageInstallerApplication"
	android:label="@string/app_name"
	android:allowBackup="false"
 	android:theme="@style/DialogWhenLarge"
 	android:supportsRtl="true"
 	android:defaultToDeviceProtectedStorage="true"
 	android:directBootAware="true">
  <!-- ... -->
  <activity android:name=".InstallStart"
 		android:exported="true"
 	  android:excludeFromRecents="true">
    <intent-filter android:priority="1">
      <action android:name="android.intent.action.VIEW" />
      <action android:name="android.intent.action.INSTALL_PACKAGE" />
      <category android:name="android.intent.category.DEFAULT" />
      <data android:scheme="file" />
      <data android:scheme="content" />
      <!-- 这里配置了 application/vnd.android.package-archive 这个字符串的mime类型 -->
      <data android:mimeType="application/vnd.android.package-archive" />
    </intent-filter>
    <intent-filter android:priority="1">
      <action android:name="android.intent.action.INSTALL_PACKAGE" />
      <category android:name="android.intent.category.DEFAULT" />
      <data android:scheme="file" />
      <data android:scheme="package" />
      <data android:scheme="content" />
    </intent-filter>
    <intent-filter android:priority="1">
      <action android:name="android.content.pm.action.CONFIRM_PERMISSIONS" />
      <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
  </activity>
    <!-- ... -->
  <activity android:name=".PackageInstallerActivity"
 		 android:exported="false" />
  <activity android:name=".InstallSuccess"
  	android:theme="@style/DialogWhenLargeNoAnimation"
  	android:exported="false" />
</application>
<!-- ... -->
```

清单文件中中配置了 `application/vnd.android.package-archive`的`mime`类型，应用安装时通过`intent-filter`过滤器寻找并激活`InstallStart`这个`Activity`。

### 1.1 `InstallStart`

[InstallStart](https://www.androidos.net.cn/android/9.0.0_r8/xref/packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallStart.java) 会启动`PackageInstallActivity`，其部分内容如下：

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
  //... 111行
  if (PackageInstaller.ACTION_CONFIRM_PERMISSIONS.equals(intent.getAction())) {
    nextActivity.setClass(this, PackageInstallerActivity.class);
  } else {
    Uri packageUri = intent.getData();

    if (packageUri != null && (packageUri.getScheme().equals(ContentResolver.SCHEME_FILE)
                               || packageUri.getScheme().equals(ContentResolver.SCHEME_CONTENT))) {
      // Copy file to prevent it from being changed underneath this process
      nextActivity.setClass(this, InstallStaging.class);
    } else if (packageUri != null && packageUri.getScheme().equals(
      PackageInstallerActivity.SCHEME_PACKAGE)) { // SCHEME_PACKAGE = "package"
      nextActivity.setClass(this, PackageInstallerActivity.class);
    } else {
      Intent result = new Intent();
      result.putExtra(Intent.EXTRA_INSTALL_RESULT,
                      PackageManager.INSTALL_FAILED_INVALID_URI);
      setResult(RESULT_FIRST_USER, result);

      nextActivity = null;
    }
  }
  //...
}
```

### 1.2 `PackageInstallerActivity`

[`PackageInstallerActivity`](https://www.androidos.net.cn/android/9.0.0_r8/xref/packages/apps/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java) 会跳转到`InstallInstalling`，其部分内容如下：

```java
//点击按钮开始安装
public void onClick(View v) { //656行
  if (v == mOk) {
    if (mOk.isEnabled()) {
      if (mOkCanInstall || mScrollView == null) {
        if (mSessionId != -1) {
          mInstaller.setPermissionsResult(mSessionId, true);
          finish();
        } else {
          startInstall();  //开始安装
        }
      } else {
        mScrollView.pageScroll(View.FOCUS_DOWN);
      }
    }
  } else if (v == mCancel) {
    // Cancel and finish
    setResult(RESULT_CANCELED);
    if (mSessionId != -1) {
      mInstaller.setPermissionsResult(mSessionId, false);
    }
    finish();
  }
}
//跳转到InstallInstalling
private void startInstall() {  //681行
  // Start subactivity to actually install the application
  Intent newIntent = new Intent();
  newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO,
                     mPkgInfo.applicationInfo);
  newIntent.setData(mPackageURI);
  newIntent.setClass(this, InstallInstalling.class); //跳转到InstallInstalling
  String installerPackageName = getIntent().getStringExtra(
    Intent.EXTRA_INSTALLER_PACKAGE_NAME);
  if (mOriginatingURI != null) {
    newIntent.putExtra(Intent.EXTRA_ORIGINATING_URI, mOriginatingURI);
  }
  if (mReferrerURI != null) {
    newIntent.putExtra(Intent.EXTRA_REFERRER, mReferrerURI);
  }
  if (mOriginatingUid != PackageInstaller.SessionParams.UID_UNKNOWN) {
    newIntent.putExtra(Intent.EXTRA_ORIGINATING_UID, mOriginatingUid);
  }
  if (installerPackageName != null) {
    newIntent.putExtra(Intent.EXTRA_INSTALLER_PACKAGE_NAME,
                       installerPackageName);
  }
  if (getIntent().getBooleanExtra(Intent.EXTRA_RETURN_RESULT, false)) {
    newIntent.putExtra(Intent.EXTRA_RETURN_RESULT, true);
  }
  newIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);
  if(localLOGV) Log.i(TAG, "downloaded app uri="+mPackageURI);
  startActivity(newIntent);
  finish();
}
```

### 1.3 `InstallInstalling`

[`InstallInstalling`](https://www.androidos.net.cn/android/9.0.0_r8/xref/packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallInstalling.java) 才真正执行安装操作，其部分内容如下：

```java
//执行安装任务
@Override
protected void onResume() { //224行
  super.onResume();

  // This is the first onResume in a single life of the activity
  if (mInstallingTask == null) {
    PackageInstaller installer = getPackageManager().getPackageInstaller();
    PackageInstaller.SessionInfo sessionInfo = installer.getSessionInfo(mSessionId);

    if (sessionInfo != null && !sessionInfo.isActive()) {
      mInstallingTask = new InstallingAsyncTask();  //执行安装任务
      mInstallingTask.execute();
    } else {
      // we will receive a broadcast when the install is finished
      mCancelButton.setEnabled(false);
      setFinishOnTouchOutside(false);
    }
  }
}

/**
 * 安装任务
 * Send the package to the package installer and then register a event result observer that
 * will call {@link #launchFinishBasedOnResult(int, int, String)}
 */
private final class InstallingAsyncTask extends AsyncTask<Void, Void,
PackageInstaller.Session> { //343行
  volatile boolean isDone;

  @Override
  protected PackageInstaller.Session doInBackground(Void... params) {
    PackageInstaller.Session session;
    try {
      session = getPackageManager().getPackageInstaller().openSession(mSessionId);
    } catch (IOException e) {
      return null;
    }

    session.setStagingProgress(0);

    try {
      File file = new File(mPackageURI.getPath());

      try (InputStream in = new FileInputStream(file)) {
        long sizeBytes = file.length();
        try (OutputStream out = session
             .openWrite("PackageInstaller", 0, sizeBytes)) {
          byte[] buffer = new byte[1024 * 1024];
          while (true) {
            int numRead = in.read(buffer);

            if (numRead == -1) {
              session.fsync(out);
              break;
            }

            if (isCancelled()) {
              session.close();
              break;
            }

            out.write(buffer, 0, numRead);
            if (sizeBytes > 0) {
              float fraction = ((float) numRead / (float) sizeBytes);
              session.addProgress(fraction);
            }
          }
        }
      }

      return session;
    } catch (IOException | SecurityException e) {
      Log.e(LOG_TAG, "Could not write package", e);

      session.close();

      return null;
    } finally {
      synchronized (this) {
        isDone = true;
        notifyAll();
      }
    }
  }

  @Override
  protected void onPostExecute(PackageInstaller.Session session) {
    if (session != null) {
      Intent broadcastIntent = new Intent(BROADCAST_ACTION);
      broadcastIntent.setFlags(Intent.FLAG_RECEIVER_FOREGROUND);
      broadcastIntent.setPackage(
        getPackageManager().getPermissionControllerPackageName());
      broadcastIntent.putExtra(EventResultPersister.EXTRA_ID, mInstallId);

      PendingIntent pendingIntent = PendingIntent.getBroadcast(
        InstallInstalling.this,
        mInstallId,
        broadcastIntent,
        PendingIntent.FLAG_UPDATE_CURRENT);

      session.commit(pendingIntent.getIntentSender());
      mCancelButton.setEnabled(false);
      setFinishOnTouchOutside(false);
    } else {
      getPackageManager().getPackageInstaller().abandonSession(mSessionId);

      if (!isCancelled()) {
        launchFailure(PackageManager.INSTALL_FAILED_INVALID_APK, null);
      }
    }
  }
}

//安装成功或失败后会回调这个方法
private void launchFinishBasedOnResult(int statusCode, int legacyStatus, String statusMessage) { //297行
  if (statusCode == PackageInstaller.STATUS_SUCCESS) {
    launchSuccess();
  } else {
    launchFailure(legacyStatus, statusMessage);
  }
}

//安装成功后跳转到 InstallSuccess
private void launchSuccess() {
  Intent successIntent = new Intent(getIntent());
  successIntent.setClass(this, InstallSuccess.class);
  successIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);

  startActivity(successIntent);
  finish();
}
```

### 1.4 `InstallSuccess`

安装成功界面  [`InstallSuccess`](https://www.androidos.net.cn/android/9.0.0_r8/xref/packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallSuccess.java) .

## 二、 `APK`无界面安装

### 2.1 `commandline`

`APK`无界面安装起点在 [commandline.c](https://www.androidos.net.cn/android/5.1.0_r3/xref/system/core/adb/commandline.c) (以`Android 5.1`为例):

```c
int adb_commandline(int argc, char **argv) { //1184行
  //...
  if (!strcmp(argv[0], "install")) { //1715行
    if (argc < 2) return usage();
    return install_app(ttype, serial, argc, argv);
  }
}
```

`install_app`:

```c
int install_app(transport_type transport, char* serial, int argc, char** argv) //2003行
{
    static const char *const DATA_DEST = "/data/local/tmp/%s";
    static const char *const SD_DEST = "/sdcard/tmp/%s";  //SDK路径
		//...

    char* apk_file = argv[last_apk]; 
    char apk_dest[PATH_MAX]; //安装路径
    snprintf(apk_dest, sizeof apk_dest, where, get_basename(apk_file));
    int err = do_sync_push(apk_file, apk_dest, 0 /* no show progress */); //将apk push到手机存储
    if (err) {
        goto cleanup_apk;
    } else {
        argv[last_apk] = apk_dest; /* destination name, not source location */
    }

    pm_command(transport, serial, argc, argv); //2051行
		//...
    return err;
}
```

`pm_command`，通过`pm`命令方式执行安装操作：

```c
static int pm_command(transport_type transport, char* serial,
                      int argc, char** argv) //1939行
{
    char buf[4096];

    snprintf(buf, sizeof(buf), "shell:pm"); //指向pm脚本文件

    while(argc-- > 0) {
        char *quoted = escape_arg(*argv++);
        strncat(buf, " ", sizeof(buf) - 1);
        strncat(buf, quoted, sizeof(buf) - 1);
        free(quoted);
    }

    send_shellcommand(transport, serial, buf);
    return 0;
}
```

### 2.2 `pm`

[`pm`](https://www.androidos.net.cn/android/5.1.0_r3/xref/frameworks/base/cmds/pm/pm) 文件内容如下：

```bash
# Script to start "pm" on the device, which has a very rudimentary
# shell.
#
base=/system
export CLASSPATH=$base/framework/pm.jar
exec app_process $base/bin com.android.commands.pm.Pm "$@"
```

`pm.jar`指向的即为 [`pm`](https://www.androidos.net.cn/android/5.1.0_r3/xref/frameworks/base/cmds/pm) 工程，其核心有一个 [`pm.java`](https://www.androidos.net.cn/android/5.1.0_r3/xref/frameworks/base/cmds/pm/src/com/android/commands/pm/Pm.java) 文件，我们关心的核心内容如下：

```java
public static void main(String[] args) { //103行
  int exitCode = 1;
  try {
    exitCode = new Pm().run(args); //调用run()方法
  } catch (Exception e) {
    Log.e(TAG, "Error", e);
    System.err.println("Error: " + e);
    if (e instanceof RemoteException) {
      System.err.println(PM_NOT_RUNNING_ERR);
    }
  }
  System.exit(exitCode);
}

```

`run()`:

```java
public int run(String[] args) throws IOException, RemoteException { //118行
  boolean validCommand = false;
  if (args.length < 1) {
    return showUsage();
  }
  mUm = IUserManager.Stub.asInterface(ServiceManager.getService("user"));
  //IPackageManager
  mPm = IPackageManager.Stub.asInterface(ServiceManager.getService("package")); 
  if (mPm == null) {
    System.err.println(PM_NOT_RUNNING_ERR);
    return 1;
  }
  mInstaller = mPm.getPackageInstaller();
  //... 149行
  if ("install".equals(op)) {
    return runInstall();
  }
  //... 174行
  if ("uninstall".equals(op)) {
    return runUninstall();
  }
	//...
}
```

`runInstall()`:

```java
private int runInstall() { //906行
	//... 1003行
  LocalPackageInstallObserver obs = new LocalPackageInstallObserver();
  try {
    VerificationParams verificationParams = new VerificationParams(verificationURI,
				originatingURI, referrerURI, VerificationParams.NO_UID, null);
		//IPackageManager
    mPm.installPackageAsUser(apkFilePath, obs.getBinder(), installFlags,
                             installerPackageName, verificationParams, abi, userId);

    synchronized (obs) {
      while (!obs.finished) {
        try {
          obs.wait();
        } catch (InterruptedException e) {
        }
      }
      if (obs.result == PackageManager.INSTALL_SUCCEEDED) {
        System.out.println("Success");
        return 0;
      } else {
        System.err.println("Failure [" + installFailureToString(obs) + "]");
        return 1;
      }
    }
  } catch (RemoteException e) {
    System.err.println(e.toString());
    System.err.println(PM_NOT_RUNNING_ERR);
    return 1;
  }
}
```

### 2.3 `PackageManagerService`

`installPackageAsUser(...)`:

```java
//11540行
@Override
public void installPackageAsUser(String originPath, IPackageInstallObserver2 observer,
                                 int installFlags, String installerPackageName, int userId) {
  mContext.enforceCallingOrSelfPermission(android.Manifest.permission.INSTALL_PACKAGES, null);

  final int callingUid = Binder.getCallingUid();
  enforceCrossUserPermission(callingUid, userId, true /* requireFullPermission */, true 
                             /* checkShell */, "installPackageAsUser");
	//... 11585行
  final File originFile = new File(originPath);
  final OriginInfo origin = OriginInfo.fromUntrustedFile(originFile);

  final Message msg = mHandler.obtainMessage(INIT_COPY); //INIT_COPY标记
  final VerificationInfo verificationInfo = new VerificationInfo(
    null /*originatingUri*/, null /*referrer*/, -1 /*originatingUid*/, callingUid);
  final InstallParams params = new InstallParams(origin, null /*moveInfo*/, observer,
    	installFlags, installerPackageName, null /*volumeUuid*/, verificationInfo, user,
			null /*packageAbiOverride*/, null /*grantedPermissions*/, null /*certificates*/);
  params.setTraceMethod("installAsUser").setTraceCookie(System.identityHashCode(params));
  msg.obj = params;

  Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "installAsUser",
                        System.identityHashCode(msg.obj));
  Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                        System.identityHashCode(msg.obj));

  mHandler.sendMessage(msg);
}
```

`doHandleMessage(msg)`:

```java
void doHandleMessage(Message msg) { //1180行
  switch (msg.what) {
    case INIT_COPY: {
      HandlerParams params = (HandlerParams) msg.obj;
      int idx = mPendingInstalls.size();
      if (DEBUG_INSTALL) Slog.i(TAG, "init_copy idx=" + idx + ": " + params);
      // If a bind was already initiated we dont really
      // need to do anything. The pending install
      // will be processed later on.
      if (!mBound) {
        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                              System.identityHashCode(mHandler));
        // If this is the only one pending we might
        // have to bind to the service again.
        if (!connectToService()) {
          Slog.e(TAG, "Failed to bind to media container service");
          params.serviceError();
          Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                              System.identityHashCode(mHandler));
          if (params.traceMethod != null) {
            Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, params.traceMethod,
                                params.traceCookie);
          }
          return;
        } else {
          // Once we bind to the service, the first
          // pending request will be processed.
          mPendingInstalls.add(idx, params);
        }
      } else {
        mPendingInstalls.add(idx, params);
        // Already bound to the service. Just make
        // sure we trigger off processing the first request.
        if (idx == 0) {
          mHandler.sendEmptyMessage(MCS_BOUND);  //MCS_BOUND
        }
      }
      break;
    }
    case MCS_BOUND: {
      if (DEBUG_INSTALL) Slog.i(TAG, "mcs_bound");
      if (msg.obj != null) {
        mContainerService = (IMediaContainerService) msg.obj;
        Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                            System.identityHashCode(mHandler));
      }
      if (mContainerService == null) {
				//...
      } else if (mPendingInstalls.size() > 0) {
        HandlerParams params = mPendingInstalls.get(0);
        if (params != null) {
          Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                              System.identityHashCode(params));
          Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "startCopy");
          if (params.startCopy()) { //真正执行拷贝工作
          	//...
          }
          Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
      } else {
        // Should never happen ideally.
        Slog.w(TAG, "Empty queue");
      }
      break;
      //...
    }
  }
}
```

`startCopy()`:

```java
final boolean startCopy() {
  boolean res;
  try {
    if (DEBUG_INSTALL) Slog.i(TAG, "startCopy " + mUser + ": " + this);

    if (++mRetries > MAX_RETRIES) { //最多尝试4次
      Slog.w(TAG, "Failed to invoke remote methods on default container service. Giving up");
      mHandler.sendEmptyMessage(MCS_GIVE_UP);
      handleServiceError();
      return false;
    } else {
      handleStartCopy();
      res = true;
    }
  } catch (RemoteException e) {
    if (DEBUG_INSTALL) Slog.i(TAG, "Posting install MCS_RECONNECT");
    mHandler.sendEmptyMessage(MCS_RECONNECT);
    res = false;
  }
  handleReturnCode(); //执行这里
  return res;
}
```

`handleReturnCode()`:

```java
@Override
void handleReturnCode() {
  if (mObserver != null) {
    try {
      mObserver.onGetStatsCompleted(mStats, mSuccess);
    } catch (RemoteException e) {
      Slog.i(TAG, "Observer no longer exists.");
    }
  }
}
```

### 2.4 总结

`APK`无界面安装方式流程如下图所示：

![image](https://github.com/tianyalu/NeApkInstallProcess/raw/master/show/apk_install_process.png)

 ## 三、 `APK`安装本质

`APK`安装的本质是把`apk`文件拷贝到对应的目录。

![image](https://github.com/tianyalu/NeApkInstallProcess/raw/master/show/apk_install_nature.png)

