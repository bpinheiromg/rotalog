## Definição de Feito: Alerta de Manutenção Preventiva

### Critérios de Aceitação:

- rotalog-api-frotas: endpoint que verifica veículo com km > limite ou data > x meses
- rotalog-api-notificacoes: chamada HTTP para o rotalog-api-notificacoes quando encontrar veículo elegível
- rotalog-api-notificacoes: endpoint que recebe pedido de alerta e envia e-mail para o gestor
- rotalog-api-notificacoes: retorna status da notificação (enviada/falha)
- rotalog-api-frotas: registra alerta no banco com status da notificação
- painel-admin: tela que lista alertas por filtro por status
- Testes: unitários em cada serviço + teste end2end do fluxo completo
- Comunicação: HTTP síncrona entre serviços
- Tratamento de falha: se o rotalog-api-notificacoes estiver fora, o alerta é registrado como status "pendente"