# Integração Hermes Agent ↔ JARVIS Cloud

- **Estado:** proposta técnica; não representa implementação concluída.
- **Versão:** 1.0
- **Última revisão:** 2026-10-18
- **Objetivo:** definir uma integração gradual, auditável e segura entre o JARVIS Cloud, o runtime Hermes Agent e o cérebro compartilhado deste repositório.

> **Regra principal:** este documento orienta a investigação e a implementação. Não comprova que qualquer componente, API, skill ou controle esteja disponível ou ativo. Não autoriza mudanças em produção.

## 1. Objetivos

A integração deve permitir que o JARVIS use capacidades de agente do Hermes sem perder o controle do produto, da identidade dos usuários, das permissões, da memória privada e da auditoria.

1. O JARVIS Cloud continua responsável por usuários, autenticação, interface, tenants, permissões, configurações e gestão das conexões.
2. O Hermes atua como runtime de agente para planejamento, execução de ferramentas e aplicação de skills realmente disponíveis.
3. Este repositório é fonte versionada de conhecimento compartilhável, não banco de conversas nem cofre de credenciais.
4. A memória privada permanece em armazenamento próprio, isolado por usuário/tenant.
5. O agente Windows/MSI é um componente separado, autenticado e limitado. O Hermes no servidor não controla automaticamente o desktop do usuário.
6. Cada tarefa tem estado, trilha de auditoria, critérios de conclusão e evidência verificável.

## 2. Escopo

### Incluído

- Consulta controlada ao cérebro compartilhado.
- Seleção de contexto pertinente à tarefa.
- Integração do JARVIS com uma interface documentada e suportada da versão instalada do Hermes.
- Separação entre conhecimento compartilhado e memória privada.
- Controle de ferramentas, permissões, aprovações e auditoria.
- Prova de conceito com testes funcionais, de segurança e isolamento.

### Não incluído na primeira fase

- Substituir todo o JARVIS pelo Hermes.
- Copiar indiscriminadamente o estado interno do Hermes para o produto.
- Conceder acesso irrestrito ao sistema operacional, navegador, terminal ou APIs.
- Executar ações externas só porque uma skill ou documento as descreve.
- Armazenar tokens, senhas, conversas privadas ou dados pessoais no Git.
- Alterar serviços existentes, SSH, firewall ou DNS sem plano aprovado, backup e rollback.

## 3. Arquitetura lógica proposta

```text
Usuário / PWA / Interface Web
            |
            v
       JARVIS Cloud
  - autenticação e sessão
  - tenant e autorização
  - política de ações
  - seleção de contexto
  - aprovação humana
  - auditoria e estado da tarefa
            |
            v
   Adaptador de Runtime JARVIS
  - valida identidade e entrada
  - aplica allowlist de ferramentas
  - normaliza respostas e erros
  - impõe timeout/cancelamento
            |
            v
       Hermes Agent
  - planejamento e execução
  - skills disponíveis
  - uso de ferramentas permitido
            |
            +---------------------+
            |                     |
            v                     v
 Cérebro compartilhado       Memória privada
 Git: rox-jarvis             Serviço/DB isolado
 conhecimento aprovado       conversas e preferências
            |
            v
 Ferramentas autorizadas / APIs / MCP
            |
            +----> Conector Windows autenticado
            +----> Integrações SaaS com escopo mínimo
            +----> Sistemas internos autorizados
```

O diagrama é conceitual. Componentes e interfaces exatos devem ser confirmados por inspeção do código do JARVIS e da versão real do Hermes.

### Responsabilidades

**JARVIS Cloud**
- Autenticar o usuário e determinar o tenant.
- Autorizar cada operação.
- Definir ferramentas permitidas, parâmetros e limites.
- Gerenciar credenciais de integrações com segurança.
- Manter estado da tarefa, aprovações e auditoria.
- Impedir que o runtime receba dados de outros usuários.

**Adaptador de Runtime**
- Isolar o produto das particularidades da interface do Hermes.
- Validar esquemas de entrada e saída.
- Associar cada execução a um identificador de tarefa e tenant.
- Aplicar limites de tempo, contexto, concorrência e cancelamento.
- Representar falhas explicitamente, sem simular sucesso.

**Hermes Agent**
- Usar apenas ferramentas e skills disponíveis e autorizadas.
- Trabalhar com o contexto fornecido para a tarefa.
- Retornar resultado, ações tentadas, erros e evidências.
- Não ser tratado como fronteira de segurança por si só.

