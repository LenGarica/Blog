## Android基础知识

> 观察App运行日志
>
> Log.e：表示错误信息
>
> Log.w：表示警告信息
>
> Log.i：表示一般消息
>
> Log.d：表示调试信息（日常使用的最多）
>
> Log.v：表示 冗余消息

AndroidManifest.xml文件实际上就是一个activity映射文件，这一点很像jsp学习时候的，有一个服务映射文件。

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

- android:layout_gravity（确定某个控件的相对位置）

### 5. 表格布局的重要属性（TableLayout）

以行列的形式布局，最常见的就是计算器

android:stretchColumns  代表可以伸展的列（可以指定要伸展的列的列号，指定所有的列可以使用*来表示）

android:shrinkColumns  代表可以收缩的列（可以指定要伸展的列的列号，指定所有的列可以使用*来表示）

android:collapseColumns  代表可以隐藏的列（可以指定要伸展的列的列号，指定所有的列可以使用*来表示）

### 6. 网格布局的重要属性（GridLayout）

与表格布局类似，但是网格布局能够自己声明行和列的数量，比表格布局灵活。不设置组件的宽高时，会自动给组件宽高。

android:rowCount（行数量）

android:columnCount（列数量）

android:layout_row（位于第几行）

android:layout_rowSpan（跨几行，并搭配android:layout_gravity=“fill”可以显示该组件能够显示的行号）

android:layout_columnSpan="列号"（设置某个组件可以跨越几个列，并搭配android:layout_gravity=“fill”可以显示该组件能够显示的列号）

android:orientation（可声明垂直或者水平属性）

### 7. 约束布局的重要属性

该布局适合拖拽组件，然后调整

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

### 6.CheckBox

复选控件，两种状态，选中以及未选中setChecked()，isChecked()。

监听状态的变化：setOnCheckedChangeListener

### 7.RadioButton

单选控件，可以和RadioGroup一起使用。

### 8.ToggleButton

切换程序中的状态，有两种状态

android:textOn

android:textOff

setChecked(boolean)

监听状态的变化：setOnCheckedChangeListener

### 9.SeekBar

可以拉的进度条，例如手机上的调节声音的那个条

setProgress：设置最初的条状进度

setOnSeekBarChangeListener：监听状态的变化

## 三、UI控件实战——点餐页面

### 1.页面设计

整个页面的大体情况如下：

![diancan](..\picture\diancan.png)

