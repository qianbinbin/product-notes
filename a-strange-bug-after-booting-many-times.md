# 手机重启若干次后，一个奇怪 Bug 的分析和解决过程

## 问题现象

公司的老化测试工具，包括了重启、存储读写、音频播放、视频播放等流程，是一个预装在手机的 APK，出厂时会去掉。

测试做老化测试挂机 24 小时后，手机概率性出现问题，现象主要包括** Home 键和 Recent 键不能用、无法下拉快捷设置和通知栏、锁屏失效、开发者模式无法打开**等。复现概率 10 台中出现 0 ~ 3 台。诡异的是，研发这边也会每天挂机，但一台都没有复现。

这个问题严重、紧急且困难。出现问题后，重启手机是不能解决的，只能恢复出厂设置。而且出现问题的都是测试使用的 user 正式版本，正式版本几天才会出一版，而我们工程师使用的 userdebug 版本从未复现，这也给 debug 造成了麻烦。

## 分析过程

### 开机向导

手机出现的这些症状，经历过 Android 5.0 ROM 开发的人可能似曾相识。如果手机预装了 Google 开机向导 SetupWizard.apk，那么你一定会记得首次开机必须登录 Google 账户的脑残设定。虽然这一做法无可厚非，但由于众所周知的原因，天朝无法访问 Google 服务器，所以作为 ROM 开发人员，如果没有能稳定爬梯的 Wi-Fi，就不得不八仙过海各显神通了。

我的做法简单粗暴，刷好 userdebug 或者 eng 版本，首次开机直接`adb shell`进去删除 SetupWizard.apk，手机就直接跳过了开机向导进入 Launcher。实际上开机向导和 Launcher 本质上一样，都为默认 Activity 添加了`android.intent.category.HOME`的 filter，只不过开机向导具有更高的优先级。

但这一做法很快就发现有问题，其症状和这次问题的现象完全一样，如果你当时也这么做过，那你也一定遇到过。原因是开机向导完成时，会写入两个重要的属性：

```java
Settings.Global.putInt(getContentResolver(), Settings.Global.DEVICE_PROVISIONED, 1);
Settings.Secure.putInt(getContentResolver(), Settings.Secure.USER_SETUP_COMPLETE, 1);
```

这两个属性表示手机已经设置完成，设备处于可用状态了。如果没有写入，那么手机很多功能都不能正常工作。在没有内置 SetupWizard.apk 的 AOSP 中，这个任务由`packages/apps/Provision/`来完成，如果内置了 SetupWizard.apk，则 Provision.apk 会被覆盖。

我对问题机测试了两个属性：

```bash
$ adb shell settings get global device_provisioned
0
$ adb shell settings get secure user_setup_complete
0
```

果然属性是不对的。再将正确的值 put 进去，手机就正常了。

考虑到 Google 的开机向导完成后，我们还会显示一个定制界面，这个界面走完后 SetupWizard.apk 才会写入这两个属性。所以我首先怀疑开机向导有问题，尤其是定制部分，并推断这个 bug 不是跑老化后出现的，一定是第一次开机就有问题了，只不过测试当时没有注意。我特意叮嘱测试，刷机后第一次开机就要确认是不是就有问题。

就在我们打算去掉定制界面、进行压力测试以验证我的想法时，我被迅速打脸。20 台机器又复现了 4 台，并且测试确认第一次开机是正常的。这就非常尴尬。

### 老化测试

既然开机向导没有问题，比较有可能的就是老化测试了，它包括很多步骤，其中最令人注意的就是重启 50 次。

之前提到问题的原因是属性值不正确，从 log 中看，此时 Keyguard 会打印错误 log：

```
KeyguardViewMediator: doKeyguard: not showing because device isn't provisioned and the sim is not locked or missing
```

