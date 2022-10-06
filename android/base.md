## Android基础知识

## 一、布局

- 线性布局（组件从上往下，从左往右，这种布局最常用）

- 相对布局（每个组件位置相对，用的比较多）
- 帧布局（层级布局）
- 表格布局（常见的是计算器）
- 网格布局（常见的就是2048游戏）
- 约束布局（当前用的比较多）

### 1. 布局添加的两种方式

利用xml文件设计和使用java代码添加

例如：使用java设置线性布局

```java
// 1. 设置根部局为线性布局
LinearLayout ll = new LinearLayout(this);
// 2. 设置宽高
ll.setLayoutParams(new LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,
                                                 ViewGroup.LayoutParams.MATCH_PARENT));
// 3. 背景设置为红色
ll.setBackgroundColor(Color.RED);
// 4. 指定此Activity的内容试图为该线性布局
setContentView(ll);
```

不推荐使用Java代码进行布局，布局或逻辑应当分开控制，即使用xml来定义布局，java代码进行逻辑控制。

### 2. 线性布局的重要属性

所有的布局都具有如下属性

- android:layout_width    宽度；通常设置为 match_parent 、 wrap_content  和一个指定的 dp 值

- android:layout_height    高度；通常设置为 match_parent 、 wrap_content  和一个指定的 dp 值

- android:layout_padding    内边距；通常指定 dp 值

- android:layout_margin   外边距；通常指定 dp 值

- android:orientation  方向；  vertical 垂直的， horizontal 水平的 ， 默认为水平，但是最好将水平写出来

- android:layout_weight 权重； 作用于控件，传一个整型的数据，实际上为了按比例划分空间 。在设置按比例调整控件位置时，需要将对应的宽或者高设置为 0dp，如果是水平就是宽，如果是垂直就是高。这样做的原因是，如果TextView内的文本长度较长，那么文本的TextView的宽度会有所增加，会挤压其他控件占有的空间。因此要设置0dp。

- android:layout_gravity  重力； 作用于控件，设置在父容器摆放重力，实际上就是设置这个控件在自己垂直领域内或者水平领域内的位置

  **案例：**做一个聊天界面

  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
      android:orientation="vertical"
      android:layout_width="match_parent"
      android:layout_height="match_parent">
  
      <LinearLayout
          android:layout_width="match_parent"
          android:layout_height="60dp"
          android:orientation="horizontal"
          android:background="#333333"
          android:paddingLeft="15dp"
          >
          <TextView
              android:layout_width="wrap_content"
              android:layout_height="wrap_content"
              android:text="<"
              android:textColor="#ffffff"
              android:textSize="26sp"
              android:layout_gravity="center_vertical" />
          <TextView
              android:layout_width="0dp"
              android:layout_height="wrap_content"
              android:text="小p"
              android:textColor="#ffffff"
              android:textSize="26sp"
              android:layout_gravity="center_vertical"
              android:layout_weight="1"
              android:gravity="center_horizontal"/>
  
          <ImageView
              android:layout_width="wrap_content"
              android:layout_height="wrap_content"
              android:src="@mipmap/ic_launcher"
              />
      </LinearLayout>
  
      <LinearLayout
          android:layout_width="match_parent"
          android:layout_height="match_parent"
          android:orientation="horizontal"
          android:layout_weight="1"></LinearLayout>
  
      <LinearLayout
          android:layout_width="match_parent"
          android:layout_height="60dp"
          android:orientation="horizontal"
          android:background="#cccccc">
      </LinearLayout>
  
  </LinearLayout>
  ```

### 3. 相对布局的重要属性

- android:layout_alignParentRight  相对于父容器右边；取值为true / false
- android:layout_alignParentLeft  相对于父容器左边；取值为true / false
- android:layout_centerInParent 相对于父容器中间；取值为true / false
- android:layout_centerInHorizontal  相对于父容器水平居中；取值为true / false
- android:layout_centerVertical  相对于父容器垂直居中；取值为true / false
- android:layout_alignParentBottom  相对于父容器下边；取值为true / false
- android:layout_alignParentTop  相对于父容器上边；取值为true / false
- android:layout_toRightOf   相对于其他控件右边；取值为其他控件的id
- android:layout_toLeftOf   相对于其他控件左边；取值为其他控件的id
- android:layout_above   相对于其他控件上面；取值为其他控件的id
- android:layout_below   相对于其他控件下面；取值为其他控件的id
- android:id="@+id/center"  控件的id
- android:layout_alignTop  相对于参照物上边线对齐；取值为其他控件的id
- android:layout_alignLeft  相对于参照物左边线对齐；取值为其他控件的id
- android:layout_alignRight 相对于参照物右边线对齐；取值为其他控件的id
- android:layout_alignBottom  相对于参照物下边线对齐；取值为其他控件的id

**案例1：**做一个五个方格相对位置

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/center"
        android:layout_width="100dp"
        android:layout_height="100dp"
        android:textSize="30sp"
        android:text="屏幕正中"
        android:background="#ff0000"
        android:layout_centerInParent="true"
        />

    <TextView
        android:layout_width="100dp"
        android:layout_height="100dp"
        android:textSize="30sp"
        android:text="左上"
        android:background="#00ff00"
        android:layout_toLeftOf="@+id/center"
        android:layout_above="@+id/center"
        />

    <TextView
        android:layout_width="100dp"
        android:layout_height="100dp"
        android:textSize="30sp"
        android:text="右上"
        android:background="#ff0000"
        android:layout_toRightOf="@+id/center"
        android:layout_above="@+id/center"
        />

    <TextView
        android:layout_width="100dp"
        android:layout_height="100dp"
        android:textSize="30sp"
        android:text="左下"
        android:background="#ff0000"
        android:layout_toLeftOf="@id/center"
        android:layout_below="@+id/center"
        />

    <TextView
        android:layout_width="100dp"
        android:layout_height="100dp"
        android:textSize="30sp"
        android:text="屏幕正中"
        android:background="#ff0000"
        android:layout_toRightOf="@id/center"
        android:layout_below="@+id/center"
        />

</RelativeLayout>
```

