# Mandro — Ciclo 1: design

## 1. Visão

Mandro é uma linguagem de programação compilada, estaticamente tipada, com palavras-chave em
português. Arquivos-fonte usam a extensão `.mndr`; a ferramenta de linha de comando é `mandro`.

**Propósito**, em ordem de prioridade:

1. **Ensinar** — fácil de aprender e de ensinar a programar em português.
2. **Aprender** — o próprio compilador é um projeto didático de construção de linguagens.
3. **Uso real** — robusta o bastante para programas de verdade, como Python é.

**Princípios da linguagem** (valem para todos os ciclos):

- **Rigor de tipos no estilo Rust, sem ownership.** Tipagem estática e forte, sem conversão
  implícita, sem nulo, tipos-soma com `escolha` exaustivo, imutável por padrão, erros como valores.
  A memória é automática: o programador nunca lida com donos nem empréstimos.
- **Português correto, sem acentos**, em palavras-chave e identificadores (`senao`, `funcao`).
  Textos entre aspas e comentários aceitam qualquer caractere.
- **Próximo do que já existe.** Na dúvida entre um nome inventado e um consagrado em outras
  linguagens, vence o consagrado (`Booleano`, `%`, `{ }`).
- **Uma forma de escrever cada coisa.** Grafias alternativas são erro com sugestão, não sinônimo.

**Estratégia de implementação.** Um compilador escrito em Rust lê `.mndr`, gera C e chama um
compilador C para produzir um executável nativo. A memória é gerenciada por contagem de referências
inserida pelo compilador.

## 2. Escopo do ciclo 1

O ciclo 1 é uma **fatia vertical**: um subconjunto pequeno da linguagem que atravessa todas as
etapas do compilador até um executável, com contagem de referências real (sobre `Texto`).

### 2.1 Entra

- Tipos `Inteiro`, `Real`, `Booleano`, `Texto`.
- Variáveis `seja` / `seja mutavel`, com tipo inferido ou anotado; redeclaração (sombra) permitida.
- Operadores aritméticos, de comparação e lógicos; concatenação de `Texto`.
- `se` / `senao se` / `senao`, `enquanto`, `para … em` com intervalos, `pare`, `continue`.
- Funções com parâmetros e retorno tipados e recursão. Funções não são valores (não são passadas
  como argumento nem guardadas em variável).
- Interpolação de texto; embutidos `escreva`, `real`, `falha`.
- Programas de um único arquivo.
- Diagnósticos completos em português, `mandro explicar`, avisos.
- CLI `mandro rodar`, `mandro compilar`, `mandro verificar`, `mandro explicar`.

### 2.2 Fica fora (roadmap na seção 11)

`enum`, `escolha`, `Opcional`, `Resultado`, `?`, `leia`, conversões que podem falhar, `Decimal`,
classes, interfaces, coleções, genéricos do usuário, módulos, constantes globais, notação
científica em literais, textos multilinha, threads, ferramentas de editor.

### 2.3 Decisões fixadas agora por compatibilidade futura

- Palavras reservadas para recursos futuros (seção 3.9) já são recusadas como identificadores.
- Literal com ponto (`0.1`) sem contexto é `Real`; o tipo `Decimal` (ciclo 2) só é usado quando
  declarado ou exigido pelo contexto.
- Erros recuperáveis serão `Resultado<T, E>` com `?`; `falha` existe só para bugs.
- Classes não terão herança: polimorfismo por interfaces, reuso por composição.
- O front-end também serve às ferramentas de editor (seção 11): o lexer preserva os comentários com
  suas posições, e a análise produz HIR mesmo quando há erros (seção 4.3).

## 3. A linguagem no ciclo 1

### 3.1 Léxico

- O arquivo-fonte é UTF-8.
- **Identificadores e palavras-chave** são ASCII: `[A-Za-z_][A-Za-z0-9_]*`, sensíveis a
  maiúsculas. Uma letra acentuada nessa posição é erro; a mensagem sugere a grafia sem acento
  (`senão` → `senao`, `preço` → `preco`). A mesma regra cobre palavras-chave e nomes.
