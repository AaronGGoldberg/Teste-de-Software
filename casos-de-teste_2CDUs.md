# Casos de Testes — CDU024: Moderar Conviventes e CDU025: Analisar Denúncias

## 1. Introdução

Este documento reúne os casos de teste dos CDU024 e CDU025 do TeVejo.

Os testes do **CDU024 — Moderar Conviventes** verificam se o moderador consegue restringir ou banir membros de sua comunidade conforme as regras definidas.

Os testes do **CDU025 — Analisar Denúncias** verificam a fila de denúncias e as decisões do moderador de aprovar ou reprovar denúncias de postagens e comentários.

### 1.1 Visão geral

* 🗂️ **1. Introdução**

  * 📑 1.1 Visão geral
* 🗂️ **2. Histórico de Revisões**
* 🗂️ **3. Testes Funcionais**

  * 📑 3.1 CDU024 — Moderar Conviventes
  * 📑 3.2 CDU025 — Analisar Denúncias

---

## 2. Histórico de Revisões

|    Data    | Versão | Descrição       | Autor(a)       |
| :--------: | :----: | :-------------- | :------------- |
| 16/09/2026 |   1.0  | Versão inicial. | Aaron Goldberg |

---

# 3. Testes Funcionais

## 3.1 CDU024 — Moderar Conviventes

### Especificação do CDU

* **Ator principal:** Moderador.
* **Resumo:** o moderador consulta membros da comunidade e pode restringir ou banir um convivente.
* **Pré-condição:** estar autenticado como moderador da comunidade; o alvo deve ser um membro diferente do moderador.
* **Pós-condição:** a ação é registrada. Na restrição, é criada uma data de expiração; no banimento, o convivente deixa `membros` e passa a `banidos`.

### Casos essenciais derivados das classes

| Cenário                     | Entrada                                                       | Resultado esperado                                                             | Resultado obtido | Situação      |
| --------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------ | ---------------- | ------------- |
| Restringir convivente       | Moderador; membro válido; `tipo=restringir`; `duracao_dias=1` | HTTP 201; ação criada, com expiração futura; convivente continua membro.       | -                | Não executado |
| Banir convivente            | Moderador; membro válido; `tipo=banir`; justificativa válida  | HTTP 201; ação criada; convivente removido dos membros e incluído nos banidos. | -                | Não executado |
| Duração inválida            | `duracao_dias=0`, nula ou ausente                             | HTTP 400; nenhuma ação é criada.                                               | -                | Não executado |
| Banimento sem justificativa | Justificativa ausente, vazia ou só espaços                    | HTTP 400; nenhum banimento é aplicado.                                         | -                | Não executado |
| Tipo inválido               | `tipo=advertir` ou ausente                                    | HTTP 400; nenhuma ação é criada.                                               | -                | Não executado |
| Alvo inválido               | ID inexistente, não membro ou do próprio moderador            | HTTP 404 para inexistente; HTTP 400 para não membro/próprio moderador.         | -                | Não executado |
| Usuário sem permissão       | Usuário anônimo ou não moderador                              | HTTP 401/403; não lista nem altera membros.                                    | -                | Não executado |

### Classes de equivalência

| Campo/condição  | Classe válida                                   | Classe inválida                            | Resultado esperado                   |
| --------------- | ----------------------------------------------- | ------------------------------------------ | ------------------------------------ |
| Moderador       | Usuário autenticado que modera a comunidade     | Anônimo ou usuário sem moderação           | Permite a ação / retorna 401 ou 403. |
| `convivente_id` | ID de membro existente e diferente do moderador | ID inexistente, não membro ou do moderador | Cria ação / retorna 404 ou 400.      |
| `tipo`          | `restringir` ou `banir`                         | Ausente ou fora das opções                 | Processa ação / retorna 400.         |
| `duracao_dias`  | Inteiro maior ou igual a 1 para restrição       | Ausente, nulo, zero, negativo ou texto     | Cria restrição / retorna 400.        |
| `justificativa` | Texto não vazio para banimento                  | Ausente, vazia ou só espaços               | Bane convivente / retorna 400.       |

### Fluxos principais

| Fluxo               | Dados de teste                               | Resultado esperado                                       | Resultado obtido | Situação      |
| ------------------- | -------------------------------------------- | -------------------------------------------------------- | ---------------- | ------------- |
| Listar membros      | Moderador da comunidade                      | HTTP 200; lista os membros e o status de moderação.      | -                | Não executado |
| Consultar histórico | Moderador; ID de membro com ações            | HTTP 200; retorna ações em ordem decrescente de criação. | -                | Não executado |
| Restrição ativa     | Convivente restrito tenta publicar           | HTTP 403; publicação bloqueada até a expiração.          | -                | Não executado |
| Banimento           | Convivente banido tenta participar novamente | HTTP 403; ingresso na comunidade é impedido.             | -                | Não executado |
| Cancelar ação       | Moderador fecha o modal antes de confirmar   | Nenhuma ação é persistida e o estado do membro não muda. | -                | Não executado |

### Análise de valor limite

O limite numérico do CDU é a duração da restrição: `duracao_dias >= 1`. Para a justificativa, a fronteira é texto vazio versus texto com pelo menos um caractere não branco.