**案例2：**做一个热门电影推荐页面

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:orientation="horizontal"
        android:background="#ff0000"
        android:layout_weight="2"
        >
        
        <RelativeLayout
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="2"
            android:background="#ff00ff">
            
            <ImageView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:src="@mipmap/ic_launcher_round"/>
            <TextView
                android:layout_width="match_parent"
                android:layout_height="60dp"
                android:text="复仇者联盟"
                android:layout_alignParentBottom="true"
                android:textSize="36sp"
                android:textColor="#ffffff"
                android:background="#666666"/>
        </RelativeLayout>
        
        
        <LinearLayout
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"
            android:orientation="vertical">

            <RelativeLayout
                android:layout_width="match_parent"
                android:layout_height="0dp"
                android:layout_weight="1"
                android:background="#ffff00"/>
            <RelativeLayout
                android:layout_width="match_parent"
                android:layout_height="0dp"
                android:layout_weight="1"
                android:background="#ff0000"/>

        </LinearLayout>
        
    </LinearLayout>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:orientation="horizontal"
        android:background="#00ff00"
        android:layout_weight="1"
        >

        <RelativeLayout
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"
            android:background="#0000ff"/>
        <RelativeLayout
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"
            android:background="#00ff00"/>
        <RelativeLayout
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"
            android:background="#00ffff"/>

    </LinearLayout>
</LinearLayout>
```

### 4. 帧布局的重要属性

使用不多。

- android:layout_gravity

### 5. 表格布局的重要属性



### 6. 网格布局的重要属性



### 7. 约束布局的重要属性



## 二、UI基础控件

### 1. View

view是所有控件的附件，代表的是一个空白区域。

- TextView ：处理文本内容的View
- Button：被点击的View
- ImageView：处理图片内容的View
- EditText：接收用户信息输入的View
- ProgressBar：进度条类的View

### 2. TextView

继承关系，属性也是继承

View <- TextView <- Button

View <- TextView <- EditText

TextView能够对长文本进行显示处理，支持HTML代码，内容有样式，链接效果

- 
  文字大小：android:textSize="22sp"

- 文字颜色：android:textColor="#00ff00"

- 文字行间距：android:lineSpacingMultiplier="1.25"

如果TextView的文本太长，当前页面显示不完全，可以使用ScrollView来布局，但是ScrollView只支持一个TextView，这时候，如果要使用多个TextView的话，应该在TextView外层多加一个布局。

### 3. Button

Button注册点击事件的方法：

- 自定义内部类（自定义一个类实现OnClickListener接口，实现OnClick方法）

- 匿名内部类（适用于操作唯一的时候使用）

- 当前Activity去实现事件接口（本类实现OnClickListener接口，实现OnClick方法）

- 在布局文件中添加点击事件属性

**案例：**实现各种方法的点击事件

  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:tools="http://schemas.android.com/tools"
      android:layout_width="match_parent"
      android:layout_height="match_parent"
      tools:context=".ButtonActivity"
      android:orientation="vertical">
  
      <Button
          android:id="@+id/btn1"
          android:layout_width="match_parent"
          android:layout_height="wrap_content"
          android:text="通过自定义内部类实现点击事件"/>
  
      <Button
          android:id="@+id/btn2"
          android:layout_width="match_parent"
          android:layout_height="wrap_content"
          android:text="通过匿名内部类实现点击事件"/>
  
      <Button
          android:id="@+id/btn3"
          android:layout_width="match_parent"
          android:layout_height="wrap_content"
          android:text="通过当前Activity去实现点击事件接口"/>
  
      <Button
          android:id="@+id/btn4"
          android:layout_width="match_parent"
          android:layout_height="wrap_content"
          android:text="在xml文件中绑定"
          android:onClick="myClick"
          />
  
  
      <Button
          android:id="@+id/btn5"
          android:layout_width="match_parent"
          android:layout_height="wrap_content"
          android:text="在xml文件中绑定2"
          android:onClick="myClick"
          />
  
  </LinearLayout>
  ```

  ```java
  import androidx.appcompat.app.AppCompatActivity;
  import android.os.Bundle;
  import android.util.Log;
  import android.view.View;
  import android.widget.Button;
  
  public class ButtonActivity extends AppCompatActivity implements View.OnClickListener {
  
      @Override
      protected void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          setContentView(R.layout.activity_button);
          // 方法一：获取按钮
          Button btn1 = findViewById(R.id.btn1);
          // 自定义内部类实现监听
          btn1.setOnClickListener(new MyClickListener());
  
          // 方法二：获取按钮
          Button btn2 = findViewById(R.id.btn2);
          btn2.setOnClickListener(new View.OnClickListener() {
              @Override
              public void onClick(View view) {
                  Log.e("TAG", "刚刚点击的按钮时，注册了匿名内部类监听器对象的按钮");
              }
          });
  
          // 方法三：获取按钮
          Button btn3 = findViewById(R.id.btn3);
          btn3.setOnClickListener(this);
  
      }
  
      @Override
      public void onClick(View view) {
          Log.e("TAG", "用本类实现监听事件");
      }
  
      /**
       * 自定义内部类实现监听器
       */
      class MyClickListener implements View.OnClickListener{
          @Override
          public void onClick(View view) {
              // 在控制台输出语句
              Log.e("TAG", "刚刚点击的按钮时，注册了内部类监听器对象的按钮");
          }
      }
  
      // 绑定xml
      public void myClick(View v){
          switch (v.getId()){
              case R.id.btn4:
                  Log.e("TAG", "btn4 通过xml绑定的点击事件");
                  break;
              case R.id.btn5:
                  Log.e("TAG", "btn5 通过xml绑定的点击事件");
                  break;
          }
      }
  }
  ```

