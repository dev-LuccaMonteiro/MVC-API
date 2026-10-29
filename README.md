# Comparativo Arquitetural em Java: CRUD Spring MVC Tradicional vs. API RESTful com Spring Boot

Este documento apresenta uma análise detalhada e técnica das diferenças entre a construção de um CRUD usando o **padrão MVC tradicional (Server-Side Rendering)** e uma **API RESTful (Desacoplada)** no ecossistema Java (Spring Framework / Jakarta EE).

---

## Tabela Comparativa Resumida (Ecossistema Java)

| Aspecto | CRUD Spring MVC Tradicional | API RESTful (Spring Boot) |
| :--- | :--- | :--- |
| **Tecnologia View/Resposta** | Thymeleaf, JSP, JSF (Renderização de HTML no servidor). | Jackson / Gson (Serialização de objetos Java para JSON). |
| **Anotações de Controller** | `@Controller` (retorna nomes de templates/views). | `@RestController` ou `@Controller` + `@ResponseBody`. |
| **Cliente Esperado** | Navegador Web (HTML/CSS renderizado). | React, Angular, Vue, Flutter, Android (Kotlin), iOS (Swift), outros microsserviços. |
| **Gerenciamento de Estado** | **Stateful**: Uso de `HttpSession`, `JSESSIONID` e Spring Security Session. | **Stateless**: Tokens JWT (Spring Security com OAuth2 Resource Server). |
| **Roteamento & Verbos** | Mapeamento de URLs com `GET` para formulários e `POST` para ações (`@GetMapping`, `@PostMapping`). | Mapeamento por recursos e verbos semânticos HTTP (`@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`). |

---

## 1. Quais as Diferenças Arquiteturais?

### A) Spring MVC Tradicional (Monolítico com Server-Side Rendering)

* **Arquitetura e Acoplamento:** Todas as camadas (*Controller*, *Service*, *Repository* e as páginas *Thymeleaf/JSP*) rodam no mesmo processo dentro da JVM e no mesmo projeto Maven/Gradle. O servidor **Spring Boot/Tomcat** processa as regras de negócio e compila o código HTML antes de enviar a resposta.
* **Modelo de Comunicação:** A cada interação no navegador (envio de formulário HTML via `<form action="/produtos" method="POST">`), o servidor Java recebe a requisição, reprocessa os dados e envia um novo documento HTML compilado de volta.
* **Gerenciamento de Estado (Stateful):** A aplicação mantém a sessão do usuário ativa usando a interface `jakarta.servlet.http.HttpSession`. O servidor injeta o cookie `JSESSIONID` no navegador do cliente para identificar a sessão armazenada na memória da JVM.

### B) API RESTful em Java (Arquitetura Cliente-Servidor Desacoplada)

* **Arquitetura e Acoplamento:** Aplica a separação *Client-Server* do padrão REST. O back-end Java funciona como uma aplicação *Headless*.
* **Modelo de Comunicação:** O `@RestController` expõe *endpoints* que aceitam e retornam **JSON** (convertidos automaticamente a partir de DTOs Java através da biblioteca **Jackson**).
* **Ausência de Estado (Statelessness):** O servidor Java **não armazena objetos de sessão HTTP em memória**. Cada requisição é autenticada de forma isolada através do envio de um token **JWT** no cabeçalho HTTP (`Authorization: Bearer <token>`). Isso permite criar réplicas da aplicação em containers Docker e gerenciá-las com Kubernetes sem necessidade de *sticky sessions*.

```
Spring MVC Tradicional (Server-Side Rendering):
[ Navegador ] <--- HTTP Request / Response (HTML) ---> [ @Controller + Thymeleaf + JPA ] <---> [ PostgreSQL ]

API RESTful Java (Desacoplada):
[ Cliente (React/App Mobile) ] <--- HTTP Request / Response (JSON) ---> [ @RestController + Service + JPA ] <---> [ PostgreSQL ]
```

---

## 2. Responsabilidades de Cada Camada no Java

### A) Camadas no Spring MVC Tradicional

