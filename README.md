# TryHackMe - XSS Vulnerability Research Lab

Repositório criado para documentar a resolução e análise prática do laboratório **XSS** da plataforma TryHackMe. O objetivo da room foi estudar a identificação, funcionamento e mitigação de vulnerabilidades de **Cross-Site Scripting (XSS)**, passando por **Reflected XSS, Stored XSS e DOM-Based XSS**, além de analisar diferentes implementações vulneráveis em aplicações web.

> **Observação:** o material utilizado para este relatório registra o conteúdo e as soluções conceituais da room, incluindo os exemplos de código em PHP, Node.js, Flask e ASP.NET/C#. Ele não contém os payloads, URLs de uma máquina-alvo específica, flags ou evidências de exploração prática das Tasks 4, 5, 7, 8 e 9. Portanto, essas informações não foram inventadas neste relatório.

---

## Stack / Tecnologias Analisadas

- **Plataforma:** TryHackMe
- **Room:** XSS (`axss`)
- **Dificuldade:** Easy
- **Tempo estimado:** 120 minutos
- **Vulnerabilidade principal:** Cross-Site Scripting (XSS)
- **Tipos abordados:** Reflected XSS, Stored XSS e DOM-Based XSS
- **Linguagens analisadas:** PHP, JavaScript/Node.js, Python/Flask e C#/ASP.NET
- **Conceitos de segurança:** validação, sanitização, escaping, encoding contextual e defesa em profundidade
- **Metodologia de análise:** identificação do fluxo de entrada controlada pelo usuário até a renderização no navegador

---

# Resolução e Análise Prática — Task por Task

## Task 1 — Introduction

A primeira etapa apresenta o conceito geral de **Cross-Site Scripting (XSS)** e estabelece a base para entender como entradas controladas pelo usuário podem acabar sendo interpretadas pelo navegador de forma não intencional.

O problema central pode ser representado pelo seguinte fluxo:

```text
Entrada controlada pelo usuário
            ↓
Processamento inseguro
            ↓
Renderização da aplicação
            ↓
Interpretação pelo navegador
            ↓
Execução de conteúdo não confiável
```

A principal premissa para a análise é que dados fornecidos pelo usuário devem ser considerados **não confiáveis** até que sejam tratados adequadamente.

---

## Task 2 — Terminology and Types

Nesta etapa foram apresentados os principais tipos de XSS.

### Reflected XSS

No **Reflected XSS**, a entrada enviada pelo usuário é refletida pela aplicação na resposta HTTP sem o tratamento adequado.

O fluxo conceitual é:

```text
Atacante
   |
   | entrada
   v
Aplicação Web
   |
   | resposta contendo a entrada
   v
Navegador
```

Diferentemente do Stored XSS, o conteúdo não precisa ser persistido no servidor.

---

### Stored XSS

No **Stored XSS**, também chamado de **Persistent XSS**, o conteúdo controlado pelo usuário é armazenado pela aplicação e posteriormente apresentado a outros usuários.

Exemplos citados pela room:

- comentários;
- posts;
- avaliações;
- informações de perfil;
- dados armazenados em banco.

Fluxo:

```text
Atacante
   ↓
Aplicação
   ↓
Banco de dados
   ↓
Página Web
   ↓
Navegador de outro usuário
```

Esse modelo é especialmente relevante porque o conteúdo malicioso pode permanecer armazenado e ser servido posteriormente para usuários que acessarem o recurso afetado.

---

### DOM-Based XSS

A room também apresenta o **DOM-Based XSS**, no qual o problema está relacionado ao processamento realizado pelo JavaScript no lado do cliente.

A análise deve considerar:

```text
Fonte controlada pelo usuário
            ↓
JavaScript da aplicação
            ↓
DOM / Sink
            ↓
Interpretação pelo navegador
```

Nesse cenário, a investigação deve procurar fontes controláveis pelo usuário e operações JavaScript que possam inserir ou interpretar esse conteúdo de forma insegura.

---

# Task 3 — Causes and Implications

A causa fundamental discutida na room é o tratamento inadequado de dados fornecidos pelo usuário.

Entre as práticas de proteção apresentadas estão:

- **validar e sanitizar a entrada**;
- **utilizar output escaping**;
- **aplicar encoding específico para o contexto**;
- **adotar defesa em profundidade**.

A room reforça que não é suficiente confiar exclusivamente em controles executados no lado do cliente.

---

# Task 4 — Reflected XSS

Nesta etapa é apresentado o conceito de **Reflected XSS**.