**Cérebro compartilhado**
- Guardar princípios gerais, procedimentos validados e decisões compartilháveis.
- Ser versionado e revisado por Git.
- Não guardar segredos, conversas privadas, memória bruta ou dados exclusivos de usuários.

**Memória privada**
- Armazenar contexto exclusivo do usuário/tenant conforme política de retenção.
- Aplicar autorização no servidor a toda leitura e escrita.
- Não ser publicada no repositório por sincronização automática.

**Conector Windows**
- Executar somente ações autorizadas no dispositivo pareado.
- Autenticar o dispositivo e validar escopo, expiração e revogação.
- Registrar comandos e resultados sem expor credenciais.
- Separar operações de leitura de ações que alterem arquivos, aplicativos ou configurações.

## 4. Interface do adaptador

Não presumir que uma interface específica do Hermes está disponível. Antes de implementar, verificar a versão instalada, a documentação oficial correspondente e as interfaces realmente expostas. Se houver gateway suportado com protocolo estruturado, avaliar essa opção; caso contrário, selecionar API documentada ou processo controlado. Evitar parsing frágil de terminal quando houver interface estruturada.

O adaptador deve oferecer ao JARVIS operações internas equivalentes a:

- **start_task:** inicia uma tarefa com identidade, objetivo, contexto autorizado e política de ferramentas.
- **get_task_status:** retorna estado e horário da última atualização.
- **get_task_result:** retorna resultado final, evidências e limitações.
- **cancel_task:** solicita cancelamento e informa se foi aceito.
- **health_check:** verifica disponibilidade sem executar tarefas com efeitos colaterais.

Esses nomes são um contrato conceitual, não endpoints existentes.

### Entrada mínima

- Identificador único da tarefa.
- Tenant e usuário internos validados pelo JARVIS.
- Objetivo da tarefa.
- Contexto mínimo necessário, filtrado pelo JARVIS.
- Lista permitida de ferramentas, calculada no servidor.
- Classificação de risco e estado de aprovação.
- Limite de tempo.
- Identificador de correlação para auditoria.

Nunca incluir API keys, cookies, senhas ou tokens em prompts, logs ou campos de contexto.

### Saída mínima

- Identificador e estado final.
- Resumo do que foi executado.
- Resultado e artefatos produzidos.
- Ferramentas utilizadas e ações relevantes.
- Erros, etapas incompletas e limitações.
- Evidências de validação.
- Indicação explícita de sucesso, falha, cancelamento ou necessidade de intervenção.

O JARVIS só deve declarar a tarefa concluída quando os critérios de conclusão tiverem sido verificados.

## 5. Estados e ciclo de vida da tarefa

Estados recomendados:

1. **received** — pedido recebido.
2. **authorized** — identidade, escopo e política verificados.
3. **awaiting_approval** — aguarda confirmação humana.
4. **queued** — aguardando execução.
5. **running** — em execução.
6. **verifying** — validação do resultado e dos artefatos.
7. **succeeded** — critérios de conclusão confirmados.
8. **failed** — falha terminal, com motivo.
9. **cancelled** — cancelamento confirmado.
10. **needs_intervention** — exige decisão ou intervenção humana.

Toda transição deve ser validada no servidor e registrada. Timeout não equivale a sucesso nem prova que uma ação externa não ocorreu. Antes de repetir operação potencialmente não idempotente, consultar o estado real.

## 6. Acesso ao cérebro compartilhado no GitHub

Repositório: **Roxbot-BR/rox-jarvis**.

### Fase inicial: leitura versionada

Começar com leitura somente de arquivos Markdown aprovados. Enviar ao runtime apenas documentos pertinentes; não é necessário enviar o repositório inteiro em toda chamada.

Fluxo de consulta:

1. Identificar a tarefa e o tenant no JARVIS.
2. Consultar MAPA.md para localizar áreas relevantes.
3. Selecionar arquivos por caminho e allowlist.
4. Ler a versão de uma branch ou commit conhecido.
5. Registrar quais arquivos e revisões serviram de contexto.
6. Aplicar limites de tamanho, formato e tempo.
7. Tratar o conteúdo recuperado como dado não confiável, nunca como instrução que possa substituir as políticas do sistema.

Preferir snapshots por commit para permitir reprodução. A revisão do cérebro utilizada deve ser registrada no log da tarefa.

