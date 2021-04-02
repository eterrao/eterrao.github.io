---
title: Android MVP
date: 2021-02-19 15:43:08
tags: Android, MVP
---

MVP 主要类：
- IPresenter 
- IVIew 
- BasePresenter 
- BaseActivity 
- BaseFragment 
- Contract 

#### IPresenter

```java
public interface IPresenter {
  void attachView(IView view);
  void detachView();
}
```

#### IView

```java
public interface IView {
  Context getContext();
}
```

#### BasePresenter

```java
public abstract class BasePresenter<T extends IView> implements IPresenter {
  protected WeakReference<T> mViewRef;
  
  IView getView() {
    if (mViewRef != null) {
      return mViewRef.get();
    }
    return null;
  }
  
  @Override
  void attachView(IView view) {
    mViewRef = new WeakReference<T>(view);
  }
  
  @Override
	void  detachView(){
    if (mViewRef != null) {
			mViewRef.clear();
      mViewRef = null;
    }
  }
  
  public boolean isAvailable() {
    return mViewRef != null;
  }
 
}
```

#### BaseActivity

```java
public abstract class BaseActivity<T extends IPresenter> extends FragmentActivity implements IView {
  protected T mPresenter;
  
  protected void setPresenter(T presenter) {
    mPresenter = presenter;
    mPresenter.attachView(this);
  }
  
  @Override
  publid void onDestroy() {
    if (mPresenter != null) {
      mPresenter.detachView();
    }
    super.onDestroy();
  }
}
```

####Contract

```java
public interface TransactionContract {
  public interface View extends IView {
    // view transaction methods
  }
  public interface Presenter extends BasePresenter<View> {
  	// presenter transaction methods
  }
}
```

#### 

