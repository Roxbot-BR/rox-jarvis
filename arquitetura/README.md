# Arquitetura — Princípios e Fronteiras

**Status:** princípios de projeto; confirmar com a implementação real antes de tratar como implementados.

## Separação conceitual

- **JARVIS Cloud:** produto, autenticação, interface, usuários, permissões, configurações e gestão de integrações/dispositivos.
- **Runtime de agente (por exemplo, Hermes):** planejamento, uso de ferramentas, execução de skills e recuperação de contexto conforme configuração.
- **Conector Windows/MSI:** canal autenticado e limitado para ações no computador autorizado. O runtime remoto não controla o desktop automaticamente.
- **Cérebro compartilhado (este repositório):** documentação e conhecimento aprovado para reutilização.
- **Memória privada por usuário:** conversas, preferências, conexões e dados exclusivos; armazenar fora deste repositório, com isolamento real.

## Requisitos de segurança

1. Nunca usar este repositório como depósito de tokens ou dados privados.
2. Aplicar autorização no servidor a cada operação; não confiar apenas no frontend ou em instruções de uma skill.
3. Isolar arquivos, memória, credenciais, tarefas e ferramentas por identidade/tenant.
4. Não considerar profiles ou nomes de agente como fronteira de segurança suficiente.
5. Proteger os conectores locais com autenticação forte, rotação/revogação de credenciais, escopo mínimo e auditoria.
6. Registrar ações e resultados sem incluir segredos em logs.
7. Definir timeout, cancelamento, idempotência e política de confirmação para operações sensíveis.

## Integração antes de migração

Validar primeiro uma prova de conceito pequena: tarefa completa com ferramenta real, retomada de contexto, logs, tratamento de falha e teste explícito de isolamento entre dois usuários. Não substituir componentes existentes antes de comprovar compatibilidade e plano de rollback.

## Estado

Este documento registra princípios e não comprova que controles específicos estejam implementados.