1. **Entity / Domain (`@Entity`):** Mapeia as tabelas do banco de dados usando **JPA / Hibernate** (ex: `Produto.java`).
2. **View (Templates Thymeleaf `.html`):** Recebe o objeto `Model` do Spring e compila o HTML dinamicamente através de atributos como `th:text`, `th:each` e `th:field`.
3. **Controller (`@Controller`):** Processa requisições HTTP, injeta dados na interface `org.springframework.ui.Model` e retorna uma `String` com o nome da View Thymeleaf (ex: `return "produtos/formulario";`).

### B) Camadas na API RESTful Java (Clean Architecture / Layered Architecture)

Nas APIs RESTful com Spring Boot, aplica-se o princípio da Responsabilidade Única (SRP - SOLID):

1. **Controller Layer (`@RestController`):**
   * Ponto de entrada HTTP (`@RequestMapping("/api/v1/produtos")`).
   * Valida objetos DTO usando o **Jakarta Bean Validation** (`@Valid`, `@NotNull`, `@Size`).
   * Retorna objetos encapsulated em `ResponseEntity<T>` com o status HTTP adequado (`HttpStatus.OK`, `HttpStatus.CREATED`).
2. **Service Layer (`@Service`):**
   * Isola a lógica de negócio da aplicação.
   * Executa regras corporativas, cálculos, orquestração de transações (`@Transactional`) e converte entidades JPA para DTOs.
3. **Repository Layer (`@Repository`):**
   * Interface que estende `JpaRepository<Entity, ID>` ou `CrudRepository`.
   * Fornece operações de persistência e consultas dinâmicas via **Spring Data JPA** e JPQL.

---

## 3. Relações Entre as Camadas (Exemplos em Código Java)

### A) Spring MVC Tradicional: Dependência Bidirecional / Circular com `Model`

No Spring MVC tradicional, o *Controller* injeta instâncias de entidades diretamente no objeto `Model` do Spring para renderização na View:

```java
@Controller
@RequestMapping("/produtos")
public class ProdutoWebController {

    @Autowired
    private ProdutoRepository repository;

    @GetMapping("/novo")
    public String exibirFormulario(Model model) {
        model.addAttribute("produto", new Produto());
        return "produtos/formulario"; // Retorna o arquivo formulario.html
    }

    @PostMapping
    public String salvar(@ModelAttribute Produto produto) {
        repository.save(produto);
        return "redirect:/produtos"; // Redireciona a página via HTTP 302
    }
}
```

* **Impacto:** A View HTML (`Thymeleaf`) fica acoplada aos nomes das propriedades da classe `@Entity`. Se uma coluna ou tipo de atributo mudar na entidade Java, o formulário HTML pode quebrar diretamente em tempo de execução.

---

### B) API RESTful em Java: Relação Unidirecional com DTOs (Data Transfer Objects)

A API utiliza DTOs (criados com **Java Records** ou Lombok) para desacoplar a camada de persistência da interface HTTP externa:

```java
// DTOs de Entrada e Saída usando Java Records
public record ProdutoRequestDTO(
    @NotBlank(message = "O nome é obrigatório") String nome,
    @NotNull @Positive BigDecimal preco,
    Long categoriaId
) {}

public record ProdutoResponseDTO(
    Long id,
    String nome,
    BigDecimal preco
) {}
```

```java
// REST Controller no Spring Boot
@RestController
@RequestMapping("/api/v1/produtos")
public class ProdutoRestController {

    private final ProdutoService service;

    public ProdutoRestController(ProdutoService service) {
        this.service = service;
    }

    @PostMapping
    public ResponseEntity<ProdutoResponseDTO> criar(@RequestBody @Valid ProdutoRequestDTO dto) {
        ProdutoResponseDTO produtoCriado = service.salvar(dto);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}")
                .buildAndExpand(produtoCriado.id())
                .toUri();
        
        return ResponseEntity.created(location).body(produtoCriado); // Status HTTP 201 Created
    }
}
```

* **Impacto:** Alterações de tabelas do banco de dados na entidade JPA não alteram o contrato público em JSON da API, mantendo a compatibilidade do sistema com os clientes consumidores.

