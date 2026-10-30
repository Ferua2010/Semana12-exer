# Conceitos de Arquitetura e Boas Práticas em PHP

## 1. Colapso do código espaguete
Manter SQL, validação e HTML no mesmo arquivo cria acoplamento total entre camadas: uma mudança visual pode quebrar regras de negócio, e uma alteração de banco pode afetar a interface. Isso dificulta testes, reuso, revisão de código e trabalho em equipe.

Em escala corporativa, o problema se multiplica: várias pessoas editam os mesmos arquivos, bugs se propagam entre apresentação, lógica e persistência, e não há separação clara de responsabilidades. A manutenção passa a exigir compreender todo o arquivo para alterar qualquer parte.

## 2. A tríade MVC

| Componente | Função específica | Responsabilidade principal |
| :--- | :--- | :--- |
| **Model** | Representa dados e regras do domínio; interage com o banco | Regras de negócio e persistência |
| **View** | Apresenta informações ao usuário; contém HTML e exibição de dados | Interface e saída visual |
| **Controller** | Recebe a requisição, interpreta a ação e coordena Model e View | Fluxo da aplicação |

As regras de negócio devem ficar no **Model** — por exemplo, calcular desconto, validar estoque ou aplicar uma regra fiscal. O **Controller** apenas orquestra: recebe dados, chama o Model e escolhe a View.

## 3. Composer: `composer.json` vs. `composer.lock`
*   **`composer.json`**: declara as dependências do projeto e faixas de versão aceitas, além da configuração de autoload. É o arquivo de *intenção* do projeto.
*   **`composer.lock`**: registra as versões exatas instaladas, incluindo dependências transitivas. Ele garante que todos os ambientes instalem o mesmo conjunto de pacotes.

A pasta `vendor/` nunca deve ir ao Git porque contém código de terceiros que pode ser regenerado com `composer install` a partir do `composer.lock`. Versioná-la inflaria o repositório, geraria conflitos e poderia expor arquivos desnecessários.

## 4. PSR-4 e autoload
O PSR-4 mapeia um prefixo de namespace para um diretório-base; depois, cada `\` do namespace torna-se um separador de diretório e o nome da classe vira o nome do arquivo `.php`.

Com uma configuração como:
```json
{
  "autoload": {
    "psr-4": {
      "App\\": "src/"
    }
  }
}
```

A classe `App\Controllers\ProdutoController` é resolvida assim:
```text
App\Controllers\ProdutoController → src/Controllers/ProdutoController.php
```
O autoloader remove o prefixo `App\`, converte `Controllers` em pasta e procura `ProdutoController.php`.

## 5. Front Controller e segurança
Centralizar as requisições em `public/index.php` cria um único ponto de entrada. Assim, arquivos internos como `src/`, configurações, classes e conexões ficam fora da raiz pública e não podem ser acessados diretamente pela URL.

Isso reduz riscos como execução direta de scripts internos, exposição de arquivos sensíveis e acesso indevido a código-fonte. O servidor web publica apenas a pasta `public/`; todo o restante permanece protegido pelo Front Controller.

## 6. Roteador HTTP
O Roteador compara dois elementos: o método HTTP e a URI.

```text
GET  /produtos → método listarProdutos()
POST /produtos → método criarProduto()
```

Embora a rota seja a mesma, o verbo muda a semântica: `GET` normalmente consulta/lista recursos, enquanto `POST` cria ou processa dados enviados pelo cliente. O Laravel usa exatamente essa distinção em arquivos como `routes/web.php`, definindo rotas por método e caminho.

## 7. `extract()` na View
No método `renderizar()`, o Controller normalmente recebe um array associativo:

```php
$this->renderizar('produtos/listar', [
    'titulo' => 'Lista de produtos',
    'produtos' => $produtos
]);
```

A função `extract()` transforma cada chave do array em uma variável no escopo local:

```php
$titulo = 'Lista de produtos';
$produtos = [...];
```

Assim, a View pode usar diretamente `<?= htmlspecialchars($titulo) ?>` e iterar sobre `$produtos`, sem manipular o array original.

## 8. Proteção contra XSS
`htmlspecialchars()` converte caracteres perigosos de HTML — como `<`, `>`, `"` e `'` — em entidades seguras. Isso impede que dados vindos do usuário sejam interpretados como código HTML ou JavaScript pelo navegador.

Sem essa conversão, um campo como:
```html
<script>alert('XSS')</script>
```
poderia ser executado na página. Portanto, toda saída dinâmica em tabelas, formulários e atributos HTML deve ser escapada:

```php
<?= htmlspecialchars($produto['nome'], ENT_QUOTES, 'UTF-8') ?>
```

## 9. Responsabilidade única no Controller
Um Controller não deve conter SQL concatenado nem criar conexões com `new PDO()` porque isso mistura controle de fluxo, persistência e segurança em uma única classe.

Além de violar o princípio da responsabilidade única, essa prática facilita SQL Injection: valores vindos do usuário nunca devem ser concatenados em comandos SQL. O correto é delegar acesso a dados ao Model ou a um repositório, usando *prepared statements* e parâmetros vinculados.

## 10. Conceitos reaproveitados no Laravel 11
O MVC artesanal prepara diretamente para o Laravel 11, que mantém os mesmos fundamentos organizacionais.

| Conceito artesanal | Equivalente no Laravel 11 |
| :--- | :--- |
| Controller que recebe a requisição e chama o Model | Controllers em `app/Http/Controllers` |
| Model com regras de negócio e acesso a dados | Models Eloquent |
| View responsável apenas pela apresentação | Blade Views em `resources/views` |
| Roteador que diferencia GET e POST | `Route::get()` e `Route::post()` em `routes/web.php` |
| Front Controller único | `public/index.php` |
| Autoload PSR-4 via Composer | Autoload configurado em `composer.json` |
| Escape de saída para evitar XSS | `{{ $variavel }}` no Blade, que escapa automaticamente |
| Injeção de dados na View | `return view('produtos.listar', ['produtos' => $produtos]);` |

A principal mudança é que o Laravel automatiza boa parte do que você implementou manualmente: roteamento, autoload, injeção de dependências, validação, ORM, templates e proteções de segurança.