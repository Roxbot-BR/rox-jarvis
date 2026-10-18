# Integração ARTEMIS ↔ JARVIS Cloud ↔ Hermes Agent

- **Estado:** proposta técnica para avaliação e prova de conceito; não implementado.
- **Versão:** 1.0
- **Última revisão:** 2026-10-18
- **Projeto upstream:** https://github.com/google/artemis
- **Objetivo:** definir como ARTEMIS pode fornecer automação de dispositivos Android ao JARVIS, complementando o Hermes sem substituir a arquitetura central do produto.

> **Resumo executivo:** ARTEMIS é um candidato adequado para uma capacidade especializada: automação de aplicações e fluxos em telefones Android físicos ou emuladores, com interface MCP, CLI, console web e SDK Python. Não é um substituto do JARVIS Cloud nem, por si só, um agente geral de automação de PCs Windows. Recomenda-se integrá-lo como serviço de ferramentas isolado, com uma prova de conceito antes de qualquer implantação operacional.

## 1. O que ARTEMIS oferece

Com base no README e na documentação do repositório upstream consultados em 2026-10-18:

- Automação de tarefas Android a partir de instruções em linguagem natural.
- Execução de fluxos que atravessam diferentes aplicativos móveis.
- Localização de elementos pela hierarquia de interface, com alternativas visuais/por coordenadas em interfaces customizadas.
- Servidor MCP com ferramentas para iniciar/gerenciar tarefas móveis, consultar estado do dispositivo, inspecionar traces e diagnosticar problemas.
- CLI, console web visual e SDK Python remoto.
- Perfis de execução denominados Flash e Pro, com diferentes compromissos entre velocidade e planejamento/verificação.
- Captura de traces e dados de diagnóstico, incluindo screenshots e Logcat conforme a configuração.

O upstream anuncia resultados acima de 99% no benchmark AndroidWorld; isso é um resultado reportado pelo projeto, não garantia de sucesso em nossos dispositivos, aplicativos ou casos de uso.

Fontes primárias:
- https://github.com/google/artemis
- https://github.com/google/artemis/blob/main/mcp_server/README.md
- https://github.com/google/artemis/blob/main/packages/artemis-client/README.md
- https://github.com/google/artemis/blob/main/LICENSE

## 2. Como complementa JARVIS e Hermes

| Componente | Responsabilidade proposta | Não deve ser tratado como |
|---|---|---|
| JARVIS Cloud | Identidade, tenants, interface, autorização, aprovação, estado de tarefas, auditoria e memória | Um executor de toda ação diretamente |
| Hermes Agent | Planejamento e orquestração geral, uso das ferramentas realmente disponíveis | Uma fronteira de segurança suficiente |
| ARTEMIS | Ferramenta especializada em tarefas e testes de UI Android | Um substituto do runtime Hermes ou automação universal de Windows |
| Adaptador ARTEMIS do JARVIS | Contrato interno, autorização, filtragem de parâmetros, limites, normalização e auditoria | Um proxy irrestrito para a API de administração |
| Cérebro compartilhado `rox-jarvis` | Documentação e procedimentos revisados | Banco de memória privada ou cofre de tokens |
| Memória privada do JARVIS | Contexto e histórico separados por usuário/tenant | Conteúdo sincronizado automaticamente ao GitHub |

### Fluxo lógico

```text
Usuário / PWA
     |
     v
JARVIS Cloud
  - autentica e identifica o tenant
  - valida permissões e risco
  - pede confirmação quando necessário
  - regista a tarefa
     |
     v
Adaptador ARTEMIS do JARVIS
  - valida pedido e dispositivo permitido
  - aplica allowlist e limites
  - chama a interface aprovada
     |
     v
ARTEMIS (host de dispositivos)
  - MCP / SDK / API de tarefas
  - agente de automação Android
  - ADB, ferramenta de dispositivo, modelo configurado
     |
     v
Dispositivo Android autorizado / emulador
     |
     v
Resultado, estado, trace e evidências
     |
     v
Adaptador verifica → JARVIS apresenta resultado
```

