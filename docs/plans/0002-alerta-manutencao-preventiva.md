# Plano: Alerta de Manutenção Preventiva — TODO List

## Contexto

O DoD (`DoD.md`) define a feature **Alerta de Manutenção Preventiva**, que verifica veículos elegíveis para manutenção (por km ou tempo) e notifica o gestor por e-mail. A análise do código revelou que já existe infraestrutura parcial:

- **api-frotas**: já tem `VeiculoManutencaoService.verificarNecessidade()` (verifica km), `NotificacaoClient` (chama api-notificacoes), `VeiculoNotificacaoService.notificarManutencao()`, `VeiculoProperties` com limite de km e intervalo em meses, `NotificacaoFalhaException`. Porém **não existe** entidade `Alerta`, tabela de alertas, endpoint de verificação automática, nem registro de status da notificação.
- **api-notificacoes** (.NET 6): já tem `POST /api/notificacoes` (cria + envia), retorna status (ENVIADO/PENDENTE/FALHA), `POST /api/notificacoes/processar` (reprocessa pendentes). Porém o envio é **fake** (simulado com delay).
- **painel-admin** (Angular 18): já tem `ManutencoesComponent` e `FrotasService`, mas **não existe** tela de alertas.

---

## TODO List

### 1. API Frotas — Modelo e Banco de Dados

- [ ] **1.1** Criar migration Flyway `V3__alertas_manutencao.sql` com tabela `alertas_manutencao`:
  - `id` (BIGSERIAL PK), `veiculo_id` (BIGINT FK → veiculos), `tipo_alerta` (VARCHAR: 'QUILOMETRAGEM' ou 'TEMPO'), `quilometragem_atual` (BIGINT), `limite_quilometragem` (BIGINT), `intervalo_meses` (INT), `data_ultima_manutencao` (TIMESTAMP), `status_notificacao` (VARCHAR: 'ENVIADA', 'PENDENTE', 'FALHA'), `notificacao_id` (BIGINT, id retornado pelo api-notificacoes), `data_alerta` (TIMESTAMP), `data_resolucao` (TIMESTAMP)
  - Índices em `veiculo_id`, `status_notificacao`, `data_alerta`

- [ ] **1.2** Criar entidade JPA `AlertaManutencao` em `domain/AlertaManutencao.java`
- [ ] **1.3** Criar enum `TipoAlerta` em `domain/TipoAlerta.java` (QUILOMETRAGEM, TEMPO)
- [ ] **1.4** Criar enum `StatusNotificacao` em `domain/StatusNotificacao.java` (ENVIADA, PENDENTE, FALHA)
- [ ] **1.5** Criar `AlertaManutencaoRepository` em `repository/AlertaManutencaoRepository.java` com queries:
  - `findByVeiculoId(Long veiculoId)`
  - `findByStatusNotificacao(String status)`
  - `existsByVeiculoIdAndStatusNotificacao(Long veiculoId, String status)` (evitar alertas duplicados pendentes)

### 2. API Frotas — Lógica de Verificação de Elegibilidade

- [ ] **2.1** Estender `VeiculoManutencaoService` (ou criar `AlertaManutencaoService`) com método `verificarVeiculosElegiveis()` que:
  - Busca veículos com `status = 'ATIVO'` e `quilometragem >= limiteManutencao` (já existe `VeiculoProperties.quilometragem.limiteManutencao = 50000`)
  - Busca veículos cuja última manutenção foi há mais de X meses (`VeiculoProperties.manutencao.intervaloMeses = 3`)
  - Filtra veículos que ainda não possuem alerta pendente/não resolvido
  - Para cada veículo elegível, chama o fluxo de alerta (item 3)

- [ ] **2.2** Adicionar query em `VeiculoRepository` para buscar veículos sem manutenção há X meses (join com `manutencoes` ou subquery verificando `data_manutencao` mais recente)