查看 log 发现，第一次复现的 3 台机器中，有两台是第一次重启后就报了这个错误，剩下一台是 50 次重启跑完后报了这个错误。手机出错时间肯定在 log 报错之前，于是我转向怀疑是 50 次重启过程中出了问题。

但是重启测试的核心代码非常简单，没有什么特别的地方：

```java
PowerManager pm = (PowerManager) getSystemService(Context.POWER_SERVICE);
if (pm != null) {
    pm.reboot("");
}
```

从 log 中分析关机和开机流程，也没发现可疑的地方。

### 初步规避方案

与同事孙克龙、王新楼讨论后，开始尝试规避方案，即在开机进入 Launcher 时判断这两个属性是否正确，如果不正确则重新写入。但老大弦哥否决了这个方案，全哥也认为不妥，原因是这个问题非常严重，应该找出问题的根源，这个根源可能会同时引起其它问题，而不仅仅是两个属性值。时间非常紧迫，看来 deadline 之前是解决不了了，但事实证明老板的想法是明智的。

与此同时我也在可疑的地方添加了 log，打印包名和调用栈信息，看到底是什么程序修改了这两个属性。Android 提供给第三方读写这两个属性的接口在`frameworks/base/core/java/android/provider/Settings.java`中，最终数据的存取是在 SettingsProvider 模块中。

但 Settings.java 中添加的 log 并没有什么卵用。

之前为了方便 debug，孙克龙建议我用 QFIL 给问题机半擦烧写了一个 userdebug 的 bootimage（只是不时会有点问题）。同事李鹏飞发现这两个属性最终保存在`/data/system/users/0/`下的 XML 文件中，他将`settings_global.xml`导出，发现写入属性的值为默认，包名为`android`：

```xml
<setting id="19" name="device_provisioned" value="0" package="android" />
```

另一个属性也是类似，但保存在`settings_secure.xml`文件中。

这说明这两个错误属性值应该是由 framework 来写的，十有八九就是 SettingsProvider 本身。这意味着我们的源码中出现了系统级别的错误，而不是什么开机向导或者第三方应用导致的。

初步规避方案宣告失败。

这时老板也安排经验丰富的同事刘伟龙来分析这个问题。

### SettingsProvider

刘伟龙把导出的 XML 文件与正常的文件做了对比，发现错误的 XML 文件总是把其它属性也写成系统默认值。也就是说，很可能是原来正常的文件丢失了，系统重新生成了一份。

Settings.java 通过 binder 机制调用 SettingsProvider 模块来完成数据的存取，最终是在 SettingsState.java 中读写 XML 文件的。

SettingsState 中维护了一个 mStatePersistFile 对象，这是一个 File 类型对象，指代当前的 XML 文件。事实上用于存取数据的 XML 文件一共有三个，每个都对应一个 SettingsState 对象：

```
/data/system/users/0/settings_system.xml
/data/system/users/0/settings_secure.xml
/data/system/users/0/settings_global.xml
```

原本这些数据保存在数据库中，Android 6.0 后会迁移到 XML 文件，默认情况下还会在迁移后删除数据库文件`settings.db`。

在分析 XML 文件如何读写之前，需要先了解一下 AtomicFile。

#### AtomicFile

AtomicFile 让文件的写入变为原子操作，但阅读源码可以发现，它并不是线程安全的。

它的原理非常简单，就是在写入文件时为原文件创建一个扩展名为`.bak`的备份文件。

