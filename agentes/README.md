# Agentes — Papéis e responsabilidades

**Status:** modelo inicial para discussão, não inventário de agentes já implementados.

Agentes especializados podem organizar o trabalho, mas devem compartilhar políticas de segurança e permanecer dentro das permissões concedidas.

## Modelo de registro de agente

Para cada agente, documentar:
- Nome e finalidade.
- Tipos de tarefa permitidos e proibidos.
- Ferramentas disponíveis e escopos.
- Dados que pode consultar e modificar.
- Dependências e handoff para outros agentes.
- Ações que exigem aprovação humana.
- Critérios objetivos de conclusão.
- Testes e limitações conhecidas.

## Regras

- Não presumir que um agente pode delegar só porque sua descrição diz isso.
- Não compartilhar memória privada entre agentes por padrão.
- Delegação não amplia permissões.
- Resumir entradas para o próximo agente sem incluir segredos desnecessários.
