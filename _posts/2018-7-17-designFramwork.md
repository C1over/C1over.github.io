---
layout:     post   				    
title:    构架  				 
subtitle:       #副标题
date:       2018-7-10			   	# 时间
author:     BY 		Cc1over				# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android
    - 构架
---

## MVC
* MVC的定义：model，view，controller三者的有机结合，分别表示：模型，视图，控制器。这个模式认为：程序不论简单还是复杂，从结构上来看，都可以分为三个层次。
* MVC构架在Android应用程序组件的担当：
  ![](https://images2015.cnblogs.com/blog/580800/201702/580800-20170211210604260-225775606.png)
1）一般来说，View层就是各种UI组件（XML布局或者Java自定义控件对象 ），只负责展示数据以及接受Controller传递过来的一些数据。
2）通常来讲，数据库SQLite、网络请求的JSON、本地XML或者Java对象数据就是最底下的Model（数据层），它代表了一些实体类，可以用来描述一些业务逻辑如何组合，同时为业务逻辑制定规则。
3）而在Android中，Activity和Fragment一般都承担起了Contoller的作用，根据从View层输入的指令，在model层选取数据，然后执行相应的操作并产生最后的结果
* 各层的关系：
  ![](https://images2015.cnblogs.com/blog/580800/201702/580800-20170211210639776-94649080.png)
###### MVC构架的优劣势：
* 优势：耦合度较低，可以实现模块的复用，维护性相对较强。
* 劣势：1、V和M之间不匹配,用户界面和流程要考虑易用性,用户体验优化同时考虑业务流程的精确和无错。 <br>           2、C和M之间界线不清,什么样的逻辑是界面逻辑,什么样的逻辑是业务逻辑,很难定义清楚。<br>           3、V的变化不能完全由Model控制,即observer模式不足以支持复杂的用户交互.这其实要求VC之间要有依赖。
## MVP
##### 为什么使用MVP构架
上面也说到了MVC定义不够明确，而由于Android的特殊性，使得Activity同时担任V和C两个角色，这样就不满足单一责任原则了，而且当业务逻辑多起来了之后，Activity中的代码就会显得有点臃肿。
  ![](https://images2015.cnblogs.com/blog/1048430/201706/1048430-20170608145822528-972361537.png)
* 相比起MVC构架，在MVP构架中view不直接与model交互，而是通过Presenter交互来与model间接交互。

* Presenter与view的交互式通过接口来进行，更有利于添加单元测试。

* 一般情况下view与Presenter是一一对应的，但是复杂得view可能绑定多个Presenter来处理逻辑。

* 在MVC构架中controller是基于行为的，并且被多个view共享，而在基于上面提到的Presenter机制也就可以避免这种情况所带来的代码膨胀。

* **在MVP构架中，Activity的定义更加明确，不再像MVC那样去同时承担V和C，而是充当V位**

* **给上一点补充：MVP的设计思想是认为VC的绑定属于不可避免的情况，但可以加强P的能力,V变成只显示,P提供数据给V,把双向依赖简化为P直接依赖于V。**
#### MVP的小案例
**以一个登陆界面的小demo为例**
###### 登录回调接口：
~~~
public interface OnLoginFinishedListener{
    void onUsernameError();

    void onPasswordError();

    void onSuccess();
}
~~~
###### Model层：
~~~
public interface LoginModel {
    void login(String username, String password, OnLoginFinishedListener listener);
}
~~~
~~~
public class LoginModelImpl implements LoginModel {

    @Override
    public void login(final String username, final String password, final OnLoginFinishedListener listener) {
       
       new Handler().postDelayed(new Runnable() {
            @Override public void run() {
                boolean error = false;
                if (TextUtils.isEmpty(username)){
                    listener.onUsernameError();//model层里面回调listener
                    error = true;
                }
                if (TextUtils.isEmpty(password)){
                    listener.onPasswordError();
                    error = true;
                }
                if (!error){
                    listener.onSuccess();
                }
            }
        }, 2000);
    }
}
~~~
###### View层：
* 主要是由Activity或Fragment，一般在这里做一些加载UI视图，设置监听的操作，而有相应的操作时就调用Presenter的相应方法。
~~~
public interface LoginView {
    void showProgress();

    void hideProgress();

    void setUsernameError();

    void setPasswordError();

    void navigateToHome();
}
~~~
~~~
public class LoginActivity extends Activity implements LoginView, View.OnClickListener {

    private ProgressBar progressBar;
    private EditText username;
    private EditText password;
    private LoginPresenter presenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);

        progressBar = (ProgressBar) findViewById(R.id.progress);
        username = (EditText) findViewById(R.id.username);
        password = (EditText) findViewById(R.id.password);
        findViewById(R.id.button).setOnClickListener(this);

        presenter = new LoginPresenterImpl(this);
    }

    @Override
    protected void onDestroy() {
        presenter.onDestroy();
        super.onDestroy();
    }

    @Override
    public void showProgress() {
        progressBar.setVisibility(View.VISIBLE);
    }

    @Override
    public void hideProgress() {
        progressBar.setVisibility(View.GONE);
    }

    @Override
    public void setUsernameError() {
        username.setError(getString(R.string.username_error));
    }

    @Override
    public void setPasswordError() {
        password.setError(getString(R.string.password_error));
    }

    @Override
    public void navigateToHome() {
        Toast.makeText(this,"login success",Toast.LENGTH_SHORT).show();
    }

    @Override
    public void onClick(View v) {
        presenter.validateCredentials(username.getText().toString(), password.getText().toString());
    }

}
~~~
###### Presenter层：
* Presenter作为一个中介者的角色，获取model层的数据之后构建view层，也可以接收到view层UI上的反馈命令，交给model层做业务操作，也可以决定view层的一些操作。
~~~
  public interface LoginPresenter {
    void validateCredentials(String username, String password);

    void onDestroy();
}
~~~
~~~
public class LoginPresenterImpl implements LoginPresenter, OnLoginFinishedListener {
    private LoginView loginView;
    private LoginModel loginModel;

    public LoginPresenterImpl(LoginView loginView) {
        this.loginView = loginView;
        this.loginModel = new LoginModelImpl();
    }

    @Override
    public void validateCredentials(String username, String password) {
        if (loginView != null) {
            loginView.showProgress();
        }

        loginModel.login(username, password, this);
    }

    @Override
    public void onDestroy() {
        loginView = null;
    }

    @Override
    public void onUsernameError() {
        if (loginView != null) {
            loginView.setUsernameError();
            loginView.hideProgress();
        }
    }

    @Override
    public void onPasswordError() {
        if (loginView != null) {
            loginView.setPasswordError();
            loginView.hideProgress();
        }
    }

    @Override
    public void onSuccess() {
        if (loginView != null) {
            loginView.navigateToHome();
        }
    }
}
LoginPresenterImpl implements LoginPresenter, OnLoginFinishedListener {
    private LoginView loginView;
    private LoginModel loginModel;

    public LoginPresenterImpl(LoginView loginView) {
        this.loginView = loginView;
        this.loginModel = new LoginModelImpl();
    }

    @Override
    public void validateCredentials(String username, String password) {
        if (loginView != null) {
            loginView.showProgress();
        }
        loginModel.login(username, password, this);
    }

    @Override
    public void onDestroy() {
        loginView = null;
    }

    @Override
    public void onUsernameError() {
        if (loginView != null) {
            loginView.setUsernameError();
            loginView.hideProgress();
        }
    }

    @Override
    public void onPasswordError() {
        if (loginView != null) {
            loginView.setPasswordError();
            loginView.hideProgress();
        }
    }

    @Override
    public void onSuccess() {
        if (loginView != null) {
            loginView.navigateToHome();
        }
    }
}
~~~
###### 该项目的UML图：
![](https://upload-images.jianshu.io/upload_images/1833901-4e682cec6dde1a24.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)