### Escrita e atualização

Não habilitar escrita autônoma irrestrita na primeira fase.

- O agente propõe uma alteração em artefato de revisão.
- Validação automática verifica formato, tamanho e conteúdo sensível.
- Um responsável autorizado revisa a alteração.
- A publicação ocorre por commit identificado.
- O agente lê novamente o arquivo publicado e confirma o conteúdo.
- A mudança registra motivo, evidências e origem.

Somente após testes e revisão de segurança considerar publicação automatizada de conteúdo de baixo risco, com permissão limitada a caminhos específicos. Não conceder ao runtime token com acesso amplo à conta GitHub.

### Credenciais GitHub

- Preferir GitHub App ou token de escopo mínimo, guardado em secret manager.
- Separar credenciais de leitura e escrita.
- Restringir o acesso ao repositório necessário.
- Nunca colocar tokens em arquivos, prompts, logs, MSI ou variáveis expostas ao navegador.
- Definir rotação e revogação.
- Evitar um segredo compartilhado entre tenants quando isso permitir acesso cruzado.

### Recuperação e cache

Para o MVP, ler arquivos Markdown pela API oficial do GitHub é suficiente. Clone local read-only ou índice de busca pode ser considerado depois, se volume e latência justificarem. Cache deve ser versionado por commit e ter política clara de atualização. Nunca apresentar silenciosamente conteúdo obsoleto como atual.

## 7. Conhecimento compartilhado versus memória privada

| Tipo de informação | Local recomendado | Pode entrar no Git compartilhado? |
|---|---|---|
| Princípios gerais do JARVIS | identidade/ | Sim, após revisão |
| Arquitetura não confidencial | arquitetura/ | Sim, após revisão |
| Procedimento reutilizável | skills/ ou operacao/ | Sim, sem segredos |
| Decisão generalizável | memoria/ | Sim, se aprovada |
| Conversa e histórico do usuário | armazenamento privado | Não |
| Preferências pessoais | memória privada | Não por padrão |
| Tokens, senhas, cookies e API keys | secret manager | Nunca |
| Dados de ERP, clientes ou faturamento | sistema de origem/armazenamento autorizado | Não por padrão |
| Logs com identificadores pessoais | observabilidade com controles | Não |

O filtro deve ocorrer antes de publicar. Remover um segredo depois do commit não garante sua remoção do histórico Git.

## 8. Isolamento multiusuário e segurança

Separar nomes de perfil ou usar prompts diferentes não é isolamento suficiente.

Requisitos obrigatórios antes de produção:

1. Validar tenant e usuário a partir da sessão autenticada no servidor.
2. Aplicar autorização em cada leitura de memória, chamada de ferramenta, acesso a artefato e integração.
3. Separar memória, conexões, credenciais, arquivos temporários e histórico de execução por tenant.
4. Não permitir que o agente escolha livremente o tenant para obter acesso a dados.
5. Usar identidade de serviço com privilégio mínimo.
6. Proteger a comunicação entre serviços com autenticação e transporte seguro.
7. Definir limites de CPU, memória, concorrência, tempo e tamanho de arquivos.
8. Aplicar allowlist de ferramentas e validar argumentos no servidor.
9. Tratar documentos, páginas web, e-mails e arquivos como entradas potencialmente hostis; mitigar prompt injection e exfiltração.
10. Auditar eventos sem guardar segredos ou conteúdo pessoal desnecessário.
11. Implementar cancelamento, timeout, rate limit e proteção contra repetição.
12. Testar explicitamente acesso cruzado entre tenants, inclusive por IDs manipulados, cache e retomada de tarefa.

### Aprovação humana

Exigir aprovação explícita, conforme política de risco, para:

- Enviar mensagens ou e-mails a terceiros.
- Publicar conteúdo.
- Alterar ou excluir dados.
- Executar comandos com privilégios elevados.
- Fazer compras, pagamentos ou alterações financeiras.
- Modificar infraestrutura, rede, firewall, DNS ou produção.
- Transmitir dados pessoais ou confidenciais a outro sistema.

A aprovação deve estar vinculada ao conteúdo exato da ação, ao alvo e ao usuário aprovador. Se os parâmetros mudarem, solicitar nova aprovação.

## 9. Skills e ferramentas

Antes de usar uma skill:

