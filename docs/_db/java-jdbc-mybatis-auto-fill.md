---
layout: post
title: MyBatis 插入数据时字段自动填充
categories: db
tags: Mybatis Interceptor
date: 2023-04-15
---
Mybatis 插入数据时自动插入雪花算法生成的 id、创建时间、更新时间，及更新数据时自动插入更新时间。这里是配合 Mybatis Generator 生成的映射文件和接口使用的
，目前只支持插入数据时使用 `insertSelective` ，更新数据时使用 `updateByPrimaryKeySelective` ，随着需求不断迭代
<!--more-->

## 1.Version1.0
使用 Mybatis 提供的 Interceptor，来实现。

定义拦截器，SnowFlake 的源码 [在这里](./db-table-id-snow-flake.html)

```java
@Intercepts(@Signature(type = Executor.class, method = "update",
        args = {MappedStatement.class, Object.class}))
public class MybatisAutoFillInterceptor implements Interceptor {
    
    @Autowired
    private SnowFlake snowFlake;

    @Autowired
    private MybatisConfig.TableAutoFillConfiguration autoFillConfiguration;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        dealAutoFilledColumn(invocation);
        return invocation.proceed();
    }

    private Field getFieldByName(Field[] fields, String fieldName) {
        final Optional<Field> first = Arrays.stream(fields)
                .filter(field -> field.getName().equals(fieldName))
                .findFirst();
        return first.orElse(null);
    }

    private void dealAutoFilledColumn(Invocation invocation) throws IllegalAccessException {
        if (!autoFillConfiguration.isOpenAutoFill()) {
            return;
        }
        if (invocation.getArgs().length < 2 && invocation.getArgs()[0] instanceof MappedStatement) {
            return;
        }
        MappedStatement mappedStatement = (MappedStatement) invocation.getArgs()[0];
        Object parameterObj = invocation.getArgs()[1];
        Field[] declaredFields = invocation.getArgs()[1].getClass().getDeclaredFields();

        switch (mappedStatement.getSqlCommandType()) {
            case INSERT:
                final Field createTime = getFieldByName(declaredFields, autoFillConfiguration.getCreateTimeFiledName());
                setCorrectDateTime(parameterObj, createTime);

                if (autoFillConfiguration.isIdUseSnowFlake()) {
                    Field id = getFieldByName(declaredFields, "id");
                    if (id != null && id.getType().isAssignableFrom(Long.class)) {
                        id.setAccessible(true);
                        id.set(parameterObj, snowFlake.nextId());
                    }
                }
            case UPDATE:
                final Field updateTime = getFieldByName(declaredFields, autoFillConfiguration.getUpdateTimeFiledName());
                setCorrectDateTime(parameterObj, updateTime);
        }
    }

    private void setCorrectDateTime(Object parameterObj, Field dateTime) throws IllegalAccessException {
        if (dateTime != null) {
            dateTime.setAccessible(true);
            if (dateTime.getType().isAssignableFrom(Date.class)) {
                dateTime.set(parameterObj, new Date());
            } else if (dateTime.getType().isAssignableFrom(LocalDateTime.class)) {
                dateTime.set(parameterObj, LocalDateTime.now());
            }
        }
    }
}

@Configuration
public class MybatisConfig {
    
    @Bean
    MybatisAutoFillInterceptor autoFilledInterceptor() {
        return new MybatisAutoFillInterceptor();
    }

    @Bean
    TableAutoFillConfiguration tableAutoFillConfiguration() {
        return new TableAutoFillConfiguration();
    }

    public static class TableAutoFillConfiguration {
        private boolean openAutoFill = true;
        private boolean idUseSnowFlake = true;
        private String createTimeFiledName = "createTime";
        private String updateTimeFiledName = "updateTime";

        public TableAutoFillConfiguration() {
        }

        public TableAutoFillConfiguration(boolean openAutoFill, boolean idUseSnowFlake, String createTimeFiledName, String updateTimeFiledName) {
            this.openAutoFill = openAutoFill;
            this.idUseSnowFlake = idUseSnowFlake;
            this.createTimeFiledName = createTimeFiledName;
            this.updateTimeFiledName = updateTimeFiledName;
        }

        public boolean isOpenAutoFill() {
            return openAutoFill;
        }

        public boolean isIdUseSnowFlake() {
            return idUseSnowFlake;
        }

        public String getCreateTimeFiledName() {
            return createTimeFiledName;
        }

        public String getUpdateTimeFiledName() {
            return updateTimeFiledName;
        }
    }
}
```

