# ADR-001 — Arquitetura e stack do Connexa: monolito modular .NET + SPA React

- **Situação:** aceita
- **Data:** 2026-11-15
- **Participantes:** Daniel de França Cunha, Gabriel Rodrigues Lopez Alvarez, Gabriela Sonvezzo Feltrin, Leonardo do Monte Serrat de Souza

## Contexto

A equipe tem de 4 a 5 integrantes e um semestre para entregar o Connexa, em sprints de duas semanas. A carga esperada é modesta (uso acadêmico, poucos usuários simultâneos) e não há equipe de operação. O código será escrito principalmente por um agente de IA a partir das Tasks do Azure Boards; por isso, as regras de arquitetura precisam ser verificadas automaticamente, e não apenas combinadas entre as pessoas.

O domínio se divide em territórios com dados próprios: Identidade, Perfil, Grupos, Encontros, Acompanhamento, Painel e Materiais. Sem fronteiras explícitas, uma parte do sistema passaria a usar os dados de outra, e cada mudança afetaria o todo. A autenticação real está fora da Sprint 1. A disciplina oferece templates prontos em Java, .NET e Node.js, com a mesma proposta de arquitetura.

## Decisão

Adotamos um **monolito modular** com duas unidades de implantação: uma **SPA em React 19 + TypeScript (Vite)** e um único back-end em **.NET 10 / ASP.NET Core (Minimal APIs)** com **PostgreSQL 17**, a partir do template `connexa-modelo-dotnet`. Cada território é um projeto `Connexa.<Modulo>`, com o interior `internal` em `Internal/`, um schema próprio no banco e migrações do EF Core aplicadas quando a aplicação sobe.

Decisões que acompanham a escolha, já presentes no template:

- erros da API no formato Problem Detail (RFC 9457) com a propriedade `codigo`;
- identidade simulada por `IUsuarioAtual`, sem ASP.NET Core Identity na Sprint 1;
- testes de aceitação com MSTest + `WebApplicationFactory` no banco `connexa_teste`;
- versões fixas em `Directory.Packages.props` e `package.json`, com `TreatWarningsAsErrors`.

## Alternativas consideradas

| Alternativa | Por que não foi escolhida |
|---|---|
| Microsserviços | Uma equipe de 4 a 5 pessoas, em um semestre e com carga modesta, não paga o custo de vários serviços, implantações separadas e um broker de mensagens |
| Monolito só em camadas (sem módulos) | Camadas não impedem que um território use os dados de outro; as fronteiras dependeriam apenas da disciplina da equipe e do agente |
| Template Java (`connexa-modelo-java`) | Mesma arquitetura; a equipe tem mais familiaridade com C#/.NET e prefere o ecossistema (EF Core, `internal` por projeto) |
| Template Node.js (`connexa-modelo-node`) | Mesma arquitetura; a fronteira entre módulos depende mais de lint do que do compilador, e a tipagem é menos rígida no back-end |
| Testcontainers para os testes | Exigiria Docker dentro do Codespace; o banco `connexa_teste` do Compose funciona igual no Codespaces, no CI e na máquina local |

## Consequências

- **Positivas:**
  - Uma única implantação de back-end, simples de rodar no Codespaces, no CI e localmente.
  - O compilador impede o uso do interior de outro módulo e recusa referências em ciclo; o NetArchTest confere o resto em todo build.
  - Cada módulo é dono dos seus dados (um schema por módulo, sem chave estrangeira entre módulos).
  - O banco só muda por migração revisada em pull request; o comportamento é o mesmo em desenvolvimento, testes e CI.
  - Quando a autenticação real chegar, só o interior de `Connexa.Identidade` muda.
  - O contrato da API fica visível em `/swagger` para quem especifica e revisa.
- **Negativas:**
  - Mais cerimônia que um projeto único: cada módulo novo exige projeto, schema e registro — e um ADR.
  - Reações entre módulos precisam passar por eventos (`IDespachanteDeEventos`), o que é menos direto que uma chamada.
  - Todos os módulos são implantados juntos; escalar só uma parte não é possível.
  - Incluir ou atualizar dependências exige ADR; o agente não pode adicioná-las.
- **Como verificar:**
  - `ArquiteturaModularTest` (NetArchTest) falha com tipo público em `Internal/`, uso do `Internal/` de outro módulo ou dependência proibida.
  - `SchemasDosModulosTest` confere um schema por módulo.
  - Regra ESLint `no-restricted-imports`: uma feature do front-end só é usada pelo seu `index.ts`.
  - `TreatWarningsAsErrors`: um aviso de compilação reprova o build.
  - CI com os três checks obrigatórios no ruleset do `main`; `./verificar` roda o mesmo localmente.
  - Revisão de pull request: dependência nova, módulo novo ou referência entre módulos sem ADR é recusada.
