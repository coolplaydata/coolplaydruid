# Druid查询之：查询内部过程

Druid查询部分源码入口为`processing`模块，`io.druid.query`包

Druid接收Http请求部分的源码在`server`模块，`io.druid.server`包

## 查询请求

当我们用HTTP POST方式发出请求后，druid接收请求的服务代码在`io.druid.server`包中的`QueryResource`类

```
curl -X POST '<queryable_host>:<port>/druid/v2/?pretty' -H 'Content-Type:application/json' -d @<query_json_file>
```
第一步，要先找到接收请求的方法

```java
@POST
  @Produces({MediaType.APPLICATION_JSON, SmileMediaTypes.APPLICATION_JACKSON_SMILE})
  @Consumes({MediaType.APPLICATION_JSON, SmileMediaTypes.APPLICATION_JACKSON_SMILE, APPLICATION_SMILE})
  public Response doPost(
      final InputStream in,
      @QueryParam("pretty") final String pretty,
      @Context final HttpServletRequest req // used to get request content-type, remote address and auth-related headers
  ) throws IOException
```
## 反序列化为Query
在`doPost`方法中，通过调用如下的`readQuery`将请求json反序列化为`Query`对象

```java
private static Query<?> readQuery(
    final HttpServletRequest req,
    final InputStream in,
    final ResponseContext context
) throws IOException
{
  Query baseQuery = context.getObjectMapper().readValue(in, Query.class);
  String prevEtag = getPreviousEtag(req);

  if (prevEtag != null) {
    baseQuery = baseQuery.withOverriddenContext(
        ImmutableMap.of(HEADER_IF_NONE_MATCH, prevEtag)
    );
  }

  return baseQuery;
}
```
## 初始化QueryLifecycle
Druid中，会通过`QueryLifecycle`类 帮助Druid服务(`broker`, `historical`等)管理查询生命周期，它确保查询按照正确的步骤执行：
* `Initialization`：初始化这个对象以执行一个特定的查询。实际上并没有执行查询
* `Authorization`：授权查询。返回一个`Access`对象，表示是否批准了查询
* `Execution`：执行查询。只有在被授权的情况下才可以调用
* `Logging`：记录日志和查询指标，若查询过程中如果出现异常也会调用这部分功能去打印

```java
final QueryLifecycle queryLifecycle = queryLifecycleFactory.factorize();


queryLifecycle.initialize(readQuery(req, in, context));
      query = queryLifecycle.getQuery();
      final String queryId = query.getId();
```

## 查询授权

`doPost`方法中初始化`QueryLifecycle`后按步骤 需要执行授权操作
```java
final Access authResult = queryLifecycle.authorize(req);
  if (!authResult.isAllowed()) {
    throw new ForbiddenException(authResult.toString());
  }
```
看一下`QueryLifecycle`中`authorize`方法的实现

首先`transition`方法会判断当前执行是否按照顺序，然后执行`doAuthorize`方法，返回`Access`对象
```java

public Access authorize(HttpServletRequest req)
  {
    transition(State.INITIALIZED, State.AUTHORIZING);
    return doAuthorize(
        AuthorizationUtils.authenticationResultFromRequest(req),
        AuthorizationUtils.authorizeAllResourceActions(
            req,
            Iterables.transform(
                queryPlus.getQuery().getDataSource().getNames(),
                AuthorizationUtils.DATASOURCE_READ_RA_GENERATOR
            ),
            authorizerMapper
        )
    );
  }}
```
## 执行查询



`QueryLifecycle`中的`QueryPlus`对象包括`Query`和`QueryRunner`中需要的`QueryMetrics`

```java
public QueryResponse execute()
{
  transition(State.AUTHORIZED, State.EXECUTING);

  final Map<String, Object> responseContext = DirectDruidClient.makeResponseContextForQuery();

  final Sequence res = queryPlus.run(texasRanger, responseContext);

  return new QueryResponse(res == null ? Sequences.empty() : res, responseContext);
}
```

`queryPlus.run`方法是通过`QueryLifecycle`中的`QuerySegmentWalker`来构造`QueryRunner` 不同的查询类型会有不同的`QueryRunner`实例



```java
public Sequence<T> run(QuerySegmentWalker walker, Map<String, Object> context)
{
  return query.getRunner(walker).run(this, context);
}
```

以查询类型是聚合查询的GroupBy类型的`GroupByQueryRunner`为例 ,

运行给定的查询并返回一个时间顺序的查询结果`Sequence`，对于特定查询类型的内部逻辑分析，会在后面的章节中详细介绍。
```java
@Override
public Sequence<Row> run(QueryPlus<Row> queryPlus, Map<String, Object> responseContext)
{
  Query<Row> query = queryPlus.getQuery();
  if (!(query instanceof GroupByQuery)) {
    throw new ISE("Got a [%s] which isn't a %s", query.getClass(), GroupByQuery.class);
  }

  return strategySelector.strategize((GroupByQuery) query).process((GroupByQuery) query, adapter);
}
```
execute方法的最后需要把`Sequence`封装为`QueryResponse`返回

## 查询结束

查询结束后，会把查询结果作为Http请求的`Response`返回给用户

同时会通过`QueryLifecycle`中的`emitLogsAndMetrics`方法记录查询成功or失败的记录条数

```java
Response.ResponseBuilder builder = Response
    .ok(
        new StreamingOutput()
        {
          @Override
          public void write(OutputStream outputStream) throws IOException, WebApplicationException
          {
            Exception e = null;

            CountingOutputStream os = new CountingOutputStream(outputStream);
            try {
              // json serializer will always close the yielder
              jsonWriter.writeValue(os, yielder);

              os.flush(); // Some types of OutputStream suppress flush errors in the .close() method.
              os.close();
            }
            catch (Exception ex) {
              e = ex;
              log.error(ex, "Unable to send query response.");
              throw Throwables.propagate(ex);
            }
            finally {
              Thread.currentThread().setName(currThreadName);

              queryLifecycle.emitLogsAndMetrics(e, req.getRemoteAddr(), os.getCount());

              if (e == null) {
                successfulQueryCount.incrementAndGet();
              } else {
                failedQueryCount.incrementAndGet();
              }
            }
          }
        },
        context.getContentType()
    )
```
## 总结
通过上面的介绍我们走完了从用户通过`HTTP POST`方式发送查询请求到获取到结果 Druid内部过程，后续我们将针对具体的查询类型进行详细介绍。
