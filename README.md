## Spring Boot Admin

# 1. Visão Geral
Spring Boot Admin é um aplicativo da web, usado para gerenciar e monitorar aplicativos Spring Boot. Cada aplicativo é considerado um cliente e se registra no servidor admin. Nos bastidores, a mágica é dada pelos endpoints Spring Boot Actuator.

Neste artigo, vamos descrever as etapas para configurar um servidor Spring Boot Admin e como um aplicativo se torna um cliente.


# 2. Configuração do servidor de administração
Em primeiro lugar, precisamos criar um aplicativo da web Spring Boot simples e também adicionar a seguinte dependência Maven:

```
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
    <version>2.4.0</version>
</dependency>
```

Após isso, o @EnableAdminServer estará disponível, então iremos adicioná-lo à classe principal, conforme mostrado no exemplo abaixo:

```
@EnableAdminServer
@SpringBootApplication
public class SpringBootAdminServerApplication(exclude = AdminServerHazelcastAutoConfiguration.class) {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootAdminServerApplication.class, args);
    }
}
```


Neste ponto, estamos prontos para iniciar o servidor e registrar os aplicativos cliente.

# 3. Configurando um cliente
Agora, depois de configurar nosso servidor de administração, podemos registrar nosso primeiro aplicativo Spring Boot como um cliente. Devemos adicionar a seguinte dependência Maven:

```
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
    <version>2.4.0</version>
</dependency>
```

Em seguida, precisamos configurar o cliente para saber sobre a URL base do servidor admin. Para que isso aconteça, basta adicionar a seguinte propriedade:

```
spring.boot.admin.client.url=http://localhost:8080
```

A partir do Spring Boot 2, pontos de extremidade diferentes de saúde e informações não são expostos por padrão.

Vamos expor todos os endpoints:

```
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
```

### 3.1. Spring Boot Admin Server

* mvn clean install
* mvn spring-boot:run
* starts on port 8080
* login with admin/admin
* to activate mail notifications uncomment the starter mail dependency
and the mail configuration from application.properties
* add some real credentials if you want the app to send emails
* to activate Hipchat notifications proceed same as for email

### 3.2. Spring Boot App Client

* mvn clean install
* mvn spring-boot:run
* starts on port 8081
* basic auth client/client


# 4. Configuração de segurança

O servidor Spring Boot Admin tem acesso aos endpoints sensíveis do aplicativo, portanto, é aconselhável adicionar alguma configuração de segurança ao aplicativo admin e cliente.

Em primeiro lugar, vamos nos concentrar na configuração da segurança do servidor de administração. Devemos adicionar as seguintes dependências do Maven:

```
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-server-ui-login</artifactId>
    <version>1.5.7</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
    <version>2.4.0</version>
</dependency>
```

Isso habilitará a segurança e adicionará uma interface de login ao aplicativo admin.

A seguir, adicionaremos uma classe de configuração de segurança, como você pode ver abaixo:

```
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    private final AdminServerProperties adminServer;

    public WebSecurityConfig(AdminServerProperties adminServer) {
        this.adminServer = adminServer;
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        SavedRequestAwareAuthenticationSuccessHandler successHandler = 
          new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setTargetUrlParameter("redirectTo");
        successHandler.setDefaultTargetUrl(this.adminServer.getContextPath() + "/");

        http
            .authorizeRequests()
                .antMatchers(this.adminServer.getContextPath() + "/assets/**").permitAll()
                .antMatchers(this.adminServer.getContextPath() + "/login").permitAll()
                .anyRequest().authenticated()
                .and()
            .formLogin()
                .loginPage(this.adminServer.getContextPath() + "/login")
                .successHandler(successHandler)
                .and()
            .logout()
                .logoutUrl(this.adminServer.getContextPath() + "/logout")
                .and()
            .httpBasic()
                .and()
            .csrf()
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .ignoringRequestMatchers(
                  new AntPathRequestMatcher(this.adminServer.getContextPath() + 
                    "/instances", HttpMethod.POST.toString()), 
                  new AntPathRequestMatcher(this.adminServer.getContextPath() + 
                    "/instances/*", HttpMethod.DELETE.toString()),
                  new AntPathRequestMatcher(this.adminServer.getContextPath() + "/actuator/**"))
                .and()
            .rememberMe()
                .key(UUID.randomUUID().toString())
                .tokenValiditySeconds(1209600);
    }
}
```

Existe uma configuração de segurança simples, mas após adicioná-la, notaremos que o cliente não pode mais se registrar no servidor.

Para registrar o cliente no servidor recém-protegido, devemos adicionar mais algumas configurações ao arquivo de propriedade do cliente:

```
spring.boot.admin.client.username=admin
spring.boot.admin.client.password=admin
```

Estamos no ponto em que protegemos nosso servidor de administração. Em um sistema de produção, naturalmente, os aplicativos que estamos tentando monitorar serão protegidos. Portanto, adicionaremos segurança ao cliente também - e notaremos na interface da IU do servidor de administração que as informações do cliente não estão mais disponíveis.

Temos que adicionar alguns metadados que enviaremos ao servidor admin. Essas informações são usadas pelo servidor para se conectar aos endpoints do cliente:

```
spring.security.user.name=client
spring.security.user.password=client
spring.boot.admin.client.instance.metadata.user.name=${spring.security.user.name}
spring.boot.admin.client.instance.metadata.user.password=${spring.security.user.password}
```

O envio de credenciais via HTTP, obviamente, não é seguro - portanto, a comunicação precisa passar por HTTPS.

5. Recursos de monitoramento e gerenciamento

O Spring Boot Admin pode ser configurado para exibir apenas as informações que consideramos úteis. Só temos que alterar a configuração padrão e adicionar nossas próprias métricas necessárias:

```
spring.boot.admin.routes.endpoints=env, metrics, trace, jolokia, info, configprops
```

À medida que avançamos, veremos que existem alguns outros recursos que podem ser explorados. Estamos falando sobre o gerenciamento de bean JMX usando Jolokia e também o gerenciamento Loglevel.

Spring Boot Admin também oferece suporte à replicação de cluster usando Hazelcast. Só precisamos adicionar a seguinte dependência Maven e deixar a configuração automática fazer o resto:

```
<dependency>
    <groupId>com.hazelcast</groupId>
    <artifactId>hazelcast</artifactId>
    <version>4.0.3</version>
</dependency>
```

Se quisermos uma instância persistente de Hazelcast, vamos usar uma configuração personalizada:

```
@Configuration
public class HazelcastConfig {

    @Bean
    public Config hazelcast() {
        MapConfig eventStoreMap = new MapConfig("spring-boot-admin-event-store")
          .setInMemoryFormat(InMemoryFormat.OBJECT)
          .setBackupCount(1)
          .setEvictionConfig(new EvictionConfig().setEvictionPolicy(EvictionPolicy.NONE))
          .setMergePolicyConfig(new MergePolicyConfig(PutIfAbsentMergePolicy.class.getName(), 100));

        MapConfig sentNotificationsMap = new MapConfig("spring-boot-admin-application-store")
          .setInMemoryFormat(InMemoryFormat.OBJECT)
          .setBackupCount(1)
          .setEvictionConfig(new EvictionConfig().setEvictionPolicy(EvictionPolicy.LRU))
          .setMergePolicyConfig(new MergePolicyConfig(PutIfAbsentMergePolicy.class.getName(), 100));

        Config config = new Config();
        config.addMapConfig(eventStoreMap);
        config.addMapConfig(sentNotificationsMap);
        config.setProperty("hazelcast.jmx", "true");

        config.getNetworkConfig()
          .getJoin()
          .getMulticastConfig()
          .setEnabled(false);
        TcpIpConfig tcpIpConfig = config.getNetworkConfig()
          .getJoin()
          .getTcpIpConfig();
        tcpIpConfig.setEnabled(true);
        tcpIpConfig.setMembers(Collections.singletonList("127.0.0.1"));
        return config;
    }
}
```

# 6. Notificações
A seguir, vamos discutir a possibilidade de receber notificações do servidor admin se algo acontecer com nosso cliente registrado. Os seguintes notificadores estão disponíveis para configuração:

- O email;
- PagerDuty;
- OpsGenie;
- Hipchat;
- Slack;
- Vamos conversar.

6.1. Notificações de e-mail
Primeiro, vamos nos concentrar na configuração de notificações de e-mail para nosso servidor de administração. Para que isso aconteça, temos que adicionar a dependência mail starter conforme mostrado abaixo:

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
    <version>2.4.0</version>
</dependency>
```

Depois disso, devemos adicionar alguma configuração de e-mail:

```
spring.mail.host=smtp.example.com
spring.mail.username=smtp_user
spring.mail.password=smtp_password
spring.boot.admin.notify.mail.to=admin@example.com
```

Agora, sempre que nosso cliente cadastrado muda seu status de UP para OFFLINE ou outro, um email é enviado para o endereço configurado acima. Para os outros notificadores, a configuração é semelhante.

### 6.2. Notificações Hipchat
Como veremos, a integração com o Hipchat é bastante direta; existem apenas algumas propriedades obrigatórias para definir:

```
spring.boot.admin.notify.hipchat.auth-token=<generated_token>
spring.boot.admin.notify.hipchat.room-id=<room-id>
spring.boot.admin.notify.hipchat.url=https://yourcompany.hipchat.com/v2/
```

Definidos estes, perceberemos na sala do Hipchat que recebemos notificações sempre que houver alteração do status do cliente.

### 6.3. Configuração de notificações personalizadas
Podemos configurar um sistema de notificação personalizado tendo à nossa disposição algumas ferramentas poderosas para isso. Podemos usar um notificador de lembrete para enviar uma notificação programada até que o status do cliente mude.

Ou talvez queiramos enviar notificações a um conjunto filtrado de clientes. Para isso, podemos usar um notificador de filtragem:

```
@Configuration
public class NotifierConfiguration {
    private final InstanceRepository repository;
    private final ObjectProvider<List<Notifier>> otherNotifiers;

    public NotifierConfiguration(InstanceRepository repository, 
      ObjectProvider<List<Notifier>> otherNotifiers) {
        this.repository = repository;
        this.otherNotifiers = otherNotifiers;
    }

    @Bean
    public FilteringNotifier filteringNotifier() {
        CompositeNotifier delegate = 
          new CompositeNotifier(this.otherNotifiers.getIfAvailable(Collections::emptyList));
        return new FilteringNotifier(delegate, this.repository);
    }

    @Bean
    public LoggingNotifier notifier() {
        return new LoggingNotifier(repository);
    }

    @Primary
    @Bean(initMethod = "start", destroyMethod = "stop")
    public RemindingNotifier remindingNotifier() {
        RemindingNotifier remindingNotifier = new RemindingNotifier(filteringNotifier(), repository);
        remindingNotifier.setReminderPeriod(Duration.ofMinutes(5));
        remindingNotifier.setCheckReminderInverval(Duration.ofSeconds(60));
        return remindingNotifier;
    }
}
```

# 7. Conclusão
Este tutorial de introdução cobre as etapas simples que é preciso fazer para monitorar e gerenciar seus aplicativos Spring Boot usando Spring Boot Admin.

A autoconfiguração nos permite adicionar apenas algumas configurações menores e, no final, ter um servidor de administração totalmente funcional.