- **Convenções de nome:** `snake_case` para funções e variáveis, `PascalCase` para tipos.
- **Números inteiros:** `42`, `1_000_000`. **Números com ponto:** `3.14`, `0.5`. Exige dígito dos
  dois lados do ponto (`.5` e `5.` são inválidos), o que mantém `1..10` sem ambiguidade.
- **Textos:** `"..."` numa única linha. Escapes: `\n`, `\t`, `\\`, `\"`. Chaves literais: `{{` e
  `}}`. Escape desconhecido é erro.
- **Comentários:** `//` até o fim da linha.
- **Fim de comando é a quebra de linha.** Não existe `;`. O lexer emite um token de fim de comando
  na quebra de linha quando o último token da linha é identificador, literal, `)`, `]`, `}` ou uma
  das palavras `retorne`, `pare`, `continue`, `verdadeiro`, `falso`. Dentro de `(` ou `[` quebras
  de linha são ignoradas. Consequências:
  - uma linha que termina em operador ou vírgula continua na seguinte;
  - `senao` fica na mesma linha do `}` que fecha o `se` (`} senao {`), e `{` fica na mesma linha
    que abre o bloco. O erro para `senao` em linha nova sugere subi-lo.

### 3.2 Tipos

| Tipo       | Valores                                   | Representação           |
|------------|-------------------------------------------|-------------------------|
| `Inteiro`  | inteiros de 64 bits com sinal             | `int64_t`               |
| `Real`     | ponto flutuante binário IEEE 754 de 64 bits | `double`              |
| `Booleano` | `verdadeiro`, `falso`                     | `bool`                  |
| `Texto`    | sequência UTF-8 imutável                  | objeto com contagem de referências |

Uma função sem `->` não devolve valor.

### 3.3 Literais e inferência

- Literais têm tipo **flexível, resolvido pelo contexto** (anotação, parâmetro, retorno, o outro
  operando de uma operação). Sem contexto: inteiro → `Inteiro`; com ponto → `Real`.
- Literal inteiro pode virar `Real`; literal com ponto nunca vira `Inteiro`.
- Por isso `1 + 2.0` é `Real` e é válido. Variáveis não se adaptam: com `idade: Inteiro`,
  `idade + 2.5` é erro, com sugestão `real(idade) + 2.5`.
- Literal inteiro fora do intervalo de 64 bits é erro de compilação (o `-` aplicado diretamente a
  um literal faz parte do literal, então `-9223372036854775808` é válido). Literal inteiro usado
  como `Real` precisa ser representável exatamente.

### 3.4 Declarações e escopo

- No nível do arquivo só existem declarações `funcao`. A ordem não importa (recursão mútua vale).
- `funcao principal()` é obrigatória, sem parâmetros e sem retorno. Ausente ou com outra
  assinatura é erro.
- Nomes de função são únicos; não existe sobrecarga. Declarar função ou variável com o nome de um
  embutido é erro.
- `seja nome = expr`, `seja nome: Tipo = expr`, `seja mutavel nome = expr`. O valor inicial é
  obrigatório.
- **Sombra:** redeclarar um nome com `seja` no mesmo bloco ou num bloco interno é permitido e pode
  mudar o tipo. Parâmetros também podem ser sombreados.
- `=`, `+=`, `-=`, `*=` só valem para variáveis `mutavel`. Parâmetros são imutáveis. Atribuição é
  comando, não expressão.
- Cada `{ … }` abre um escopo.

### 3.5 Expressões e operadores

Precedência, da menor para a maior:

| Nível | Operadores                       | Observação                         |
|-------|----------------------------------|------------------------------------|
| 1     | `ou`                             | curto-circuito                     |
| 2     | `e`                              | curto-circuito                     |
| 3     | `nao` (prefixo)                  | como `not` do Python               |
| 4     | `==` `!=` `<` `<=` `>` `>=`      | não associativos: `a < b < c` é erro |
| 5     | `+` `-`                          |                                    |
| 6     | `*` `/` `div` `%`                |                                    |
| 7     | `-` (prefixo)                    |                                    |
| 8     | chamada `f(…)`, `( … )`          |                                    |

Tipos dos operandos (os dois lados sempre do mesmo tipo):

- `+` `-` `*`: `Inteiro` → `Inteiro` (com checagem de overflow); `Real` → `Real`.
- `+` também concatena `Texto`.
- `/`: `Inteiro` ou `Real` → **sempre `Real`** (`5 / 2 == 2.5`).
- `div` e `%`: só `Inteiro`, com **arredondamento para baixo** (como Python):
  `-7 div 2 == -4`, `-7 % 2 == 1`. `i64::MIN div -1` é overflow.
- Divisão por zero (`/`, `div`, `%`, inclusive com `Real`) para o programa.
- `-` prefixo: `Inteiro` (checado) e `Real`.
- `==` `!=`: qualquer um dos quatro tipos. `<` `<=` `>` `>=`: `Inteiro`, `Real`, `Texto`
  (ordem dos bytes UTF-8).
- `e`, `ou`, `nao`: só `Booleano`. Não existe "valor verdadeiro" implícito: `se x` com `x: Inteiro`
  é erro.
- `Real` segue IEEE 754 fora da divisão: overflow dá `inf`; `inf - inf` dá `nan`.

### 3.6 Comandos

- `se cond { … } senao se cond { … } senao { … }` — `cond` é `Booleano`; parênteses opcionais.
- `enquanto cond { … }`.
- `para i em a..b { … }` vai de `a` a `b` **incluindo** `b`; `para i em a..<b { … }` para antes de
  `b`. `a` e `b` são `Inteiro`, avaliados uma vez antes do laço. `i` é imutável. Com `a` maior que
  o fim, o laço não executa. Intervalos só existem dentro de `para`.
- `pare` e `continue` só dentro de laço.
- `retorne` / `retorne expr`.
- Como comando isolado, só é aceita uma chamada de função. Outra expressão solta é erro. Ignorar o
  valor devolvido por uma função gera aviso; `seja _ = f()` descarta de propósito.
- **Toda função com retorno devolve valor em todos os caminhos.** Um bloco devolve se seu último
  comando devolve; um `se` devolve se tem `senao` e todos os ramos devolvem. `enquanto verdadeiro`
  sem `pare` e `falha(…)` nunca terminam normalmente, então satisfazem a regra.

### 3.7 Texto e interpolação

- `"Olá, {nome}! Você tem {idade + 1} anos."` — entre chaves vai qualquer expressão de um tipo
  **exibível**. No ciclo 1, exibíveis são os quatro tipos primitivos.
- `escreva` usa a mesma regra de "exibível" da interpolação: uma regra, dois usos. No ciclo 3 essa
  regra vira uma interface.
- Formato de exibição:
  - `Inteiro`: decimal (`-42`).
  - `Real`: o mesmo formato de `repr(float)` do Python — o menor texto que relê o mesmo número
    (`0.1`, `2.0`, `0.30000000000000004`, `1e+16`, `1e-05`, `inf`, `-inf`, `nan`).
  - `Booleano`: `verdadeiro` / `falso`.
  - `Texto`: como está.

### 3.8 Embutidos

Uma tabela única define cada embutido: nome, assinatura (consumida pela checagem de tipos) e
símbolo C (consumido pelo gerador).

| Embutido                        | Efeito                                                     |
|---------------------------------|------------------------------------------------------------|
| `escreva(valor: exibível)`      | escreve o valor e uma quebra de linha na saída padrão      |
| `real(i: Inteiro) -> Real`      | converte para o `Real` mais próximo                        |
| `falha(mensagem: Texto)`        | para o programa (código 101) com a mensagem e a posição; nunca retorna |

