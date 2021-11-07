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
 
 ## 부록
 ### createLogProxy() 메소드 상세
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