O Hermes pode decidir quando invocar a ferramenta ARTEMIS por meio do adaptador MCP ou de uma integração equivalente. A autorização deve continuar sob controle do JARVIS; não confiar apenas no raciocínio do agente ou nas instruções de uma skill.

## 3. Casos de uso possíveis

Após prova de conceito, ARTEMIS poderá ajudar em casos como:

1. Testar aplicações Android e percorrer fluxos de login, configuração, navegação e formulários com contas de teste autorizadas.
2. Reproduzir bugs relatados, capturando traces e evidências.
3. Verificar uma aplicação após uma nova versão, observando alertas ou diálogos inesperados.
4. Executar roteiros móveis internos autorizados para suporte/diagnóstico.
5. Apoiar pipelines de QA e regressão em dispositivos dedicados ou emuladores.

Não assumir que todo aplicativo será automatizável. Aplicativos bancários, autenticação multifator, CAPTCHA, políticas anti-bot e interfaces protegidas podem exigir intervenção humana ou impedir automação.

## 4. Escolha da interface de integração

O upstream documenta mais de um modo. Recomenda-se escolher uma única via inicial para manter o escopo pequeno.

### Opção A — Python SDK remoto

O pacote `artemis-client` foi descrito pelo upstream como um cliente remoto sem dependências de runtime, que comunica com o daemon ARTEMIS por HTTP. A documentação lista operações como health, readiness, listar dispositivos, submeter tarefa, consultar seu estado, aguardar o resultado e interrompê-la.

**Vantagens:** adaptador tipado em Python, fluxo de tarefa assíncrono e possibilidade de integração com os testes já existentes.

**Ponto de atenção:** a documentação do cliente afirma explicitamente que a API administrativa atual não possui autenticação remota integrada. Não expor o daemon diretamente à internet. Usar túnel SSH na fase inicial ou um proxy seguro validado antes de qualquer uso remoto. O proxy deve exigir HTTPS, autenticação bearer, controle de escopo e exposição apenas das rotas necessárias.

### Opção B — Servidor MCP nativo

ARTEMIS inclui ferramentas MCP, entre elas operações relacionadas a tarefas móveis, estado de dispositivos, inspeção de traces e diagnóstico.

**Vantagens:** pode permitir que o runtime Hermes/um adaptador de ferramentas invoque as capacidades sem o JARVIS reimplementar todas as ações de dispositivo.

**Pontos de atenção:** validar compatibilidade real entre a versão do Hermes instalada e a implementação MCP do ARTEMIS; inspecionar transporte, processo de inicialização, ciclo de vida, política de timeout e autorização. Não presumir que um servidor MCP iniciado localmente esteja automaticamente seguro para uso multiusuário.

### Recomendação

Começar pelo SDK remoto dentro de um adaptador Python do JARVIS se isso for compatível com o ambiente atual. Testar MCP em seguida, se a conexão direta pelo runtime trouxer benefício claro. Não habilitar simultaneamente diversos caminhos para a mesma capacidade antes de definir responsabilidades, controle de acesso e observabilidade.

## 5. Requisito crítico: segurança de rede

A documentação upstream do SDK informa que o daemon atual não implementa autenticação remota integrada e orienta a não expor sua porta diretamente à internet. Isto é um bloqueador para exposição pública desprotegida.

Requisitos mínimos:

1. Executar inicialmente em máquina/VM de teste ou host de dispositivos segregado.
2. Manter o serviço ligado a loopback ou a uma interface privada sempre que possível.
3. Para acesso inicial remoto, usar túnel SSH com contas e chaves controladas, sem abrir a porta pública do daemon.
4. Para ambiente persistente, colocar proxy/reverse proxy à frente, com HTTPS, autenticação forte e autorização por operação e dispositivo.
5. Não considerar bearer token simples suficiente sem TLS, rotação, expiração, proteção contra replay e política de escopo.
6. Aplicar firewall com allowlist de origem e não alterar regras da infraestrutura existente sem plano e autorização.
7. Separar os dispositivos de teste dos dispositivos pessoais ou de produção.
8. Não guardar credenciais de aplicação Android em prompts, traces ou logs.
9. Não permitir que um tenant liste, controle ou inspecione o dispositivo de outro tenant.
10. Auditar tarefas e resultados sem armazenar conteúdo sensível além do necessário.

