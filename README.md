# NotePad

## 1.添加时间戳

（1）想要在每个列表项中添加时间戳，需要在noteslist_item.xml中创建一个TextView控件来显示时间戳

```
<TextView
    android:id="@+id/uptime"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_marginEnd="3dp"
    android:layout_marginBottom="3dp"
    android:text=""
    app:layout_constraintBottom_toBottomOf="@android:id/text1"
    app:layout_constraintEnd_toEndOf="parent" />
```

（2）要将毫秒数转换为时间的形式yy.MM.dd HH:mm:ss，需要在NotePadProvider中的insert方法下修改时间戳格式，并将COLUMN_NAME_MODIFICATION_DATE赋值为修改后的时间dateFormat

```
Long now = Long.valueOf(System.currentTimeMillis());
Date date = new Date(now);
SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yy-MM-dd HH:mm:ss");
String dateFormat = simpleDateFormat.format(date);

if (values.containsKey(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE) == false) {
	values.put(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE, dateFormat);
}
```

（3）在NoteEditor中的updateNote方法下也进行相应修改

```
// 添加时间戳
long now = System.currentTimeMillis();
Date date = new Date(now);
SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yy-MM-dd HH:mm:ss");
String dateFormat = simpleDateFormat.format(date);
values.put(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE, dateFormat);
```

（4）如果要显示时间戳，必须要将COLUMN_NAME_MODIFICATION_DATE也投影出来. 所以我们在NoteList的PROJECTION中添加上修改时间

```
private static final String[] PROJECTION = new String[]{
    NotePad.Notes._ID, // 0
    NotePad.Notes.COLUMN_NAME_TITLE, // 1
    NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE, // 2
};
```

（5）PROJECTION只是定义了需要被取出来的数据列，而之后用Cursor进行数据库查询，再之后用Adapter进行装填，需要把COLUMN_NAME_MODIFICATION_DATE这一属性加入dataColumns和viewIDs中

```
String[] dataColumns = {NotePad.Notes.COLUMN_NAME_TITLE, NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE};
int[] viewIDs = {android.R.id.text1, R.id.uptime};
```

（6）最终效果如下：