O cenário ocorre quando uma entrada controlada pelo usuário é recebida pela aplicação e posteriormente inserida na resposta sem o tratamento necessário.

Uma representação simplificada:

```text
Request
   |
   | parâmetro controlado
   v
Web Application
   |
   | conteúdo refletido
   v
HTTP Response
   |
   v
Browser
```

O ponto de análise é identificar onde o dado recebido pelo servidor é incorporado à resposta.

### Ponto de atenção

Uma aplicação segura deve tratar corretamente o dado antes de inseri-lo em HTML, JavaScript, URL ou outro contexto.

---

# Task 5 — Vulnerable Web Application 1

A room passa então para uma análise de uma aplicação vulnerável.

O objetivo é compreender como uma aplicação pode aceitar conteúdo fornecido pelo usuário e posteriormente inseri-lo em uma página sem escaping adequado.

Essa lógica é aprofundada na etapa seguinte através de exemplos de **Stored XSS**.

---

# Task 6 — Stored XSS

Esta foi uma das etapas mais importantes do laboratório.

O Stored XSS ocorre quando a aplicação:

1. recebe conteúdo controlado pelo usuário;
2. armazena esse conteúdo;
3. recupera posteriormente o conteúdo;
4. insere o conteúdo na página sem tratamento adequado.

---

## Exemplo em PHP

A room apresenta o seguinte padrão vulnerável:

```php
// Storing user comment
$comment = $_POST['comment'];
mysqli_query($conn, "INSERT INTO comments (comment) VALUES ('$comment')");

// Displaying user comment
$result = mysqli_query($conn, "SELECT comment FROM comments");

while ($row = mysqli_fetch_assoc($result)) {
    echo $row['comment'];
}
```

### Análise

O problema relevante para XSS está na renderização:

```php
echo $row['comment'];
```

O conteúdo armazenado no banco é inserido diretamente na resposta.

Assim, uma entrada originalmente fornecida como dado pode acabar sendo interpretada pelo navegador como parte do HTML.

A room também aponta que o código possui um problema de SQL Injection, mas deixa claro que esse problema está fora do escopo da análise de XSS naquele exercício.

---

## Correção apresentada em PHP

A solução utiliza:

```php
$sanitizedComment = htmlspecialchars($row['comment']);
echo $sanitizedComment;
```

A função:

```php
htmlspecialchars()
```

converte caracteres especiais para entidades HTML.

O objetivo é impedir que a entrada armazenada seja interpretada como marcação HTML executável.

---

# Stored XSS em Node.js

A room apresenta também uma aplicação Node.js:

```javascript
app.get('/comments', (req, res) => {
  let html = '<ul>';

  for (const comment of comments) {
    html += `<li>${comment}</li>`;
  }

  html += '</ul>';
  res.send(html);
});
```

### Vulnerabilidade

O conteúdo de:

```javascript
comment
```

é inserido diretamente no HTML:

```javascript
html += `<li>${comment}</li>`;
```

Não existe sanitização antes da montagem da resposta.

---

## Correção em Node.js

A solução apresentada utiliza o pacote:

```javascript
sanitize-html
```

Exemplo:

```javascript
const sanitizeHtml = require('sanitize-html');

app.get('/comments', (req, res) => {
  let html = '<ul>';

  for (const comment of comments) {
    const sanitizedComment = sanitizeHtml(comment);
    html += `<li>${sanitizedComment}</li>`;
  }

  html += '</ul>';
  res.send(html);
});
```

A abordagem permite aplicar uma política de conteúdo, removendo elementos HTML considerados perigosos.

---

# Stored XSS em Python / Flask

Outro exemplo apresentado utiliza Flask e SQLAlchemy.

A aplicação recebe o comentário:

```python
comment_content = request.form['comment']
```

Depois persiste o valor:

```python
comment = Comment(content=comment_content)
db.session.add(comment)
db.session.commit()
```

Na renderização, o conteúdo é concatenado diretamente ao HTML:

```python
comments = Comment.query.all()

return render_template_string(
    ''.join(['<div>' + c.content + '</div>' for c in comments])
)
```

### Vulnerabilidade

O valor:

```python
c.content
```

é inserido diretamente na resposta HTML.

Isso cria o mesmo padrão observado nos exemplos anteriores:

```text
User Input
    ↓
Database
    ↓
HTML sem escaping
    ↓
Browser
```

---

## Correção em Flask

A room utiliza:

```python
from markupsafe import escape
```

E aplica:

```python
sanitized_comments = [escape(c.content) for c in comments]
```

