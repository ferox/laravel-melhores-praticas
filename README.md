# Desenvolvendo as Melhores Práticas com o Laravel

Esta biblioteca é uma tradução para o Português do Brasil do:

[Espanhol](https://github.com/alexeymezenin/laravel-best-practices/blob/master/spanish.md) (escrito por [César Escudero](https://github.com/cedaesca))

A biblioteca reúne as melhores práticas para codificação de aplicativos web com o framework Laravel e não é uma adaptação dos cinco princípios do S.O.L.I.D `(Single Responsiblity Principle, Open-Closed Principle, Liskov Substitution Principle, Interface Segregation Principle e o Dependency Inversion Principle)`, muito menos dos atuais padrões de projetos. Aqui você encontrará as melhores práticas de codificação que geralmente são ignoradas em projetos reais.

## Índice

[Princípio da responsabilidade única](#princípio-da-responsabilidade-única)

[Modelos gordos, Controladoras magras](#modelos-gordos-controladoras-magras)

[Validação](#validação)

[A lógica de negócios deve ficar em classe de serviço](#a-lógica-de-negócios-deve-ficar-em-classe-de-serviço)

[Não escreva código duplicado (do princípio de Don't Repeat Yourself)](#não-escreva-código-duplicado-do-princípio-de-dont-repeat-yourself)

[Priorize o uso do Eloquent ORM (o ActiveRecord do Framework). Priorize também o uso das Collections ao invés dos arrays](#priorize-o-uso-do-eloquent-orm-o-activerecord-do-framework-priorize-também-o-uso-das-collections-ao-invés-dos-arrays-vetores)

[Atribuição em massa](#atribuição-em-massa)

[Não execute queries diretamente nos templates do Blade, faça o uso de carregamento prematuro (eager loading). Esse é um problema do tipo N + 1](#não-execute-queries-diretamente-nos-templates-do-blade-faça-o-uso-de-carregamento-prematuro-eager-loading-esse-é-um-problema-do-tipo-n--1)

[Comente seu código, mas faça o uso de métodos (funções) e de nomes de variáveis descritivos](#comente-seu-código-mas-faça-o-uso-de-métodosfunções-e-de-nomes-de-variáveis-descritivos)

[Não coloque código de JavaScript e de CSS nos templates do Blade. Jamais coloque código HTML em classes PHP](#não-coloque-código-de-javascript-e-de-css-nos-templates-do-blade-jamais-coloque-código-html-em-classes-php)

[Faça uso de arquivos de configuração e de linguagens ao invés de mensagens de textos no código](#faça-uso-de-arquivos-de-configuração-e-de-linguagens-ao-invés-de-mensagens-de-textos-no-código)

[Use somente ferramentas padronizadas desenvolvidas pela comunidade Laravel](#use-somente-ferramentas-padronizadas-desenvolvidas-pela-comunidade-laravel)

[Siga a convenção de nomes do PHP e do Laravel](#siga-a-convenção-de-nomes-do-php-e-do-laravel)

[Use sintaxes curtas e legíveis sempre que possível](#use-sintaxes-curtas-e-legíveis-sempre-que-possível)

[Use contêineres IoC (inversão de controle) ou Facades (interfaces estáticas) no lugar de Classes](#use-contêineres-ioc-inversão-de-controle-ou-facades-interfaces-estáticas-no-lugar-de-classes)

[Não extraia informações diretamente do arquivo .env](#não-extraia-informações-diretamente-do-arquivo-env)

[Salve datas em formatos padronizados. Use os modificadores de atributos das tabelas chamados de "accessors" e "mutators" para formatar datas](#salve-datas-em-formatos-padronizados-use-os-modificadores-de-atributos-das-tabelas-chamados-de-accessors-e-mutators-para-formatar-as-datas)

[Outras boas práticas](#outras-boas-práticas)

### **Princípio da responsabilidade única**

Classes e métodos devem ter apenas um propósito.

Péssimo:

```php
public function getNomeCompletoAttribute()
{
    if (auth()->usuario() && auth()->usuario()->hasRole('editor') && auth()->usuario()->isVerified()) {
        return 'Sr. ' . $this->primeiro_nome . ' ' . $this->nome_do_meio . ' ' . $this->sobrenome;
    } else {
        return $this->primeiro_nome[0] . '. ' . $this->sobrenome;
    }
}
```

Ótimo:

```php
public function getNomeCompletoAttribute()
{
    return $this->isVerifiedEditor() ? $this->getNomeCompletoLong() : $this->getNomeCompletoShort();
}

public function isVerifiedEditor()
{
    return auth()->usuario() && auth()->usuario()->hasRole('editor') && auth()->usuario()->isVerified();
}

public function getNomeCompletoLong()
{
    return 'Sr. ' . $this->primeiro_nome . ' ' . $this->nome_do_meio . ' ' . $this->sobrenome;
}

public function getNomeCompletoShort()
{
    return $this->primeiro_nome[0] . '. ' . $this->sobrenome;
}
```

[🔝 Voltar ao índice](#)

### **Modelos gordos, Controladoras magras**

Coloque toda a lógica relacionada ao banco de dados em modelos do próprio Eloquent ou em uma Classe do tipo Repositório se você estiver usando o Query Builder ou consultas puras em SQL (Raw Expressions).

Péssimo:

```php
public function index()
{
    $artistas = Artistas::verificado()
        ->with(['obras_encomendadas' => function ($q) {
            $q->where('created_at', '>', Carbon::today()->subWeek());
        }])
        ->get();

    return view('index', ['artistas' => $artistas]);
}
```

Ótimo:

```php
public function index()
{
    return view('index', ['artistas' => $this->artista->getWithObrasEncomendadas()]);
}

class Client extends Model
{
    public function getWithObrasEncomendadas()
    {
        return $this->verificado()
            ->with(['obras_encomendadas' => function ($q) {
                $q->where('created_at', '>', Carbon::today()->subWeek());
            }])
            ->get();
    }
}
```

[🔝 Voltar ao índice](#)

### **Validação**

Remova todas as validações das Classes Controladoras e coloque-as em Classes de Requisições (Request)

Péssimo:

```php
public function store(Request $request)
{
    $request->validate([
        'titulo'       => 'required|max:255',
        'descricao'    => 'required',
        'publicado_em' => 'nullable|date',
    ]);

    ....
}
```

Ótimo:

```php
public function store(PostRequest $request)
{    
    ....
}

class PublicacaoRequest extends Request
{
    public function rules()
    {
        return [
            'titulo'       => 'required|max:255',
            'descricao'    => 'required',
            'publicado_em' => 'nullable|date',
        ];
    }
}
```

[🔝 Voltar ao índice](#)

### **A lógica de negócios deve ficar em Classe de Serviço**

A classe Controladora deve ter apenas um propósito, portanto, mova a lógica de negócios dessa classe para uma nova classe de serviço.

Péssimo:

```php
public function store(Request $request)
{
    if ($request->hasFile('foto')) {
        $request->file('foto')->move(public_path('fotos') . 'temporario');
    }
    
    ....
}
```

Ótimo:

```php
public function store(Request $request)
{
    $this->publicacaoService->handleUploadedFoto($request->file('foto'));

    ....
}

class PublicacaoService
{
    public function handleUploadedFoto($foto)
    {
        if (!is_null($foto)) {
            $foto->move(public_path('fotos') . 'temporario');
        }
    }
}
```

[🔝 Voltar ao índice](#)

### **Não escreva código duplicado (do princípio de Don't Repeat Yourself)**

Reutilize seu código sempre que possível. A ideia da responsabilidade única ajuda você a evitar duplicação de código. Isso serve também para templates do Blade. Use também os "scopes" para filtrar consultas pelo Eloquent.

Péssimo:

```php
public function getPublicado()
{
    return $this->where('publicado', 1)->whereNotNull('deleted_at')->get();
}

public function getPublicacoes()
{
    return $this->whereHas('artista', function ($q) {
            $q->where('publicado', 1)->whereNotNull('deleted_at');
        })->get();
}
```

Ótimo:

```php
public function scopePublicado($q)
{
    return $q->where('publicado', 1)->whereNotNull('deleted_at');
}

public function getPublicado()
{
    return $this->publicado()->get();
}

public function getPublicacoes()
{
    return $this->whereHas('artista', function ($q) {
            $q->publicado();
        })->get();
}
```

[🔝 Voltar ao índice](#)

### **Priorize o uso do Eloquent ORM (o ActiveRecord do Framework). Priorize também o uso das Collections ao invés dos arrays (vetores)**

O Eloquent permite que você escreva código legível e de fácil manutenção. Ele também possui ótimas ferramentas como, o deletar de forma suave (soft delete), os eventos, os "scopes", entre outros. Leia a documentação oficial do Laravel e conheça todas essas ferramentas.

Péssimo:

```sql
SELECT *
FROM `publicacoes`
WHERE EXISTS (SELECT *
              FROM `editores`
              WHERE `publicacoes`.`editor_id` = `editores`.`id`
              AND EXISTS (SELECT *
                          FROM `perfis`
                          WHERE `perfis`.`editor_id` = `editores`.`id`) 
              AND `editores`.`deleted_at` IS NULL)
AND `verificado` = '1'
AND `ativo` = '1'
ORDER BY `created_at` DESC
```

Ótimo:

```php
Publicacao::has('editor.perfil')->verificado()->latest()->get();
```

[🔝 Voltar ao índice](#)

### **Atribuição em massa**

O método mais eficaz para lidar com ataques de atribuição em massa é passar apenas os campos que foram validados ao invés de passar todos os dados da solicitação (requisição).

Péssimo:

```php
class Usuario extends Authenticatable {
 protected $fillable = ['nome', 'email', 'senha'];
  ...
}

class UsuarioRequest extends Request {
 public function rules() {
   return [
       'nome'              => 'string|required',
       'email'             => 'email|required',
       'senha'             => 'string|required|min:6',
       'confirmacao_senha' => 'same:password',
   ];
 }
}

class UsuarioController extends Controller {
   public function store(UsuarioRequest $request) {
       $usuario = new Usuario();
       $usuario->fill($request->all());
       $usuario->save();
       
       return response()->json([
         'success' => true,
       ],201);
   }
```

Ótimo:

```php
class Usuario extends Authenticatable {
 protected $fillable = ['nome', 'email', 'senha'];
  ...
}

class UsuarioRequest extends Request {
 public function rules() {
   return [
       'nome'              => 'string|required',
       'email'             => 'email|required',
       'senha'             => 'string|required|min:6',
       'confirmacao_senha' => 'same:password',
   ];
 }
}

class UsuarioController extends Controller {
    public function store(UsuarioRequest $request) {
        $usuario = new Usuario();
        $usuario->fill($request->validated());
        $usuario->save();
        
        return response()->json([
            'success' => true,
            ],201);
    }
}
```

[🔝 Voltar ao índice](#)

### **Não execute queries diretamente nos templates do Blade, faça o uso de carregamento prematuro (eager loading). Esse é um problema do tipo N + 1**

Péssimo - Para 100 usuários, 101 consultas foram executadas:

```php
@foreach (Usuario::all() as $usuario)
    {{ $usuario->perfil->nome }}
@endforeach
```

Ótimo - Para 100 usuários, 2 consultas foram executadas:

```php
$usuarios = Usuario::with('perfil')->get();

...

@foreach ($usuarios as $usuario)
    {{ $usuario->perfil->nome }}
@endforeach
```

[🔝 Voltar ao índice](#)

### **Comente seu código, mas faça o uso de métodos (funções) e de nomes de variáveis descritivos**

Péssimo:

```php
if (count((array) $builder->getQuery()->joins) > 0)
```

Ótimo:

```php
// Determina se existe algum método de junção
if (count((array) $builder->getQuery()->joins) > 0)

// hasJoins() - Método descritivo que verifica a existência de alguma junção
if ($this->hasJoins())
```

[🔝 Voltar ao índice](#)

### **Não coloque código de JavaScript e de CSS nos templates do Blade. Jamais coloque código HTML em classes PHP**

Péssimo:

```js
let publicacao = `{{ json_encode($publicacao) }}`;
```

```php
class Publicacao extends Model
{
    
    ...
    
    public function laratablesRowData()
    {
        return [
            'acoes' =>
                "<div class=\"btn-group\">
                    <button type=\"button\" id=\"btnDropDown_{$this->id}\" class=\"btn btn-primary btn-sm dropdown-toggle\" data-toggle=\"dropdown\" aria-haspopup=\"true\" aria-expanded=\"false\">
                        <i class=\"fa fa-cogs fa-lg fa-fw fa-inverse\"></i> Opções&nbsp;
                        <span class=\"fa fa-caret-down fa-inverse fa-fw\"></span>
                    </button>
                    <ul class=\"dropdown-menu myDropDown\" aria-labelledby=\"btnDropDown_{$this->id}\">
                        <li data-toggle=\"tooltip\" data-placement=\"right\" title=\"Exibir {$this->titulo}\">
                            <a href=\"javascript:modalGlobalOpen('".route('publicacao.show', ["id" => $this->id])."')\">
                                <i class=\"fa fa-eye fa-lg fa-fw\"></i> Visualizar
                            </a>
				        </li>
                        <li data-toggle=\"tooltip\" data-placement=\"right\" title=\"Editar {$this->titulo}\">
                            <a href=\"".route('publicacao.edit', ["id" => $this->id])."\">
                                <i class=\"fa fa-edit fa-lg fa-fw\"></i> Editar
                            </a>
                        </li>
                        <li data-toggle=\"tooltip\" data-placement=\"right\" title=\"Deletar {$this->titulo}\">
                           <a href=\"javascript:void(0)\" onclick=\"doDelete('{$this->id}')\">
                               <i class=\"fa fa-ban fa-lg fa-fw\"></i> Deletar
                           </a>
                        </li>
                    </ul>
                </div>"
        ];
      
    }
}
}
            
```

Ótimo:

```js
let publicacao = $('#publicacao').val();
```

```php
<input id="publicacao" type="hidden" value='@json($publicacao)'>

Ou

<button class="btn btn-primary" data-publicacao='@json($publicacao)'>{{ $publicacao->titulo }}<button>
```

A maneira ideal para lidar com essa questão é usar algum pacote especializado para transferir informações do PHP para JavaScript.

[🔝 Voltar ao índice](#)

### **Faça uso de arquivos de configuração e de linguagens ao invés de mensagens de textos no código**

Péssimo:

```php
public function isNormal()
{
    return $artigo->tipo === 'normal';
}

return back()->with('mensagem', 'Seu artigo foi publicado com sucesso!');
```

Ótimo:

```php
public function isNormal()
{
    return $artigo->tipo === Artigo::TIPO_NORMAL;
}

return back()->with('mensagem', __('app.artigo_publicado'));
```

[🔝 Voltar ao índice](#)

### **Use somente ferramentas padronizadas desenvolvidas pela comunidade Laravel**

Priorize o uso das classes e métodos do próprios framework e de pacotes fornecidos pela comunidade em vez de usar pacotes ou ferramentas de terceiros, pois qualquer novo desenvolvedor que trabalhará em seu aplicativo no futuro precisará aprender como usar essas ferramentas. Além disso, as chances de receber ajuda da comunidade são menores quando você usa ferramentas ou pacotes de terceiros. Não faça seu cliente pagar por isso.

Tarefa | Ferramenta padrão | Pacotes de terceiros
------------ | ------------- | -------------
Autorização | Policies | Entrust, Sentinel, entre outros
Compilação de assets | Laravel Mix | Grunt, Gulp, entre outros
Ambiente de desenvolvimento | Homestead e Laradock | Docker
Deployment | Laravel Forge e Laravel Vapor | Deployer, Anistrano, Storm.dev, entre outros
Testes unitários | PHPUnit, PestPHP e Mockery | PHPSpec
Teste em navegador | Laravel Dusk | Codeception e Nerrvana
ORM - ActiveRecord | Eloquent | Raw SQL e Doctrine
Templates | Blade e Vue.js | Twig
Trabalhando com dados | Laravel Collections | Arrays (Vetores)
Validação de formulários | Classes Request e Factory | pacotes de terceiros, validação pela controladora, entre outros
Autenticação | Laravel | pacotes de terceiros, entre outros
Autenticação API | Laravel Passport | JWT e OAuth2 
Criação API | Laravel e Laravel 7 Sanctum | Dingo API, entre outros
Trabalhando com estruturas de Bancos de Dados | Migrações | Laravel SQL Migrations de Peter Matseykanets, em SQL puro, entre outros
Localização | Laravel | pacotes de terceiros
Interface em tempo real | Laravel Echo e Pusher | pacotes de terceiros e trabalhar com WebSockets diretamente
Gerar dados de teste | Classes Seeders, Modeles Factories e o Faker | Criar testes manualmente
Agendador de tarefas | Laravel Task Scheduler | Scripts e pacotes de terceiros
Banco de Dados | Laravel Drivers para MySQL, PostgreSQL, SQLite, SQL Server | Laravel MongoDB de Jens Segers e o Laravel Oracle OCI8 

[🔝 Voltar ao índice](#)

### **Siga a convenção de nomes do PHP e do Laravel**

 Confira o Guia para estilo para codificação [PSR-12](http://www.php-fig.org/psr/psr-12/).
 
Siga também a convenção de nomes aceita pelo a comunidade Laravel:

O que? | Como? | Ótimo | Péssimo
------------ | ------------- | ------------- | -------------
Controladora | singular | ArtigoController | ~~ArtigosController~~
Rota | plural | artigos/1 | ~~artigo/1~~
Nomes da rota | snake_case com notação de ponto | usuarios.show_active | ~~usuarios.show-active, show-active-usuarios~~
Modelo | singular | Usuario | ~~Usuarios~~
Relacionamento de Tabelas hasOne ou belongsTo | singular | artigoComnetario | ~~artigoComentarios, artigo_comentario~~
Qualquer outro tipo de relacionamento | plural | artigoComentarios | ~~artigoComentario, artigo_comentarios~~
Tabela do Banco de Dados | plural | artigo_comentarios | ~~artigo_comentario, artigoComentarios~~
Tabela Pivô | Nomes de Modelos no singular e em ordem alfabética | artigo_usuario | ~~usuario_artigo, artigos_usuarios~~
Coluna das tabela | snake_case sem o nome da tabela (Modelo) | descricao_titulo | ~~DescricaoTitulo; artigo_descricao_titulo~~
Propriedade do Modelo | snake_case | $model->created_at | ~~$model->createdAt~~
Chave Estrangeira | Nome do modelo em singular com sufixo _id | usuario_id | ~~UsuarioId, id_usuario, usuarios_id~~
Chave primária | - | id | ~~id_usuario~~
Migração | - | 2020_01_04_174100_create_usuarios_table | ~~2020_01_04_174100_usuarios~~
Método | camelCase | getNomeAluno | ~~get_nome_aluno~~
Método na controladora de recursos | [Documentação do Laravel - Resource Controllers](https://laravel.com/docs/master/controllers#resource-controllers) | store | ~~storeArtigo~~
Método na classe de Teste | camelCase | testeUsuarioSemFuncaoAcessaSolicitarServico | ~~teste_usuario_sem_funcao_acessa_solicitar_servico~~
Variável | camelCase | $alunoMatriculado | ~~$aluno_matriculado~~
Coleção | descritivo e no plural | $usuariosAtivos = Usuario::ativo()->get() | ~~$ativo, $dado~~
Objeto | descritivo e no singular | $usuarioAtivo = Usuario::ativo()->first() | ~~$usuarios, $obj~~
Índice de arquivo de configuração e de linguagem | snake_case | artigos_permitidos | ~~ArtigosPermitidos; artigos-permitidos~~
Visualização | kebab-case | mostra-filtros.blade.php | ~~mostraFiltros.blade.php, mostra_filtros.blade.php~~
Configuração | snake_case | sso_wso2.php | ~~ssoWso2.php, sso-wso2.php~~
Contrato (interface) | adjetivo ou substantivo | Esquematico | ~~EsquematizarInterface, IEsquematizar~~
Trait | adjetivo | Negador | ~~NegacaoTrait, NegarTrait~~

[🔝 Voltar ao índice](#)

### **Use sintaxes curtas e legíveis sempre que possível**

Péssimo:

```php
$request->session()->get('usuario');
$request->input('nome');
```

Ótimo:

```php
session('usuario');
$request->nome;
```

Outros exemplos

Sintaxe comum | Sintaxe curta e legível
------------ | -------------
`Session::get('usuario')` | `session('usuario')`
`$request->session()->get('usuario')` | `session('usuario')`
`Session::put('usuario', $dados)` | `session(['usuario' => $dados])`
`$request->input('nome'), Request::get('nome')` | `$request->nome, request('nome')`
`return Redirect::back()` | `return back()`
`is_null($object->relation) ? null : $object->relation->id` | `optional($object->relation)->id`
`return view('index')->with('titulo', $titulo)->with('aluno', $aluno)` | `return view('index', compact('titulo', 'aluno'))`
`$request->has('foto_perfil') ? $request->foto_perfil : 'default';` | `$request->get('foto_perfil', 'default')`
`Carbon::now(), Carbon::today()` | `now(), today()`
`App::make('Curso')` | `app('Curso')`
`->where('aprovado', '=', 1)` | `->where('aprovado', 1)`
`->orderBy('created_at', 'desc')` | `->latest()`
`->orderBy('idade', 'desc')` | `->latest('idade')`
`->orderBy('created_at', 'asc')` | `->oldest()`
`->select('id', 'nome')->get()` | `->get(['id', 'nome'])`
`->first()->nome` | `->value('nome')`

[🔝 Voltar ao índice](#)

### **Use contêineres IoC (inversão de controle) ou Facades (interfaces estáticas) no lugar de Classes**

A sintaxe 'new Class' cria acoplamentos estreitos complicando os testes. Use contêineres IoC ou as Facades.

Péssimo:

```php
$aluno = new Aluno;
$aluno->create($request->validated());
```

Ótimo:

```php
public function __construct(Aluno $Aluno)
{
    $this->aluno = $aluno;
    ...
}

...

$this->aluno->create($request->validated());
```

[🔝 Voltar ao índice](#)

### **Não extraia informações diretamente do arquivo `.env`**

Em vez disso, coloque as informações em um arquivo de configuração e use o helper `config ()` para obter as informações em sua aplicação.

Péssimo:

```php
$chaveApi = env('API_KEY');
```

Ótimo:

```php
// config/api.php
'chave' => env('API_KEY'),

// Fazendo uso da chave da api pelo helper config
$chaveApi = config('api.chave');
```

[🔝 Voltar ao índice](#)

### **Salve datas em formatos padronizados. Use os modificadores de atributos das tabelas chamados de "accessors" e "mutators" para formatar as datas**

Péssimo:

```php
{{ Carbon::createFromFormat('Y-d-m H-i', $objeto->ordered_at)->toDateString() }}
{{ Carbon::createFromFormat('Y-d-m H-i', $objeto->ordered_at)->format('m-d') }}
```

Ótimo:

```php
// Modelo
protected $datas = ['ordered_at', 'created_at', 'updated_at'];
public function getAtributoData($datas)
{
    return $datas->format('m-d');
}

// Visualização
{{ $objeto->ordered_at->toDateString() }}
{{ $objeto->ordered_at->ano_publicacao }}
```

[🔝 Voltar ao índice](#)

### **Outras boas práticas**

Não coloque nenhum tipo de lógica nos arquivos de rotas.

Minimize o uso de códigos PHP puros (vanilla) nos modelos em Blade (template engine).

[🔝 Voltar ao índice](#)