### 2.页面布局代码

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity"
    >

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:orientation="vertical">

        <LinearLayout
            android:layout_marginLeft="15dp"
            android:gravity="center_vertical"
            android:layout_width="match_parent"
            android:orientation="horizontal"
            android:layout_height="40dp">
            <TextView
                android:textSize="22sp"
                android:text="@string/name"
                android:layout_width="60dp"
                android:layout_height="wrap_content"/>
            <EditText
                android:id="@+id/nameEditText"
                android:layout_width="200dp"
                android:layout_height="wrap_content"
                android:hint="@string/edittext_input_hint_name"
                />
        </LinearLayout>

        <LinearLayout
            android:layout_marginLeft="15dp"
            android:gravity="center_vertical"
            android:layout_width="match_parent"
            android:orientation="horizontal"
            android:layout_height="40dp">
            <TextView
                android:textSize="22sp"
                android:text="@string/sex"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"/>
            <RadioGroup
                android:id="@+id/sexRadioGroup"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="horizontal"
                >
                <RadioButton
                    android:id="@+id/manRadioButton"
                    android:layout_width="75dp"
                    android:layout_height="wrap_content"
                    android:text="@string/sex_man"
                    android:textSize="22sp"
                    />
                <RadioButton
                    android:id="@+id/womanRadioButton"
                    android:layout_width="75dp"
                    android:layout_height="wrap_content"
                    android:text="@string/sex_woman"
                    android:textSize="22sp"
                    />
            </RadioGroup>
        </LinearLayout>

        <LinearLayout
            android:layout_marginLeft="15dp"
            android:gravity="center_vertical"
            android:layout_width="match_parent"
            android:orientation="horizontal"
            android:layout_height="40dp">
            <TextView
                android:textSize="22sp"
                android:text="@string/hobby"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"/>

            <LinearLayout
                android:orientation="horizontal"
                android:layout_width="match_parent"
                android:layout_height="wrap_content">
                <CheckBox
                    android:id="@+id/hotCheckBox"
                    android:text="@string/hobby_hot"
                    android:textSize="22sp"
                    android:layout_width="60dp"
                    android:layout_height="wrap_content"/>
                <CheckBox
                    android:id="@+id/seafoodCheckBox"
                    android:text="@string/hobby_seafood"
                    android:textSize="22sp"
                    android:layout_width="80dp"
                    android:layout_height="wrap_content"/>
                <CheckBox
                    android:id="@+id/garlicCheckBox"
                    android:text="@string/hobby_garlic"
                    android:textSize="22sp"
                    android:layout_width="80dp"
                    android:layout_height="wrap_content"/>
            </LinearLayout>

        </LinearLayout>

        <LinearLayout
            android:layout_marginLeft="15dp"
            android:gravity="center_vertical"
            android:layout_width="match_parent"
            android:orientation="horizontal"
            android:layout_height="40dp">
            <TextView
                android:textSize="22sp"
                android:text="@string/budget"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"/>


            <TextView
                android:textSize="20sp"
                android:text="@string/zero_yuan"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"/>

            <SeekBar
                android:id="@+id/seekBar"
                android:textSize="20sp"
                android:layout_width="160dp"
                android:max="100"
                android:layout_height="wrap_content"
            />

            <TextView
                android:textSize="20sp"
                android:text="@string/hundred_yuan"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"/>

        </LinearLayout>

        <Button
            android:id="@+id/searchButton"
            android:layout_width="300dp"
            android:layout_height="50dp"
            android:text="@string/find_menu_button"
            android:layout_gravity="center_horizontal"
            android:gravity="center_horizontal"
            android:textSize="22sp"
            />
    </LinearLayout>

    <LinearLayout
        android:layout_width="match_parent"
        android:orientation="vertical"
        android:layout_height="0dp"
        android:layout_weight="1"
        >

        <ImageView
            android:id="@+id/foodImageView"
            android:src="@drawable/ic_launcher_foreground"
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_weight="3"/>

        <ToggleButton
            android:id="@+id/showToggleButton"
            android:textOff="下一个"
            android:textOn="显示信息"
            android:layout_gravity="center_horizontal"
            android:gravity="center_horizontal"
            android:textSize="22sp"
            android:layout_width="300dp"
            android:layout_height="50dp"
            android:text="@string/find_menu_button"
            />


    </LinearLayout>


</LinearLayout>
```

### 3.业务逻辑代码

```java
package com.example.baseandroid;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;

import android.text.Editable;
import android.text.TextWatcher;
import android.util.Log;
import android.widget.Button;
import android.widget.CheckBox;
import android.widget.EditText;
import android.widget.ImageView;
import android.widget.RadioGroup;
import android.widget.SeekBar;
import android.widget.Toast;
import android.widget.ToggleButton;


import com.example.baseandroid.model.Food;
import com.example.baseandroid.model.Person;

import java.util.ArrayList;
import java.util.List;

public class MainActivity extends AppCompatActivity {

    // 文本编辑框对象
    private EditText mNameEditText;
    // 单选组件
    private RadioGroup mSexRadioGroup;
    // 多选组件
    private CheckBox mHotCheckBox, mSeaFoodCheckBox, mGarlicCheckBox;
    // 调节条组件
    private SeekBar mSeekBar;
    // 按钮
    private Button mSearchButton;
    // 图片
    private ImageView mFoodImageView;
    // 可变按钮
    private ToggleButton mToggleButton;
    private List<Food> mFoods;
    private Person mPerson;
    private List<Food> mFoodResult;
    private boolean mIsFish;
    private boolean mIsGarlic;
    private boolean mIsHot;
    private int mPrice;
    private int mCurrentIndex;


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        // 初始化控件
        findViews();
        // 菜品初始化数据
        initData();
        // 为控件添加监听器，实现基本功能
        setListeners();