O conteúdo passa a ser escapado antes de ser enviado ao navegador.

Exemplos apresentados na room:

```text
&  -> &amp;
<  -> &lt;
>  -> &gt;
'  -> &#39;
"  -> &quot;
```

---

# Stored XSS em ASP.NET / C#

O último exemplo de Stored XSS apresentado na Task utiliza ASP.NET/C#.

Código vulnerável:

```csharp
public void DisplayComments()
{
    var reader = new SqlCommand(
        "SELECT Comment FROM Comments",
        connection
    ).ExecuteReader();

    while (reader.Read())
    {
        Response.Write(reader["Comment"].ToString());
    }
}
```

O problema ocorre porque o conteúdo recuperado do banco é enviado diretamente para a resposta:

```csharp
Response.Write(reader["Comment"].ToString());
```

---

## Correção em ASP.NET

A solução apresentada utiliza:

```csharp
HttpUtility.HtmlEncode()
```

Exemplo:

```csharp
var comment = reader["Comment"].ToString();
var sanitizedComment = HttpUtility.HtmlEncode(comment);

Response.Write(sanitizedComment);
```

O método realiza o encoding do conteúdo antes da renderização.

---

# Task 7 — Vulnerable Web Application 2

A room possui uma segunda etapa dedicada à análise de uma aplicação web vulnerável.

O material fornecido registra a existência da Task, porém não contém os detalhes da exploração, payload utilizado ou resultados específicos dessa aplicação.

Por isso, esta seção não atribui achados que não estejam presentes nas evidências disponíveis.

---

# Task 8 — DOM-Based XSS

Nesta etapa é introduzido o **DOM-Based XSS**.

Diferentemente de um cenário em que o servidor necessariamente gera a resposta vulnerável, o problema pode ocorrer quando o JavaScript do navegador manipula dados controlados pelo usuário e os insere no DOM de forma insegura.

Para uma análise prática, o fluxo deve ser investigado como:

```text
Source
  ↓
JavaScript
  ↓
Processamento
  ↓
Sink
  ↓
DOM
```

A identificação exige descobrir tanto a origem dos dados quanto o ponto onde esses dados são utilizados.

---

# Task 9 — Context and Evasion

A etapa final de exploração aborda **contexto e evasão**.

Um dos principais conceitos aprendidos é que XSS não deve ser analisado somente pelo conteúdo da entrada, mas também pelo **contexto no qual essa entrada será utilizada**.

Possíveis contextos incluem:

```text
HTML
Atributo HTML
JavaScript
URL
DOM
```

Cada contexto possui requisitos diferentes de encoding e proteção.

Assim, durante uma análise de segurança, deve-se determinar:

```text
Onde o input entra?
        ↓
Como é processado?
        ↓
Em qual contexto é inserido?
        ↓
Existe escaping adequado?
        ↓
Existe sanitização?
```

---

# Task 10 — Conclusion

A conclusão da room consolida os principais conceitos relacionados à identificação e mitigação de XSS.

A análise dos exemplos mostrou que o mesmo padrão de vulnerabilidade pode existir em diferentes tecnologias:

| Tecnologia | Vulnerabilidade | Proteção apresentada |
|---|---|---|
| PHP | Stored XSS | `htmlspecialchars()` |
| Node.js | Stored XSS | `sanitize-html` |
| Flask | Stored XSS | `escape()` |
| ASP.NET/C# | Stored XSS | `HttpUtility.HtmlEncode()` |

---

# Análise Técnica dos Achados

## 1. Falta de Output Encoding

O padrão mais recorrente nos exemplos foi:

```text
Input do usuário
      ↓
Armazenamento
      ↓
Recuperação
      ↓
HTML
```

sem uma etapa adequada de encoding.

A correção é realizar o escaping apropriado antes da saída.

---

## 2. Sanitização inadequada

Quando a aplicação precisa aceitar determinado subconjunto de HTML, a room apresenta a sanitização como alternativa.

No exemplo Node.js:

```javascript
sanitizeHtml(comment)
```

A aplicação pode definir quais elementos são permitidos e remover conteúdo que não faça parte da política esperada.

---

## 3. Ausência de proteção contextual

A room destaca que o mecanismo utilizado deve considerar o contexto.

Por exemplo:

```text
HTML       → HTML encoding
JavaScript → JavaScript encoding
URL        → URL encoding
```

A proteção precisa ser escolhida de acordo com o local em que o dado será inserido.

---

# Impacto de Segurança

Os impactos possíveis apresentados no contexto da room incluem:

