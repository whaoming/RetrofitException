1.前言
==================
强烈建议在了解Retrofit，RxJava，gson的情况下看此篇博文

2.异常的类型
============
在使用Retrofit+RxJava访问RestFul服务器的过程中，总免不了要处理错误，错误有很多种，例如服务器端的错误：密码错误，cookie过期等等。客户端中产生的错误：网络异常，解析数据出错等。但是这里就会出现一个问题，假如服务器返回的是统一数据格式：

```
/**
 * 公共数据格式
 * @author Mr.W
 * @param <T>
 */
public class Result<T> {
	public int state;
	public String error;
	public T infos;

}
```
那么此时，如果是网络错误等客户端错误，在使用Retrofit+RxJava时会直接调用subscribe的onError事件，但是如果是服务器端的错误，比如说密码错误等，此时还是会执行subscribe的onnext事件，而且必须在subscribe里面处理异常信息，这对于一个对架构有强迫症的人是难以忍受的，就好像下面的代码：

```
UserEngine.initUserInfo("username", "password")
                .subscribe(new Observer<Result<Login>>() {
                    @Override
                    public void onCompleted() {
                        
                    }

                    @Override
                    public void onError(Throwable e) {
                            
                    }

                    @Override
                    public void onNext(Result<Login> data) {
                        if(data.state == 200){
                            //.....
                        }else if(data.state == 404){
                            
                        }
                    }
                });
```
那么现在我希望在发生任何错误的情况下，都会调用onError事件，并且由model来处理错误信息。在此之前，我们必须去分析Retrofit+RxJava的工作流程

3.Retrofit+RxJava的工作流程
==============

```
Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://192.168.56.1:8080/ElectricBicycleServer/")
                .addConverterFactory(GsonConverterFactory.create())
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .build();
```
从上面的代码我们可以看出在retrofit中，我们必须用到俩个工厂，一个就是RxJava的，一个就是Gson的。我们可以这样理解：
RxJavaCallAdapterFactory可以理解成告诉retrofit我们想要的工作方式，GsonConverterFactory是告诉retrofit我们怎么去解析数据，有可能是xml，有可能是json。
RxJavaCallAdapterFactory其实很容易理解，就是在这个工厂里面自定义一个OnSubscribe，然后把retrofit的工作放到Observable的工作流里，可以看看核心代码：

```
static final class CallOnSubscribe<T> implements Observable.OnSubscribe<Response<T>> {
    private final Call<T> originalCall;

    private CallOnSubscribe(Call<T> originalCall) {
      this.originalCall = originalCall;
    }

    @Override public void call(final Subscriber<? super Response<T>> subscriber) {
      // Since Call is a one-shot type, clone it for each new subscriber.
      final Call<T> call = originalCall.clone();

      // Attempt to cancel the call if it is still in-flight on unsubscription.
      subscriber.add(Subscriptions.create(new Action0() {
        @Override public void call() {
          call.cancel();
        }
      }));

      if (subscriber.isUnsubscribed()) {
        return;
      }

      try {
        Response<T> response = call.execute();
        if (!subscriber.isUnsubscribed()) {
          subscriber.onNext(response);
        }
      } catch (Throwable t) {
        Exceptions.throwIfFatal(t);
        if (!subscriber.isUnsubscribed()) {
          subscriber.onError(t);
        }
        return;
      }

      if (!subscriber.isUnsubscribed()) {
        subscriber.onCompleted();
      }
    }
}
```
其中，call是retrofit对okhttp的一个代理，call.execute()就是执行。
然而，我们的重点在GsonConverterFactory中，先po源码看一下：

```
@Override public T convert(ResponseBody value) throws IOException {
    Reader reader = value.charStream();
    try {
      return gson.fromJson(reader, type);
    } finally {
      Utils.closeQuietly(reader);
    }
  }
```
这就是工厂里最核心的一个方法(剔除了很多其他代码)，在Retrofit中，把json数据解析成object就发生在这个方法里。
好了，那说了这些工作流程有什么用呢？
4.自定义解析工厂
===================
可以看出，**整个流程都是在RxJava的工作流中**，所以RxJava对于Retrofit的适配器(也就是工厂)并不用做什么改变，因为**只要在流程中抛出异常，便会执行onError方法**，这是永恒不变的。我们最终的目的，就是解析服务器返回的json，判断请求是否成功，如果不成功，那么抛出异常，因为在RxJava的工作流中，只要抛出异常，便会执行subscribe的onError事件。所以，我们必须要**自定义一个gson的工厂**。这个工厂能在内部先解析服务器返回的数据，**根据数据的成功与否来判断是否需要抛出异常或者进行object转换**。
这是GsonConverterFactory的包结构：
![这里写图片描述](http://img.blog.csdn.net/20160531103422905)
需要把着三个类都复制到自己的代码下面，然后我们需要更改的是GsonResponseBodyConverter类，这是我自己更改的：

```
/**
 * Created by 12262 on 2016/5/30.
 */
class MyGsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
    private final Gson gson;
    private final Type type;

    MyGsonResponseBodyConverter(Gson gson, Type type) {
        this.gson = gson;
        this.type = type;
    }

    @Override
    public T convert(ResponseBody value) throws IOException {
        try {
            String response = value.string();
            Result<T> resultResponse = JsonUtil.fromJson(response,type);
            //对返回码进行判断，如果是200，便返回object
            if (resultResponse.state == 200) {
                return resultResponse.infos;
            } else {
	            //抛出自定义服务器异常
                throw new ServerException(resultResponse.state, resultResponse.error);
            }
        }finally {
//            Utils.closeQuietly(reader);
        }
    }
}
```
其中ServerException对应服务器的错误码

```
/**
 * Created by 12262 on 2016/5/30.
 */
public class ServerException extends Exception {

    /**
     * 登录失败
     */
    public final int SERVER_ERROR_LOGIN_FAIL = 1001;

    /**
     * 服务器发生未知错误
     */
    public final int SERVER_ERROR_UNKNOW = 1002;

    /**
     * 账号已经被注册
     */
    public final int SERVER_ERROR_ACOUNT_REGISTER = 1003;

    private int errCode = 0;

    /**
     *
     * @param errCode  错误码
     * @param msg    错误信息
     */
    public ServerException(int errCode, String msg) {
        super(msg);
        this.errCode = errCode;
    }

    public int getErrCode() {
        return errCode;
    }
}
```
到这里，我们就已经完成了我们最开始的目的，就是在gson工厂中处理服务器的错误，所以当服务器发出错误信息的时候，在我们的subscribe的onError事件便能捕获。
再回头看看我们的题目，说的是在MVP中处理异常，那么，当发生连接不上服务器，或者解析json数据失败的时候，我们并不能让presenter去接触这些错误，正确的做法是在model中处理这些错误并统一成一个ApiException，下一节讲讲把！