        // 自测

    }

    /**
     * 初始化所有的菜品
     */
    private void initData() {
        mFoods = new ArrayList<>();
        // 初始化添加所有的数据
        mFoods.add(new Food("麻辣香锅", 55, R.drawable.malaxiangguo, true, false, false));
        mFoods.add(new Food("水煮鱼", 48, R.drawable.shuizhuyu, true, true, false));
        mFoods.add(new Food("麻辣火锅", 80, R.drawable.malahuoguo, true, true, false));
        mFoods.add(new Food("清蒸鲈鱼", 68, R.drawable.qingzhengluyu, false, true, false));
        mFoods.add(new Food("桂林米粉", 15, R.drawable.guilin, false, false, false));
        mFoods.add(new Food("上汤娃娃菜", 28, R.drawable.wawacai, false, false, false));
        mFoods.add(new Food("红烧肉", 60, R.drawable.hongshaorou, false, false, false));
        mFoods.add(new Food("木须肉", 40, R.drawable.muxurou, false, false, false));
        mFoods.add(new Food("酸菜牛肉面", 35, R.drawable.suncainiuroumian, false, false, true));
        mFoods.add(new Food("西芹炒百合", 38, R.drawable.xiqin, false, false, false));
        mFoods.add(new Food("酸辣汤", 40, R.drawable.suanlatang, true, false, true));

        mPerson = new Person();

        mFoodResult = new ArrayList<>();
    }

    /**
     * 设置所有的组件
     */
    private void findViews() {
        mNameEditText = findViewById(R.id.nameEditText);
        mSexRadioGroup = findViewById(R.id.sexRadioGroup);
        mHotCheckBox = findViewById(R.id.hotCheckBox);
        mSeaFoodCheckBox = findViewById(R.id.seafoodCheckBox);
        mGarlicCheckBox = findViewById(R.id.garlicCheckBox);
        mSeekBar = findViewById(R.id.seekBar);
        mSeekBar.setProgress(30);
        mSearchButton = findViewById(R.id.searchButton);
        mToggleButton = findViewById(R.id.showToggleButton);
        mToggleButton.setChecked(true);
        mFoodImageView = findViewById(R.id.foodImageView);
    }

    /**
     * 设置监听器
     */
    private void setListeners() {
        mNameEditText.addTextChangedListener(new TextWatcher() {
            /**
             * 即将改变
             * @param s
             * @param start
             * @param count
             * @param after
             */
            @Override
            public void beforeTextChanged(CharSequence s, int start, int count, int after) {

            }

            /**
             * 改变完了
             * @param s
             * @param start
             * @param before
             * @param count
             */
            @Override
            public void onTextChanged(CharSequence s, int start, int before, int count) {

            }

            @Override
            public void afterTextChanged(Editable s) {
                if(mPerson != null) {
                    mPerson.setName(s.toString());
                }
            }
        });
        // 性别
        mSexRadioGroup.setOnCheckedChangeListener((radioGroup, checkedId) -> {
            switch (checkedId) {
                case R.id.manRadioButton:
                    mPerson.setSex("男");
                    break;
                case R.id.womanRadioButton:
                    mPerson.setSex("女");
                    break;
            }
            Log.d("用户选择的性别是：", mPerson.getSex());
        });
        // 海鲜选择器
        mSeaFoodCheckBox.setOnCheckedChangeListener((compoundButton, isChecked) -> {
            mIsFish = isChecked;
            Log.d("用户是否选择了海鲜:", String.valueOf(mIsFish));
        });
        // 烤肉选择器
        mGarlicCheckBox.setOnCheckedChangeListener((compoundButton, isChecked) -> {
            mIsGarlic = isChecked;
            Log.d("用户是否选择了烤肉:", String.valueOf(mIsGarlic));
        });
        // 火锅选择器
        mHotCheckBox.setOnCheckedChangeListener((compoundButton, isChecked) -> {
            mIsHot = isChecked;
            Log.d("用户是否选择了火锅:", String.valueOf(mIsHot));
        });
        // 拉条选择器
        mSeekBar.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
            @Override
            public void onProgressChanged(SeekBar seekBar, int i, boolean b) {

            }

            @Override
            public void onStartTrackingTouch(SeekBar seekBar) {

            }

            @Override
            public void onStopTrackingTouch(SeekBar seekBar) {
                mPrice = seekBar.getProgress();
                Toast.makeText(MainActivity.this, "价格" + mPrice, Toast.LENGTH_SHORT).show();
            }
        });
        // 查询选择器
        mSearchButton.setOnClickListener(view -> search());
        // 状态改变选择器（通常可以用来作为下一个，上一个按钮）
        mToggleButton.setOnClickListener(view -> {
            if (mToggleButton.isChecked()) {
                mCurrentIndex++;
                if (mCurrentIndex < mFoodResult.size()) {
                    mFoodImageView.setImageResource(mFoodResult.get(mCurrentIndex).getPic());
                }
            } else {
                if (mCurrentIndex < mFoodResult.size()) {
                    String foodName = mFoodResult.get(mCurrentIndex).getName();
//                    String personName = mNameEditText.getText().toString();
                    String sex = mPerson.getSex();
                    Toast.makeText(MainActivity.this, "菜名" + foodName + ",人名" + mPerson.getName() + ",性别" + sex, Toast.LENGTH_SHORT).show();
                } else {
                    Toast.makeText(MainActivity.this, "没有啦", Toast.LENGTH_SHORT).show();
                }
            }
        });

    }

    /**
     * 查找菜品
     */
    private void search() {
        // 结果列表清空
        // 先初始化
        if (mFoodResult == null) {
            mFoodResult = new ArrayList<>();
        }
        // 清除
        mFoodResult.clear();
        // 显示结果中的第几个菜
        mCurrentIndex = 0;

        // 遍历所有菜品
        // 如果符合条件，则加入到结果列表中
        for (int i = 0; i < mFoods.size(); i++) {
            Food food = mFoods.get(i);
            if (food != null) {
                // 价格小于设定的价格，同时是顾客选择的口味
                if (food.getPrice() <= mPrice
                        && (food.isHot() == mIsHot || food.isFish() == mIsFish || food.isGarlic() == mIsGarlic)) {
                    mFoodResult.add(food);
                }
            }
        }
        // 返回图片信息
        if (mCurrentIndex < mFoodResult.size()) {
            mFoodImageView.setImageResource(mFoodResult.get(mCurrentIndex).getPic());
        }
    }

}
```

补上两个model

```java
package com.example.baseandroid.model;

