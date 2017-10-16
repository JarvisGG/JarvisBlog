---
title: GreenDao 数据库升级数据迁移
date: 2017-04-22 11:53:56
tags: Android 第三方
comments: true
---

## 前言
最近需要解决GreenDao 在迭代升级数据库时需要做数据库迁移。经过查资料解决了，立个flag备忘。。

## 问题切入点
在GreenDao apt自动生成的DaoMaster代码
<!--more-->

``` Java
/** WARNING: Drops all table on Upgrade! Use only during development. */
public static class DevOpenHelper extends OpenHelper {
    public DevOpenHelper(Context context, String name, CursorFactory factory) {
        super(context, name, factory);
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        Log.i("greenDAO", "Upgrading schema from version " + oldVersion + " to " + newVersion + " by dropping all tables");
        dropAllTables(db, true);
        onCreate(db);
    }
}
```
GreenDao 在根据数据库版本号升级的时候会执行DevOpenHelper的onUpgrade方法，我们依次看到dropAllTables，onCreate，先清除所有的数据表，然后重建，这样我们上一个版本是数据表里的老数据就被drop掉了。所以我们的切入点就是着这里。

## 解决办法
这里我们的基本思路是：
1.对老的数据表重新命名，作为临时数据表
2.建立新的数据表
3.将临时数据表的所有数据迁移到新的数据表
4.删除临时数据表
这里我们统筹用MigrationHelper管理
``` Java
public static void migrate(SQLiteDatabase db, Class<? extends AbstractDao<?, ?>>... daoClasses) {
    // 1 新建临时表
    generateTempTables(db, daoClasses);
    // 2 创建新表
    createAllTables(db, false, daoClasses);
    // 3 临时表数据写入新表，删除临时表
    restoreData(db, daoClasses);
}
```
### step.1
将老的数据表重命名 **_TEMP
``` Java
private static void generateTempTables(SQLiteDatabase db, Class<? extends AbstractDao<?, ?>>... daoClasses) {
    for (int i = 0; i < daoClasses.length; i++) {
        DaoConfig daoConfig = new DaoConfig(db, daoClasses[i]);
        String tableName = daoConfig.tablename;
        if (!checkTable(db, tableName))
            continue;
        String tempTableName = daoConfig.tablename.concat("_TEMP");
        StringBuilder insertTableStringBuilder = new StringBuilder();
        insertTableStringBuilder.append("alter table ")
                .append(tableName)
                .append(" rename to ")
                .append(tempTableName)
                .append(";");
        db.execSQL(insertTableStringBuilder.toString());
    }
}

private static boolean checkTable(SQLiteDatabase db, String tableName) {
    StringBuilder query = new StringBuilder();
    query.append("SELECT count(*) FROM sqlite_master WHERE type='table' AND name='")
            .append(tableName)
            .append("'");
    Cursor c = db.rawQuery(query.toString(), null);
    if (c.moveToNext()) {
        int count = c.getInt(0);
        if (count > 0) {
            return true;
        }
        return false;
    }
    return false;
}
```
这里从系统表 sqlite_master 里面查需要迁移数据的表是否存在
### step.2
``` Java
private static void createAllTables(SQLiteDatabase db, boolean ifNotExits, @NonNull Class<? extends AbstractDao<?, ?>>... daoClasses) {
    reflectMethod(db, "createTable", ifNotExits, daoClasses);
}

private static void reflectMethod(SQLiteDatabase db, String methodName, boolean isExists, Class<? extends AbstractDao<?, ?>>... daoClasses) {
    if (daoClasses.length < 1) {
        return;
    }
    try {
        for (Class cls : daoClasses) {
            Method method = cls.getDeclaredMethod(methodName, SQLiteDatabase.class, boolean.class);
            method.invoke(null, db, isExists);
        }
    } catch (NoSuchMethodException e) {
        e.printStackTrace();
    } catch (InvocationTargetException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    }
}
```
由于父类AbstractDao不存在createTable这个方法，而是在每个子类的***Dao实现了createTable这个方法，所以这里我们采用反射来获得这个方法，来做建立新的表结构操作。
### step.3
``` Java
private static void restoreData(SQLiteDatabase db, Class<? extends AbstractDao<?, ?>>... daoClasses) {
    for (int i = 0; i < daoClasses.length; i++) {
        DaoConfig daoConfig = new DaoConfig(db, daoClasses[i]);
        String tableName = daoConfig.tablename;
        String tempTableName = daoConfig.tablename.concat("_TEMP");
        if (!checkTable(db,tempTableName))
            continue;
        List<String> columns = getColumns(db, tempTableName);
        ArrayList<String> properties = new ArrayList<>(columns.size());
        for (int j = 0; j < daoConfig.properties.length; j++) {
            String columnName = daoConfig.properties[j].columnName;
            if (columns.contains(columnName)) {
                properties.add(columnName);
            }
        }
        if (properties.size() > 0) {
            final String columnSQL = TextUtils.join(",", properties);
            StringBuilder insertTableStringBuilder = new StringBuilder();
            insertTableStringBuilder.append("INSERT INTO ")
                    .append(tableName)
                    .append(" (")
                    .append(columnSQL)
                    .append(") SELECT ")
                    .append(columnSQL)
                    .append(" FROM ")
                    .append(tempTableName)
                    .append(";");

            db.execSQL(insertTableStringBuilder.toString());
        }
        StringBuilder dropTableStringBuilder = new StringBuilder();
        dropTableStringBuilder.append("DROP TABLE ").append(tempTableName);
        db.execSQL(dropTableStringBuilder.toString());
    }
}

private static List<String> getColumns(SQLiteDatabase db, String tableName) {
    List<String> columns = null;
    Cursor cursor = null;
    try {
        cursor = db.rawQuery("SELECT * FROM " + tableName + " limit 0", null);
        if (null != cursor && cursor.getColumnCount() > 0) {
            columns = Arrays.asList(cursor.getColumnNames());
        }
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        if (cursor != null)
            cursor.close();
        if (null == columns)
            columns = new ArrayList<>();
    }
    return columns;
}
```
这里就是简单的从老表拿数据insert到新的表结构里面，这里可以根据具体项目来进行插入。