1. Confirmar que existe na versão instalada.
2. Ler suas instruções e dependências.
3. Verificar se as ferramentas necessárias estão disponíveis.
4. Conferir escopos e permissões do tenant.
5. Executar testes em ambiente controlado.
6. Verificar o resultado e registrar limitações.

Uma skill não substitui uma integração API, um runtime de automação de navegador ou um agente Windows. Não declarar suporte a controle de GUI, ERP, navegador ou planilhas até que a capacidade esteja instalada e testada no ambiente correspondente.

## 10. Integrações SaaS e conector Windows

### SaaS

- Usar OAuth/API oficial quando disponível.
- Solicitar escopos mínimos e explicar o uso.
- Associar cada conexão ao tenant/usuário correto.
- Armazenar tokens no servidor em mecanismo seguro, criptografado e revogável.
- Respeitar limites de taxa, paginação, expiração e erros transitórios.
- Registrar operações sem gravar tokens ou conteúdo sensível.

Um agregador de integrações pode reduzir o trabalho de conectividade, mas não substitui autorização por tenant, gestão de credenciais, auditoria nem ferramentas locais.

### Windows/MSI

- Tratar o conector como agente separado.
- Exigir pareamento explícito, identidade de dispositivo, expiração e revogação.
- Validar autenticidade das mensagens e autorização no servidor.
- Implementar allowlist de ações e confirmação para operações de risco.
- Evitar comandos arbitrários de shell como interface padrão.
- Usar canal seguro, heartbeat limitado e auditoria.
- Não reutilizar tokens embutidos em instaladores ou configurações distribuídas.
- Confirmar o resultado no computador-alvo; não presumir execução em caso de desconexão.

## 11. Observabilidade e auditoria

Para cada tarefa, registrar no mínimo:

- Identificador da tarefa e correlação.
- Tenant e identidade de serviço em forma interna apropriada.
- Horários de início, transições e término.
- Versão do runtime e do adaptador.
- Commit do cérebro consultado.
- Ferramentas chamadas e escopos aplicados.
- Aprovações concedidas ou recusadas.
- Resultado da validação, erros e motivo da falha.

Não registrar segredos, tokens, cookies, senhas ou transcrições completas por padrão. Definir retenção, acesso, exportação e exclusão de logs. Proteger os logs contra alteração indevida.

## 12. Plano de implementação por fases

### Fase 0 — Descoberta e inventário

- Identificar versões reais do JARVIS, Hermes e bibliotecas.
- Consultar a documentação oficial da versão instalada.
- Mapear APIs, CLI, gateway, ferramentas e skills disponíveis.
- Documentar topologia, portas, autenticação e armazenamento.
- Não alterar produção nesta fase.

**Saída:** relatório do estado atual, interfaces suportadas, riscos e plano de teste.

### Fase 1 — Leitura do cérebro

- Implementar leitura somente dos arquivos Markdown autorizados.
- Usar allowlist de caminhos e impedir path traversal.
- Limitar tamanho, tempo e tipos de arquivo.
- Fixar commit/revisão por execução.
- Testar rede indisponível, arquivo inexistente e conteúdo inválido.

**Aceite:** recuperar arquivo correto, registrar revisão e falhar com segurança se GitHub estiver indisponível.

### Fase 2 — Adaptador Hermes em teste

- Criar contrato interno independente da interface específica do Hermes.
- Implementar health check, início, consulta de estado, resultado e cancelamento.
- Validar esquemas, limites, timeouts e logs sem segredos.
- Não expor o runtime diretamente ao navegador.

**Aceite:** tarefa simples executa e retorna estado verificável; falhas são representadas corretamente.

### Fase 3 — Memória privada e isolamento

- Integrar o armazenamento privado já escolhido pelo JARVIS.
- Aplicar tenant no servidor, sem confiar em parâmetros do cliente.
- Testar dois usuários com dados e conexões distintos.
- Testar IDs manipulados, cache, artefatos e retomada de tarefa.

**Aceite:** dados privados do usuário A não podem ser acessados pelo usuário B.

### Fase 4 — Ferramentas e aprovações

- Habilitar uma ferramenta não destrutiva por vez.
- Criar allowlist por tenant e validação de argumentos.
- Implementar aprovação para ações de risco.
- Validar saída com evidência independente.
- Testar timeout, cancelamento e repetição.

**Aceite:** ações não autorizadas são bloqueadas no servidor; ações aprovadas deixam trilha auditável.

### Fase 5 — Escrita controlada no GitHub

