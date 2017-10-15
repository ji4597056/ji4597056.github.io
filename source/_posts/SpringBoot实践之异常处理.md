---
title: SpringBoot实践之异常处理
categories: 技术
tags: [java,springboot]
---

## start

&emsp;&emsp;对于开发java web应用而言,异常处理一直不是java新手重视的环节,新手很少注意定义好固定的响应体格式,并且对于逻辑异常和系统异常不知道怎么做区别处理.当出现异常时应当依然符合固定的响应体格式响应给客户端.
&emsp;&emsp;使用`spring boot`框架已经一年多了,结合自己一些实践的经验,通过这篇文章分享一下自己是如何处理异常响应.

- - -

<!--more-->

- - -

## content

### 定义响应格式

&emsp;&emsp;首先需要定义一个规范统一的响应格式,这个格式除了能够给予客户端需要的数据,也需要附带一些其他有用的信息,例如响应是否成功,响应出错的异常码,响应出错的信息.具体格式如下:
```json
{
  "code" : 1011000, 
  "success" : true,
  "message" : "操作成功",
  "data" : "响应数据"
}
```
- **code** : 响应状态码.由于我们现在习惯使用微服务的开发方式,所有响应码使用**<服务id + 响应id>**的形式,每个服务拥有自己独一无二的id(例如101),响应id为该服务自定义的一些成功/异常的状态码(例如1000).
- **success** : 操作是否成功标识(true/false).即不管是系统异常还是逻辑错误都响应`false`,仅当操作处理成功时才响应`true`,作为客户端判断操作是否成功的标识.
- **message** : 操作结果信息.存放当客户端得到`success`为`false`的响应时,需要获取操作失败的逻辑原因信息.
- **data** : 响应结果.客户端请求的具体响应.

&emsp;&emsp;以上为定义的响应格式规范,服务端的任何 响应都需要符合该格式.

### 响应处理

&emsp;&emsp;为了实现上述响应格式,我们需要定义响应码的枚举,已经响应实体类.
```java
public enum RespCodeEnum {

    // 成功
    SUCCESS(10000, "success"),
    // 系统错误
    SYSTEM_ERROR(20001, "system error"),
    // 登录超时
    LOGIN_TIMEOUT(20002, "login timeout"),
    // 无权限
    UNAUTHORIZED(20003, "unauthorized"),
    // 参数校验错误
    PARAM_VALIDATE_ERROR(20004, "parameter validation error"),
    // 自定义错误
    CUSTOM_ERROR(30001, "custom error");

    // response code map
    private static final Map<String, Map<String, Object>> respCodeMap = new HashMap<>();

    static {
        for (RespCodeEnum codeEnum : values()) {
            Map<String, Object> map = new HashMap<>(3);
            map.put("code", codeEnum.code);
            map.put("value", codeEnum.value);
            respCodeMap.put(codeEnum.toString(), map);
        }
    }

    // 响应码
    private int code;

    // value
    private String value;

    RespCodeEnum(int code, String value) {
        this.code = code;
        this.value = value;
    }

    public int code() {
        return code;
    }

    public String value() {
        return value;
    }

    public static Map<String, Map<String, Object>> getEnumValues() {
        return respCodeMap;
    }
}
```
```java
public class WebResponse {

    // response code
    private String code;
    // response message
    private String message;
    // success(操作逻辑是否成功)
    private boolean success;
    // response data
    private Object data;

    private WebResponse() {}

    private WebResponse(String code, String message, boolean success, Object data) {
        this.code = code;
        this.message = message;
        this.success = success;
        this.data = data;
    }

    private static WebResponse create(String code, String message, boolean success, Object data) {
        return new WebResponse(code, message, success, data);
    }

    /**
     * 创建响应
     *
     * @param respCodeEnum RespCodeEnum
     * @param success is success
     * @param data response data
     * @return WebResponse
     */
    public static WebResponse createResp(RespCodeEnum respCodeEnum, boolean success, Object data) {
        return createResp(respCodeEnum.code(), respCodeEnum.value(), success, data);
    }

    /**
     * 创建响应
     *
     * @param code code
     * @param message message
     * @param success is success
     * @param data response data
     * @return WebResponse
     */
    public static WebResponse createResp(int code, String message, boolean success, Object data) {
        return create(getServiceId() + code, message, success, data);
    }

    /**
     * 创建成功响应
     *
     * @param data response data
     * @return WebResponse
     */
    public static WebResponse createSuccessResp(Object data) {
        return createResp(RespCodeEnum.SUCCESS, true, data);
    }

    /**
     * 获取服务id
     *
     * @return service id
     */
    private static String getServiceId() {
        return PropertyUtils.getValue(SysConstant.GATEWAY_ID_KEY);
    }

    public String getCode() {
        return code;
    }

    public void setCode(String code) {
        this.code = code;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public boolean isSuccess() {
        return success;
    }

    public void setSuccess(boolean success) {
        this.success = success;
    }

    public Object getData() {
        return data;
    }

    public void setData(Object data) {
        this.data = data;
    }

    @Override
    public String toString() {
        final StringBuilder sb = new StringBuilder("WebResponse{");
        sb.append("code='").append(code).append('\'');
        sb.append(", message='").append(message).append('\'');
        sb.append(", success=").append(success);
        sb.append(", data=").append(data);
        sb.append('}');
        return sb.toString();
    }
}
```
&emsp;&emsp;在`WebResponse`响应实体类中,对于操作成功的响应通过`WebResponse.createSuccessResp(Object data)`构造,对于操作失败的响应提供了两个`WebResponse.createResp()`方法构造.由于所有响应码需要和服务id拼接,所以定义了获取服务id的私有方法.

