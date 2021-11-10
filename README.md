# zzimkkong-trouble-shooting

 전체 과정을 코드를 통해 설명드리겠습니다. (실제 코드는 인자값이 하드코딩되어있지 않습니다)

 `MemberRepository`라는 회원에 대한 JpaRepository가 있다고 가정하겠습니다. 빈 컨테이너로부터 이 클래스에 해당하는 빈 인스턴스를 반환받습니다.
 ``` Java
 String[] beanNames = beanFactory.getBeanNamesForType(MemberRepository.class);
 // beanNames에 대해 루프를 돕니다.
 Object target = beanFactory.getBean("memberRepository");
 ```

 그리고 로깅 프록시 객체를 생성합니다.

 ``` Java
 Object logProxy = LogAspect.createLogProxy(target, MemberRepository.class);
 // 위 정적 팩토리 메소드는 인자로 넘겨진 클래스명으로 로깅을 하는 프록시를 만듭니다.
 ```

 기존의 BeanDefinition을 삭제하고 로깅 프록시를 등록합니다.

 ``` Java
// 빈 정의 삭제
 beanDefinitionRegistry.removeBeanDefinition("memberRepository");

// 새로운 빈 정의 등록
 beanDefinitionRegistry.registerBeanDefinition("memberRepository", proxyBeanDefinition);

// 프록시 객체를 등록
 beanFactory.registerSingleton("memberRepository", logProxy);
 ```
 
 -----------
 
 ## 부록

 ### createLogProxy() 메소드 상세 (원본)
 - 실제 코드에는 메소드와 내부 클래스가 정의된 순서가 조금 다릅니다.
 
 ``` Java
     static <T> T createLogProxy(Object target, Class<T> typeToLog, String logGroup) {
        final LogProxyHandler logProxyHandler = new LogProxyHandler(target, typeToLog, logGroup);
        return typeToLog.cast(
                Proxy.newProxyInstance(
                        typeToLog.getClassLoader(),
                        new Class[]{typeToLog},
                        logProxyHandler));
    }

    private static class LogProxyHandler implements InvocationHandler {
        private final Object target;
        private final Class<?> typeToLog;
        private final String logGroup;

        private LogProxyHandler(Object target, Class<?> typeToLog, String logGroup) {
            this.target = target;
            this.typeToLog = typeToLog;
            this.logGroup = logGroup;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            long startTime = System.currentTimeMillis();
            final Object invokeResult = method.invoke(target, args);
            long endTime = System.currentTimeMillis();
            long timeTaken = endTime - startTime;

            logExecutionInfo(typeToLog, method, timeTaken, logGroup);

            return invokeResult;
        }
    }
    
    private static void logExecutionInfo(Class<?> typeToLog, Method method, long timeTaken, String logGroup) {
        log.info("{} took {} ms. (info group by '{}')",
                value("method", typeToLog.getName() + "." + method.getName() + "()"),
                value("execution_time", timeTaken),
                value("group", logGroup));
    }
 ```


### 빈 후처리기, `ProxyFactory`를 이용하는 방식으로 업데이트 (현 상황)
 
 ``` Java
     static Object createLogProxy(Object target, Class<?> typeToLog, String logGroup) {
        // Advisor 생성
        AspectJExpressionPointcutAdvisor advisor = new AspectJExpressionPointcutAdvisor();
      
        // 적용될 클래스와 메소드의 기준 설정, 내부적으로 PointCut 객체를 만들어냅니다.
        advisor.setExpression("execution(public * com.woowacourse.zzimkkong..*(..))");

        // 부가기능을 지정, ExecutionTimeLogAdvice는 수행 시간을 측정하라는 부가기능을 담고 있습니다.
        ExecutionTimeLogAdvice advice = new ExecutionTimeLogAdvice(typeToLog, logGroup);
        advisor.setAdvice(advice);

        // 프록시를 생성합니다.
        ProxyFactory proxyFactory = new ProxyFactory(target);
        proxyFactory.addAdvisor(advisor);
        proxyFactory.setProxyTargetClass(true);

        return proxyFactory.getProxy();
    }


    // 참고. ExecutionTimeLogAdvice 클래스 (LogAspect 내부로 감추었습니다.)
    private static class ExecutionTimeLogAdvice implements MethodInterceptor {
        private final Class<?> typeToLog;
        private final String logGroup;

        private ExecutionTimeLogAdvice(Class<?> typeToLog, String logGroup) {
            this.typeToLog = typeToLog;
            this.logGroup = logGroup;
        }

        @Override
        public Object invoke(MethodInvocation invocation) throws Throwable {
            long startTime = System.currentTimeMillis();
            
            // 타겟 인스턴스의 메소드를 실행합니다.
            final Object result = invocation.proceed();

            long endTime = System.currentTimeMillis();
            long timeTaken = endTime - startTime;

            Method method = invocation.getMethod();
            
            // LogAspect의 로거를 이용하여 로깅하라는 static 메소드
            logExecutionInfo(typeToLog, method, timeTaken, logGroup);

            return result;
        }
    }
 ```