**Não expor a porta HTTP do ARTEMIS diretamente à internet.** A implantação não deve prosseguir enquanto não existir canal protegido e controle de autorização.

## 6. Identidade, dispositivos e multi-tenancy

O JARVIS deve manter um registro de dispositivos com, no mínimo:

- ID interno estável.
- Serial/identificador do dispositivo, guardado com controles adequados.
- Host ARTEMIS responsável.
- Tenant proprietário e usuários autorizados.
- Ambiente: teste, homologação ou produção.
- Estado de pareamento e saúde.
- Permissões permitidas.
- Data da última verificação.

Todas as chamadas devem resolver o dispositivo permitido pelo servidor. O usuário/agente não pode enviar um serial arbitrário e passar a controlar qualquer telefone acessível pelo host.

Se a instalação ARTEMIS não oferecer isolamento nativo por tenant suficiente, isolar os hosts/dispositivos por instância de serviço, credenciais e rede. Não presumir que sessões diferentes ou um ID de tarefa forneçam uma fronteira de segurança.

## 7. Política de execução e aprovação

Classificar as tarefas:

- **Baixo risco:** ler estado de um app de teste, abrir páginas de demonstração, capturar screenshot de tela sem dados pessoais.
- **Risco moderado:** preencher formulários de teste, alterar preferências em ambiente de homologação ou realizar ações reversíveis.
- **Alto risco:** enviar mensagens, publicar, realizar compras/pagamentos, apagar dados, alterar configurações críticas ou operar contas reais com dados sensíveis.

As de alto risco devem exigir aprovação humana explícita vinculada ao dispositivo, ação, aplicação e parâmetros exatos. A aprovação expira se esses parâmetros forem alterados.

Aplicar allowlist de ações e aplicativos, timeout, limite de passos, limite de duração e botão de cancelamento. A capacidade de parar uma tarefa não garante que efeitos externos já realizados sejam revertidos.

## 8. Contrato interno recomendado no JARVIS

Criar um serviço/adaptador com interface interna equivalente a:

- `artemis_health(device_group)`
- `artemis_list_authorized_devices()`
- `artemis_submit_task(task_id, device_id, objective, profile, policy)`
- `artemis_get_task(task_id)`
- `artemis_stop_task(task_id)`
- `artemis_get_trace(task_id)`

Estes nomes são ilustrativos; não afirmam a existência desses métodos na API upstream. Implementar o mapeamento para os métodos/endpoints que a versão instalada realmente suporta.

### Entradas

- ID único da tarefa gerado pelo servidor.
- Tenant e usuário derivados da sessão autenticada, nunca aceitos como autoridade a partir do cliente.
- ID interno do dispositivo autorizado.
- Objetivo e contexto minimizados.
- Perfil aprovado pela política do JARVIS.
- Limite de tempo e de execução.
- Estado de aprovação.
- ID de correlação de auditoria.

### Saídas

- Estado real da tarefa.
- ID de tarefa/trace upstream.
- Dispositivo resolvido.
- Resumo e resultado estruturado.
- Erro ou limitação, se houver.
- Artefatos de evidência autorizados.
- Indicação de verificação concluída ou pendente.

O adaptador deve normalizar exceções e timeouts e impedir que uma resposta textual do modelo seja interpretada como prova de sucesso. Quando a tarefa falhar ou o host ficar inacessível, exibir estado inconclusivo/falha em vez de sucesso.

## 9. Perfis Flash e Pro

O README upstream distingue os perfis:

- **Flash:** prioriza reação e velocidade; a documentação lista limitações, incluindo ausência de plano de tarefa e de verificação final/checkpoints.
- **Pro:** oferece planejamento e verificação mais profundos, com mais custo/latência por etapa.

Política sugerida para avaliação:

- Usar Flash somente em tarefas curtas, reversíveis, determinísticas e de baixo risco.
- Usar Pro em fluxos multi-etapas que precisam de plano, checkpoints e revisão final.
- Não permitir que o agente escolha livremente o nível de risco ou desative proteções.
- Fixar o perfil por política do servidor e registrar a seleção na auditoria.
- Validar a disponibilidade e o comportamento dos perfis na versão instalada.

