# Custom-Okhttp
###对OKHttp进行过封装。具体的注意点有下面几点：  
1、首先，OKHttp官方要求我们最好用单例模式去使用OKHttpClient类的，因此我们自定义一个OKHttpHelper类，并且使用单例模式。  
2、对get以及post方法进行封装，主要的思想是把共同的代码抽取出来，例如代码中被抽取出来的request方法。  
3、对外公开一些静态方法，包括get和post方法等。  
4、Callback基类，对OKHttp的回调进行封装。这个类用里面有一个type，是方便回调中使用Gson对JSON进行解析的封装。使用Callback的时候只需要在泛型中传入类似Data 、List<Data>即可以方便地使用JSON。  
5、由于原来的回调不在主线程，因此我们需要使用Handler来将回调放入主线程。  
其余的可以参照代码，有详细注释。
###核心代码  
```java
package com.wyuxks.okhttp.okhttp;

import android.os.Handler;
import android.os.Looper;

import com.google.gson.Gson;
import com.google.gson.JsonParseException;
import com.wyuxks.okhttp.okhttp.callback.BaseCallback;

import org.json.JSONException;
import org.json.JSONObject;

import java.io.IOException;
import java.util.Iterator;
import java.util.Map;
import java.util.concurrent.TimeUnit;

import okhttp3.Call;
import okhttp3.Callback;
import okhttp3.FormBody;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.Response;

/**
 * Created by Xie on 2017/1/6.
 */
public class OkHttpHelper {
    /**
     * 采用单例模式使用OkHttpClient
     */
    private static final Class STRING_TYPE = String.class;
    private static final String TAG = OkHttpHelper.class.getSimpleName();
    private static OkHttpHelper mOkHttpHelperInstance;
    private static OkHttpClient mClientInstance;
    private Handler mHandler;
    private Gson mGson;

    /**
     * 单例模式，私有构造函数，构造函数里面进行一些初始化
     */
    private OkHttpHelper() {
        mClientInstance = new OkHttpClient.Builder()
                .connectTimeout(10, TimeUnit.SECONDS) //设置连接超时
                .writeTimeout(10, TimeUnit.SECONDS)   //设置写超时
                .readTimeout(30, TimeUnit.SECONDS)    //设置读超时
                .retryOnConnectionFailure(true)       //是否自动重连
                .build();
        mGson = new Gson();
        mHandler = new Handler(Looper.getMainLooper());
    }

    /**
     * 获取单例
     *
     * @return
     */
    public static OkHttpHelper getInstance() {

        if (mOkHttpHelperInstance == null) {
            synchronized (OkHttpHelper.class) {
                if (mOkHttpHelperInstance == null) {
                    mOkHttpHelperInstance = new OkHttpHelper();
                }
            }
        }
        return mOkHttpHelperInstance;
    }

    /**
     * 封装一个request方法，不管post或者get方法中都会用到
     */
    public void request(final Request request, final Class type, final BaseCallback callback) {

        //在请求之前所做的事，比如弹出对话框等
        mClientInstance.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                //返回失败
                callbackFailure(request, callback, e);
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                if (response.isSuccessful()) {
                    //返回成功回调
                    String resString = response.body().string();
                    if (resString.contains("false")) {
                        //返回结果 result为false(根据实际需求请求返回的string判断)
                        try {
                            JSONObject jsonObject = new JSONObject(resString);
                            String err_msg = jsonObject.getString("err_msg");
                            callbackFalse(response, err_msg, callback);
                        } catch (JSONException e) {
                            e.printStackTrace();
                            callbackError(response, callback, e);
                        }
                    } else {
                        if (type == STRING_TYPE) {
                            //如果我们需要返回String类型
                            callbackSuccess(response, resString, callback);
                        } else {
                            //如果返回的是其他类型，则利用Gson去解析
                            try {
                                Object o = mGson.fromJson(resString, type);
                                callbackSuccess(response, o, callback);
                            } catch (JsonParseException e) {
                                e.printStackTrace();
                                callbackError(response, callback, e);
                            }
                        }
                    }
                } else {
                    //返回错误
                    callbackError(response, callback, null);
                }
            }
        });
    }

    /**
     * 在主线程中执行的回调
     *
     * @param response
     * @param o
     * @param callback
     */
    private void callbackSuccess(final Response response, final Object o, final BaseCallback callback) {
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                callback.onSuccess(response, o);
            }
        });
    }

    /**
     * 在主线程中执行的回调
     *
     * @param response
     * @param callback
     * @param e
     */
    private void callbackError(final Response response, final BaseCallback callback, final Exception e) {
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                callback.onError(response, response.code(), e);
            }
        });
    }

    /**
     * 在主线程中执行的回调
     *
     * @param request
     * @param callback
     * @param e
     */
    private void callbackFailure(final Request request, final BaseCallback callback, final Exception e) {
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                callback.onFailure(request, e);
            }
        });
    }

    /**
     * 在主线程中执行的回调
     *
     * @param response
     * @param callback
     * @param err_msg
     */
    private void callbackFalse(final Response response, final String err_msg, final BaseCallback callback) {
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                callback.onFalse(response, err_msg);
            }
        });
    }

    /**
     * 对外公开的get方法
     *
     * @param url
     * @param callback
     */
    public void get(String url, Class type, BaseCallback callback) {
        Request request = buildRequest(url, null, HttpMethodType.GET);
        request(request, type, callback);
    }

    /**
     * 对外公开的post方法
     *
     * @param url
     * @param params
     * @param callback
     */
    public void post(String url, Class type, Map<String, Object> params, BaseCallback callback) {
        Request request = buildRequest(url, params, HttpMethodType.POST);
        request(request, type, callback);
    }

    /**
     * 对外公开的手动释放内存线程池方法，在内存不足时可调用
     */
    public void closeThreadPools() {
        mClientInstance.dispatcher().executorService().shutdown();   //清除并关闭线程池
        mClientInstance.connectionPool().evictAll();                 //清除并关闭连接池
        try {
            if (mClientInstance.cache() != null)
                mClientInstance.cache().close();                             //清除cache
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 对外公开的取消当前请求
     */
    public void cancleRequest(Call call) {
        if (call.isCanceled()) {
            call.cancel();  //取消请求
        }
    }

    /**
     * 构建请求对象
     *
     * @param url
     * @param params
     * @param type
     * @return
     */
    private Request buildRequest(String url, Map<String, Object> params, HttpMethodType type) {
        Request.Builder builder = new Request.Builder();
        builder.url(url)
                .addHeader("Accept", "application/json; q=0.5")                                    //添加header
                .addHeader("Content-Type", "application/x-www-form-urlencoded");
        if (type == HttpMethodType.GET) {
            builder.get();
        } else if (type == HttpMethodType.POST) {
            RequestBody body = buildRequestBody(params);
            builder.post(body);
            try {
                long length = body.contentLength();
                builder.addHeader("Content-Length", length + "");
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return builder.build();
    }

    /**
     * 通过Map的键值对构建post请求对象的body
     *
     * @param params
     * @return
     */
    private RequestBody buildRequestBody(final Map<String, Object> params) {
        FormBody.Builder builder = new FormBody.Builder();
        Iterator<Map.Entry<String, Object>> iterator = params.entrySet().iterator();
        while (iterator.hasNext()) {
            Map.Entry<String, Object> next = iterator.next();
            String key = next.getKey();
            Object value = next.getValue();
            builder.add(key, value + "");
        }
        return builder.build();
    }

    /**
     * 这个枚举用于指明是哪一种提交方式
     */
    enum HttpMethodType {
        GET,
        POST
    }

}
```
###回调的封装  
```java
package com.wyuxks.okhttp.okhttp.callback;

import okhttp3.Request;
import okhttp3.Response;

/**
 * Created by Xie on 2017/1/6.
 * 基本的回调
 */
public interface BaseCallback<T> {

    /**
     * 请求失败调用（网络问题）
     *
     * @param request
     * @param e
     */
     void onFailure(Request request, Exception e);

    /**
     * 请求成功没有错误且返回result为true的时候调用
     *
     * @param response
     * @param t
     */
    void onSuccess(Response response, T t);

    /**
     * 请求成功没有错误但返回result为false的时候调用 例如：参数非法、时间戳非法、签名不一致等提示信息
     * @param err_msg
     */
    void onFalse(Response response, String err_msg);

    /**
     * 请求成功但是有错误的时候调用，例如Gson解析错误等
     *
     * @param response
     * @param errorCode
     * @param e
     */
     void onError(Response response, int errorCode, Exception e);
}
```  
###封装后的使用  
首先得到OkHttpHelper的单例，然后调用get方法就可以了。由于继承了Gson，因此需要在BaseCallback的泛型中传入JSON对应的数据类型。 
```java
mHttpHelper=OkHttpHelper.getinstance();
mHttpHelper.get(请求url, new BaseCallback<List<Banner>>() {

    @Override
    public void onRequestBefore() {

    }

    @Override
    public void onFailure(Request request, Exception e) {

    }

    @Override
    public void onSuccess(Response response, List<Banner> banners) {
        initBanners(banners);
    }

    @Override
    public void onError(Response response, int errorCode, Exception e) {

    }
});
```