```java
// frameworks/base/core/java/android/util/AtomicFile.java

public class AtomicFile {
    private final File mBaseName;
    private final File mBackupName;

    /**
     * Create a new AtomicFile for a file located at the given File path.
     * The secondary backup file will be the same file path with ".bak" appended.
     */
    public AtomicFile(File baseName) {
        mBaseName = baseName;
        mBackupName = new File(baseName.getPath() + ".bak");
    }

    // ...

    /**
     * Start a new write operation on the file.  This returns a FileOutputStream
     * to which you can write the new file data.  The existing file is replaced
     * with the new data.  You <em>must not</em> directly close the given
     * FileOutputStream; instead call either {@link #finishWrite(FileOutputStream)}
     * or {@link #failWrite(FileOutputStream)}.
     *
     * <p>Note that if another thread is currently performing
     * a write, this will simply replace whatever that thread is writing
     * with the new file being written by this thread, and when the other
     * thread finishes the write the new write operation will no longer be
     * safe (or will be lost).  You must do your own threading protection for
     * access to AtomicFile.
     */
    public FileOutputStream startWrite() throws IOException {
        // Rename the current file so it may be used as a backup during the next read
        if (mBaseName.exists()) {
            if (!mBackupName.exists()) {
                if (!mBaseName.renameTo(mBackupName)) {
                    Log.w("AtomicFile", "Couldn't rename file " + mBaseName
                            + " to backup file " + mBackupName);
                }
            } else {
                mBaseName.delete();
            }
        }
        FileOutputStream str = null;
        try {
            str = new FileOutputStream(mBaseName);
        } catch (FileNotFoundException e) {
            File parent = mBaseName.getParentFile();
            if (!parent.mkdirs()) {
                throw new IOException("Couldn't create directory " + mBaseName);
            }
            FileUtils.setPermissions(
                parent.getPath(),
                FileUtils.S_IRWXU|FileUtils.S_IRWXG|FileUtils.S_IXOTH,
                -1, -1);
            try {
                str = new FileOutputStream(mBaseName);
            } catch (FileNotFoundException e2) {
                throw new IOException("Couldn't create " + mBaseName);
            }
        }
        return str;
    }

    /**
     * Call when you have successfully finished writing to the stream
     * returned by {@link #startWrite()}.  This will close, sync, and
     * commit the new data.  The next attempt to read the atomic file
     * will return the new file stream.
     */
    public void finishWrite(FileOutputStream str) {
        if (str != null) {
            FileUtils.sync(str);
            try {
                str.close();
                mBackupName.delete();
            } catch (IOException e) {
                Log.w("AtomicFile", "finishWrite: Got exception:", e);
            }
        }
    }

    /**
     * Call when you have failed for some reason at writing to the stream
     * returned by {@link #startWrite()}.  This will close the current
     * write stream, and roll back to the previous state of the file.
     */
    public void failWrite(FileOutputStream str) {
        if (str != null) {
            FileUtils.sync(str);
            try {
                str.close();
                mBaseName.delete();
                mBackupName.renameTo(mBaseName);
            } catch (IOException e) {
                Log.w("AtomicFile", "failWrite: Got exception:", e);
            }
        }
    }

    // ...

    /**
     * Open the atomic file for reading.  If there previously was an
     * incomplete write, this will roll back to the last good data before
     * opening for read.  You should call close() on the FileInputStream when
     * you are done reading from it.
     *
     * <p>Note that if another thread is currently performing
     * a write, this will incorrectly consider it to be in the state of a bad
     * write and roll back, causing the new data currently being written to
     * be dropped.  You must do your own threading protection for access to
     * AtomicFile.
     */
    public FileInputStream openRead() throws FileNotFoundException {
        if (mBackupName.exists()) {
            mBaseName.delete();
            mBackupName.renameTo(mBaseName);
        }
        return new FileInputStream(mBaseName);
    }

    // ...

}
```

读取方法`openRead()`返回的是一个原文件的文件输入流。如果发现备份文件存在，则说明当前有写入操作未完成，直接把备份文件恢复为原文件，这样原文件中总是干净的数据。

写入方法`startWrite()`返回的是一个原文件的文件输出流。如果原文件存在，但备份文件不存在，则将创建一个备份。如果原文件存在且备份文件也存在，说明当前有写入操作未完成，直接把原文件删除。这样备份文件中总是干净的数据。当写入成功后，应调用`finishWrite()`来删除备份文件；当写入失败时，应调用`failWrite()`来将备份文件恢复为原文件。

