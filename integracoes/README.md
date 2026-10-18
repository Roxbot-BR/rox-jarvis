# Integrações

Documente aqui integrações com APIs e serviços externos, por exemplo, Google Workspace, ERP, mensageria e gateways de ferramentas.

## Para cada integração

- Serviço e finalidade.
- Estado: proposta, em teste, validada, desativada.
- Documentação oficial consultada e data.
- Fluxo de autenticação e escopos necessários.
- Dados acessados e ações permitidas.
- Armazenamento e revogação de credenciais (descrever o mecanismo, nunca o valor).
- Limites de taxa, paginação, timeouts e idempotência.
- Tratamento de erros e observabilidade.
- Testes e evidências.

## Segurança

Credenciais devem ficar num cofre ou mecanismo seguro do servidor, associadas ao tenant/usuário apropriado. Nunca registrar segredos neste repositório, commits, exemplos, issues ou logs.