public class Food {
    private String name;
    private int price;
    private int pic;
    private boolean hot;
    private boolean fish;
    private boolean garlic;

    public Food() {
    }

    public Food(String name, int price, int pic, boolean hot, boolean fish, boolean garlic) {
        this.name = name;
        this.price = price;
        this.pic = pic;
        this.hot = hot;
        this.fish = fish;
        this.garlic = garlic;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getPrice() {
        return price;
    }

    public void setPrice(int price) {
        this.price = price;
    }

    public int getPic() {
        return pic;
    }

    public void setPic(int pic) {
        this.pic = pic;
    }

    public boolean isHot() {
        return hot;
    }

    public void setHot(boolean hot) {
        this.hot = hot;
    }

    public boolean isFish() {
        return fish;
    }

    public void setFish(boolean fish) {
        this.fish = fish;
    }

    public boolean isGarlic() {
        return garlic;
    }

    public void setGarlic(boolean garlic) {
        this.garlic = garlic;
    }

    @Override
    public String toString() {
        return "Food{" +
                "name='" + name + '\'' +
                ", price='" + price + '\'' +
                ", pic=" + pic +
                ", hot=" + hot +
                ", fish=" + fish +
                ", garlic=" + garlic +
                '}';
    }
}
```

```java
package com.example.baseandroid.model;

public class Person {

    private String name;

    private String sex;

    private Food food;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getSex() {
        return sex;
    }

    public void setSex(String sex) {
        this.sex = sex;
    }

    public Food getFood() {
        return food;
    }