- [ ] **2.3** Criar endpoint `GET /veiculos/alertas-manutencao` em `VeiculoController` (ou novo `AlertaManutencaoController`) que invoca `verificarVeiculosElegiveis()` e retorna a lista de alertas

### 3. API Frotas — Comunicação com API Notificações e Registro

- [ ] **3.1** Modificar `NotificacaoClient.enviarNotificacao()` para retornar o objeto de resposta do api-notificacoes (inclusive o `id` e `status`), em vez de `void`. Atualmente o método é `void` e não captura a resposta.

- [ ] **3.2** No fluxo de alerta, após chamar `NotificacaoClient`:
  - **Sucesso**: registrar alerta com `status_notificacao = 'ENVIADA'` e salvar `notificacao_id` retornado
  - **Falha** (api-notificacoes fora): registrar alerta com `status_notificacao = 'PENDENTE'` (capturar exceção, não propagar)

- [ ] **3.3** Garantir que `VeiculoNotificacaoService.notificarManutencao()` propague o status retornado do `NotificacaoClient` para o chamador decidir o status do alerta

### 4. API Notificações — Endpoint de Recebimento

- [ ] **4.1** O endpoint `POST /api/notificacoes` já existe e atende ao requisito (recebe pedido de alerta, envia e-mail, retorna status). Confirmar que o retorno inclui `id` e `status` — **já implementado** via `NotificacaoResponse`.

- [ ] **4.2** Verificar se o DTO `NotificacaoRequest` suporta campo `ServicoOrigem = "api-frotas"` e `Tipo = "ALERTA_MANUTENCAO"` — **já suportado**, ambos os campos existem.

- [ ] **4.3** (Opcional) Adicionar template de notificação específico para `ALERTA_MANUTENCAO` na tabela `templates_notificacao` via seed SQL, com variáveis `{{placa}}`, `{{quilometragem}}`, `{{limite_km}}`

### 5. Painel Admin — Tela de Alertas

- [ ] **5.1** Criar interface `AlertaManutencao` em `models/index.ts` com campos: `id`, `veiculo_id`, `veiculo_placa`, `tipo_alerta`, `quilometragem_atual`, `limite_quilometragem`, `status_notificacao`, `data_alerta`

- [ ] **5.2** Criar componente `AlertasComponent` em `components/alertas/alertas.component.ts` com:
  - Tabela listando alertas com colunas: Veículo, Tipo de Alerta, KM Atual, Limite, Status Notificação, Data do Alerta
  - Filtro/combobox por `status_notificacao` (ENVIADA, PENDENTE, FALHA)
  - Badge de status estilizado (verde p/ ENVIADA, amarelo p/ PENDENTE, vermelho p/ FALHA)

- [ ] **5.3** Adicionar rota `{ path: 'alertas', component: AlertasComponent }` em `app.routes.ts`

- [ ] **5.4** Adicionar link "Alertas" no sidebar (`sidebar.component.ts`)

- [ ] **5.5** Criar serviço `AlertasService` (ou estender `FrotasService`) com métodos:
  - `getAlertas(status?: string): Promise<AlertaManutencao[]>`
  - `verificarElegiveis(): Promise<AlertaManutencao[]>` (chama GET /veiculos/alertas-manutencao)

### 6. Testes

- [ ] **6.1** Testes unitários em `api-frotas` (JUnit + Mockito):
  - `AlertaManutencaoServiceTest`: verificar que veículos acima do limite de km são detectados como elegíveis
  - `AlertaManutencaoServiceTest`: verificar que veículos sem manutenção há X meses são detectados
  - `AlertaManutencaoServiceTest`: verificar que não cria alerta duplicado para veículo com alerta pendente
  - `AlertaManutencaoServiceTest`: verificar que falha no api-notificacoes gera alerta com status PENDENTE
  - `AlertaManutencaoServiceTest`: verificar que sucesso no api-notificacoes gera alerta com status ENVIADA