| Campo                             | Entrada         | Resultado esperado                                |
| --------------------------------- | --------------- | ------------------------------------------------- |
| `duracao_dias` — abaixo do mínimo | `0`             | HTTP 400.                                         |
| `duracao_dias` — mínimo válido    | `1`             | HTTP 201; expiração em aproximadamente um dia.    |
| `duracao_dias` — acima do mínimo  | `2`             | HTTP 201; expiração em aproximadamente dois dias. |
| `justificativa` — limite inválido | `""` ou `"   "` | HTTP 400 no banimento.                            |
| `justificativa` — mínimo válido   | `"a"`           | HTTP 201; banimento registrado.                   |

---

## 3.2 CDU025 — Analisar Denúncias

### Especificação do CDU

* **Ator principal:** Moderador.
* **Resumo:** o moderador visualiza as denúncias pendentes da comunidade e as aprova ou reprova.
* **Pré-condição:** estar autenticado como moderador e existir denúncia pendente na comunidade.
* **Pós-condição:** a denúncia recebe status, moderador, data e ação registrada. A aprovação remove a postagem/comentário; a reprovação mantém o conteúdo.

### Casos essenciais derivados das classes

| Cenário                        | Entrada                                        | Resultado esperado                                                     | Resultado obtido | Situação      |
| ------------------------------ | ---------------------------------------------- | ---------------------------------------------------------------------- | ---------------- | ------------- |
| Aprovar denúncia de postagem   | Moderador; denúncia pendente de postagem       | HTTP 200; status `aprovada`, auditoria preenchida e postagem removida. | -                | Não executado |
| Aprovar denúncia de comentário | Moderador; denúncia pendente de comentário     | HTTP 200; status `aprovada`, comentário removido e postagem mantida.   | -                | Não executado |
| Reprovar denúncia              | Moderador; denúncia pendente                   | HTTP 200; status `reprovada`, auditoria preenchida e conteúdo mantido. | -                | Não executado |
| Denúncia já analisada          | Denúncia `aprovada` ou `reprovada`             | HTTP 400; não altera a decisão anterior.                               | -                | Não executado |
| Denúncia inválida              | ID inexistente ou denúncia de outra comunidade | HTTP 404; nenhuma denúncia é alterada.                                 | -                | Não executado |
| Ação inválida                  | Ação diferente de `aprovar`/`reprovar`         | HTTP 400; denúncia continua pendente.                                  | -                | Não executado |
| Usuário sem permissão          | Usuário anônimo ou não moderador               | HTTP 401/403; não acessa a fila nem analisa denúncia.                  | -                | Não executado |

### Classes de equivalência

| Campo/condição  | Classe válida                               | Classe inválida                                | Resultado esperado                                                        |
| --------------- | ------------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------- |
| Moderador       | Usuário autenticado que modera a comunidade | Anônimo ou usuário sem moderação               | Permite a operação / retorna 401 ou 403.                                  |
| ID da denúncia  | Denúncia existente na comunidade            | ID inexistente ou denúncia de outra comunidade | Analisa denúncia / retorna 404.                                           |
| `status`        | `pendente`                                  | `aprovada` ou `reprovada`                      | Permite decisão / retorna 400.                                            |
| Ação            | `aprovar` ou `reprovar`                     | Qualquer outro valor                           | Atualiza denúncia / retorna 400.                                          |
| Alvo denunciado | Postagem ou comentário existente            | Alvo já removido                               | Remove o alvo existente; registra decisão sem falhar se ele estiver nulo. |

### Fluxos principais

| Fluxo                | Dados de teste                                | Resultado esperado                                             | Resultado obtido | Situação      |
| -------------------- | --------------------------------------------- | -------------------------------------------------------------- | ---------------- | ------------- |
| Visualizar pendentes | Moderador; comunidade com denúncias pendentes | HTTP 200; retorna somente denúncias pendentes e os contadores. | -                | Não executado |
| Sem pendências       | Moderador; comunidade sem denúncias pendentes | HTTP 200; lista vazia e mensagem de que não há pendências.     | -                | Não executado |
| Aprovar              | Denúncia pendente de postagem ou comentário   | Conteúdo denunciado removido e denúncia sai da fila.           | -                | Não executado |
| Reprovar             | Denúncia pendente                             | Conteúdo permanece e denúncia sai da fila.                     | -                | Não executado |
| Repetir decisão      | Denúncia já aprovada/reprovada                | HTTP 400; decisão e auditoria não são sobrescritas.            | -                | Não executado |

### Análise de valor limite

Os IDs das rotas representam chaves positivas. A AVL testa o valor abaixo do primeiro ID possível, um ID válido e o seguinte. Para `status`, a fronteira é a transição de `pendente` para um estado analisado.

| Campo                                        | Entrada                   | Resultado esperado                    |
| -------------------------------------------- | ------------------------- | ------------------------------------- |
| ID da comunidade/denúncia — abaixo do limite | `0`                       | HTTP 404.                             |
| ID da comunidade/denúncia — limite válido    | ID existente              | Operação executada normalmente.       |
| ID da comunidade/denúncia — acima do limite  | Próximo ID inexistente    | HTTP 404.                             |
| `status` — antes da decisão                  | `pendente`                | Aprovar ou reprovar retorna HTTP 200. |
| `status` — após a decisão                    | `aprovada` ou `reprovada` | Nova decisão retorna HTTP 400.        |