    public void setFood(Food food) {
        this.food = food;
    }
}

```

## 四、Activity

> 安卓开发有四大组件：Activity、Service、BroadcastReceiver、ContentProvider

### 1.Activity入门

实际上Activity就是一个活动，一个个能看到的页面就是Activity，能够给用户直接展示的界面，和接受用户交互的数据。

### 2.Activity之间的跳转

```java
Intent intent = new Intent(当前Activity.this, newActivity.class);
startActivity(intent);
```

### 3.Activity启动模式

- standard：标准模式（默认的模式）
- singleTop：可复用顶部activity，如果不在顶部重启一个Activity
- singleTask：清除在自己上面的Activity，运行自己的Activity
- singleInstance：重启一个栈，存放这个Activity，全局只有这一个Activity，可复用。

## 五、Handler

主要的功能，就是在其他线程更新UI。

每个应用启动的时候，会启动一个主进程然后开启要给主线程，这个主线程也叫UI线程，负责处理界面上的UI消息分发。例如，点击要给Button，这个点击事件就是UI线程来处理的。但这个线程不线程安全。

![handler](..\picture\handler.png)

### 1.常用方法

handler.sendMessage()

Handler.post()

### 2.实现异步下载并更新进度条

主线程start-》点击按键-》发起下载-》开启子线程做下载-》下载过程中通知主线程-》主线程更新进度条

页面代码：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">

    <Button
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Button" />

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <ProgressBar
            android:id="@+id/progressBar"
            style="?android:attr/progressBarStyleHorizontal"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:max="100"
            android:layout_gravity="center_horizontal"/>
    </LinearLayout>
</LinearLayout>
```

业务逻辑

```java
package com.example.hadnlerbase;

import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.os.Environment;
import android.os.Handler;
import android.os.Message;
import android.widget.ProgressBar;

import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLConnection;

public class MainActivity extends AppCompatActivity {

    public static final int DOWNLOAD_MESSAGE_CODE = 100001;

    private static final int DOWNLOAD_MESSAGE_FAIL_CODE = 100002;

    private static final String APP_URL = "http://download.lgdsunday.club/imooc/jdApp/%E6%85%95%E8%AF%BE%E7%BD%91%E9%AB%98%E4%BB%BF%E4%BA%AC%E4%B8%9C%E5%95%86%E5%9F%8E.apk";

    public static Handler mHandler = new Handler() {
        @Override
        public void handleMessage(@NonNull Message msg) {
            super.handleMessage(msg);
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ProgressBar progressBar = findViewById(R.id.progressBar);

        findViewById(R.id.button).setOnClickListener(view -> {
            new Thread(() -> download(APP_URL)).start();
            download("");
        });

        mHandler = new Handler() {
            @Override
            public void handleMessage(@NonNull Message msg) {
                super.handleMessage(msg);
                switch (msg.what) {
                    case DOWNLOAD_MESSAGE_CODE:
                        progressBar.setProgress((Integer) msg.obj);
                        break;
                    case DOWNLOAD_MESSAGE_FAIL_CODE:
                        break;
                }
            }
        };

    }

    /**
     * 根据URL下载文件
     * @param URL
     */
    private void download(String URL) {
        try {
            URL url = new URL(URL);
            URLConnection connection = url.openConnection();
            InputStream inputStream = connection.getInputStream();
            /**
             * 获取文件的总长度
             */
            int contentLength = connection.getContentLength();
            String downloadFileName = Environment.getExternalStorageDirectory() + File.separator + "imooc" + File.separator;

            File file = new File(downloadFileName);
            if(!file.exists()) {
                file.mkdir();
            }

            String fileName = downloadFileName + "imooc.apk";
            File apkFile = new File(fileName);
            if(apkFile.exists()) {
                apkFile.delete();
            }

            int downloadSize = 0;
            byte[] bytes = new byte[1024];

            int length = 0;

            OutputStream outputStream = new FileOutputStream(fileName);
            while ((length = inputStream.read(bytes)) != -1) {
                outputStream.write(bytes, 0, length);
                downloadSize += length;

                Message message = Message.obtain();
                message.obj = downloadSize * 100 / contentLength;
                message.what = DOWNLOAD_MESSAGE_CODE;

                mHandler.sendMessage(message);
            }

            inputStream.close();
            outputStream.close();

        } catch (IOException e) {
            Message message = Message.obtain();
            message.what = DOWNLOAD_MESSAGE_FAIL_CODE;
            mHandler.sendMessage(message);
            e.printStackTrace();
        }
    }
}
```

## 六、ListView

简言之，就是瀑布流，可下滑页面。在很多app中，我们都看到了这样的，比如通讯录。ListView实际上就是一行行横向的View构成一个整体的可以上下滑动的View。每一行View又能够设置多种试图组件。

本质上，就是一个View，和TextView、Button是一样的。 

### 1.实战一——自定义数据展示ListView

> 需求说明：实现App商城的样式，即一行里，左侧是App图标，右侧是App名称。

首先定义主页面，代码如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <ListView
        android:id="@+id/app_listView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        />
</LinearLayout>
```

其次，我们使用可以复用的页面，也就是ListView中的每一行内容，即左边是图标，右边是app的名称。其代码如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <ImageView
        android:id="@+id/app_icon_image_view"
        android:layout_width="60dp"
        android:layout_height="60dp"
        android:src="@mipmap/ic_launcher"
        />

    <TextView
        android:id="@+id/app_name_text_view"
        android:layout_width="match_parent"
        android:layout_height="60dp"
        android:gravity="left|center"
        android:paddingLeft="6dp"
        android:textSize="20sp"
        android:text="@string/app_name"
        />

</LinearLayout>
```

最后，编写业务逻辑，主要使用了Adapter和LayoutInflater，如下所示：

```java
package com.example.listviewbase;

import androidx.appcompat.app.AppCompatActivity;

import android.content.Context;
import android.os.Bundle;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.BaseAdapter;
import android.widget.ImageView;
import android.widget.ListView;
import android.widget.TextView;

import java.util.ArrayList;
import java.util.List;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ListView appListView = findViewById(R.id.app_listView);
        List<String> appName = new ArrayList<>();
        appName.add("QQ");
        appName.add("wechat");
        appName.add("imooc");
        appName.add("douyin");
        appName.add("wifi");
        appName.add("5G");
        appName.add("taobao");
        appName.add("Timao");
        appName.add("Tencent");
        appName.add("Huawei");

        appListView.setAdapter(new AppListAdapter(appName));
    }

    /**
     * 将数据和视图适配的Adapter
     */
    public class AppListAdapter extends BaseAdapter {

        List<String> mAppNames;

        public AppListAdapter(List<String> appNames) {
            mAppNames = appNames;
        }

        @Override
        public int getCount() {
            // 有多少条数据
            return mAppNames.size();
        }

        @Override
        public Object getItem(int position) {
            // 获取当前position位置的一条
            return mAppNames.get(position);
        }

        @Override
        public long getItemId(int position) {
            // 返回当前position位置的这一条ID
            return position;
        }

        /**
         * 这个方法，实际上就是在一个Activity中引入其他的xml文件，因为一个Activity对应一个xml，
         * 但是，有时候可能会有复用的xml文件，这个xml文件并不与单一的Activity对应。
         * 因此，需要先创建一个LayoutInflater对象，在调用inflate方法调用这个可复用的xml文件
         * LayoutInflater layoutInflater = (LayoutInflater) getSystemService(Context.LAYOUT_INFLATER_SERVICE);
         * convertView = layoutInflater.inflate(R.layout.item_app_list_view,null);
         *
         * @param position
         * @param convertView
         * @param parent
         * @return
         */
        @Override
        public View getView(int position, View convertView, ViewGroup parent) {
            // 处理 view -- data 填充数据的一个过程
            LayoutInflater layoutInflater = (LayoutInflater) getSystemService(Context.LAYOUT_INFLATER_SERVICE);
            // 解析Layout，使其能够被操控
            convertView = layoutInflater.inflate(R.layout.item_app_list_view, null);

            ImageView appIconImageView = convertView.findViewById(R.id.app_icon_image_view);
            TextView appNameTextView = (TextView) convertView.findViewById(R.id.app_name_text_view);

            appNameTextView.setText(mAppNames.get(position));
            return convertView;
        }
    }

}
```

> 需求扩展：可在右侧App名称下，加入App所属类别。

### 2.实战——获取系统中的应用列表

> 当前很多app都在后台收集了用户在手机上安装的app应用，从而达到精准推荐

定义一个方法，获取系统中的App

```java
    /**
     * 获取系统中的App
     * @return
     */
    private List<ResolveInfo> getAppInfos() {
        Intent intent = new Intent(Intent.ACTION_MAIN, null);
        intent.addCategory(Intent.CATEGORY_LAUNCHER);
        return getPackageManager().queryIntentActivities(intent,0);
    }
```

修改Adapter的方法

```java
/**
     * 将数据和视图适配的Adapter
     */
    public class AppListAdapter extends BaseAdapter {

        List<ResolveInfo> mAppNames;

        public AppListAdapter(List<ResolveInfo> appNames) {
            mAppNames = appNames;
        }

        @Override
        public int getCount() {
            // 有多少条数据
            return mAppNames.size();
        }

        @Override
        public Object getItem(int position) {
            // 获取当前position位置的一条
            return mAppNames.get(position);
        }

        @Override
        public long getItemId(int position) {
            // 返回当前position位置的这一条ID
            return position;
        }

        /**
         * 这个方法，实际上就是在一个Activity中引入其他的xml文件，因为一个Activity对应一个xml，
         * 但是，有时候可能会有复用的xml文件，这个xml文件并不与单一的Activity对应。
         * 因此，需要先创建一个LayoutInflater对象，在调用inflate方法调用这个可复用的xml文件
         * LayoutInflater layoutInflater = (LayoutInflater) getSystemService(Context.LAYOUT_INFLATER_SERVICE);
         * convertView = layoutInflater.inflate(R.layout.item_app_list_view,null);
         *
         * @param position
         * @param convertView
         * @param parent
         * @return
         */
        @Override
        public View getView(int position, View convertView, ViewGroup parent) {
            // 处理 view -- data 填充数据的一个过程
            LayoutInflater layoutInflater = (LayoutInflater) getSystemService(Context.LAYOUT_INFLATER_SERVICE);
            // 解析Layout，使其能够被操控
            convertView = layoutInflater.inflate(R.layout.item_app_list_view, null);

            ImageView appIconImageView = convertView.findViewById(R.id.app_icon_image_view);
            TextView appNameTextView = (TextView) convertView.findViewById(R.id.app_name_text_view);

            appNameTextView.setText(mAppNames.get(position).activityInfo.loadLabel(getPackageManager()));
                        appIconImageView.setImageDrawable(mAppNames.get(position).activityInfo.loadIcon(getPackageManager()));

            return convertView;
        }
    }
```

点击这一行，使得能够跳转到这个app的页面，对AppListAdapter中重写的getView方法最后面，添加如下代码，其逻辑如下：

```java

            convertView.setOnClickListener(view -> {
                String packageName = mAppNames.get(position).activityInfo.packageName;
                String classString = mAppNames.get(position).activityInfo.name;

                ComponentName componentName = new ComponentName(packageName,classString);
                startActivity(new Intent().setComponent(componentName));

            });
```

也可以在Oncreate()中，设置每行的Item点击事件。代码如下：

```java
        appListView.setOnItemClickListener((parent, view, position, id) -> {
            String packageName = mAppNames.get(position).activityInfo.packageName;
            String classString = mAppNames.get(position).activityInfo.name;

            ComponentName componentName = new ComponentName(packageName,classString);
            startActivity(new Intent().setComponent(componentName));

        });
```

也可以设置长按这一行，然后可以实现移除这个App的操作，代码如下：

```java
appListView.setOnItemLongClickListener((parent, view, position, id) -> {

	return false;
});
```

在应用列表上面，添加上一个图片。

页面xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="60dp">

    <ImageView
        android:layout_width="match_parent"
        android:layout_height="60dp"
        android:background="@drawable/ic_launcher_foreground" />
</LinearLayout>
```

业务逻辑，在Oncreate()方法中添加如下代码：

```java
        // 添加一个header View
        LayoutInflater layoutInflater = (LayoutInflater) getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        View headerView = layoutInflater.inflate(R.layout.headr_list_view , null);
        appListView.addHeaderView(headerView);
```

如果，有成百上千个应用，那么下滑上拉，会很卡。因此，这里需要优化。

```

```