Estas recomendações precisam ser confirmadas em testes com os dispositivos e modelos escolhidos.

## 10. Dados, traces e privacidade

Screenshots, hierarquias de interface, traces, Logcat, históricos de ações e vídeos podem conter dados pessoais, tokens temporários, mensagens ou informações de clientes.

- Guardar traces em armazenamento protegido e separado por tenant.
- Aplicar prazo de retenção e exclusão.
- Não publicar traces brutos no cérebro compartilhado.
- Não enviar traces automaticamente a provedores de IA sem política de dados aprovada.
- Redigir ou evitar capturar campos secretos sempre que possível.
- Permitir que o resultado compartilhe links controlados para evidências, em vez de anexar conteúdo sensível ao histórico de chat.
- Garantir que cada link de evidência verifique novamente a autorização.

## 11. Plano de prova de conceito

### Fase 0 — Inspeção

- Ler documentação do commit/release que será usado.
- Identificar sistema operacional host, versão Python, `uv`, ADB, dispositivo/emulador e modelo/provedor necessário.
- Validar licença, dependências e procedimento de atualização.
- Verificar a configuração de rede e ausência de exposição pública indevida.
- Nenhuma modificação no servidor JARVIS em produção.

**Saída:** ficha de compatibilidade e riscos.

### Fase 1 — Teste isolado local

- Clonar ARTEMIS em máquina de laboratório.
- Instalar dependências segundo o guia upstream.
- Conectar um emulador ou telefone de teste com depuração USB autorizada.
- Executar diagnóstico e uma tarefa simples que apenas leia uma configuração não sensível.
- Registrar duração, status, trace e resultado verificado.

**Aceite:** execução reproduzível, sucesso/falha inequívocos e dispositivo de laboratório isolado.

### Fase 2 — Adaptador de integração

- Implementar o contrato interno no JARVIS em ambiente de desenvolvimento.
- Adicionar timeout, cancelamento, polling e tratamento de host indisponível.
- Permitir apenas dispositivos registrados e ferramentas aprovadas.
- Adicionar testes unitários e de integração com cliente mockado e serviço real de laboratório.
- Não habilitar execução autônoma com contas pessoais ou produção.

**Aceite:** o JARVIS consegue enviar tarefa, consultar estado e recuperar resultado sem contornar permissões.

### Fase 3 — Segurança e isolamento

- Colocar o serviço atrás de canal privado e autenticação.
- Testar acesso sem credencial, credencial expirada, tenant incorreto e dispositivo não autorizado.
- Testar rede interrompida, tarefa duplicada, cancelamento e trace inacessível.
- Conferir que logs não contêm tokens ou dados sensíveis.
- Testar a revogação de dispositivo e credencial.

**Aceite:** nenhum dispositivo ou trace pode ser acessado por tenant não autorizado.

### Fase 4 — Integração com Hermes

- Expor ao Hermes apenas as ferramentas do adaptador aprovadas.
- Confirmar que o runtime Hermes não pode ignorar a autorização do JARVIS.
- Validar o caminho completo: intenção → autorização → aprovação quando necessária → tarefa ARTEMIS → evidência → verificação → resposta.
- Testar injeção de prompt em conteúdo da tela e verificar que não altera política de sistema nem escopo de ferramentas.

**Aceite:** fluxo completo demonstrado com logs e evidência, incluindo ao menos um cenário de falha controlada.

### Fase 5 — Piloto controlado

- Habilitar apenas usuários e dispositivos de teste.
- Medir sucesso por tipo de tarefa, duração, falha, intervenção humana e custo.
- Definir suporte, retenção e rollback.
- Ampliar escopo apenas após revisão.

**Aceite:** critérios de segurança, confiabilidade e operação aprovados pelo responsável do produto.

## 12. Matriz mínima de testes