![images](https://github.com/Yechuizz/NotePad/blob/main/pictures/exp1.png)

## 2.添加搜索功能

（1）搜索组件在主页面的菜单选项中，所以应在menu文件夹下的list_options_menu.xml布局文件中添加搜索功能的item控件

```
<item
    android:id="@+id/menu_search"
    android:icon="@drawable/ic_menu_search"
    android:title="Search"
    android:showAsAction="always" />
```

（2）新建一个查找笔记内容的布局文件note_search.xml

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="50dp"
        android:orientation="horizontal">

        <ImageView
            android:id="@+id/search_back"
            android:layout_width="30dp"
            android:layout_height="20dp"
            android:layout_gravity="center_vertical"
            android:src="@drawable/ic_back"/>

        <SearchView
            android:id="@+id/search_txt"
            android:layout_width="match_parent"
            android:layout_height="50dp" />

    </LinearLayout>
    
    <ListView
        android:id="@+id/search_list"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</LinearLayout>
```

（3）在NoteList类中的onOptionsItemSelected方法中添加search查询的处理

```
case R.id.menu_search:
    Intent intent = new Intent(this, NoteSearch.class);
    this.startActivity(intent);
    return true;
```

（4）新建一个NoteSearch类用于search功能的功能实现，当点击search_back时返回主界面，在onQueryTextChange方法中设置查询语句，并为搜索项设置onItemClick事件跳转到相应笔记编辑界面

```
public class NoteSearch extends Activity implements SearchView.OnQueryTextListener
{
    ListView listView;
    SQLiteDatabase sqLiteDatabase;
    private static final String[] PROJECTION = new String[]{
            NotePad.Notes._ID, // 0
            NotePad.Notes.COLUMN_NAME_TITLE, // 1
            NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE, //2
    };
    public boolean onQueryTextSubmit(String query) {
        Toast.makeText(this, "搜索内容为："+query, Toast.LENGTH_SHORT).show();
        return false;
    }
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.note_search);
        SearchView searchView = (SearchView) findViewById(R.id.search_txt);
        Intent intent = getIntent();
        if (intent.getData() == null) {
            intent.setData(NotePad.Notes.CONTENT_URI);
        }
        listView = (ListView) findViewById(R.id.search_list);
        imageView = (ImageView) findViewById(R.id.search_back);
        sqLiteDatabase = new NotePadProvider.DatabaseHelper(this).getReadableDatabase();
        //设置该SearchView显示搜索按钮
        searchView.setSubmitButtonEnabled(true);
        //设置该SearchView内默认显示的提示文本
        searchView.setQueryHint("查找");
        searchView.setOnQueryTextListener(this);
        imageView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                finish();
            }
        });
    }
    public boolean onQueryTextChange(String string) {
        String selection1 = NotePad.Notes.COLUMN_NAME_TITLE+" like ? or "+NotePad.Notes.COLUMN_NAME_NOTE+" like ?";
        String[] selection2 = {"%"+string+"%","%"+string+"%"};
        Cursor cursor = sqLiteDatabase.query(
                NotePad.Notes.TABLE_NAME,
                PROJECTION, 
                selection1, 
                selection2, 
                null,         
                null,      
                NotePad.Notes.DEFAULT_SORT_ORDER
        );
        String[] dataColumns = {
                NotePad.Notes.COLUMN_NAME_TITLE,
                NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE,
        } ;
        int[] viewIDs = {
                android.R.id.text1,
                android.R.id.text2
        };
        SimpleCursorAdapter adapter
                = new SimpleCursorAdapter(
                this,                             // The Context for the ListView
                R.layout.noteslist_item,         // Points to the XML for a list item
                cursor,                           // The cursor to get items from
                dataColumns,
                viewIDs
        );
        listView.setAdapter(adapter);
        listView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
            @Override
            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
                Uri uri = ContentUris.withAppendedId(getIntent().getData(), id);
                String action = getIntent().getAction();
                if (Intent.ACTION_PICK.equals(action) || Intent.ACTION_GET_CONTENT.equals(action)) {
                    setResult(RESULT_OK, new Intent().setData(uri));
                } else {
                    startActivity(new Intent(Intent.ACTION_EDIT, uri));
                }
            }
        });
        return true;
    }
}
```

（5）最后在清单文件AndroidManifest.xml里面注册NoteSearch实现界面的跳转

```
<activity android:name=".NoteSearch" android:label="@string/menu_search" />
```

（6）最终效果如下：

![images](https://github.com/Yechuizz/NotePad/blob/main/pictures/exp2.png)

## 3.扩展功能一：更改记事本的背景

（1）首先在NotePadProvider中数据库的onCreate方法下为数据表添加颜色字段

```
@Override
public void onCreate(SQLiteDatabase db) {
    db.execSQL("CREATE TABLE " + NotePad.Notes.TABLE_NAME + " ("
        + NotePad.Notes._ID + " INTEGER PRIMARY KEY,"
        + NotePad.Notes.COLUMN_NAME_TITLE + " TEXT,"
        + NotePad.Notes.COLUMN_NAME_NOTE + " TEXT,"
        + NotePad.Notes.COLUMN_NAME_CREATE_DATE + " INTEGER,"
        + NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE + " INTEGER,"
        + NotePad.Notes.COLUMN_NAME_BACK_COLOR + " INTEGER" //color
        + ");");
}
```

（2）在契约类NotePad中添加颜色列并定义五种不同颜色

```
public static final String COLUMN_NAME_BACK_COLOR = "color";
public static final int DEFAULT_COLOR = 0; //white
public static final int YELLOW_COLOR = 1; //yellow
public static final int BLUE_COLOR = 2; //blue
public static final int GREEN_COLOR = 3; //green
public static final int PINK_COLOR = 4; //pink
```

（3）由于数据库中多了一个字段，所以要在NotePadProvider中添加对其相应的处理，在static{}中添加以下字段：

```
sNotesProjectionMap.put(
    NotePad.Notes.COLUMN_NAME_BACK_COLOR,
    NotePad.Notes.COLUMN_NAME_BACK_COLOR);