1. **Execução de JavaScript no navegador** no contexto da aplicação;
2. **Manipulação do conteúdo apresentado ao usuário**;
3. **Execução de ações em nome de usuários**, dependendo do contexto e das proteções existentes;
4. **Acesso a informações disponíveis para o contexto da página**;
5. **Comprometimento de sessões em determinadas condições**.

O impacto real depende da aplicação vulnerável, do contexto de execução e dos controles adicionais existentes.

---

# Metodologia de Análise XSS

A partir do conteúdo da room, uma metodologia prática pode ser estruturada da seguinte maneira:

### 1. Identificar fontes de entrada

```text
GET
POST
Formulários
Comentários
Perfil
Parâmetros
URL
JavaScript
```

### 2. Mapear o fluxo

```text
Input
  ↓
Processamento
  ↓
Armazenamento
  ↓
Renderização
  ↓
Browser
```

### 3. Identificar o contexto

```text
HTML
Attribute
JavaScript
URL
DOM
```

### 4. Procurar controles

```text
Validation
Sanitization
Output Encoding
Contextual Encoding
Server-side Controls
```

### 5. Validar em ambiente autorizado

A confirmação deve ser realizada somente dentro do escopo autorizado do laboratório ou teste de segurança.

### 6. Documentar

Registrar:

```text
Endpoint
Parâmetro
Tipo de XSS
Contexto
Evidência
Impacto
Correção
```

---

# Checklist para Pentest / Bug Bounty

```text
[ ] Mapear entradas controladas pelo usuário
[ ] Identificar parâmetros GET/POST
[ ] Verificar campos persistentes
[ ] Verificar Reflected XSS
[ ] Verificar Stored XSS
[ ] Identificar fontes DOM
[ ] Identificar sinks DOM
[ ] Determinar contexto de saída
[ ] Verificar output encoding
[ ] Verificar sanitização
[ ] Verificar validação server-side
[ ] Avaliar mecanismos adicionais de defesa
[ ] Registrar evidência
[ ] Classificar impacto
[ ] Recomendar correção
```

---

# Perguntas Respondidas da Room

### Qual função JavaScript foi utilizada para sanitizar o input?

```text
sanitizeHtml()
```

### Qual método foi utilizado em ASP.NET C#?

```text
HttpUtility.HtmlEncode()
```

---

# Principais Lições Técnicas

### 1. Input do usuário é não confiável

O fato de um dado ser armazenado em banco não o transforma em conteúdo confiável.

### 2. O ponto crítico pode estar na saída

Uma aplicação pode armazenar um valor sem executar nada naquele momento e ainda assim permanecer vulnerável quando o valor for posteriormente renderizado.

### 3. O contexto importa

HTML, JavaScript, URL e DOM possuem diferentes requisitos de proteção.

### 4. Validação e escaping têm funções diferentes

Validação restringe o que pode entrar.

Escaping impede que dados sejam interpretados como código no contexto de saída.

Sanitização remove ou restringe conteúdo potencialmente perigoso de acordo com uma política.

### 5. Defesa em profundidade

A aplicação não deve depender de uma única camada de segurança.

---

# Conclusão Extraída da Experiência

O laboratório **XSS** demonstrou, através de diferentes stacks, que uma vulnerabilidade de Cross-Site Scripting não está necessariamente ligada à linguagem utilizada, mas principalmente à forma como dados não confiáveis percorrem a aplicação até chegar ao navegador.

A análise dos exemplos de **PHP, Node.js, Flask e ASP.NET/C#** mostrou o mesmo padrão recorrente:

```text
Entrada controlada pelo usuário
              ↓
Processamento / armazenamento
              ↓
Renderização insegura
              ↓
Interpretação pelo navegador
```

A correção depende de aplicar o mecanismo apropriado ao contexto, utilizando **validação, sanitização e output encoding**, além de uma estratégia de **defesa em profundidade**.

Para uma atuação em **Pentest, AppSec ou Bug Bounty**, o principal aprendizado da room é deixar de procurar somente por um payload específico e passar a analisar o **data flow** completo:

```text
SOURCE → PROCESSAMENTO → SINK → CONTEXTO → IMPACTO
```

Essa abordagem permite identificar tanto vulnerabilidades refletidas quanto persistentes e baseadas em DOM, além de facilitar a documentação técnica da causa raiz e da correção.

---

## Referência

- **TryHackMe — XSS**
- Room: `axss`
- Conteúdo utilizado: material da própria room fornecido para elaboração deste relatório.
