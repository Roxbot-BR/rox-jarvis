# Operação e Runbooks

Documente procedimentos operacionais reproduzíveis e seguros.

## Modelo mínimo de runbook

- Objetivo e sistemas afetados.
- Pré-requisitos e autorização necessária.
- Avaliação de risco.
- Passos numerados, com comandos claramente identificados.
- Resultado esperado após cada etapa.
- Verificação independente.
- Procedimento de rollback/recuperação.
- Logs e evidências que devem ser coletados.
- Data, responsável e versões testadas.

## Regras para produção

- Não modificar SSH, firewall, DNS, dados ou serviços sem escopo e autorização explícitos.
- Preferir leitura/diagnóstico antes de alterações.
- Preservar backups e dados existentes.
- Não executar comandos destrutivos copiados de documentação sem revisar contexto e impacto.
