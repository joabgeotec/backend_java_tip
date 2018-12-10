### **1. Para implantar serviços spring**

1. Gerar o .jar do projeto (**será criado na pasta /target do projeto**) 

```
> ./mvnw clean install -Dmaven.test.skip=true
```

2. Rodar o .jar

```
> java -jar nome-do-arquivo.jar
```

### **2. Implantar backend no linux CentOS**

1. Criar .sh com comando para rodar serviço:

```
> java -jar nome-do-arquivo.jar
```

2. Chamar arquivo.sh no arquivo de inicialização do sistema:  **/etc/rc.local**

> OBS 1: Adicionar comando **nohup** antes da chamada do script para não printar o console e **&** após o comando caso tenha mais de uma chamada para não travar.


> OBS 2: No caso de substituição de um .jar (alterações no sistema) deve-se derrubar todos os serviços que estiverem sendo chamados no **rc.local** antes de substituir algum jar e rodar novamente, utilizando os comandos:


```
> netstat -plten | grep NUMERO_DA_PORTA
``` 


```
> kill -9 ID_DO_PROCESSO
``` 


### **3. Autenticação JWT**

 1. Incluir dependência do **spring-security** no pom.xml

> Ao incluir esta dependência, todas as requisições serão automaticamente bloqueadas, retornando **403 (Forbidden)**


> Também é necessário incluir a dependência do JWT no pom para validação e geração de token:


```
<dependency>
	<groupId>io.jsonwebtoken</groupId>
	<artifactId>jjwt</artifactId>
	<version>0.7.0</version>
</dependency>
```


 2. Para remover bloqueio automático, deve-se criar uma classe de configuração que estende **WebSecurityConfigurerAdapter** com as seguintes anotações acima da classe:

```
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled=true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    ...
 ```

3. Os endpoints a desbloqueados devem ser incluídos no método **configure**, exemplo:


```
@Override
protected void configure(HttpSecurity httpSecurity) throws Exception {
		
	httpSecurity.csrf().disable().authorizeRequests()
        .antMatchers(HttpMethod.POST, "/api/usuarios/autenticar").permitAll()
        .antMatchers(HttpMethod.OPTIONS, "/api/usuarios/autenticar").permitAll()
        .antMatchers(HttpMethod.GET, "/NOVO-ENDPOINT-DESBLOQUEADO").permitAll()
        .anyRequest().authenticated()
        .and()

	.addFilterBefore(new JWTAuthenticationFilter(),
		UsernamePasswordAuthenticationFilter.class);
}
```

4. Os filtros **JWTAuthenticationFilter** e **JWTLoginFilter** servem para validar a existência de um token na requisição e validar os dados de autenticação enviados.


> Ver implementação padrão no tecadmin


5. No método para criação de token, é possível modificar o tempo de expiração do token em milisegundos, por exemplo, no código abaixo o tempo está setado como 7200000 que equivale a 2 horas:


```
public String generateToken(String chave) {
	return Jwts.builder()
		.setSubject(chave)
		.setExpiration(new Date(System.currentTimeMillis() + 7200000))
		.signWith(SignatureAlgorithm.HS512, this.secret)
		.compact();
}
```


6. Para enviar requisições utilizando o token obtido, basta incluir na requisição o header **Authorization** com o valor **Bearer eyJhbGciOiJIUzUxMiJ9.eyJz...**

### **4. Injeção de dependência do Spring**

A injeção de dependência do spring funciona com a anotação **@Autowired**, por exemplo:


```
@Autowired
private IUsuarioRepository usuarioRep;
```


Adicionando esta anotação, a classe IUsuarioRepository estará pronta para ser utilizada, pois será injetada antes da classe mãe ser instanciada.

> OBS: Para que uma classe seja **elegível** a ser injetada com @Autowired, ela precisa ser um componente Spring. Para isso, basta anotar a classe com **@Component** ou suas especializações: @Service, @Controller, @Repository etc...

### **5. Utilizando propriedades externas**

Para utilizar propriedades descritas no arquivo **application.properties**

1. Incluir a propriedade no arquivo, exemplo:


```
setup.usuario.username = Administrador
```

2. Incluir a injeção de dependência do arquivo externo na classe onde será utilizada:


```
@Autowired
Environment env;
```

3. Utilizar a propriedade:

```
env.getProperty("setup.usuario.username");
```

> OBS: Para obter a configuração presente em outro arquivo de configuração (**o application.properties é o padrão**), basta seguir o mesmo procedimento descrito acima, adicionando apenas o caminho do novo arquivo em cima da classe, exemplo:

```
@PropertySource("file:propriedades.yml")
public class Teste {
```

### **6. Executando código ao iniciar a aplicação spring**

Para executar um trecho de código sempre que a aplicação iniciar, basta fazer com que a classe principal da aplicação (anotada com @SpringBootApplication) implemente a interface **CommandLineRunner**, exemplo:

```
@SpringBootApplication
public class TecAdminApplication implements CommandLineRunner { 
	...
```

Feito isso, basta incluir qualquer código no método run:


```
@Override
public void run(String... args) throws Exception {
	System.out.println("Teste");
}
```

### **7. Conexão com múltiplos bancos de dados no Spring**

Para conectar mais de um banco de dados (**dataSource**) no spring:

1. Adicionar as múltiplas configurações no arquivo application.properties, com a chave de configuração diferente, exemplo:


```
master.spring.datasource.url=jdbc:postgresql://187.84.228.69;databaseName=teste
master.spring.datasource.username=root
master.spring.datasource.password=teste

destino.spring.datasource.url=jdbc:sqlserver://187.84.228.69;databaseName=sjr_cadastro
destino.spring.datasource.username=sa
destino.spring.datasource.password=abc
```

> É necessário que a dependência de cada SGBD utilizado esteja no pom.xml

2. Criar uma classe dataSource

> Ver exemplo de implementação padrão na classe **MasterConfig** do **Geoitbi backend**

3. As classes e pacotes que estiverem dentro do mesmo pacote desta classe dataSource poderão manipular o banco de dados configurado, para utilizar em pactes diferentes, deve-se modificar a linha no dataSource:

```
factoryBean.setPackagesToScan(this.getClass().getPackage().getName());
```

Para:

```
factoryBean.setPackagesToScan("com.nome.do.pacote");
```

### **8. Ignorando atributos na serialização JSON**

1. Um problema comum que acontece quando se tem um relacionamento bidirecional nos models é cair no looping infinito das propriedades, por exemplo:

> Um Grupo possui uma propriedade list de recursos e um recurso possui uma propriedade list de grupos.

2. Ao retornar uma classe relacionada para o response, vai acontecer um looping infinito, pois o grupo vai apresentar os recursos e cada recurso vai apresentar os grupos e assim vai repetindo.

3. Para contar este problema deve-se utilizar a anotação **@JsonIgnore** acima da propriedade que você deseja ignorar durante a conversão para JSON efetuada quando se inclui a classe no ResponseEntity do Spring.

### **9. Fluxo de migração do Geoitbi backend**

1. O processo inicia recebendo a requisição **get** - /integracao no controller

2. É chamado o processo de autenticação no Tecadmin e o Token é gravado em um singleton (a instância única desse objeto pode ser chamada em qualquer lugar do programa)

> Isso acontece porque as chamadas de escrita em Log do Tecadmin que serão chamadas durante a migração necessitam de autenticação prévia.

3. Com o singleton de autenticação gravado, o service de migração é chamado dentro de uma **Thread**.

> Isso acontece para que o usuário que chame a rota de migração tenha a resposta que a migração iniciou e não precise ela esperar para receber o response, o processo acontecerá em background (assíncrono).

4. No método iniciarMigracao presente no MigracaoService o processo é iniciado, o log de inicialização é gravado e a contagem do tempo de duração é iniciada.

5. O processo de ETL (migração) das entidades é feita utilizando o padrão strategy, ou seja, o migracaoService não tem acoplamento com todas as classes de ETL, ele está ligado com uma **interface** chamada ***IETLStrategy***, que é instanciada por um Factory (***FactoryETLService***).

6. O Factory retorna a instância correta da classe de ETL, injetando ao mesmo tempo o DAO correto, por exemplo, sendo enviada a string "lote" para o factory, ele retornaria utilizando o seguinte procedimento:

```
switch (nome) {
	
	case LOTE_FACT: 		
		return new ETLLote(daoLote);
		... 
```

> O DAO passado para a classe de ETL contém todos os repositórios necessários para o processo, bem como alguns métodos de manipulação do banco de dados, no mínimo deve conter o método de obter os dados da origem e salvar no repositório de destino.

7. Cada classe de ETL implementa a interface **IETLStrategy**, sendo assim, deve conter a implementação de, no mínimo, três métodos: ***extrairDados***, ***processarDados*** e ***carregarDados***.

8. Cada método será implementado de acordo com a necessidade de cada entidade (strategy), sendo assim, o método migrar do MigracaoService, apenas irá chamar os métodos pela interface independente de qual instância concreta esteja lidando.

9. Ao final do processo o tempo de duração é calculado e o log de finalização é gravado.

**Observações**

> É importante que o Tecadmin esteja rodando corretamente para que os logs funcionem durante a migração.

> Cada tabela de destino ou origem utilizada na migração deve ter sido previamente mapeada no **pacote** do dataSource correto.

> É importante verificar o ddl-auto nos dataSources antes de rodar a migração, pois se estiver como **create** ou **update** os dados da tabela serão apagados, deve-se ter cuidado para não apagar os dados da origem em produção, por isso é bom deixar sempre na opção **none**.

```
properties.setProperty("hibernate.hbm2ddl.auto", "none");
```

> As entidades de DSA são gravadas no meio do processo e são cópias as entidades de destino, a cada processo de migração os dados são apagados e substituídos pelos dados da migração atual.

> O método getDadosOrigem nos DAO's está chamando o método do repository findTop100 para limitar o retorno e ficar melhor de testar. Deve ser modificado para findAll() quando for ser utilizado em produção.

