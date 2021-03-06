# 验证码机制
    验证码直接网关异步生成，基于webflux

>webflux 生成验证码的handler

    public class ImageCodeHandler implements HandlerFunction<ServerResponse> {
        private final Producer producer;
        private final RedisTemplate redisTemplate;
    
        @Override
        public Mono<ServerResponse> handle(ServerRequest serverRequest) {
            //生成验证码
            String text = producer.createText();
            BufferedImage image = producer.createImage(text);
    
            //保存验证码信息
            String randomStr = serverRequest.queryParam("randomStr").get();
            redisTemplate.opsForValue().set(CommonConstants.DEFAULT_CODE_KEY + randomStr, text, 60, TimeUnit.SECONDS);
    
            // 转换流信息写出
            FastByteArrayOutputStream os = new FastByteArrayOutputStream();
            try {
                ImageIO.write(image, "jpeg", os);
            } catch (IOException e) {
                log.error("ImageIO write err", e);
                return Mono.error(e);
            }
    
            return ServerResponse
                .status(HttpStatus.OK)
                .contentType(MediaType.IMAGE_JPEG)
                .body(BodyInserters.fromResource(new ByteArrayResource(os.toByteArray())));
        }
    }
    
>webflux 请求处理入口

    public class RouterFunctionConfiguration {
        private final HystrixFallbackHandler hystrixFallbackHandler;
        private final ImageCodeHandler imageCodeHandler;
    
        @Bean
        public RouterFunction routerFunction() {
            return RouterFunctions.route(RequestPredicates.GET("/code")
                    .and(RequestPredicates.accept(MediaType.TEXT_PLAIN)), imageCodeHandler);
    
        }
    
    }
    
>校验逻辑,通过oauth2 终端的client-id 来确定是否校验验证码

    public class ValidateCodeGatewayFilter extends AbstractGatewayFilterFactory {
        @Override
        public GatewayFilter apply(Object config) {
            return (exchange, chain) -> {
                ServerHttpRequest request = exchange.getRequest();
    
    
                // 终端设置不校验， 直接向下执行
                String[] clientInfos = WebUtils.getClientId(request);
                if (filterIgnorePropertiesConfig.getClients().contains(clientInfos[0])) {
                    return chain.filter(exchange);
                }
    
                //校验验证码
                checkCode(request);
    
                return chain.filter(exchange);
            };
        }
    }
>配置终端不校验验证码
>修改base-gateway-dev.yml

    不校验验证码终端
    ignore:
      clients:
        - test
      swagger-providers:
        - ${AUTH-HOST:base-auth}