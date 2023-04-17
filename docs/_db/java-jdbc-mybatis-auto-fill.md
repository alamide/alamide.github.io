---
layout: post
title: MyBatis 插入数据时字段自动填充
categories: db
tags: Mybatis Interceptor
date: 2023-04-15
isHidden: true
---
Mybatis 插入数据时自动插入雪花算法生成的 id、创建时间、更新时间，及更新数据时自动插入更新时间。这里是配合 Mybatis Generator 生成的映射文件和接口使用的
，目前只支持插入数据时使用 `insertSelective` ，更新数据时使用 `updateByPrimaryKeySelective` ，随着需求不断迭代
<!--more-->

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