| ID | Cenário | Resultado esperado |
|---|---|---|
| ART-001 | Health do host | Estado real devolvido |
| ART-002 | Dispositivo autorizado | Visível apenas aos usuários autorizados |
| ART-003 | Serial não autorizado | Bloqueado no servidor |
| ART-004 | Pedido sem autenticação | Negado |
| ART-005 | Porta do daemon sem proxy | Não acessível publicamente |
| ART-006 | Tarefa curta em dispositivo de teste | Estado e resultado verificáveis |
| ART-007 | Aplicativo inesperado ou diálogo de crash | Falha/alerta refletido no resultado |
| ART-008 | Host fica indisponível durante tarefa | Estado inconclusivo/falha, nunca sucesso presumido |
| ART-009 | Cancelamento | Estado de cancelamento informado, efeitos parciais reconhecidos |
| ART-010 | Repetição de requisição | Sem duplicar tarefa de forma inadvertida |
| ART-011 | Tenant A consulta trace de B | Acesso negado e evento auditado |
| ART-012 | Trace contém dado sensível | Não publicado no Git ou memória compartilhada |
| ART-013 | Ação de alto risco sem aprovação | Não executada |
| ART-014 | Tela contém prompt injection | Não eleva privilégios nem altera allowlist |
| ART-015 | Perfil incompatível | Falha explícita com sugestão segura |
| ART-016 | Atualização de ARTEMIS | Regressões detectadas antes de atualizar produção |

## 13. Licença e dependências

O repositório upstream declara licença Apache 2.0 e menciona código de terceiros em diretórios próprios com licenças/metadados específicos. Antes de distribuir, empacotar ou modificar ARTEMIS junto com o JARVIS:

- Ler o arquivo LICENSE da versão selecionada.
- Verificar licenças dos componentes de terceiros usados no build.
- Manter avisos e atribuições exigidos.
- Não assumir que todos os modelos, drivers ou aplicativos móveis usados durante execução têm a mesma licença.
- Fixar versão ou commit conhecido e revisar mudanças antes de atualizar.

## 14. Rollback

- Manter ARTEMIS como serviço opcional e desacoplado.
- Usar feature flag para desligar chamadas pelo JARVIS.
- Se houver incidente, bloquear acesso no proxy/rede e revogar credenciais, sem apagar evidências necessárias.
- Preservar o funcionamento de chat e ferramentas existentes do JARVIS.
- Validar que a desativação da integração não interrompa Hermes ou outros componentes.
- Não reiniciar ou alterar o servidor compartilhado sem avaliar o impacto e obter autorização.

## 15. Decisões pendentes

1. ARTEMIS será executado em um computador de testes dedicado, em VM ou em host Windows/Linux já existente?
2. Qual dispositivo Android físico ou emulador será o primeiro alvo?
3. O caso de uso principal é QA de aplicações, suporte operacional ou automação pessoal?
4. A interface inicial será SDK remoto ou MCP?
5. Qual provedor/modelo e quais custos serão aceitos?
6. Onde traces e screenshots serão armazenados e por quanto tempo?
7. Como será feita a autenticação do usuário que solicita a execução e a autorização do dispositivo?
8. Qual limite inicial de duração, passos e categorias de ação?
9. Qual versão exata do Hermes e do JARVIS está sendo usada no ambiente-alvo?

Não preencher esses pontos por suposição.

## 16. Decisão recomendada

**Prosseguir com uma prova de conceito isolada**, não com instalação direta no servidor principal.

Ordem recomendada:

1. Validar ARTEMIS localmente com um dispositivo de teste.
2. Confirmar interface e dependências na versão selecionada.
3. Construir adaptador seguro do JARVIS, com autenticação e autorização fora do daemon.
4. Integrar a ferramenta ao Hermes apenas depois de validar o adaptador.
5. Testar isolamento de tenant, traces, cancelamento, falha de rede e prompt injection.
6. Só então avaliar hospedagem central, começando com dispositivos de laboratório.

A documentação upstream ressalta que a API administrativa atual do daemon não oferece autenticação remota integrada; portanto, não expor a porta do serviço à internet diretamente.

**Estado final:** proposta de arquitetura e plano de POC. Nenhum software foi instalado, nenhum serviço foi alterado e nenhuma automação de dispositivo foi executada como parte da criação deste documento.