---

## 4. Persistência em um Banco de Dados com Java

Tanto no MVC quanto na API RESTful, o acesso ao banco em Java utiliza **JPA (Jakarta Persistence)** e **Hibernate** gerenciados pelo **Spring Data JPA**.

```java
@Entity
@Table(name = "tb_produtos")
public class Produto {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String nome;

    @Column(nullable = false)
    private BigDecimal preco;

    // Getters, Setters e Construtores
}

public interface ProdutoRepository extends JpaRepository<Produto, Long> {
    List<Produto> findByNomeContainingIgnoreCase(String nome);
}
```

### Diferenças no Controle de Persistência e Transação:

* **Spring MVC Tradicional:** O contexto do usuário autenticado é obtido diretamente de `SecurityContextHolder.getContext().getAuthentication()` na sessão do servidor.
* **API RESTful:**
  * **Gerenciamento de Transação:** O serviço usa a anotação `@Transactional` do Spring para controlar os limites de abertura, commit e rollback de transações ACID na JVM.
  * **Paginação Semântica:** A consulta utiliza a interface `Pageable` do Spring Data JPA e retorna um objeto `Page<T>` serializado em JSON com os metadados de paginação:

```java
@GetMapping
public ResponseEntity<Page<ProdutoResponseDTO>> listar(Pageable pageable) {
    Page<ProdutoResponseDTO> pagina = service.buscarTodos(pageable);
    return ResponseEntity.ok(pagina); // Retorna JSON com conteúdo, número de páginas e total de elementos
}
```

---

## 5. Qual o Padrão Arquitetural Mais Utilizado no Mercado Java?

**O padrão de API RESTful com Spring Boot é o padrão dominante no ecossistema Java comercial.**

### Justificativas Técnicas:

1. **Adoção do Spring Boot e Microservices:** O ecossistema Java evoluiu do modelo de servidores de aplicação pesados (JBoss, WebLogic) para microserviços leves empacotados em JARs executáveis contendo Tomcat embutido.
2. **Integração com Clientes Modernos:** A camada visual em Java (JSF, JSP) foi amplamente substituída por frameworks de mercado como React, Angular e Vue no web, ou Kotlin/Swift no mobile.
3. **Escalabilidade Cloud Native e GraalVM:** APIs Spring Boot Stateless em formato de imagem nativa (compiladas via GraalVM) possuem tempos de inicialização (*startup time*) na casa dos milissegundos e menor consumo de memória RAM, sendo ideais para ambientes serverless ou clusters Kubernetes.

---

## 6. Como Testar esta API Java?

No ecossistema Java, o ecossistema de testes é composto por **JUnit 5**, **Mockito**, **Spring Boot Test** e **RestAssured / MockMvc**.

### A) Teste Unitário (Service Layer) com JUnit 5 e Mockito

Testa a regra de negócio isolando o banco de dados via Mocks:

```java
@ExtendWith(MockitoExtension.class)
class ProdutoServiceTest {

    @Mock
    private ProdutoRepository repository;

    @InjectMocks
    private ProdutoService service;

    @Test
    void deveSalvarProdutoComSucesso() {
        ProdutoRequestDTO dto = new ProdutoRequestDTO("Notebook", new BigDecimal("3500.00"), 1L);
        Produto produtoSalvo = new Produto(1L, "Notebook", new BigDecimal("3500.00"));

        Mockito.when(repository.save(Mockito.any(Produto.class))).thenReturn(produtoSalvo);

        ProdutoResponseDTO resultado = service.salvar(dto);

        Assertions.assertNotNull(resultado.id());
        Assertions.assertEquals("Notebook", resultado.nome());
        Mockito.verify(repository, Mockito.times(1)).save(Mockito.any());
    }
}
```

### B) Teste de Integração / End-to-End (E2E) do Controller com `MockMvc`

Valida as rotas HTTP, conversão JSON, validação `@Valid` e os status codes sem precisar subir um servidor real:

```java
@SpringBootTest
@AutoConfigureMockMvc
class ProdutoRestControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void deveCriarProdutoERetornarStatus201() throws Exception {
        ProdutoRequestDTO dto = new ProdutoRequestDTO("Teclado Java", new BigDecimal("150.00"), 1L);

        mockMvc.perform(post("/api/v1/produtos")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(dto)))
                .andExpect(status().isCreated())
                .andExpect(header().exists("Location"))
                .andExpect(jsonPath("$.nome").value("Teclado Java"));
    }
}
```

---

## 7. Como Documentar esta API Java?

A documentação automática de APIs Spring Boot é realizada utilizando a biblioteca **Springdoc OpenAPI** (Swagger 3).

### A) Configuração via Dependência Maven:

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.5.0</version>
</dependency>
```

### B) Anotações do Swagger no Controller Java:

```java
@Tag(name = "Produtos", description = "Endpoints para gerenciamento do catálogo de produtos")
@RestController
@RequestMapping("/api/v1/produtos")
public class ProdutoRestController {

    @Operation(summary = "Busca produto por ID", description = "Retorna os detalhes do produto cadastrado")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Produto encontrado com sucesso"),
        @ApiResponse(responseCode = "404", description = "Produto não localizado no banco de dados")
    })
    @GetMapping("/{id}")
    public ResponseEntity<ProdutoResponseDTO> buscarPorId(@PathVariable Long id) {
        // Implementação...
        return ResponseEntity.ok().build();
    }
}
```

Ao executar a aplicação Spring Boot, a documentação e o ambiente de testes interativo ficam disponíveis na URL: `http://localhost:8080/swagger-ui.html`.

---

## 8. Como Consumir esta API no Front-end?

O consumo do backend Java é feito via requisições HTTP assíncronas assinadas com token JWT no cabeçalho da requisição.

### A) Exemplo de Consumo no Front-end Web (JavaScript / Axios):

```javascript
import axios from 'axios';

const api = axios.create({
  baseURL: 'http://localhost:8080/api/v1',
  timeout: 5000,
});

// Interceptor para injeção automática do Token JWT gerado pelo Spring Security
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('jwt_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

async function cadastrarProduto(novoProduto) {
  try {
    const response = await api.post('/produtos', novoProduto);
    console.log('Produto criado com ID:', response.data.id);
  } catch (error) {
    if (error.response && error.response.status === 400) {
      console.error('Erros de validação do Spring Validation:', error.response.data);
    } else {
      console.error('Erro na requisição para a API Java:', error.message);
    }
  }
}
```

### B) Exemplo de Consumo em outro serviço Java (HttpClient nativo do Java 11+):

```java
HttpClient client = HttpClient.newHttpClient();

HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("http://localhost:8080/api/v1/produtos"))
        .header("Content-Type", "application/json")
        .header("Authorization", "Bearer " + jwtToken)
        .GET()
        .build();

HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println("Status: " + response.statusCode());
System.out.println("Body JSON: " + response.body());
```

---

## Referências Bibliográficas

1. **WALLS, Craig.** *Spring in Action*. 6th ed. Manning Publications, 2022. *(Referência principal para Spring Boot, Spring MVC, Spring Data JPA e Spring Security)*.
2. **FIELDING, Roy Thomas.** *Architectural Styles and the Design of Network-based Software Architectures*. Tese de Doutorado – University of California, Irvine, 2000.
3. **MARTIN, Robert C.** *Clean Architecture: A Craftsman's Guide to Software Structure and Design*. Prentice Hall, 2017.
4. **SPRING PROJECT.** *Spring Framework Documentation / Spring Boot Reference Documentation*. VMware Tanzu, 2026. Disponível em: <https://spring.io/projects/spring-boot>.
5. **BAELDUNG.** *REST with Spring Tutorial Series*. Disponível em: <https://www.baeldung.com/rest-with-spring-series>.
6. **IETF (Internet Engineering Task Force).** *RFC 7519: JSON Web Token (JWT)*. Disponível em: <https://datatracker.ietf.org/doc/html/rfc7519>.
7. **SPRINGDOC INITIATIVE.** *OpenAPI 3 Library for Spring Boot*. Disponível em: <https://springdoc.org/>.