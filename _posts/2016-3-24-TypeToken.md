## Type 接口
Type is the common superinterface for all types in the Java programming language. These include raw types, parameterized types, array types, type variables and primitive types.

`Type`接口是java语言中所有类型的父类，包含原始类型（对应`Class`类），参数化类型，数组类型和基本类型。其比较常用的子接口和子类有`GenericArrayType`, `ParameterizedType`, `TypeVariable<D>`, `WildcardType`，`Class`分别表示数组类型，参数化类型（如`Collection`），类型变量（如`T`），通配符类型（如`？ extend Object`）和其他类型.

## Class 类
Class类的实例代表了运行中的Java程序的类和接口。枚举也作为一种类，同时注解作为一种接口。所有具有相同的元素类型和维数的数组对象共享一个Class实例。Java基本类型和关键字void也由Class对象表示。Class 对象是在加载类时由 Java 虚拟机以及通过调用类加载器中的defineClass 方法自动构造的。

## TypeToken

Guava提供了TypeToken, 它使用了基于反射的技巧甚至让你在运行时都能够巧妙的操作和查询泛型类型。想象一下TypeToken是创建，操作，查询泛型类型（以及，隐含的类）对象的方法。

一种代表泛型的类型。个人理解就是封装了一个Type对象，能够保存泛型信息，同时又提供类似于Class类的操作。

提供了一些以前只能在`Class`类使用的方法来支持`Type`，如`isSubTypeOf`，`isArray`，`getComponentType`，此外还提供了额外的工具方法，如`getTypes`,`resolveType`.



Guava提供了三种方式获取TypeToken的实例

- 封装`Type`对象，如`TypeToken.of(method.getGenericReturnType())`
- 用匿名子类捕获泛型类，如`new TypeToken<List<String>>() {}`
- 还是使用匿名子类，并且在已知类型参数的上下文类中解析它，如

         abstract class IKnowMyType<T> {
          TypeToken<T> type = new TypeToken<T>(getClass()) {};
        }

`TypeToken`类继承自`TypeCapture`抽象类，`TypeCapture`只实现了一个方法。

    /** Returns the captured type. */
      final Type capture() {
        Type superclass = getClass().getGenericSuperclass();
        checkArgument(superclass instanceof ParameterizedType,
            "%s isn't parameterized", superclass);
        return ((ParameterizedType) superclass).getActualTypeArguments()[0];
      }

返回继承该类的类的泛型信息。这个方法之所以能够生效，因为`TypeToken`是一个抽象类，因此它的子类能够通过`getClass().getGenericSuperclass()`获取到`TypeToken`类的类型，由此获得泛型类的类型。

    /**
       * Constructs a new type token of {@code T}.
       *
       * <p>Clients create an empty anonymous subclass. Doing so embeds the type
       * parameter in the anonymous class's type hierarchy so we can reconstitute
       * it at runtime despite erasure.
       *
       * <p>For example: <pre>   {@code
       *   TypeToken<List<String>> t = new TypeToken<List<String>>() {};}</pre>
       */
      protected TypeToken() {
        this.runtimeType = capture();
        checkState(!(runtimeType instanceof TypeVariable),
            "Cannot construct a TypeToken for a type variable.\n"
            + "You probably meant to call new TypeToken<%s>(getClass()) "
            + "that can resolve the type variable for you.\n"
            + "If you do need to create a TypeToken of a type variable, "
            + "please use TypeToken.of() instead.", runtimeType);
      }

无参构造函数，将泛型类型捕获。如果泛型变量为类型变量，抛出异常。

    /**
       * <p>Resolves the given {@code type} against the type context represented by this type.
       * For example: <pre>   {@code
       *   new TypeToken<List<String>>() {}.resolveType(
       *       List.class.getMethod("get", int.class).getGenericReturnType())
       *   => String.class}</pre>
       */
      public final TypeToken<?> resolveType(Type type) {
        checkNotNull(type);
        TypeResolver resolver = typeResolver;
        if (resolver == null) {
          resolver = (typeResolver = TypeResolver.accordingTo(runtimeType));
        }
        return of(resolver.resolveType(type));
      }

`resolveType()`方法是一个查询操作，将参数的`Type`类型转换为当前`TypeToken`对象持有的泛型信息的方法。举个例子：

    /**
       * Constructs a new type token of {@code T} while resolving free type variables in the context of
       * {@code declaringClass}.
       *
       * <p>Clients create an empty anonymous subclass. Doing so embeds the type
       * parameter in the anonymous class's type hierarchy so we can reconstitute
       * it at runtime despite erasure.
       *
       * <p>For example: <pre>   {@code
       *   abstract class IKnowMyType<T> {
       *     TypeToken<T> getMyType() {
       *       return new TypeToken<T>(getClass()) {};
       *     }
       *   }
       *
       *   new IKnowMyType<String>() {}.getMyType() => String}</pre>
       */
      protected TypeToken(Class<?> declaringClass) {
        Type captured = super.capture();
        if (captured instanceof Class) {
          this.runtimeType = captured;
        } else {
          this.runtimeType = of(declaringClass).resolveType(captured).runtimeType;
        }
      }

`TypeToken`的带`Class`参数的构造函数，首先执行父类的`capture()`方法获取当前泛型类型，如果获取的结果为类型变量时，再通过参数里的`Class`对象构造`TypeToken`对象，调用`resolveType`方法查询该类型变量真实的类型。