- [ ] **6.2** Testes unitários em `api-notificacoes` (xUnit):
  - `NotificacaoServiceTest`: verificar que notificação do tipo ALERTA_MANUTENCAO é criada e processada
  - `NotificacaoServiceTest`: verificar que falha no envio marca status como FALHA após max tentativas

- [ ] **6.3** Teste end-to-end do fluxo completo:
  - Registrar veículo com km alto → chamar endpoint de verificação → verificar alerta criado com status ENVIADA
  - Simular api-notificacoes indisponível → verificar alerta com status PENDENTE

- [ ] **6.4** Testes do componente Angular (Jest):
  - `AlertasComponent`: renderiza tabela de alertas, aplica filtro por status

### 7. Tratamento de Falha e Resiliência

- [ ] **7.1** Implementar lógica de fallback no `NotificacaoClient`: se api-notificacoes estiver fora, capturar exceção e não propagar — permitir registro do alerta como PENDENTE
- [ ] **7.2** (Opcional) Adicionar endpoint `POST /alertas-manutencao/{id}/reenviar` que re-tenta a notificação para alertas com status PENDENTE
- [ ] **7.3** (Opcional) Adicionar endpoint `POST /alertas-manutencao/processar-pendentes` que re-tenta todos os alertas PENDENTE

---

## Arquivos Críticos a Modificar

| Serviço | Arquivo | Ação |
|---------|---------|------|
| api-frotas | `src/main/resources/db/migration/V3__alertas_manutencao.sql` | **Criar** |
| api-frotas | `domain/AlertaManutencao.java` | **Criar** |
| api-frotas | `domain/TipoAlerta.java` | **Criar** |
| api-frotas | `domain/StatusNotificacaoAlerta.java` | **Criar** |
| api-frotas | `repository/AlertaManutencaoRepository.java` | **Criar** |
| api-frotas | `service/AlertaManutencaoService.java` | **Criar** |
| api-frotas | `controller/AlertaManutencaoController.java` | **Criar** |
| api-frotas | `service/NotificacaoClient.java` | **Modificar** — retornar resposta |
| api-frotas | `service/VeiculoNotificacaoService.java` | **Modificar** — propagar status |
| api-frotas | `repository/VeiculoRepository.java` | **Modificar** — query veículos sem manutenção há X meses |
| api-notificacoes | `Data/seed.sql` | **Modificar** — template ALERTA_MANUTENCAO |
| painel-admin | `models/index.ts` | **Modificar** — interface AlertaManutencao |
| painel-admin | `components/alertas/alertas.component.ts` | **Criar** |
| painel-admin | `services/alertas.service.ts` | **Criar** |
| painel-admin | `app.routes.ts` | **Modificar** — rota /alertas |
| painel-admin | `components/layout/sidebar.component.ts` | **Modificar** — link Alertas |

## Ordem de Implementação Sugerida

1. Migration + Entidade + Repository (itens 1.1–1.5)
2. Service de verificação de elegibilidade (itens 2.1–2.2)
3. Modificar NotificacaoClient para retornar resposta (item 3.1)
4. Lógica de alerta com registro de status (itens 3.2–3.3)
5. Controller endpoint (item 2.3)
6. Seed de template no api-notificacoes (item 4.3)
7. Tela de alertas no painel-admin (itens 5.1–5.5)
8. Testes unitários (itens 6.1–6.4)
9. Tratamento de falha e resiliência (itens 7.1–7.3)

## Verificação End-to-End

1. Subir api-frotas + api-notificacoes + painel-admin
2. Chamar `GET /veiculos/alertas-manutencao` — deve retornar veículos com km > 50000 ou sem manutenção há > 3 meses
3. Verificar no banco `alertas_manutencao` que alertas foram criados com `status_notificacao = 'ENVIADA'`
4. Derrubar api-notificacoes → chamar endpoint novamente → verificar alertas com `status_notificacao = 'PENDENTE'`
5. No painel-admin: navegar para `/alertas`, filtrar por status, verificar renderização
6. Re-subir api-notificacoes → chamar reenvio → verificar status mudou para ENVIADA