### 4. ImageView

- android:src   指定前景图片资源

- android:background  设置背景

### 5. ProgressBar

- 设置进度条风格：style="?android:attr/progressBarStyleHorizontal"

- 设置默认进度：android:progress="30"

- 设置最大值：android:max="200"

- 是否永恒滚动：android:indeterminate="true"

**案例一：**

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".ProgressBarActivity"
    android:orientation="vertical"
    android:background="#fff">

    <!--
        进度条：默认样式是转圈。修改样式需设置风格
        style 设置风格progressBarStyleHorizontal（水平进度条）
        android:progress=""   设置进度
        android:max=""      设置最大值，默认100
        android:indeterminate="true"   设置进度条一直滚动
    -->
    <ProgressBar
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

    <ProgressBar
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        style="?android:attr/progressBarStyleHorizontal"
        android:progress="30"
        android:max="200"/>

    <ProgressBar
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        style="?android:attr/progressBarStyleHorizontal"
        android:indeterminate="true"/>

    <ProgressBar
        android:id="@+id/progress"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        style="?android:attr/progressBarStyleHorizontal"/>
</LinearLayout>
```

```java
import androidx.appcompat.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.ProgressBar;

public class ProgressBarActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_progress_bar);

        ProgressBar progressBar = findViewById(R.id.progress);
        progressBar.setProgress(80);

        // android中，4.0以后不能直接在线程中操作控件，但是进度条是个特例
        new Thread() {
            @Override
            public void run() {
                for (int i = 0; i <= 100; i++) {
                    progressBar.setProgress(i);
                    try {
                        Thread.sleep(30);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }.start();

    }
}
```

**案例二：**完善注册页面

```java
public void register(View v){
        //1.判断姓名、密码是否为空
        EditText nameEdt = findViewById(R.id.name);
        EditText pwdEdt = findViewById(R.id.pwd);
        final ProgressBar proBar = findViewById(R.id.pro_bar);

        String name = nameEdt.getText().toString();
        String pwd = pwdEdt.getText().toString();
        if(name.equals("") || pwd.equals("")) {
            //2.如果为空，则提示
            //无焦点提示
            //参数1：环境上下文     参数2：提示性文本    参数3：提示持续时间
            Toast.makeText(this,"姓名或密码不能为空",Toast.LENGTH_SHORT).show();
        }else {
            //3.都不为空，则出现进度条
            proBar.setVisibility(View.VISIBLE);
            new Thread(){
                @Override
                public void run() {
                    for(int i = 0 ; i <= 100 ; i++){
                        proBar.setProgress(i);
                        try {
                            Thread.sleep(30);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }.start();
        }
    }
```

