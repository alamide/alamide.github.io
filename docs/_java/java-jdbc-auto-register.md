---
layout: post
title: JDBC 数据库驱动自动装配原理
categories: java
tags: Java JDBC
date: 2023-03-17
---

当加载 `DriverManager.class` 时，会自动装配 `com.mysql.cj.jdbc.Driver`，所以不需要自己注册 `mysql` 驱动。
<!--more-->
## 1.代码跟踪
```java
package java.sql;

public class DriverManager {
  
  static {
      loadInitialDrivers();
  }

  private static void loadInitialDrivers() {
      ....
      AccessController.doPrivileged(new PrivilegedAction<Void>() {
          public Void run() {

              ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
              Iterator<Driver> driversIterator = loadedDrivers.iterator();
              try{
                  while(driversIterator.hasNext()) {
                      driversIterator.next();
                  }
              } catch(Throwable t) {
              // Do nothing
              }
              return null;
          }
      });
      ...
  }
}
```

```java
package java.util;

public final class ServiceLoader<S> implements Iterable<S> {

  private static final String PREFIX = "META-INF/services/";

  private Iterator<String> parse(Class<?> service, URL u) throws ServiceConfigurationError {
      InputStream in = null;
      BufferedReader r = null;
      ArrayList<String> names = new ArrayList<>();
      try {
          in = u.openStream();
          r = new BufferedReader(new InputStreamReader(in, "utf-8"));
          int lc = 1;
          while ((lc = parseLine(service, u, r, lc, names)) >= 0);
      } catch (IOException x) {
          fail(service, "Error reading configuration file", x);
      } finally {
          try {
              if (r != null) r.close();
              if (in != null) in.close();
          } catch (IOException y) {
              fail(service, "Error closing configuration file", y);
          }
      }
      return names.iterator();//[com.mysql.cj.jdbc.Driver]
  }

  private class LazyIterator implements Iterator<S> {

      public S next() {
          if (acc == null) {
              return nextService();
          } else {
              PrivilegedAction<S> action = new PrivilegedAction<S>() {
                  public S run() { return nextService(); }
              };
              return AccessController.doPrivileged(action, acc);
          }
      }

      private S nextService() {
          if (!hasNextService())
              throw new NoSuchElementException();
          String cn = nextName;//com.mysql.cj.jdbc.Driver
          nextName = null;
          Class<?> c = null;
          try {
              c = Class.forName(cn, false, loader);//加载 com.mysql.cj.jdbc.Driver
          } catch (ClassNotFoundException x) {
              fail(service,
                    "Provider " + cn + " not found");
          }
      }

      private boolean hasNextService() {
        if (nextName != null) {
            return true;
        }
        if (configs == null) {
            try {
                String fullName = PREFIX + service.getName();// META-INF/services/java.sql.Driver
                if (loader == null)
                    configs = ClassLoader.getSystemResources(fullName);
                else
                    configs = loader.getResources(fullName);
            } catch (IOException x) {
                fail(service, "Error locating configuration files", x);
            }
        }
        while ((pending == null) || !pending.hasNext()) {
            if (!configs.hasMoreElements()) {
                return false;
            }
            pending = parse(service, configs.nextElement());
        }
        nextName = pending.next();//com.mysql.cj.jdbc.Driver
        return true;
    }
  }
}
```

```java
package com.mysql.cj.jdbc;

public class Driver extends NonRegisteringDriver implements java.sql.Driver {
  static {
      try {
          java.sql.DriverManager.registerDriver(new Driver());//注册
      } catch (SQLException E) {
          throw new RuntimeException("Can't register driver!");
      }
  }
}
```

## 2.总结
1. `java.sql.DriverManager.class` 被类加载器加载时会去类路径下加载 `META-INF/services/java.sql.Driver` 这个文件，并读取内容

2. 文件中写有一行数据 `com.mysql.cj.jdbc.Driver`

3. 加载 `com.mysql.cj.jdbc.Driver.class`

4. 注册  `java.sql.DriverManager.registerDriver(new Driver())`