```

在insert方法下添加以下字段：

```
if (values.containsKey(NotePad.Notes.COLUMN_NAME_BACK_COLOR) == false) {
    values.put(NotePad.Notes.COLUMN_NAME_BACK_COLOR, NotePad.Notes.DEFAULT_COLOR);
}
```

（4）自定义一个MyCursorAdapter.java继承SimpleCursorAdapter，将颜色填充到ListView

```
public class MyCursorAdapter extends SimpleCursorAdapter {
    public MyCursorAdapter(Context context, int layout, Cursor c,
                           String[] from, int[] to) {
        super(context, layout, c, from, to);
    }
    @Override
    public void bindView(View view, Context context, Cursor cursor){
        super.bindView(view, context, cursor);
        //Get the color data corresponding to the note list from the cursor read from the database, and set the note color
        int x = cursor.getInt(cursor.getColumnIndexOrThrow(NotePad.Notes.COLUMN_NAME_BACK_COLOR));
        switch (x){
            case NotePad.Notes.YELLOW_COLOR:
                view.setBackgroundColor(Color.rgb(247, 216, 133));
                break;
            case NotePad.Notes.BLUE_COLOR:
                view.setBackgroundColor(Color.rgb(165, 202, 237));
                break;
            case NotePad.Notes.GREEN_COLOR:
                view.setBackgroundColor(Color.rgb(161, 214, 174));
                break;
            case NotePad.Notes.PINK_COLOR:
                view.setBackgroundColor(Color.rgb(251, 208, 233));
                break;
            default:
                view.setBackgroundColor(Color.rgb(255, 255, 255));
                break;
        }
    }
}
```

（5）在NoteList.java中的PROJECTION中添加颜色项

```
private static final String[] PROJECTION = new String[]{
    NotePad.Notes._ID, // 0
    NotePad.Notes.COLUMN_NAME_TITLE, // 1
    NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE, // 2
    NotePad.Notes.COLUMN_NAME_BACK_COLOR, //3
};
```

（6）将NotesList.java中用的SimpleCursorAdapter改为使用MyCursorAdapter

```
MyCursorAdapter adapter = new MyCursorAdapter(
    this,
    R.layout.noteslist_item,
    cursor,
    dataColumns,
    viewIDs
);
```

（7）将NoteSearch.java中用的SimpleCursorAdapter也改为使用MyCursorAdapter，并添加NotePad.Notes.COLUMN_NAME_BACK_COLOR到PROJECTION中，来实现搜索时同样显示笔记背景色

（8）在menu文件下的布局文件editor_options_menu.xml中添加一个控件来显示颜色选项

```
<item
    android:id="@+id/ic_bg_color"
    android:title="change color"
    android:icon="@drawable/ic_bg_color"
    android:showAsAction="always">
</item>
```

（9）在NoteEditor类中的onOptionsItemSelected方法中添加addcolor的处理

```
case R.id.ic_bg_color:
    changeNoteColor();
    break;
```

（10）设置changeNoteColor方法

```
private void changeNoteColor() {
    Intent intent = new Intent(null,mUri);
    intent.setClass(NoteEditor.this,NoteColor.class);
    NoteEditor.this.startActivity(intent);
}
```

（11）新建布局color_layout.xml，对选择颜色界面进行布局

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal" android:layout_width="match_parent"
    android:layout_height="match_parent">
    <ImageButton
        android:id="@+id/color_white"
        android:layout_width="0dp"
        android:layout_height="50dp"
        android:layout_weight="1"
        android:background="@color/colorWhite"
        android:onClick="white"/>
    <ImageButton
        android:id="@+id/color_yellow"
        android:layout_width="0dp"
        android:layout_height="50dp"
        android:layout_weight="1"
        android:background="@color/colorYellow"
        android:onClick="yellow"/>
    <ImageButton
        android:id="@+id/color_blue"
        android:layout_width="0dp"
        android:layout_height="50dp"
        android:layout_weight="1"
        android:background="@color/colorBlue"
        android:onClick="blue"/>
    <ImageButton
        android:id="@+id/color_green"
        android:layout_width="0dp"
        android:layout_height="50dp"
        android:layout_weight="1"
        android:background="@color/colorGreen"
        android:onClick="green"/>
    <ImageButton
        android:id="@+id/color_pink"
        android:layout_width="0dp"
        android:layout_height="50dp"
        android:layout_weight="1"
        android:background="@color/colorPink"
        android:onClick="pink"/>
</LinearLayout>
```

（12）在values文件夹下的colors.xml中添加所需颜色