- Começar com proposta de alteração, sem commit automático.
- Validar formato e conteúdo sensível.
- Exigir revisão humana.
- Usar credencial de escopo mínimo.
- Confirmar conteúdo publicado e registrar commit.

**Aceite:** publicar somente no caminho autorizado, sem segredos ou dados privados.

### Fase 6 — Piloto

- Executar tarefas reais de baixo risco.
- Medir latência, falhas, custo e qualidade de recuperação.
- Testar carga e isolamento.
- Documentar rollback e operação.
- Aprovar produção somente após revisão dos critérios.

**Aceite:** testes críticos aprovados, sem falhas conhecidas de isolamento e com recuperação validada.

## 13. Matriz mínima de testes

| ID | Cenário | Resultado esperado |
|---|---|---|
| INT-001 | Consultar arquivo existente | Conteúdo correto e commit registrado |
| INT-002 | Arquivo inexistente | Erro explícito; sem fallback silencioso |
| INT-003 | GitHub indisponível | Falha controlada; cache só se versionado e sinalizado |
| INT-004 | Caminho com ../ | Requisição bloqueada |
| INT-005 | Documento com instrução hostil | Conteúdo tratado como dado; política de sistema preservada |
| INT-006 | Usuário A consulta memória de B | Acesso negado e evento auditado |
| INT-007 | Token ausente ou expirado | Erro seguro; nenhum segredo no log |
| INT-008 | Runtime não responde | Timeout e estado recuperável |
| INT-009 | Tarefa cancelada | Cancelamento reportado honestamente; verificar efeitos já iniciados |
| INT-010 | Ferramenta não permitida | Bloqueio no servidor |
| INT-011 | Ação de risco sem aprovação | Não executar |
| INT-012 | Saída afirma sucesso sem evidência | Não marcar como concluída |
| INT-013 | Escrita fora de caminho autorizado | Bloqueio |
| INT-014 | Segredo de teste no arquivo | Publicação bloqueada |
| INT-015 | Repetição não idempotente | Sem duplicação ou com estado verificado antes de repetir |
| INT-016 | Conector Windows desconectado | Falha segura; sem presumir execução |
| INT-017 | Tenant manipulado pelo cliente | Identidade validada pela sessão do servidor |
| INT-018 | Cache com versão antiga | Versão explicitada; não apresentada como atual |

## 14. Rollback e recuperação

Antes de qualquer implantação:

1. Registrar versão atual e configuração relevante sem segredos.
2. Fazer backup conforme política do ambiente.
3. Manter o adaptador desacoplado para desativá-lo sem remover o JARVIS.
4. Usar feature flag para habilitar/desabilitar a integração.
5. Em falha, desativar o caminho Hermes e preservar as funcionalidades anteriores.
6. Revogar credenciais comprometidas e revisar logs.
7. Validar saúde do serviço e integridade dos dados após rollback.

Não executar rollback destrutivo nem reiniciar serviços críticos sem compreender o impacto e obter a autorização necessária.

## 15. Decisões pendentes

- Qual versão exata do Hermes será usada?
- Qual interface oficial e suportada existe nessa versão?
- Como o JARVIS autentica usuários e representa tenants hoje?
- Onde a memória privada está armazenada e quais garantias de isolamento existem?
- Qual mecanismo de secrets está disponível no servidor?
- Quais ferramentas serão habilitadas primeiro?
- O conector Windows atual possui autenticação, autorização, assinatura e revogação adequadas?
- Qual política de retenção de conversas e logs será aplicada?
- Qual fluxo de aprovação humana existirá na interface?

Esses pontos devem ser verificados no código e no ambiente real; não devem ser preenchidos por suposição.

## 16. Definição de pronto

A integração só pode ser considerada pronta para piloto quando:

- A versão e a interface do Hermes estiverem documentadas.
- O adaptador tiver contrato e testes.
- A leitura do cérebro for versionada e restrita.
- Memória e credenciais estiverem isoladas por tenant.
- Ferramentas tiverem allowlist e validação no servidor.
- Ações de risco exigirem aprovação adequada.
- Auditoria e logs não expuserem segredos.
- Testes de isolamento e falha estiverem passando.
- Houver rollback testado e responsável definido.
- A interface comunicar sucesso, falha e tarefas incompletas com clareza.

---

**Estado final:** especificação inicial para orientar investigação e implementação. Não comprova que qualquer componente, interface, skill ou controle acima já esteja disponível ou ativo.