## 2.Version2.0
有一个批量插入的需求，原本准备继续使用 Interceptor 来实现的，写到一半，突然想到了使用 AOP 来实现，效果还是不错的，比拦截器好多了，灵活多了。后面会在 2.0 版本上继续迭代。这个版本可以支持 `insert*(T entity)` 、`insert*(List<T> entities)` 、`update*(T entity，..)`
```java
@Aspect
@Slf4j
@Component
@ConditionalOnProperty(name = "enable.field-auto-fill", havingValue = "true")
public class MybatisSemiAutoFill {

    @Autowired
    private SnowFlake snowFlake;

    @Autowired
    private MybatisConfig.TableAutoFillConfiguration autoFillConfiguration;

    //查询数据时默认不选择被删除的数据，为 isDeleted=0
    @Before("execution(* com.alamide.*.mapper.*Mapper.selectByExample(..))")
    public void beforeSelectMethod(JoinPoint joinPoint) throws IllegalAccessException, NoSuchMethodException, InvocationTargetException, ClassNotFoundException {
        final Object[] args = joinPoint.getArgs();

        log.info("查询操作：{}，{}", joinPoint.getSignature(), joinPoint.getArgs());

        if (args.length != 1 || args[0] == null) return;
        Object param0 = args[0];

        final Method getOredCriteria = param0.getClass().getDeclaredMethod("getOredCriteria");

        List<Object> criterias = (List<Object>) getOredCriteria.invoke(param0);
        if (criterias.size() == 0) {
            final Method createCriteria = param0.getClass().getDeclaredMethod("createCriteria");
            createCriteria.invoke(param0);
        }
        final Optional<Method> andIsDeletedEqualTo = Arrays.stream(criterias.get(0).getClass().getSuperclass().getDeclaredMethods())
                .filter(item -> item.getName().equals("andIsDeletedEqualTo"))
                .findFirst();
        if (andIsDeletedEqualTo.isEmpty()) return;
        for (Object c : criterias) {
            andIsDeletedEqualTo.get().invoke(c, 0);
        }
    }

    @Before("execution(* com.alamide.*.mapper.*Mapper.insert*(..))")
    public void beforeInsertMethod(JoinPoint joinPoint) throws IllegalAccessException {
        final Object[] args = joinPoint.getArgs();

        log.info("插入操作：{}，{}", joinPoint.getSignature(), joinPoint.getArgs());

        if (args.length != 1 || args[0] == null) return;
        Object param0 = args[0];

        Consumer<Object> insertFill = (row) -> {
            try {
                fillId(row);
                fillCreateTime(row);
                fillUpdateTime(row);
                fillCreateUserId(row);
                fillUpdateUserId(row);
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            }
        };
        //批量插入更新
        if (param0 instanceof Collection<?>) {
            log.info("批量插入.......");
            Collection<?> rows = (Collection<?>) param0;
            for (Object row : rows) {
                insertFill.accept(row);
            }
        } else {//单条插入更新
            log.info("单条插入.......");
            insertFill.accept(param0);
        }
    }

    @Before("execution(* com.alamide.*.mapper.*Mapper.update*(..))")
    public void beforeUpdateMethod(JoinPoint joinPoint) throws IllegalAccessException {
        final Object[] args = joinPoint.getArgs();

        log.info("更新操作：{}，{}", joinPoint.getSignature(), joinPoint.getArgs());
        //updateByExampleSelective(@Param("row") Employee row, @Param("example") EmployeeExample example);
        //两个参数
        if (args.length == 0 || args[0] == null) return;
        Object param0 = args[0];

        Consumer<Object> updateFill = (row) -> {
            try {
                fillUpdateTime(row);
                fillUpdateUserId(row);
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            }
        };
        //批量插入更新
        if (param0 instanceof Collection<?>) {
            log.info("批量更新.......");
            Collection<?> rows = (Collection<?>) param0;
            for (Object row : rows) {
                updateFill.accept(row);
            }
        } else {//单条插入更新
            log.info("单条更新.......");
            updateFill.accept(param0);
        }
    }

    private void fillUserId(Object row, String filedName) throws IllegalAccessException {
        Field userId = getFieldByName(row, filedName);
        if (userId != null && userId.getType().isAssignableFrom(Long.class)) {
            userId.setAccessible(true);
            userId.set(row, BaseContext.getCurrentId());
        }
    }

    private void fillCreateUserId(Object row) throws IllegalAccessException {
        fillUserId(row, "createUser");
    }

    private void fillUpdateUserId(Object row) throws IllegalAccessException {
        fillUserId(row, "updateUser");
    }

    private void fillId(Object row) throws IllegalAccessException {
        if (autoFillConfiguration.isIdUseSnowFlake()) {
            Field id = getFieldByName(row, autoFillConfiguration.getIdFiledName());
            if (id != null && id.getType().isAssignableFrom(Long.class)) {
                id.setAccessible(true);
                id.set(row, snowFlake.nextId());
            }
        }
    }

    private void fillCreateTime(Object row) throws IllegalAccessException {
        final Field createTime = getFieldByName(row, autoFillConfiguration.getCreateTimeFiledName());
        setCorrectDateTime(row, createTime);
    }

    private void fillUpdateTime(Object row) throws IllegalAccessException {
        final Field updateTime = getFieldByName(row, autoFillConfiguration.getUpdateTimeFiledName());
        setCorrectDateTime(row, updateTime);
    }

    private Field getFieldByName(Object row, String fieldName) {

        Field[] fields = row.getClass().getDeclaredFields();
        Optional<Field> first = Arrays.stream(fields)
                .filter(field -> field.getName().equals(fieldName))
                .findFirst();

        //向上寻找
        Class<?> superClass = null;
        if (first.isEmpty()) {
            superClass = row.getClass().getSuperclass();
        }
        while (first.isEmpty() && superClass != Object.class) {
            fields = superClass.getDeclaredFields();
            first = Arrays.stream(fields)
                    .filter(field -> field.getName().equals(fieldName))
                    .findFirst();
            superClass = superClass.getSuperclass();
        }

        return first.orElse(null);
    }

    private void setCorrectDateTime(Object parameterObj, Field dateTime) throws IllegalAccessException {
        if (dateTime != null) {
            dateTime.setAccessible(true);
            if (dateTime.getType().isAssignableFrom(Date.class)) {
                dateTime.set(parameterObj, new Date());
            } else if (dateTime.getType().isAssignableFrom(LocalDateTime.class)) {
                dateTime.set(parameterObj, LocalDateTime.now());
            }
        }
    }
}
```