```
<resources>
    <color name="colorWhite">#fff</color>
    <color name="colorYellow">#FFD885</color>
    <color name="colorBlue">#A5CAED</color>
    <color name="colorGreen">#A1D6AE</color>
    <color name="colorPink">#FBD0E9</color>
</resources>
```

（13）创建NoteColor.java，用来选择颜色

```
public class NoteColor extends Activity {
    private Cursor mCursor;
    private Uri mUri;
    private int color;
    private static final int COLUMN_INDEX_TITLE = 1;
    private static final String[] PROJECTION = new String[] {
            NotePad.Notes._ID, 
            NotePad.Notes.COLUMN_NAME_BACK_COLOR,
    };
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.color_layout);
        mUri = getIntent().getData();
        mCursor = managedQuery(
                mUri,        
                PROJECTION,  
                null,       
                null,        
                null        
        );
    }
    @Override
    protected void onResume(){
        if (mCursor != null) {
            mCursor.moveToFirst();
            color = mCursor.getInt(COLUMN_INDEX_TITLE);
        }
        super.onResume();
    }
    @Override
    protected void onPause() {
        super.onPause();
        ContentValues values = new ContentValues();
        values.put(NotePad.Notes.COLUMN_NAME_BACK_COLOR, color);
        getContentResolver().update(mUri, values, null, null);
    }
    public void white(View view){
        color = NotePad.Notes.DEFAULT_COLOR;
        finish();
    }
    public void yellow(View view){
        color = NotePad.Notes.YELLOW_COLOR;
        finish();
    }
    public void blue(View view){
        color = NotePad.Notes.BLUE_COLOR;
        finish();
    }
    public void green(View view){
        color = NotePad.Notes.GREEN_COLOR;
        finish();
    }
    public void pink(View view){
        color = NotePad.Notes.PINK_COLOR;
        finish();
    }
}
```

（14）最后在清单文件AndroidManifest.xml里面注册NoteColor实现界面的跳转

```
<activity
    android:name=".NoteColor"
    android:label="ChangeNoteColor"
    android:theme="@android:style/Theme.Holo.Light.Dialog"
    android:windowSoftInputMode="stateVisible" />
```

（15）最终效果如下：

![images](https://github.com/Yechuizz/NotePad/blob/main/pictures/exp3-1.png)

![images](https://github.com/Yechuizz/NotePad/blob/main/pictures/exp3-2.png)

## 4.扩展功能二：笔记导出

（1）在menu文件下的布局文件editor_options_menu.xml中添加一个控件来显示笔记导出选项

```
<item
    android:id="@+id/ic_note_out"
    android:title="note export"
    android:icon="@drawable/ic_note_out"
    android:showAsAction="always">
    </item>
```

（2）在NoteEditor类中的onOptionsItemSelected方法中添加noteexport的处理,将文件保存到本地的data/data/目录下

```
case R.id.ic_note_out:
    try {
        String fileName = createFileName();
        FileOutputStream fileout=openFileOutput(fileName, MODE_PRIVATE);
        OutputStreamWriter outputWriter=new OutputStreamWriter(fileout);
        outputWriter.write(mOriginalContent);
        outputWriter.close();
        Toast.makeText(getBaseContext(), "File saved successfully!",
        Toast.LENGTH_SHORT).show();
    } catch (Exception e) {
    e.printStackTrace();
    }
    break;
```

（3）设置createFileName方法获取随机文件名

```
public String  createFileName(){
    Random random = new Random();
    int length  = 10;
    String numstr = "123456789";
    String chastr_b = "ABCDEFGHIJKLMNOPQRSTUVWXYZ" ;
    String chastr_s = "abcdefghijklmnopqrstuvwxyz";
    String specil = "_";
    String base = numstr+chastr_b+chastr_s+specil;
    StringBuffer sb =  new StringBuffer();
    sb.append(chastr_b.charAt(random.nextInt(chastr_b.length())));
    for(int i =0 ;i <length-2;i++){
    int num = random.nextInt(base.length());
    sb.append(base.charAt(num));
    }
    sb.append(numstr.charAt(random.nextInt(numstr.length())));
    return sb.toString();
}
```

（4）最终可以在data/data/目录下找到文件

![images](https://github.com/Yechuizz/NotePad/blob/main/pictures/exp4-1.png)

![images](https://github.com/Yechuizz/NotePad/blob/main/pictures/exp4-2.png)