### 3.9 Palavras-chave

- **Ativas:** `funcao` `retorne` `seja` `mutavel` `se` `senao` `enquanto` `para` `em` `pare`
  `continue` `e` `ou` `nao` `div` `verdadeiro` `falso`.
- **Reservadas:** `classe` `interface` `implementa` `enum` `escolha` `caso` `este` `com` `publico`
  `privado` `importe` `constante`.
- Nomes de tipo (`Inteiro`, `Real`, `Booleano`, `Texto`) e embutidos são identificadores
  resolvidos pelo escopo, não palavras-chave.
- Uma tabela única de palavras-chave alimenta o lexer e as sugestões de grafia.

### 3.10 Exemplos

```
funcao fatorial(n: Inteiro) -> Inteiro {
    se n <= 1 {
        retorne 1
    }
    retorne n * fatorial(n - 1)
}

funcao principal() {
    seja mutavel soma = 0
    para i em 1..10 {
        soma += i
        se i % 2 == 0 {
            escreva("{i}! = {fatorial(i)}")
        }
    }
    seja media = soma / 10
    escreva("média: {media}")
}
```

```
funcao principal() {
    para i em 1..15 {
        se i % 15 == 0 {
            escreva("FizzBuzz")
        } senao se i % 3 == 0 {
            escreva("Fizz")
        } senao se i % 5 == 0 {
            escreva("Buzz")
        } senao {
            escreva(i)
        }
    }
}
```

## 4. Arquitetura do compilador

### 4.1 Pipeline

```
app.mndr
  │ lex         texto → tokens
  │ parse       tokens → AST
  │ resolve     cada uso de nome ligado à sua declaração
  │ typeck      → HIR tipado (todo nó com tipo concreto)
  ├──── fronteira: daqui para baixo nenhum erro é do usuário ────
  │ lower       HIR → MIR
  │ rc          MIR → MIR com retenções e liberações explícitas
  │ codegen_c   MIR → app.c
  │ cc          app.c + runtime → executável
```

### 4.2 Regras da arquitetura

1. **Só o front-end fala com o usuário.** `lex`, `parse`, `resolve` e `typeck` produzem
   diagnósticos. Qualquer falha depois da fronteira — inclusive o compilador C recusar o código
   gerado — é erro interno do compilador (seção 5.7).
2. **Cada etapa é uma transformação pura.** Recebe a representação anterior, devolve a seguinte e
   diagnósticos. Sem estado global. Cada etapa é testável isoladamente.
3. **Cada fato tem uma fonte.** Posições são offsets de byte num arquivo registrado; linha e coluna
   são derivadas. Palavras-chave, embutidos e o catálogo de erros são tabelas únicas.

### 4.3 Crates

O repositório é um workspace Cargo. A direção de dependência segue o pipeline: nenhum crate
depende de um crate posterior.

| Crate                | Responsabilidade                                                              |
|----------------------|-------------------------------------------------------------------------------|
| `mandro_diagnostics` | `SourceMap`, `SourceFile`, `Span`, `Diagnostic`, catálogo de códigos e explicações, renderização |
| `mandro_syntax`      | tokens, tabela de palavras-chave, lexer, AST, parser                          |
| `mandro_sema`        | resolução de nomes, inferência e checagem de tipos, análises de fluxo, HIR, tabela de embutidos |
| `mandro_mir`         | MIR, `lower`, inserção de contagem de referências                             |
| `mandro_codegen_c`   | MIR → texto C                                                                 |
| `mandro_driver`      | encadeia as etapas; único ponto que invoca o compilador C; runtime embutido   |
| `mandro`             | binário da CLI                                                                |
| `runtime/` (C)       | `mandro.h`, `mandro.c`                                                        |

Interfaces principais (esboço):

- `mandro_syntax::lex(&SourceFile) -> Lexed` — tokens, comentários (lista separada, com posições)
  e diagnósticos. Os comentários não chegam ao parser; existem para o formatador e o editor.
