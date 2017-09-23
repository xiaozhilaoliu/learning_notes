#MultiDex原理
## 一、Android中常见的类加载器

### 1、PathClassLoader
	加载/data/app目录下的apk文件，从这个目录可以看出，PathClassLoader主要用来加载已经安装了apk。
   
### 2、DexClassLoader
	加载路径需要在创建DexClassLoader时传入，可以加载任何路径下下的apk/dex/*.jar；插件化的原理就在此	
## 二、实现动态加载dex的原理
* 需要将dex拷贝到系统的dex文件目录下！
* 创建odex的文件目录，用来存储优化后的文件
* 将DexClassLoader插入到PathClassLoader和BootstrapClassloader中间，这个服务副委托模式！apk加载类的顺序，PathClassloader->DexClassloader->bootstrapClassloader；
	* 插入方法：在application中，创建一个DexClassLoader，他的parent为当前PathClassloader的父（bootstrapClassloader） ，然后再通过反射，将PathClassLoader的父设置为DexClassLoader;

## MultiDex源码分析
### application调运的代码
```java
public static void install(Context context) {
    Log.i("MultiDex", "Installing application");
    if(IS_VM_MULTIDEX_CAPABLE) {   //如果虚拟机支持分包，那么久不需要动态加载
        Log.i("MultiDex", "VM has multidex support, MultiDex support library is disabled.");
    } else if(VERSION.SDK_INT < 4) {  //在这个版本下面，不支持动态加载
        throw new RuntimeException("MultiDex installation failed. SDK " + VERSION.SDK_INT + " is unsupported. Min SDK version is " + 4 + ".");
    } else {
        try {
            ApplicationInfo applicationInfo = getApplicationInfo(context);
            if(applicationInfo == null) {
                Log.i("MultiDex", "No ApplicationInfo available, i.e. running on a test Context: MultiDex support library is disabled.");
                return;
            }

            doInstallation(context, new File(applicationInfo.sourceDir), new File(applicationInfo.dataDir), "secondary-dexes", "");     //动态加载耳机目录下的文件
        } catch (Exception var2) {
            Log.e("MultiDex", "MultiDex installation failure", var2);
            throw new RuntimeException("MultiDex installation failed (" + var2.getMessage() + ").");
        }

        Log.i("MultiDex", "install done");
    }
}
```
### 动态加载apk文件
```java
private static void doInstallation(Context mainContext, File sourceApk, File dataDir, String secondaryFolderName, String prefsKeyPrefix) throws IOException, IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException {
    Set var5 = installedApk;   //已经加载的文件集合
    synchronized(installedApk) {
        if(!installedApk.contains(sourceApk)) {
            installedApk.add(sourceApk);
            if(VERSION.SDK_INT > 20) {
                Log.w("MultiDex", "MultiDex is not guaranteed to work in SDK version " + VERSION.SDK_INT + ": SDK version higher than " + 20 + " should be backed by " + "runtime with built-in multidex capabilty but it's not the " + "case here: java.vm.version=\"" + System.getProperty("java.vm.version") + "\"");
            }

            ClassLoader loader;
            try {
                loader = mainContext.getClassLoader();   //获取加载器（一定集成于BaseDexClassLoader）
            } catch (RuntimeException var11) {
                Log.w("MultiDex", "Failure while trying to obtain Context class loader. Must be running in test mode. Skip patching.", var11);
                return;
            }

            if(loader == null) {
                Log.e("MultiDex", "Context class loader is null. Must be running in test mode. Skip patching.");
            } else {
                try {
                    clearOldDexDir(mainContext);  //清除二级缓存以前的加载信息
                } catch (Throwable var10) {
                    Log.w("MultiDex", "Something went wrong when trying to clear old MultiDex extraction, continuing without cleaning.", var10);
                }

                File dexDir = getDexDir(mainContext, dataDir, secondaryFolderName);
                List<? extends File> files = MultiDexExtractor.load(mainContext, sourceApk, dexDir, prefsKeyPrefix, false);   //获取需要加载的二级文件列表
                installSecondaryDexes(loader, dexDir, files);
            }
        }
    }
}
```

### 获取需要动态加载的二级文件列表
```java
static List<? extends File> load(Context context, File sourceApk, File dexDir, String prefsKeyPrefix, boolean forceReload) throws IOException {
    Log.i("MultiDex", "MultiDexExtractor.load(" + sourceApk.getPath() + ", " + forceReload + ", " + prefsKeyPrefix + ")");
    long currentCrc = getZipCrc(sourceApk);
    File lockFile = new File(dexDir, "MultiDex.lock");
    RandomAccessFile lockRaf = new RandomAccessFile(lockFile, "rw");
    FileChannel lockChannel = null;
    FileLock cacheLock = null;
    IOException releaseLockException = null;

    List files;
    try {
        lockChannel = lockRaf.getChannel();
        Log.i("MultiDex", "Blocking on lock " + lockFile.getPath());
        cacheLock = lockChannel.lock();
        Log.i("MultiDex", lockFile.getPath() + " locked");
        if(!forceReload && !isModified(context, sourceApk, currentCrc, prefsKeyPrefix)) {
            try {
                files = loadExistingExtractions(context, sourceApk, dexDir, prefsKeyPrefix);
            } catch (IOException var21) {
                Log.w("MultiDex", "Failed to reload existing extracted secondary dex files, falling back to fresh extraction", var21);
                files = performExtractions(sourceApk, dexDir);
                putStoredApkInfo(context, prefsKeyPrefix, getTimeStamp(sourceApk), currentCrc, files);
            }
        } else {
            Log.i("MultiDex", "Detected that extraction must be performed.");
            files = performExtractions(sourceApk, dexDir);
            putStoredApkInfo(context, prefsKeyPrefix, getTimeStamp(sourceApk), currentCrc, files);
        }
    } finally {
        if(cacheLock != null) {
            try {
                cacheLock.release();
            } catch (IOException var20) {
                Log.e("MultiDex", "Failed to release lock on " + lockFile.getPath());
                releaseLockException = var20;
            }
        }

        if(lockChannel != null) {
            closeQuietly(lockChannel);
        }

        closeQuietly(lockRaf);
    }

    if(releaseLockException != null) {
        throw releaseLockException;
    } else {
        Log.i("MultiDex", "load found " + files.size() + " secondary dex files");
        return files;
    }
}
```

### 安装二级目录
```java
private static void installSecondaryDexes(ClassLoader loader, File dexDir, List<? extends File> files) throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException, IOException {
    if(!files.isEmpty()) {
        if(VERSION.SDK_INT >= 19) {
            MultiDex.V19.install(loader, files, dexDir);
        } else if(VERSION.SDK_INT >= 14) {
            MultiDex.V14.install(loader, files, dexDir);
        } else {
            MultiDex.V4.install(loader, files);
        }
    }

}



private static final class V4 {
    private V4() {
    }

    private static void install(ClassLoader loader, List<? extends File> additionalClassPathEntries) throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException, IOException {
        int extraSize = additionalClassPathEntries.size();
        Field pathField = MultiDex.findField(loader, "path");
        StringBuilder path = new StringBuilder((String)pathField.get(loader));
        String[] extraPaths = new String[extraSize];
        File[] extraFiles = new File[extraSize];
        ZipFile[] extraZips = new ZipFile[extraSize];
        DexFile[] extraDexs = new DexFile[extraSize];

        String entryPath;
        int index;
        for(ListIterator iterator = additionalClassPathEntries.listIterator(); iterator.hasNext(); extraDexs[index] = DexFile.loadDex(entryPath, entryPath + ".dex", 0)) {
            File additionalEntry = (File)iterator.next();
            entryPath = additionalEntry.getAbsolutePath();
            path.append(':').append(entryPath);
            index = iterator.previousIndex();
            extraPaths[index] = entryPath;
            extraFiles[index] = additionalEntry;
            extraZips[index] = new ZipFile(additionalEntry);
        }

        pathField.set(loader, path.toString());
        MultiDex.expandFieldArray(loader, "mPaths", extraPaths);
        MultiDex.expandFieldArray(loader, "mFiles", extraFiles);
        MultiDex.expandFieldArray(loader, "mZips", extraZips);
        MultiDex.expandFieldArray(loader, "mDexs", extraDexs);
    }
}

private static final class V14 {
    private V14() {
    }

    private static void install(ClassLoader loader, List<? extends File> additionalClassPathEntries, File optimizedDirectory) throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException {
        Field pathListField = MultiDex.findField(loader, "pathList");  //反射pathList字段
        Object dexPathList = pathListField.get(loader);  //获取pathlist的值
        MultiDex.expandFieldArray(dexPathList, "dexElements", makeDexElements(dexPathList, new ArrayList(additionalClassPathEntries), optimizedDirectory));  //将获取到的需要动态加载的dex，存放到pathList的dex数据集合中，这个系统就可以加载额外的dex了
    }

    private static Object[] makeDexElements(Object dexPathList, ArrayList<File> files, File optimizedDirectory) throws IllegalAccessException, InvocationTargetException, NoSuchMethodException {
        Method makeDexElements = MultiDex.findMethod(dexPathList, "makeDexElements", new Class[]{ArrayList.class, File.class});
        return (Object[])((Object[])makeDexElements.invoke(dexPathList, new Object[]{files, optimizedDirectory}));
    }
}

private static final class V19 {
    private V19() {
    }

    private static void install(ClassLoader loader, List<? extends File> additionalClassPathEntries, File optimizedDirectory) throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException {
        Field pathListField = MultiDex.findField(loader, "pathList");
        Object dexPathList = pathListField.get(loader);
        ArrayList<IOException> suppressedExceptions = new ArrayList();
        MultiDex.expandFieldArray(dexPathList, "dexElements", makeDexElements(dexPathList, new ArrayList(additionalClassPathEntries), optimizedDirectory, suppressedExceptions));
        if(suppressedExceptions.size() > 0) {
            Iterator var6 = suppressedExceptions.iterator();

            while(var6.hasNext()) {
                IOException e = (IOException)var6.next();
                Log.w("MultiDex", "Exception in makeDexElement", e);
            }

            Field suppressedExceptionsField = MultiDex.findField(dexPathList, "dexElementsSuppressedExceptions");
            IOException[] dexElementsSuppressedExceptions = (IOException[])((IOException[])suppressedExceptionsField.get(dexPathList));
            if(dexElementsSuppressedExceptions == null) {
                dexElementsSuppressedExceptions = (IOException[])suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
            } else {
                IOException[] combined = new IOException[suppressedExceptions.size() + dexElementsSuppressedExceptions.length];
                suppressedExceptions.toArray(combined);
                System.arraycopy(dexElementsSuppressedExceptions, 0, combined, suppressedExceptions.size(), dexElementsSuppressedExceptions.length);
                dexElementsSuppressedExceptions = combined;
            }

            suppressedExceptionsField.set(dexPathList, dexElementsSuppressedExceptions);
        }

    }

    private static Object[] makeDexElements(Object dexPathList, ArrayList<File> files, File optimizedDirectory, ArrayList<IOException> suppressedExceptions) throws IllegalAccessException, InvocationTargetException, NoSuchMethodException {
        Method makeDexElements = MultiDex.findMethod(dexPathList, "makeDexElements", new Class[]{ArrayList.class, File.class, ArrayList.class});
        return (Object[])((Object[])makeDexElements.invoke(dexPathList, new Object[]{files, optimizedDirectory, suppressedExceptions}));
    }
}
```

### BaseDexClassLoader
```java

public class BaseDexClassLoader extends ClassLoader {
   private final DexPathList pathList;  //这个字段存放dex的文件信息
}

/*package*/ final class DexPathList {
    private static final String DEX_SUFFIX = ".dex";

    /** class definition context */
    private final ClassLoader definingContext;

    /**
     * List of dex/resource (class path) elements.
     * Should be called pathElements, but the Facebook app uses reflection
     * to modify 'dexElements' (http://b/7726934).
     */
    private final Element[] dexElements;  //每个classloader寻找类的dex元素集合

    /** List of native library directories. */
    private final File[] nativeLibraryDirectories;
 }
```