### 异常处理

&emsp;&emsp;接下来就是重点了,遇到逻辑异常,可以通过抛异常并由最外层拦截异常构造`WebResponse`的方式响应给前端,在使用`sprinfg boot`框架时需要注意分别处理进入`Controller`的异常拦截和未进入的异常拦截(比如访问不存在的路由,就会跳转到`/error`路由,然后输出404响应.再比如当使用`spring cloud zuul`时,如果出现异常也会请求转发到`/error`路由,然后解析异常输出响应).
&emsp;&emsp;当然,还有一个需要处理的问题,当正常响应时可以统一给客户端响应的http状态码为200(这里有点不符合`REST`规范,比如删除成功的操作,响应码应该为204,可以在`Controller`下的路由里通过注解自定义响应码),但是具体不同的异常可能需要不同的响应码(400~500),所以在定义逻辑异常类时需要声明`http code`字段.
```java
public class BaseException extends RuntimeException {

    // default http code
    private static final HttpStatus DEFAULT_HTTP_STATUS = HttpStatus.INTERNAL_SERVER_ERROR;
    // default response code
    private static final int DEFAULT_RESPONSE_CODE = RespCodeEnum.CUSTOM_ERROR.code();
    // http status
    protected final HttpStatus httpStatus;
    // response code
    protected final int respCode;
    // message
    protected final String message;

    public BaseException(String message) {
        super(message);
        this.message = message;
        this.httpStatus = DEFAULT_HTTP_STATUS;
        this.respCode = DEFAULT_RESPONSE_CODE;
    }

    public BaseException(String message, HttpStatus httpStatus) {
        super(message);
        this.message = message;
        this.httpStatus = httpStatus;
        this.respCode = DEFAULT_RESPONSE_CODE;
    }

    public BaseException(String message, HttpStatus httpStatus, RespCodeEnum respCodeEnum) {
        super(message);
        this.message = message;
        this.httpStatus = httpStatus;
        this.respCode = respCodeEnum.code();
    }

    public BaseException(String message, Exception e) {
        super(message, e);
        this.message = message;
        this.httpStatus = DEFAULT_HTTP_STATUS;
        this.respCode = DEFAULT_RESPONSE_CODE;
    }

    public BaseException(String message, HttpStatus httpStatus, Exception e) {
        super(message, e);
        this.message = message;
        this.httpStatus = httpStatus;
        this.respCode = DEFAULT_RESPONSE_CODE;
    }

    public BaseException(String message, HttpStatus httpStatus, int respCode, Exception e) {
        super(message, e);
        this.message = message;
        this.httpStatus = httpStatus;
        this.respCode = respCode;
    }

    public BaseException(String message, HttpStatus httpStatus, RespCodeEnum respCodeEnum,
        Exception e) {
        super(message, e);
        this.message = message;
        this.httpStatus = httpStatus;
        this.respCode = respCodeEnum.code();
    }

    public HttpStatus getHttpStatus() {
        return httpStatus;
    }

    public int getRespCode() {
        return respCode;
    }

    @Override
    public String getMessage() {
        return message;
    }

    @Override
    public String toString() {
        final StringBuilder sb = new StringBuilder("BaseException{");
        sb.append("httpStatus=").append(httpStatus);
        sb.append(", respCode=").append(respCode);
        sb.append(", message='").append(message).append('\'');
        sb.append('}');
        return sb.toString();
    }
}
```
&emsp;&emsp;如果每次主动`new BaseException()`创建异常,必然很麻烦(这里由于类构造方法很多,所以没有写静态工厂方法),所以使用枚举定义常见的逻辑异常.
```java
public enum ExceptionEnum {

    SYSTEM_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, RespCodeEnum.SYSTEM_ERROR, "系统错误,请稍后再试!"),
    REQUEST_PARAM_NOT_VALIDATE(HttpStatus.BAD_REQUEST, RespCodeEnum.PARAM_VALIDATE_ERROR,
        "参数校验不合法!");

    //response code map
    private static final Map<String, Map<String, Object>> exceptionMap = new HashMap<>();

    static {
        for (ExceptionEnum exceptionEnum : values()) {
            Map<String, Object> map = new HashMap<>(5);
            map.put("httpStatus", exceptionEnum.httpStatus.value());
            map.put("respCode", exceptionEnum.respCode());
            map.put("message", exceptionEnum.message);
            exceptionMap.put(exceptionEnum.toString(), map);
        }
    }

    // http code
    private HttpStatus httpStatus;
    // RespCodeEnum
    private RespCodeEnum respCodeEnum;
    // response message
    private String message;
    // BaseException
    private BaseException exception;

    ExceptionEnum(HttpStatus httpStatus, RespCodeEnum respCodeEnum, String message) {
        this.httpStatus = httpStatus;
        this.respCodeEnum = respCodeEnum;
        this.message = message;
        this.exception = new BaseException(message, httpStatus, respCodeEnum);
    }

    public BaseException exception() {
        return this.exception;
    }

    public int respCode() {
        return this.respCodeEnum.code();
    }

    public HttpStatus httpStatus() {
        return this.httpStatus;
    }

    public static Map<String, Map<String, Object>> getEnumValues() {
        return exceptionMap;
    }
}
```
&emsp;&emsp;通过`GlobalExceptionHandler`类对进入`Controller`的异常进行拦截,非`BaseException`为系统异常,需要打印异常堆栈信息,方便进行差错.`BaseException`为逻辑异常,只打印异常信息(在具体使用时,仍然需要在声明`BaseException`时打印具体导致产生逻辑异常原因的堆栈信息,方便差错).
```java
@ControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    private static final String LOG_MESSAGE = "--- system error! --- Host: {}, Url: {}, ERROR: {}";

    /**
     * 未定义系统异常
     */
    @ExceptionHandler(Exception.class)
    @ResponseBody
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public WebResponse sysErrorHandler(HttpServletRequest req, Exception e) {
        printErrorLog(req, e);
        return WebResponse
            .createResp(ExceptionEnum.SYSTEM_ERROR.respCode(), e.getMessage(), false, null);
    }

    /**
     * 处理BaseException
     */
    @ExceptionHandler(BaseException.class)
    @ResponseBody
    public WebResponse serviceErrorHandler(HttpServletRequest req, HttpServletResponse res,
        BaseException e) {
        printWarnLog(req, e);
        res.setStatus(e.getHttpStatus().value());
        return WebResponse.createResp(e.getRespCode(), e.getMessage(), false, null);
    }

    /**
     * 打印错误日志(会打印出堆栈信息)
     */
    private void printErrorLog(HttpServletRequest req, Exception e) {
        logger.error(LOG_MESSAGE, req.getRemoteHost(), req.getRequestURL(), e);
    }

    /**
     * 打印警告日志(仅打印错误信息)
     */
    private void printWarnLog(HttpServletRequest req, Exception e) {
        logger.warn(LOG_MESSAGE, req.getRemoteHost(), req.getRequestURL(), e.getMessage());
    }
}
```
&emsp;&emsp;`spring boot`框架对于未进入`Controller`的请求都会进入`/error`路由(可通过配置修改该路径),再解析异常构造响应,所以需要重写`/error`路由的逻辑处理.
```java
@RestController
public class CustomErrorController implements ErrorController {

    private static final String ERROR_PATH = "/error";

    @Autowired
    private ErrorAttributes errorAttributes;

    @RequestMapping(ERROR_PATH)
    public WebResponse error(HttpServletRequest request, HttpServletResponse response) {
        Throwable throwable = getException(request);
        response.setStatus(getErrorStatus(response));
        return WebResponse
            .createResp(getErrorRespCode(throwable), getErrorMessage(throwable), false, null);
    }

    @Override
    public String getErrorPath() {
        return ERROR_PATH;
    }

    /**
     * 获取异常
     *
     * @param request HttpServletRequest
     * @return Throwable
     */
    private Throwable getException(HttpServletRequest request) {
        ServletRequestAttributes requestAttributes = new ServletRequestAttributes(request);
        Throwable throwable = errorAttributes.getError(requestAttributes);
        if (throwable == null) {
            throwable = new Throwable(
                (String) errorAttributes.getErrorAttributes(requestAttributes, false)
                    .get("message"));
        }
        return throwable;
    }

    /**
     * 获取错误状态码
     *
     * @param response HttpServletResponse
     * @return response http status
     */
    private int getErrorStatus(HttpServletResponse response) {
        return response.getStatus();
    }

    /**
     * 获取错误状态码
     *
     * @param throwable Throwable
     * @return response http status
     */
    private int getErrorRespCode(Throwable throwable) {
        return ExceptionEnum.SYSTEM_ERROR.respCode();
    }

    /**
     * 获取错误信息
     *
     * @param throwable Throwable
     * @return error message
     */
    private String getErrorMessage(Throwable throwable) {
        return throwable.getMessage();
    }
}
```

### end

&emsp;&emsp;异常处理往往被新手java程序员忽略,也有一些程序员觉得通过抛异常的方式来处理逻辑错误效率很低,理论上来说构造异常对象确实效率比较低,但是如果具体写逻辑错误响应,不管对于开发还是维护而言都很麻烦,牺牲一点效率换来更好的代码可读性和开发效率应该是可以接受的.追求极致的效率往往需要写极多的代码,对于懒人而言,写的少、看得懂似乎是更好的选择.