- `mandro_syntax::parse(&SourceFile, &[Token]) -> (ast::Module, Vec<Diagnostic>)`
- `mandro_sema::analyze(&ast::Module) -> Analysis` — `Analysis` traz o `hir::Module` **sempre**,
  mesmo com erros (nós com erro têm o tipo *erro*), e os diagnósticos, avisos incluídos. Isso
  permite ao editor mostrar tipos e navegar num arquivo que ainda tem erros.
- `Analysis::checked(self) -> Result<CheckedModule, Vec<Diagnostic>>` — só uma análise sem erros
  vira `CheckedModule`.
- `mandro_mir::lower(&CheckedModule) -> mir::Program` — o tipo garante que o back-end nunca recebe
  um programa com erro.
- `mandro_mir::insert_rc(mir::Program) -> mir::Program`
- `mandro_codegen_c::emit(&mir::Program, &SourceMap) -> String`
- `mandro_driver::compile(&Path, &Options) -> Result<Artifact, Failure>`

### 4.4 Escolhas de implementação

- **Parser escrito à mão**: descida recursiva para comandos e declarações, Pratt para expressões.
  Dá controle total sobre mensagens e recuperação de erro.
- **MIR desde o ciclo 1.** Funções com variáveis locais tipadas, blocos básicos e terminadores
  explícitos (`goto`, desvio condicional, `return`). `para`, interpolação, atribuição composta e
  curto-circuito já chegam desaçucarados. É onde a contagem de referências é inserida, onde a ordem de avaliação é
  fixada e onde otimizações futuras entram. Toda operação que pode falhar em execução carrega a
  posição do `.mndr`.
- **Idioma do código:** identificadores em inglês; comentários e documentação em português.
- **Dependências previstas:** `annotate-snippets` (renderização de diagnósticos), `clap` (CLI),
  `tempfile` (driver); para testes, `insta`, `proptest`, `libtest-mimic`.

## 5. Diagnósticos

### 5.1 Formato

```
erro[E0004]: tipos diferentes na operação `+`
 --> app.mndr:7:18
  |
7 |     seja total = idade + 2.5
  |                  ----- ^ --- Real
  |                  |
  |                  Inteiro
  |
  = ajuda: converta explicitamente: `real(idade) + 2.5`
  = detalhes: mandro explicar E0004
```

Um `Diagnostic` tem: gravidade (`erro` ou `aviso`), código, título, rótulo principal com `Span`,
rótulos secundários, notas e uma sugestão opcional com texto de substituição. (Códigos nos
exemplos são ilustrativos; a numeração é a do catálogo.)

### 5.2 Catálogo e `mandro explicar`

- Cada código existe uma vez no catálogo, com título e explicação longa em português (exemplo
  errado, exemplo corrigido, o porquê da regra). As explicações são arquivos Markdown dentro de
  `mandro_diagnostics`, embutidos no binário.
- `mandro explicar E0004` mostra a explicação.
- Um teste falha se um código não tiver explicação ou se nenhum caso de teste o disparar.

### 5.3 Tom

