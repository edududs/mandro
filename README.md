# Mandro

Linguagem de programação em português, fácil de aprender e de ensinar, com o rigor de tipos de
Rust e compilada para código nativo.

> **Estado: em design.** O compilador ainda não existe. A especificação do primeiro ciclo está em
> [`docs/superpowers/specs/2026-10-21-mandro-cycle-1-design.md`](docs/superpowers/specs/2026-10-21-mandro-cycle-1-design.md).

## Como é

Programas Mandro usam a extensão `.mndr`:

```mndr
funcao fatorial(n: Inteiro) -> Inteiro {
    se n <= 1 {
        retorne 1
    }
    retorne n * fatorial(n - 1)
}

funcao principal() {
    para i em 1..10 {
        escreva("{i}! = {fatorial(i)}")
    }
}
```

O compilador explica os erros em português, aponta onde estão e sugere a correção:

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

## Princípios

- **Português sem acentos** em palavras-chave e nomes (`senao`, `funcao`); textos e comentários
  aceitam qualquer caractere.
- **Tipagem estática e forte**: sem conversões implícitas, sem nulo, imutável por padrão.
- **Memória automática** por contagem de referências: o programador não lida com donos nem
  empréstimos.
- **Erros que ensinam**: mensagens em português, sugestões de correção e `mandro explicar <código>`.
- **Executável nativo**: o compilador, escrito em Rust, gera C e o entrega a um compilador C.
- **Próximo do que já existe**: na dúvida, vale o nome ou símbolo consagrado (`Booleano`, `%`, `{ }`).

## Roteiro

A linguagem cresce em ciclos; cada um tem especificação, plano e implementação próprios.

| Ciclo | Conteúdo                                                              | Estado            |
|-------|-----------------------------------------------------------------------|-------------------|
| 1     | `Inteiro`, `Real`, `Booleano`, `Texto`, funções, controle de fluxo, CLI | em especificação |
| 2     | `enum`, `escolha`, `Opcional`, `Resultado`, `leia`, `Decimal`         | planejado         |
| 3     | classes, interfaces                                                   | planejado         |
| 4     | coleções, genéricos, módulos                                          | planejado         |
| Editor | extensão VS Code e Cursor: realce, LSP, formatador, lints, depuração | planejado         |

O detalhe de cada ciclo está na seção 11 da especificação.