线程安全问题由调用方考虑。

#### XML 文件的读取

XML 文件的读取是在 SettingsState.java 的`readStateSyncLocked()`中：

```java
// frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsState.java

private void readStateSyncLocked() {
    FileInputStream in;
    if (!mStatePersistFile.exists()) {
        Slog.i(LOG_TAG, "No settings state " + mStatePersistFile);
        addHistoricalOperationLocked(HISTORICAL_OPERATION_INITIALIZE, null);
        return;
    }
    try {
        in = new AtomicFile(mStatePersistFile).openRead();
    } catch (FileNotFoundException fnfe) {
        String message = "No settings state " + mStatePersistFile;
        Slog.wtf(LOG_TAG, message);
        Slog.i(LOG_TAG, message);
        return;
    }
    try {
        XmlPullParser parser = Xml.newPullParser();
        parser.setInput(in, StandardCharsets.UTF_8.name());
        parseStateLocked(parser);
    } catch (XmlPullParserException | IOException e) {
        String message = "Failed parsing settings file: " + mStatePersistFile;
        Slog.wtf(LOG_TAG, message);
        throw new IllegalStateException(message , e);
    } finally {
        IoUtils.closeQuietly(in);
    }
}
```

#### 写入是在`doWriteState()`中：

```java
private void doWriteState() {
    if (DEBUG_PERSISTENCE) {
        Slog.i(LOG_TAG, "[PERSIST START]");
    }

    AtomicFile destination = new AtomicFile(mStatePersistFile);

    final int version;
    final ArrayMap<String, Setting> settings;

    synchronized (mLock) {
        version = mVersion;
        settings = new ArrayMap<>(mSettings);
        mDirty = false;
        mWriteScheduled = false;
    }

    FileOutputStream out = null;
    try {
        out = destination.startWrite();

        XmlSerializer serializer = Xml.newSerializer();
        serializer.setOutput(out, StandardCharsets.UTF_8.name());
        serializer.setFeature("http://xmlpull.org/v1/doc/features.html#indent-output", true);
        serializer.startDocument(null, true);
        serializer.startTag(null, TAG_SETTINGS);
        serializer.attribute(null, ATTR_VERSION, String.valueOf(version));

        final int settingCount = settings.size();
        for (int i = 0; i < settingCount; i++) {
            Setting setting = settings.valueAt(i);

            writeSingleSetting(mVersion, serializer, setting.getId(), setting.getName(),
                    setting.getValue(), setting.getPackageName());

            if (DEBUG_PERSISTENCE) {
                Slog.i(LOG_TAG, "[PERSISTED]" + setting.getName() + "=" + setting.getValue());
            }
        }

        serializer.endTag(null, TAG_SETTINGS);
        serializer.endDocument();
        destination.finishWrite(out);

        synchronized (mLock) {
            addHistoricalOperationLocked(HISTORICAL_OPERATION_PERSIST, null);
        }

        if (DEBUG_PERSISTENCE) {
            Slog.i(LOG_TAG, "[PERSIST END]");
        }
    } catch (Throwable t) {
        Slog.wtf(LOG_TAG, "Failed to write settings, restoring backup", t);
        destination.failWrite(out);
    } finally {
        IoUtils.closeQuietly(out);
    }
}
```

我在`doWriteState()`中添加了 log。几天后测试复现问题，分析 log 发现，出错时确实走到了这里，并且写入的属性为默认值，包名为`android`，这与 XML 文件中是一致的。但是为什么 SettingsProvider 会写入默认值，仍然不清楚。

问题迟迟不能解决，引起了康总的注意。康总拉我们开了一个小会。由于怀疑是 50 次重启过程中出了问题，我们在重启前获取属性值，开机启动 ProviderService 时分别打印`/data/system/users/0/`下的文件，结果发现重启之前文件存在