Diz o que aconteceu, onde e como resolver. Sem jargão de compilador ("esperava `)`", não "token
inesperado"). Mensagens em português com acentuação normal.

### 5.4 Categorias de erro do ciclo 1

- **Léxicas:** letra acentuada em nome ou palavra-chave; caractere inválido; texto ou interpolação
  não fechados; escape inválido; número malformado; literal fora do intervalo.
- **Sintáticas:** símbolo esperado ausente; `senao` sem `se` ou em linha nova; bloco não fechado;
  palavra reservada usada como nome; comparação encadeada; expressão solta como comando.
- **Nomes:** nome não declarado (com sugestão do nome mais parecido); função duplicada; embutido
  redefinido; `principal` ausente ou com assinatura errada.
- **Tipos:** tipos diferentes numa operação; operação não suportada pelo tipo; condição não
  `Booleano`; número ou tipo de argumentos errado; valor não exibível; literal que não cabe no tipo.
- **Fluxo:** função sem retorno em algum caminho; `retorne` com valor em função sem retorno (e
  vice-versa); atribuição a variável não `mutavel` ou a parâmetro; `pare`/`continue` fora de laço.

### 5.5 Cascatas

- O parser se recupera no próximo fim de comando ou `}` e continua.
- Expressões com erro recebem o tipo interno *erro*, compatível com tudo, e não geram novos erros.
- Mostra no máximo **10 erros**, seguidos de "e mais N erros".

### 5.6 Avisos

Não impedem a compilação:

- variável `mutavel` nunca alterada;
- variável nunca usada (nome começando com `_` silencia);
- código inalcançável depois de `retorne`, `pare`, `continue` ou `falha`;
- valor devolvido por função ignorado.

### 5.7 Erros em execução e erro interno

- Overflow de `Inteiro`, divisão por zero e `falha(…)` imprimem na saída de erro:
  ```
  erro durante a execução: divisão por zero
   --> app.mndr:12:20
  ```
  e terminam com código **101**.
- **Erro interno do compilador:** "erro interno do compilador — isto é um bug do Mandro, não do seu
  código", com a etapa que falhou, o caminho do `.c` gerado (preservado) e como reportar. Código
  **70**.

### 5.8 Códigos de saída

| Código | Significado                                    |
|--------|------------------------------------------------|
| 0      | sucesso                                        |
| 1      | o programa tem erros de compilação             |
| 2      | uso incorreto da CLI                           |
| 70     | erro interno do compilador                     |
| 101    | o programa parou em execução (`mandro rodar` repassa o código do programa) |

## 6. Geração de C, runtime e memória

### 6.1 Exemplo

```
funcao saudacao(nome: Texto) -> Texto {
    retorne "Olá, " + nome
}

funcao principal() {
    seja s = saudacao("Ana")
    escreva(s)
}
```

```c
static MndrText *mndr_fn_saudacao(MndrText *nome) {     /* nome: emprestado */
    MndrText *t0 = mndr_text_concat(&MNDR_LIT_0, nome);  /* t0: novo, com dono */
    return t0;                                           /* o dono passa a quem chamou */
}

void mndr_fn_principal(void) {
    MndrText *s = mndr_fn_saudacao(&MNDR_LIT_1);
    mndr_print_text(s);
    mndr_text_release(s);                                /* fim do escopo de s */
}
```

Funções do usuário viram `mndr_fn_<nome>`; o `main` do C fica no runtime e chama
`mndr_fn_principal`.

### 6.2 Contagem de referências

- **Parâmetros são emprestados**: quem recebe não libera. **Retornos têm dono**: quem chama libera.
- Toda variável ou temporário que é dono de um valor gerenciado é liberado **ao fim do seu escopo**,
  inclusive em saídas antecipadas (`retorne`, `pare`, `continue`). O `lower` gera os caminhos de
  limpeza explicitamente na MIR.
- Copiar um valor emprestado para um dono (guardar em variável, devolver) faz uma retenção.
- Reatribuir uma variável `mutavel` libera o valor anterior.
- **Literais de texto são imortais**: estáticos, sem alocação; reter e liberar não os afetam.
- Contagem não atômica (o ciclo 1 não tem threads).
- Ciclos de referência são impossíveis no ciclo 1: `Texto` não referencia outros objetos.
- O esquema prioriza correção. Eliminar pares retenção/liberação e reuso in-place (Perceus) são
  otimizações futuras, feitas com medição.

### 6.3 Garantias do C gerado

- **Ordem de avaliação da esquerda para a direita.** A MIR avalia cada operando e argumento num
  temporário, em ordem; o C gerado nunca depende da ordem de avaliação do C.
- **Sem comportamento indefinido.** Toda aritmética de `Inteiro` passa por funções do runtime com
  checagem; não há variável sem valor inicial.
- Padrão C11, apenas `stdint.h`, `stdbool.h` e a biblioteca padrão.
- Compila sem avisos com `-Wall -Wextra`.

### 6.4 Runtime

- `runtime/mandro.h` e `runtime/mandro.c`: representação de `Texto`, concatenação, formatação dos
  quatro tipos, escrita, aritmética checada, `falha` e `main`.
- Único lugar com aritmética checada: usa `__builtin_*_overflow` quando disponível e um fallback
  portável caso contrário.
- Embutido no binário `mandro` (`include_str!`) e escrito ao lado do `.c` gerado na compilação.

### 6.5 Invocação do compilador C

- Ponto único no `mandro_driver`. Compilador padrão: `cc`; `MANDRO_CC` substitui.
- Perfis: `rodar` usa `-O0 -g`; `compilar` usa `-O2`; testes usam
  `-O0 -g -fsanitize=address,undefined -Wall -Wextra -Werror`.

## 7. CLI

- `mandro rodar <arquivo.mndr> [--emitir-c]` — compila numa pasta temporária e executa,
  repassando o código de saída do programa.
- `mandro compilar <arquivo.mndr> [-o <saida>] [--emitir-c]` — gera o executável (padrão: nome do
  arquivo sem extensão, no diretório atual).
- `mandro verificar <arquivo.mndr>` — roda só o front-end e mostra os diagnósticos, sem gerar
  nada nem exigir compilador C. Serve para CI e editores.
- `mandro explicar <codigo>` — mostra a explicação de um código de erro.
- `--emitir-c` grava o `.c` ao lado do `.mndr`, com um comentário `/* app.mndr:LINHA */` antes do
  código de cada comando.
- A pasta temporária é apagada em caso de sucesso e preservada em erro interno.
- Ajuda e mensagens da CLI em português.

## 8. Testes

### 8.1 Casos de interface (UI)

- `tests/ui/**/<caso>.mndr` acompanhado de `<caso>.expected`. Cada caso é um teste separado.
- O `.expected` é a única descrição do resultado: código de saída, saída padrão e saída de erro,
  exatamente como aparecem.
  ```
  código de saída: 101
  --- saída ---
  --- erros ---
  erro durante a execução: divisão por zero
   --> division_by_zero.mndr:3:14
  ```
- `MANDRO_UPDATE_EXPECTED=1 cargo test` regrava os `.expected`; a revisão acontece no diff do git.
- Todo programa de teste é compilado com o perfil de testes (6.5): AddressSanitizer,
  LeakSanitizer e UBSan ativos, avisos do C como erro. Vazamento, liberação dupla, uso após
  liberação ou aviso do C reprovam o caso.

### 8.2 Snapshots por etapa

Com `insta`: tokens, AST, HIR, e a MIR antes e depois de `insert_rc`. O snapshot da MIR mostra onde
cada retenção e liberação foi inserida.

### 8.3 Propriedades

Com `proptest`:

- `lex` e `parse` nunca entram em pânico com entrada arbitrária.
- Aritmética do runtime (overflow, `div`, `%`, divisão por zero) concorda com uma implementação de
  referência em Rust.
- Exibir um `Real` e relê-lo devolve o mesmo valor, no formato do `repr(float)` do Python.

### 8.4 Catálogo

Todo código do catálogo tem explicação e é disparado por ao menos um caso de UI.

### 8.5 Disciplina do código do compilador

`#![forbid(unsafe_code)]` em todos os crates; `clippy::pedantic` com avisos como erro; `cargo fmt`.
Desenvolvimento em TDD: cada recurso começa por um caso de UI que falha.

## 9. Riscos

| Risco | Mitigação |
|-------|-----------|
| Bugs de contagem de referências (vazamento, liberação dupla) | sanitizers em todo caso de UI; snapshots da MIR após `insert_rc` |
| Regra de fim de linha surpreender (`senao` em linha nova) | erro específico com sugestão de correção |
| Formato de `Real` divergir do esperado | testes de propriedade de ida e volta e tabela de casos do `repr` |
| Crescimento de escopo | seção 2.2; cada recurso novo pertence a um ciclo com spec própria |
| Falta de compilador C (Windows) | fora do ciclo 1; ciclo de distribuição embute tcc ou `zig cc` |

## 10. Critérios de conclusão

1. `mandro rodar` executa os exemplos da seção 3.10, e programas de primos e tabuada, com a saída
   correta.
2. Cada categoria de erro da seção 5.4 tem ao menos um caso de UI com mensagem exata, e cada código
   tem explicação acessível por `mandro explicar`.
3. Todos os casos de UI passam com sanitizers ativos e sem avisos do compilador C.
4. `cargo clippy` (pedantic, avisos como erro), `cargo fmt --check` e `cargo test` passam.
5. `--emitir-c` produz C legível com comentários de posição.

## 11. Roadmap depois do ciclo 1

Cada ciclo tem spec, plano e implementação próprios. Ordem provável:

- **Ciclo 2 — tipos-soma e erros:** `enum` com dados, `escolha` exaustivo, `Opcional`,
  `Resultado<T, E>` com `?`, `leia()`, conversões que podem falhar (`inteiro("42")`), e `Decimal`:
  - base 10, 128 bits, até 28 dígitos significativos;
  - `0.1 + 0.2 == 0.3`; preserva casas (`7.50` exibe `7.50`);
  - `+` `-` `*` exatos ou o programa para; nunca arredondam em silêncio;
  - `/` exato quando a divisão termina, 28 dígitos com meio-para-par quando não termina;
  - arredondamento para casas sempre explícito, com os modos de mercado (meio-para-par,
    meio-para-cima, truncar e outros);
  - literais adotam `Decimal` pelo contexto e são lidos direto do texto; nunca mistura com `Real`
    sem conversão explícita.
- **Ciclo 3 — objetos:** `classe`, métodos, `este`, cópia com alteração `valor com { campo: … }`,
  construtores que validam invariantes, `publico`/`privado`, `interface`, `implementa`.
- **Ciclo 4 — coleções e módulos:** `Lista`, `Mapa`, genéricos do usuário, múltiplos arquivos,
  `importe`.
- **Etapa de editor (VS Code e Cursor)** — separada dos ciclos da linguagem, com spec própria:
  - extensão VS Code com realce de sintaxe (gramática TextMate), configuração de linguagem
    (comentários, pares de chaves, indentação) e trechos prontos; as palavras-chave da gramática são
    geradas a partir da tabela única de `mandro_syntax` ou verificadas contra ela por teste;
  - servidor de linguagem `mandro lsp` (protocolo LSP) sobre o mesmo front-end: erros enquanto se
    digita, tipo ao passar o mouse, ir para a definição, autocompletar, renomear, realce semântico.
    Cumpre o papel que o pyright tem para Python e o rust-analyzer para Rust; no Mandro o
    verificador de tipos é o próprio compilador;
  - formatador `mandro formatar` (como gofmt e ruff format), também exposto pelo LSP;
  - lints configuráveis além dos avisos do compilador (como clippy e ruff);
  - publicação no Visual Studio Marketplace e no Open VSX, de onde Cursor e VSCodium instalam;
  - depuração: diretivas `#line` no C gerado deixam gdb e lldb pararem em linhas do `.mndr`,
    integradas ao editor pelo protocolo de depuração (DAP);
  - ordem sugerida: realce de sintaxe logo após o ciclo 1; LSP e formatador após o ciclo 2 ou 3.
- **Depois:** `Dinheiro` na biblioteca padrão (valor + moeda, casas por moeda, rateio sem perda de
  centavos); remoção de retenções redundantes e Perceus; referências fracas ou coleta de ciclos;
  distribuição (Windows, macOS, compilador C embutido); benchmarks contra Java e Python.
