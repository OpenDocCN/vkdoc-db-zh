# 第 7 章 ■ 在 Android/Java 中使用 SQLite

以下是`NoteBookProvider.java`文件中`NoteBookProvider`类的一些静态值。

```java
private static final String TAG = "NotePadProvider";

private static final String DATABASE_NAME = "notepad.db";

private static final int DATABASE_VERSION = 2;

private static final String NOTES_TABLE_NAME = "notes";

private static HashMap<String, String> sNotesProjectionMap;

private static HashMap<String, String> sLiveFolderProjectionMap;

private static final int NOTES = 1;

private static final int NOTE_ID = 2;

private static final int LIVE_FOLDER_NOTES = 3;

private static final UriMatcher sUriMatcher;
```

## 扩展 SQLiteOpenHelper

你需要声明一个继承自`SQLiteOpenHelper`的类，并实现自己的`onCreate`函数。以下是示例中你需要使用的代码：

```java
private static class DatabaseHelper extends SQLiteOpenHelper {

    DatabaseHelper(Context context) {
        super(context, DATABASE_NAME, null, DATABASE_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL("CREATE TABLE " + NOTES_TABLE_NAME + " ("
                + NoteColumns._ID + " INTEGER PRIMARY KEY,"
                + NoteColumns.TITLE + " TEXT,"
                + NoteColumns.NOTE + " TEXT,"
                + NoteColumns.CREATED_DATE + " INTEGER,"
                + NoteColumns.MODIFIED_DATE + " INTEGER"
                + ");");
    }

    // 上面的代码将创建如下表结构，其中 COLid 在运行时是 NoteColumns._ID 的值
    // CREATE TABLE notes (COLid INTEGER PRIMARY KEY, title TEXT,
    // note TEXT, created INTEGER, modified INTEGER);

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        Log.w(TAG, "Upgrading database from version " + oldVersion + " to "
                + newVersion + ", which will destroy all old data");
        db.execSQL("DROP TABLE IF EXISTS notes");
        onCreate(db);
    }
}
```

`private DatabaseHelper mOpenHelper;`

上面这行代码很容易被忽略。它在本文件的示例代码中贯穿始终：它是`SQLiteOpenHelper`的本地实例。采用这种结构，如果你重用这段代码，可以在`DatabaseHelper`内部进行更改（例如，使用你自己的字符串名称），而对其他代码只需做极少的改动。

接下来是示例中使用你将重用的样板代码的部分。注意`mOpenHelper`的使用，它就是`DatabaseHelper`。如你所见，它建立在已经构建好的结构之上。如果你检查示例中的完整代码，会发现大部分代码处理的是要存储的数据，而不是 SQLite 接口。

**粗体**显示的代码是实际更新数据库的代码。`db.insert`方法使用本章前面描述的静态值构建必要的 SQLite 语法。紧接在粗体代码下方的代码获取`db.insert`的返回值并将其放入一个名为`rowId`的局部变量中。这正是它的含义——新行的唯一 ID。如果操作失败，则返回-1。

```java
@Override
public Uri insert(Uri uri, ContentValues initialValues) {
    // 验证请求的 URI
    if (sUriMatcher.match(uri) != NOTES) {
        throw new IllegalArgumentException("Unknown URI " + uri);
    }

    ContentValues values;
    if (initialValues != null) {
        values = new ContentValues(initialValues);
    } else {
        values = new ContentValues();
    }

    Long now = Long.valueOf(System.currentTimeMillis());

    // 确保字段都已设置
    if (values.containsKey(NoteColumns.CREATED_DATE) == false) {
        values.put(NoteColumns.CREATED_DATE, now);
    }

    if (values.containsKey(NoteColumns.MODIFIED_DATE) == false) {
        values.put(NoteColumns.MODIFIED_DATE, now);
    }

    if (values.containsKey(NoteColumns.TITLE) == false) {
        Resources r = Resources.getSystem();
        values.put(NoteColumns.TITLE, r.getString(android.R.string.untitled));
    }

    if (values.containsKey(NoteColumns.NOTE) == false) {
        values.put(NoteColumns.NOTE, "");
    }

    SQLiteDatabase db = mOpenHelper.getWritableDatabase();
    long rowId = db.insert(NOTES_TABLE_NAME, NoteColumns.NOTE, values);

    if (rowId > 0) {
        Uri noteUri = ContentUris.withAppendedId(NoteColumns.CONTENT_URI, rowId);
        getContext().getContentResolver().notifyChange(noteUri, null);
        return noteUri;
    }

    throw new SQLException("Failed to insert row into " + uri);
}
```

## 总结

本章向你展示了在 Android 中使用内置 SQLite 数据库的基础知识。尽管这里的示例中包含大量代码，但实际你需要为 SQLite 接口编写的只有几行代码。当你为自己的目的重用这些代码时，你编写的大部分代码将与你自己的应用及其数据相关，而不是与数据库本身相关。

