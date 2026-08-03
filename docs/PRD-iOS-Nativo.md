# PRD — Chatwoot Agent para iOS (aplicativo nativo)

**Versão do documento:** 1.0
**Data:** 01/08/2026
**Baseline analisada:** `@chatwoot/mobile-app` v4.7.0 (React Native 0.79.6 / Expo SDK 53), branch `develop`, commit `8696302`
**Servidor Chatwoot suportado pela baseline:** ≥ 3.13.0 (aviso de upgrade abaixo de `EXPO_PUBLIC_MINIMUM_CHATWOOT_VERSION`, hoje 4.1.0)

---

## 1. Sumário executivo

O aplicativo mobile do Chatwoot é o console do **agente de atendimento** — não do cliente final. Ele se conecta a qualquer instalação Chatwoot (cloud ou self-hosted) informada pelo usuário, autentica via devise_token_auth, e entrega caixa de entrada, conversas em tempo real, composição de mensagens multicanal, ações de conversa (status, atribuição, prioridade, rótulos, participantes), notificações push, busca, macros e um assistente de IA (Captain/Copilot).

Este PRD especifica a **reescrita nativa em Swift/SwiftUI para iOS**, em duas fases:

- **Fase 1 — Paridade:** reproduzir 100% das funcionalidades da versão React Native atual, com contrato de API idêntico (sem exigir nenhuma mudança no servidor Chatwoot).
- **Fase 2 — Evolução:** funcionalidades que só se tornam viáveis (ou muito melhores) em nativo — widgets, Live Activities, Handoff, Siri/Atalhos, iPad/Mac, offline real, notificações acionáveis.

O documento é escrito para ser suficiente como especificação de implementação: contém o inventário funcional completo extraído do código, o contrato de API endpoint a endpoint, o protocolo de tempo real, o modelo de dados e os critérios de aceite.

### 1.1 Por que reescrever

| Dor na baseline RN | Efeito |
|---|---|
| New Architecture desabilitada (`newArchEnabled: false`) | Preso à ponte legada; ganhos de performance do RN moderno indisponíveis |
| 60+ dependências nativas, 2 com patch manual (`ffmpeg-kit`, `react-native-ios-utilities`) | Cada upgrade de SDK é um projeto; ffmpeg-kit está descontinuado |
| Transcodificação de áudio via ffmpeg (~30 MB no binário) | `AVAudioConverter`/`AVAudioEngine` resolve nativamente com custo zero |
| Estado 100% em memória + `redux-persist` que **descarta tudo** ao subir a versão do schema | Sem offline real; reabertura fria refaz todas as chamadas |
| Renderização de listas longas via `FlashList` sobre a ponte | Scroll de conversas com anexos ainda apresenta jank |
| Componentes de menu/contexto iOS emulados (`zeego`, `react-native-ios-context-menu`) | UX próxima, mas não idêntica ao sistema |

### 1.2 Não-objetivos

- Não é escopo desta reescrita o widget de chat para o **cliente final** (`@chatwoot/react-native-widget`), usado hoje apenas dentro da tela de Configurações para "Fale conosco".
- Não é escopo alterar o backend Chatwoot. Toda funcionalidade de Fase 1 deve funcionar contra uma instalação Chatwoot 4.1+ sem modificações.
- Android não faz parte deste PRD.

---

## 2. Contexto: como o app atual funciona

### 2.1 Estrutura em números

- 617 arquivos TypeScript/TSX em `src/`
- 27 slices Redux (`src/store/reducers.ts`)
- 42 idiomas (`src/i18n/`), sincronizados via Crowdin
- ~40 arquivos de teste unitário (apenas store e utils; nenhum teste de UI/E2E)
- 1 cliente HTTP singleton, 1 conexão WebSocket (ActionCable)

### 2.2 Fluxo de inicialização

```
App.tsx → Sentry.wrap → src/app.tsx (Provider + PersistGate)
  → AppNavigator (GestureHandler, Keyboard, Refs, SafeArea)
    → NavigationContainer (linking: deep link + SSO + FCM)
      → AppTabs → isLoggedIn ? RootStack : AuthStack
```

Ao entrar no estado logado, `src/navigation/tabs/AppTabs.tsx` dispara em paralelo: `getProfile`, `saveDeviceDetails` (registro FCM), `fetchInboxes`, `fetchLabels`, `dashboardApps.index`, `customAttributes.index`, inicializa ActionCable, identifica usuário no Sentry e no analytics, limpa notificações entregues e checa a versão do servidor.

### 2.3 Camada de rede (comportamento a replicar exatamente)

`src/services/APIService.ts` — singleton axios com dois interceptors:

1. **`baseURL` dinâmico:** lido a cada request de `state.settings.installationUrl`. Não existe host de API em tempo de compilação.
2. **Escopo de conta automático:** `get('conversations')` vira `api/v1/accounts/{account_id}/conversations`. Exceções (allowlist `nonAccountRoutes`) viram `api/v1/{url}`: `profile`, `profile/availability`, `notification_subscriptions`, `profile/set_active_account`.
3. **Headers de autenticação:** `access-token`, `uid`, `client` (devise_token_auth), mais headers de telemetria de dispositivo enviados em toda request:
   `X-Chatwoot-Client-Name: Chatwoot Mobile`, `X-Chatwoot-Client-Version`, `X-Chatwoot-Platform`, `X-Chatwoot-Platform-Version`, `X-Chatwoot-Device-Model`, e `User-Agent: Chatwoot Mobile/{v} ({platform} {os}; {model})`.
4. **401 → logout global.** Qualquer outro erro → toast genérico (`ERRORS.COMMON_ERROR`).

---

## 3. Personas e permissões

| Persona | Descrição | Impacto no app |
|---|---|---|
| **Agente** | Atende conversas atribuídas a si, ao seu time, ou não atribuídas | Persona principal |
| **Administrador** | Agente + visão total da conta | Recebe alerta de upgrade de servidor com instrução técnica; agente recebe alerta genérico |
| **Usuário sem permissão de conversa** | Ex.: perfil restrito a relatórios | Abas *Inbox* e *Conversas* **não são montadas**; app abre em Configurações |

A checagem usa `CONVERSATION_PERMISSIONS = [administrator, agent, conversation_manage, conversation_unassigned_manage, conversation_participating_manage]` contra as permissões da conta ativa (`src/utils/permissionUtils.ts`). **RF-PERM-01:** a versão nativa deve aplicar a mesma regra, e deve reavaliar ao trocar de conta.

Um mesmo usuário pode pertencer a **múltiplas contas** (`user.accounts[]`); trocar de conta limpa conversas, contatos, notificações e busca em memória, chama `PUT profile/set_active_account` e recria a navegação.

---

## 4. Arquitetura alvo (iOS nativo)

### 4.1 Stack

| Camada | Decisão | Justificativa |
|---|---|---|
| Linguagem | Swift 6, strict concurrency | Elimina classes inteiras de bug de concorrência que hoje existem no fluxo ActionCable ↔ Redux |
| UI | SwiftUI (iOS 17+ como mínimo), UIKit pontual | iOS 17 dá `ScrollView` com âncoras, `Observable`, `ContentUnavailableView`. A baseline suporta iOS 13.4 — ver §4.2 |
| Estado | `@Observable` + stores por feature, injetados por `Environment` | Mapeia 1:1 os slices Redux atuais sem o custo do Redux |
| Persistência | SQLite via **GRDB** | Precisamos de queries relacionais (conversa ↔ mensagens ↔ contato ↔ labels) e de migração incremental — que é exatamente o que o `redux-persist` atual não tem |
| Rede | `URLSession` + `async/await`, camada `APIClient` com interceptors equivalentes | §2.3 |
| Tempo real | `URLSessionWebSocketTask` + implementação enxuta do protocolo ActionCable | Sem dependência de terceiros |
| Push | APNs direto (`UNUserNotificationCenter`) — ver §9 | Remove Firebase Messaging do caminho crítico |
| Áudio | `AVAudioRecorder` / `AVAudioEngine` / `AVAudioConverter` | Substitui ffmpeg-kit |
| Imagens | `Nuke` ou `URLCache` + `AsyncImage` | Substitui `expo-image` |
| Markdown | `AttributedString(markdown:)` nativo | Substitui `react-native-markdown-display` |
| Crash/telemetria | Sentry Cocoa SDK | Mantém paridade e continuidade de dashboards |
| Testes | Swift Testing + XCUITest | A baseline não tem teste de UI; é uma lacuna a corrigir |

### 4.2 Decisão pendente — versão mínima do iOS

A baseline declara suporte a **iOS 13.4+**. Um alvo em SwiftUI moderno quer **iOS 17+**. A recomendação é **iOS 17.0 como mínimo** (cobertura >90% da base instalada no lançamento previsto) e comunicar a mudança nas notas de versão. Caso o negócio exija cauda mais longa, o piso viável sem reescrever a camada de UI é **iOS 16.0**, com custo de reimplementar ancoragem de scroll e `Observable` manualmente. *Esta decisão deve ser fechada antes do início da Fase 1.*

### 4.3 Camadas

```
Features/            Telas + view models (@Observable), 1 pasta por épico
 ├─ Auth/  Conversations/  Chat/  Inbox/  Search/  ContactDetails/  Settings/
Domain/              Modelos puros (Conversation, Message, Contact, …) + regras
Data/
 ├─ API/             APIClient, endpoints tipados, DTOs snake_case + mapeadores
 ├─ Realtime/        ActionCableClient, roteamento de eventos
 ├─ Persistence/     GRDB: schema, migrations, repositórios, observação reativa
 └─ Push/            Registro de device, parsing de payload, deep link
DesignSystem/        Tokens (cor, tipografia, espaçamento), componentes reutilizáveis
Core/                Logging, feature flags, versionamento de servidor, i18n
```

**Regra de dependência:** `Features → Domain ← Data`. Nenhum DTO snake_case pode vazar para `Features` — a conversão acontece exclusivamente em `Data/API` (é o mesmo contrato que hoje existe em `src/utils/camelCaseKeys.ts`, mas passa a ser garantido pelo compilador).

### 4.4 Estratégia offline (mudança relevante de comportamento)

Hoje o app é *online-first* com cache volátil. A versão nativa deve ser **local-first para leitura**:

- Conversas, mensagens, contatos, labels, inboxes, times, macros e respostas prontas são persistidos em SQLite.
- A UI lê **sempre** do banco, via observação reativa; a rede apenas atualiza o banco.
- Ao abrir o app sem rede: lista de conversas e histórico já carregado ficam disponíveis em modo leitura, com banner de offline (a baseline já tem `components-next/no-network`).
- Mensagens compostas offline entram em fila de envio com estado `pending` e retry automático ao voltar a conectividade (a baseline hoje falha e mostra toast).

---

## 5. Inventário funcional — Fase 1 (paridade)

Numeração `RF-<épico>-<n>`. Cada item foi extraído do código da baseline; a coluna *Origem* aponta o arquivo de referência.

### 5.1 Épico A — Onboarding e autenticação

| ID | Requisito | Origem |
|---|---|---|
| RF-AUTH-01 | Tela **Configurar URL**: usuário informa a URL da instalação. Validação faz `GET {url}api`; sucesso = URL válida. A URL é persistida e sobrevive ao logout. | `ConfigURLScreen.tsx`, `settingsService.verifyInstallationUrl` |
| RF-AUTH-02 | Padrão inicial `https://app.chatwoot.com` (`EXPO_PUBLIC_CHATWOOT_BASE_URL`); app detecta se é cloud ou self-hosted e ajusta textos | `selectIsChatwootCloud` |
| RF-AUTH-03 | **Login** e-mail + senha → `POST auth/sign_in`. Tokens `access-token`/`uid`/`client` vêm nos **headers** da resposta e devem ir para o Keychain (hoje ficam em AsyncStorage — melhoria obrigatória) | `authService.login` |
| RF-AUTH-04 | **MFA**: se a resposta trouxer `mfa_required: true` + `mfa_token`, navegar para tela de verificação com duas abas — *App autenticador* (código TOTP) e *Código de backup*. Verificação usa `POST auth/sign_in` com o token de MFA | `MFAScreen.tsx`, `authService.verifyMfa` |
| RF-AUTH-05 | **SSO / SAML**: abre navegador do sistema; callback volta por `chatwootapp://auth/saml`. O callback é interceptado antes da navegação e não altera a rota. Tratar sucesso e erro (`SSO_AUTH_FAILED`, `SSO_INVALID_RESPONSE`, `SSO_UNEXPECTED_ERROR`) | `SsoUtils`, `navigation/index.tsx` |
| RF-AUTH-06 | **Esqueci minha senha**: `POST auth/password` com e-mail, mensagem de sucesso | `ForgotPassword.tsx` |
| RF-AUTH-07 | Troca de idioma disponível **antes** do login (nas telas de URL e login) | `LOGIN.CHANGE_LANGUAGE` |
| RF-AUTH-08 | Logout limpa todo o estado **exceto** as configurações (URL da instalação e idioma), remove o device de `notification_subscriptions` e desconecta o WebSocket | `store/index.ts` rootReducer |
| RF-AUTH-09 | 401 em qualquer chamada força logout imediato | `APIService` response interceptor |
| RF-AUTH-10 | Se o usuário não tem nenhuma conta, exibir `ERRORS.NO_ACCOUNTS_MESSAGE` | `en.json` |

### 5.2 Épico B — Lista de conversas

| ID | Requisito | Origem |
|---|---|---|
| RF-CONV-01 | Lista paginada via `GET conversations` com `status`, `assignee_type`, `page`, `sort_by`, `inbox_id` | `conversationService.getConversations` |
| RF-CONV-02 | Filtro **status**: `open`, `pending`, `snoozed`, `resolved`, `all` | `CONVERSATION_STATUSES` |
| RF-CONV-03 | Filtro **atribuição**: `mine`, `unassigned`, `all`, com contadores (`mineCount`, `unassignedCount`, `allCount`) vindos do `meta` | `ConversationListMeta` |
| RF-CONV-04 | Filtro **inbox**: "Todas as caixas" ou uma caixa específica | `InboxFilters.tsx` |
| RF-CONV-05 | Ordenação: `latest`, `sort_on_created_at`, `sort_on_priority` | `SORT_TYPES` |
| RF-CONV-06 | Item da lista exibe: avatar do contato + indicador de presença, nome, ID da conversa, prévia da última mensagem não-atividade, timestamp relativo, badge de não lidas, indicador de canal, prioridade, rótulos e indicador de SLA | `conversation-item/` |
| RF-CONV-07 | "Digitando…" em tempo real substitui a prévia da mensagem | `TypingMessage.tsx` |
| RF-CONV-08 | Ações por gesto (swipe) e seleção múltipla com barra de ações em lote | `ConversationSelect.tsx`, `bulk_actions` |
| RF-CONV-09 | **Ações em lote** via `POST bulk_actions` sobre a seleção | `conversationService.bulkAction` |
| RF-CONV-10 | Ações rápidas na lista: alterar status, responsável, time, prioridade, rótulos | `conversation-actions/` |
| RF-CONV-11 | Pull-to-refresh, scroll infinito e estado "todas as conversas carregadas" | `ALL_CONVERSATION_LOADED` |
| RF-CONV-12 | Estado vazio dedicado | `CONVERSATION.EMPTY` |
| RF-CONV-13 | Lista se atualiza por WebSocket sem refetch (nova conversa, mudança de status, atribuição, leitura) | `actionCable.ts` |

### 5.3 Épico C — Tela de conversa (chat)

Este é o épico de maior superfície. A baseline tem 56 componentes só aqui.

**C.1 — Histórico de mensagens**

| ID | Requisito | Origem |
|---|---|---|
| RF-CHAT-01 | Carregamento por cursor: `GET conversations/{id}/messages?before=&after=` | `fetchPreviousMessages` |
| RF-CHAT-02 | Tipos de mensagem: `incoming` (0), `outgoing` (1), `activity` (2), `template` (3) | `MESSAGE_TYPES` |
| RF-CHAT-03 | Agrupamento visual de mensagens consecutivas do mesmo remetente (`groupWithNext`, `groupWithPrevious`, `shouldRenderAvatar`) | `MessageItem.tsx` |
| RF-CHAT-04 | Separadores de data com "Hoje"/"Ontem" | `CONVERSATION.TODAY/YESTERDAY` |
| RF-CHAT-05 | Status de entrega em mensagens de saída: `sent`, `delivered`, `read`, `failed` — com motivo do erro quando houver (`externalError`) | `DeliveryStatus.tsx`, `ErrorInformation.tsx` |
| RF-CHAT-06 | Notas privadas com tratamento visual distinto | `PrivateTextCell.tsx` |
| RF-CHAT-07 | Mensagens de atividade (sistema) em estilo próprio | `ActivityBubble.tsx` |
| RF-CHAT-08 | Renderização de Markdown quando o conteúdo é markdown (detecção em `isMarkdown.ts`) | `MarkdownBubble.tsx` |
| RF-CHAT-09 | Mensagem citada/respondida renderizada inline (`contentAttributes.inReplyTo`) | `ReplyMessageBubble.tsx` |
| RF-CHAT-10 | Mensagens não suportadas exibem placeholder | `UnsupportedBubble.tsx` |
| RF-CHAT-11 | Mensagem excluída (`contentAttributes.deleted`) exibe placeholder | `MessageTextCell.tsx` |

**C.2 — Anexos e mídia**

| ID | Requisito | Origem |
|---|---|---|
| RF-CHAT-12 | Tipos: `image`, `video`, `audio`, `file`, `ig_reel`, localização, contato | `ImageMetadata.fileType`, `CONVERSATION.ATTACHMENTS.*` |
| RF-CHAT-13 | Imagem: thumbnail na bolha, visualizador em tela cheia com zoom/gestos | `ImageBubble.ios.tsx`, `galeria` |
| RF-CHAT-14 | Vídeo: player inline | `VideoBubble.tsx` |
| RF-CHAT-15 | Áudio: player com forma de onda, controle de velocidade e progresso; **transcodificação** de ogg/aac para reprodução (usar `AVAudioConverter` no lugar do ffmpeg) | `AudioBubble.tsx`, `audioConverter.ios.ts` |
| RF-CHAT-16 | Arquivo: nome, extensão, download e abertura no visualizador do sistema | `FileBubble.tsx` |
| RF-CHAT-17 | Localização: mapa/coordenadas (`coordinatesLat`, `coordinatesLong`) | `LocationBubble.tsx` |
| RF-CHAT-18 | Limite de upload: **20 MB** com mensagem de erro dedicada | `MAXIMUM_FILE_UPLOAD_SIZE`, `FILE_SIZE_LIMIT` |
| RF-CHAT-19 | E-mail: cabeçalho expansível com De/Para/Cc/Cco/Assunto e corpo HTML renderizado | `Email.tsx`, `EmailMeta.tsx` |

**C.3 — Composição**

| ID | Requisito | Origem |
|---|---|---|
| RF-CHAT-20 | Campo de texto multilinha com limite `MESSAGE_MAX_LENGTH` | `MessageTextInput.tsx` |
| RF-CHAT-21 | Alternância **Responder / Nota privada** | `togglePrivateMessage` |
| RF-CHAT-22 | **Respostas prontas** por `/atalho` → `GET canned_responses?search=` com filtro incremental | `CannedResponses.tsx` |
| RF-CHAT-23 | **Menção de agente** por `@` (apenas em notas privadas), a partir de `assignable_agents` | `MentionUser.tsx` |
| RF-CHAT-24 | **Variáveis de mensagem** (ex.: `{{contact.name}}`) com aviso bloqueante quando restarem variáveis não resolvidas | `messageVariableUtils.ts`, `UNDEFINED_VARIABLES_TITLE` |
| RF-CHAT-25 | Anexos: galeria de fotos, câmera e seletor de documentos, com pré-visualização removível antes do envio | `PhotosCommandButton.tsx`, `AttachedMedia.tsx` |
| RF-CHAT-26 | **Gravação de voz** com temporizador, cancelamento e pré-escuta antes do envio | `AudioRecorder.tsx` |
| RF-CHAT-27 | Cabeçalho de e-mail editável (Para/Cc/Cco) quando a inbox é de e-mail | `ReplyEmailHead.tsx` |
| RF-CHAT-28 | Citar mensagem (responder a) quando a inbox suporta `replyTo`/`replyToOutgoing` | `QuoteReply.tsx`, `INBOX_FEATURES` |
| RF-CHAT-29 | **Envio otimista**: mensagem aparece imediatamente com `echoId`; a resposta do servidor reconcilia; falha marca como `failed` com opção de reenvio | `messageUtils.ts` |
| RF-CHAT-30 | Indicador de digitação **de saída**: `POST conversations/{id}/toggle_typing_status` com debounce | `toggleTyping` |
| RF-CHAT-31 | Indicador de digitação **de entrada** com nomes dos usuários digitando e expiração automática em 30 s | `TypingIndicator.tsx`, `typingUtils.ts` |
| RF-CHAT-32 | **Banners de janela de resposta**: janela de 24 h (WhatsApp/Twilio) e "não é possível responder" quando `canReply == false` | `ReplyWarning.tsx`, `BANNER.*` |

**C.4 — Ações sobre a mensagem (menu de contexto)**

| ID | Requisito | Origem |
|---|---|---|
| RF-CHAT-33 | Copiar texto | `LONG_PRESS_ACTIONS.COPY` |
| RF-CHAT-34 | Responder (citar) | `LONG_PRESS_ACTIONS.REPLY` |
| RF-CHAT-35 | Excluir → `DELETE conversations/{id}/messages/{messageId}`, com confirmação | `deleteMessage` |
| RF-CHAT-36 | **Traduzir** → `POST conversations/{id}/messages/{messageId}/translate` com `target_language`; alternar entre original e traduzido | `translateMessage`, `VIEW_ORIGINAL/VIEW_TRANSLATED` |

**C.5 — Cabeçalho e ações de conversa**

| ID | Requisito | Origem |
|---|---|---|
| RF-CHAT-37 | Cabeçalho: avatar + nome do contato, canal de origem, responsável atual, com colapso animado no scroll | `ChatHeader.tsx`, `useHeaderAnimation.ts` |
| RF-CHAT-38 | **Eventos de SLA** no cabeçalho: FRT, NRT, RT com estados *due*/*missed* e lista expansível de violações | `SlaEvents.tsx`, `SLA.*` |
| RF-CHAT-39 | Alterar status: aberto, pendente, resolvido, adiado → `POST conversations/{id}/toggle_status` | `toggleConversationStatus` |
| RF-CHAT-40 | **Adiar (snooze)** com opções: até a próxima resposta, até amanhã, até a próxima semana, e data/hora personalizada | `SNOOZE_UNTIL_*` |
| RF-CHAT-41 | Atribuir responsável (com busca) e auto-atribuir/desatribuir → `POST conversations/{id}/assignments` | `AssigneePanel.tsx` |
| RF-CHAT-42 | Atribuir time → `POST conversations/{id}/assignments?team_id=` | `assignTeam` |
| RF-CHAT-43 | Alterar prioridade: nenhuma, urgente, alta, média, baixa → `POST conversations/{id}/toggle_priority` | `PriorityPanel.tsx` |
| RF-CHAT-44 | Adicionar/remover rótulos → `POST conversations/{id}/labels` | `ConversationLabelActions.tsx` |
| RF-CHAT-45 | **Participantes**: listar, adicionar, remover, e "acompanhar esta conversa" (auto-inclusão) → `GET`/`PUT conversations/{id}/participants` | `AddParticipantList.tsx` |
| RF-CHAT-46 | Silenciar/reativar → `POST conversations/{id}/mute` \| `/unmute` | `muteConversation` |
| RF-CHAT-47 | Marcar como não lida → `POST conversations/{id}/unread`; marcar como lida → `POST conversations/{id}/update_last_seen` (automático ao abrir) | `markMessagesUnread`, `markMessageRead` |
| RF-CHAT-48 | **Macros**: listar (`GET macros`), ver detalhes das ações e executar (`POST macros/{id}/execute`) | `MacrosList.tsx` |
| RF-CHAT-49 | Metadados da conversa: navegador, SO, iniciada em, origem, IP, ID — com cópia para a área de transferência | `ConversationMetaInformation.tsx` |
| RF-CHAT-50 | Compartilhar conversa | `CONVERSATION.SHARE` |
| RF-CHAT-51 | Estado "conversa não encontrada" com ações de retry e voltar ao início | `NOT_FOUND.*` |

**C.6 — Copilot (Captain AI)**

| ID | Requisito | Origem |
|---|---|---|
| RF-COP-01 | **Melhorar resposta** → `POST captain/tasks/rewrite` com `operation: improve` | `copilotService.rewrite` |
| RF-COP-02 | **Corrigir ortografia/gramática** → `rewrite` com `fix_spelling_grammar` | idem |
| RF-COP-03 | **Mudar tom**: profissional, casual, direto, confiante, amigável → `rewrite` com o tom escolhido | `TONE_ACTIONS` |
| RF-COP-04 | **Sugerir resposta** → `POST captain/tasks/reply_suggestion` | `replySuggestion` |
| RF-COP-05 | **Resumir conversa** → `POST captain/tasks/summarize` | `summarize` |
| RF-COP-06 | **Follow-up** conversacional sobre o resultado gerado → `POST captain/tasks/follow_up` | `followUp` |
| RF-COP-07 | Estado "pensando", cancelamento da requisição em andamento (`AbortSignal` → `Task.cancel()`), e tratamento de falha | `COPILOT.THINKING`, `GENERATION_FAILED` |
| RF-COP-08 | Aceitar/descartar o texto gerado antes de inserir no campo de resposta | `CopilotEditorSection.tsx` |

### 5.4 Épico D — Inbox de notificações

| ID | Requisito | Origem |
|---|---|---|
| RF-NOTIF-01 | Lista paginada: `GET notifications?sort_order=&includes[]=snoozed&includes[]=read&page=` | `notificationService` |
| RF-NOTIF-02 | Tipos exibidos com ícone próprio: menção, criação de conversa, atribuição, nova mensagem em conversa atribuída, nova mensagem em conversa participante, SLA perdido (primeira resposta / próxima resposta / resolução) | `NOTIFICATION.TYPES.*` |
| RF-NOTIF-03 | Ordenação ascendente/descendente | `notificationFilterSlice` |
| RF-NOTIF-04 | Marcar todas como lidas → `POST notifications/read_all` | `markAllAsRead` |
| RF-NOTIF-05 | Marcar item como lido → `POST notifications/read_all` com `primary_actor_id`/`primary_actor_type` | `markAsRead` |
| RF-NOTIF-06 | Marcar como não lida → `POST notifications/{id}/unread` | `markAsUnread` |
| RF-NOTIF-07 | Excluir → `DELETE notifications/{id}`; ações em massa "excluir todas" e "excluir lidas" | `NOTIFICATION.ALERTS.*` |
| RF-NOTIF-08 | Tocar na notificação abre a conversa correspondente | `InboxItem.tsx` |
| RF-NOTIF-09 | Lista atualiza em tempo real por `notification.created` / `notification.deleted` | `actionCable.ts` |
| RF-NOTIF-10 | Badge do ícone do app reflete a contagem de não lidas | `updateBadgeCount` |

### 5.5 Épico E — Busca

| ID | Requisito | Origem |
|---|---|---|
| RF-SEARCH-01 | Busca unificada em três seções: **Contatos** (`GET search/contacts`), **Conversas** (`GET search/conversations`), **Mensagens** (`GET search/messages`) — parâmetros `q` e `page` | `search/config.ts` |
| RF-SEARCH-02 | Visão "todos os resultados" com prévia por seção + "ver mais" para a lista completa da seção | `AllResultsView.tsx` |
| RF-SEARCH-03 | Destaque do termo buscado no resultado | `HighlightedText.tsx`, `highlightText.ts` |
| RF-SEARCH-04 | **Buscas recentes** persistidas localmente, com "limpar tudo" | `recentSearches.ts` |
| RF-SEARCH-05 | Debounce + cancelamento da requisição anterior a cada tecla | `useSearchScreen.ts` |
| RF-SEARCH-06 | Estados: vazio, sem resultados, erro genérico, cancelado, e "toque para tentar de novo" | `SEARCH.*` |

### 5.6 Épico F — Detalhes do contato

| ID | Requisito | Origem |
|---|---|---|
| RF-CONT-01 | Avatar, nome, status de presença, e-mail, telefone | `ContactDetailsScreen.tsx` |
| RF-CONT-02 | Ações rápidas: ligar, enviar e-mail, copiar | `ContactBasicActions.tsx` |
| RF-CONT-03 | Rótulos do contato: ler (`GET contacts/{id}/labels`) e atualizar (`POST contacts/{id}/labels`) | `contactService` |
| RF-CONT-04 | Atributos personalizados renderizados conforme `GET custom_attribute_definitions` (texto, número, lista, data, checkbox) | `ContactMetaInformation.tsx` |
| RF-CONT-05 | Outras conversas do mesmo contato → `GET contacts/{id}/conversations`, navegáveis | `getContactConversations` |
| RF-CONT-06 | Apresentado como sheet modal (`formSheet` no iOS) | `AppTabs.tsx` |

### 5.7 Épico G — Configurações

| ID | Requisito | Origem |
|---|---|---|
| RF-SET-01 | Perfil: avatar, nome, e-mail, conta ativa | `SettingsHeader.tsx` |
| RF-SET-02 | **Disponibilidade**: online / ocupado / offline → `POST profile/availability`; reflete em tempo real via `presence.update` | `AvailabilityStatusList.tsx` |
| RF-SET-03 | **Trocar de conta** (quando `accounts.length > 1`) → `PUT profile/set_active_account`, com limpeza de estado e recarga completa | `SwitchAccount.tsx` |
| RF-SET-04 | **Idioma**: 42 opções → aplicado imediatamente e persistido | `LanguageList.tsx`, `LANGUAGES` |
| RF-SET-05 | **Preferências de notificação** → `GET`/`PUT notification_settings`, com canais e-mail e push por tipo de evento (criação, atribuição, nova mensagem em atribuída, menção, nova mensagem em participante, e os três tipos de SLA perdido) | `NotificationPreferences.tsx` |
| RF-SET-06 | Acesso à documentação (navegador in-app) | `HELP_URL` |
| RF-SET-07 | "Fale conosco" — widget de suporte com identificação do usuário (`identifier_hash`) e atributos do app | `ChatWootWidget` |
| RF-SET-08 | Exibir versão do app e build | `Application.nativeApplicationVersion` |
| RF-SET-09 | Logout com confirmação | `SETTINGS.LOGOUT` |
| RF-SET-10 | Ações de debug (build interna): exibir token de push, forçar erro, limpar cache | `DebugActions.tsx` |

### 5.8 Épico H — Dashboard Apps

| ID | Requisito | Origem |
|---|---|---|
| RF-DASH-01 | `GET dashboard_apps` lista apps configurados na conta | `dashboardAppService` |
| RF-DASH-02 | Renderizar o app em WebView modal, passando o contexto da conversa | `DashboardScreen.tsx` |

### 5.9 Épico I — Transversais

| ID | Requisito | Origem |
|---|---|---|
| RF-SYS-01 | **Deep links**: `https://{installation}/app/accounts/{accountId}/conversations/{conversationId}` (+ `primaryActorId`/`primaryActorType` opcionais) e esquema `chatwootapp://` — via Universal Links (`associatedDomains: applinks:app.chatwoot.com`) | `navigation/index.tsx`, `app.config.ts` |
| RF-SYS-02 | **Checagem de versão do servidor** no boot: se `< EXPO_PUBLIC_MINIMUM_CHATWOOT_VERSION`, alerta — texto técnico para administrador, genérico para agente. Deve tolerar versões não-semver (`semver.coerce`) sem crashar | `serverUtils.ts` |
| RF-SYS-03 | Detecção de conectividade com banner de offline | `no-network/` |
| RF-SYS-04 | Toasts para erro genérico e confirmações | `toastUtils.ts` |
| RF-SYS-05 | Feedback háptico em seleções e ações | `useHaptic.ts` |
| RF-SYS-06 | Suporte a tema claro e escuro | `theme/colors/{light,dark}.ts` |
| RF-SYS-07 | Telemetria de erros (Sentry) com identificação de usuário, conta e URL da instalação | `AppTabs.tsx` |
| RF-SYS-08 | Analytics de produto com os eventos já catalogados | `analyticsEvents.ts` |
| RF-SYS-09 | 42 idiomas, mantidos via Crowdin — apenas `en.json` é editável na origem | `crowdin.yml` |
| RF-SYS-10 | Headers de telemetria `X-Chatwoot-*` em toda requisição (§2.3) | `APIService.deviceHeaders` |

---

## 6. Contrato de API (Fase 1)

Base: `{installationUrl}/api/v1/accounts/{accountId}/` salvo indicação em contrário.
Autenticação: headers `access-token`, `uid`, `client`.

### 6.0 Montadas na raiz da instalação (**sem** `api/v1/`)

O devise_token_auth é montado fora do namespace da API. Estes dois caminhos
vão direto para `{installationUrl}/auth/…`:

| Método | Caminho | Uso |
|---|---|---|
| POST | `{installationUrl}auth/sign_in` | Login, verificação de MFA, SSO. Tokens vêm nos headers da resposta |
| POST | `{installationUrl}auth/password` | Redefinição de senha |

Na baseline isso acontece por consequência da lógica do interceptor, não por
regra explícita: no login ainda não há `account_id` **e** `auth/sign_in` não
está na allowlist `nonAccountRoutes`, então nenhum dos dois ramos executa e a
URL permanece intacta. Uma implementação que trate `auth/*` como
"rota sem escopo de conta" e prefixe `api/v1/` recebe **404**.

### 6.1 Fora do escopo de conta (`api/v1/`)

| Método | Caminho | Uso |
|---|---|---|
| GET | `profile` | Perfil + lista de contas + `pubsub_token` |
| POST | `profile/availability` | Alterar disponibilidade |
| PUT | `profile/set_active_account` | Trocar conta ativa |
| POST | `notification_subscriptions` | Registrar device para push |
| DELETE | `notification_subscriptions` | Remover device (payload no corpo) |

### 6.2 Sem autenticação

| Método | Caminho | Uso |
|---|---|---|
| GET | `{installationUrl}api` | Validação da URL e leitura da versão do servidor |

### 6.3 Conversas

| Método | Caminho | Parâmetros |
|---|---|---|
| GET | `conversations` | `inbox_id`, `assignee_type`, `status`, `page`, `sort_by` |
| GET | `conversations/{id}` | — |
| GET | `conversations/{id}/messages` | `before`, `after` |
| POST | `conversations/{id}/messages` | corpo multipart quando há anexo |
| DELETE | `conversations/{id}/messages/{messageId}` | — |
| POST | `conversations/{id}/messages/{messageId}/translate` | `target_language` |
| POST | `conversations/{id}/toggle_status` | retorna `current_status`, `snoozed_until` |
| POST | `conversations/{id}/toggle_priority` | `priority` |
| POST | `conversations/{id}/toggle_typing_status` | `typing_status`, `is_private` |
| POST | `conversations/{id}/assignments` | `assignee_id`, `team_id` |
| POST | `conversations/{id}/assignments?team_id=` | atribuição de time |
| POST | `conversations/{id}/labels` | `labels[]` |
| POST | `conversations/{id}/mute` \| `/unmute` | — |
| POST | `conversations/{id}/unread` | marcar não lida |
| POST | `conversations/{id}/update_last_seen` | marcar lida |
| GET / PUT | `conversations/{id}/participants` | PUT recebe `user_ids[]` |
| POST | `bulk_actions` | ações em lote |

### 6.4 Contatos, metadados e catálogos

| Método | Caminho |
|---|---|
| GET / POST | `contacts/{id}/labels` |
| GET | `contacts/{id}/conversations` |
| GET | `inboxes` |
| GET | `labels` |
| GET | `teams` |
| GET | `assignable_agents?inbox_ids[]=` |
| GET | `canned_responses?search=` |
| GET | `macros` · POST `macros/{id}/execute` (`conversation_ids[]`) |
| GET | `custom_attribute_definitions` |
| GET | `dashboard_apps` |

### 6.5 Notificações

| Método | Caminho |
|---|---|
| GET | `notifications?sort_order=&includes[]=snoozed&includes[]=read&page=` |
| POST | `notifications/read_all` (opcionalmente com `primary_actor_id`/`primary_actor_type`) |
| POST | `notifications/{id}/unread` |
| DELETE | `notifications/{id}` |
| GET / PUT | `notification_settings` |

### 6.6 Busca e IA

| Método | Caminho |
|---|---|
| GET | `search/contacts` · `search/conversations` · `search/messages` — `q`, `page` |
| POST | `captain/tasks/rewrite` — `content`, `operation`, `conversation_display_id` |
| POST | `captain/tasks/summarize` — `conversation_display_id` |
| POST | `captain/tasks/reply_suggestion` — `conversation_display_id` |
| POST | `captain/tasks/follow_up` — `follow_up_context`, `message`, `conversation_display_id` |

---

## 7. Protocolo de tempo real

**Transporte:** ActionCable (Rails) sobre WebSocket, em `{webSocketUrl}` derivado da URL da instalação.

**Assinatura:**
```json
{ "channel": "RoomChannel",
  "pubsub_token": "<do perfil>",
  "account_id": <id>,
  "user_id": <id> }
```

**Presença:** comando `update_presence` a cada **20 segundos**.

**Validação:** toda mensagem recebida é descartada se `data.account_id` ≠ conta ativa.

**Eventos consumidos** (`src/utils/actionCable.ts`):

| Evento | Efeito esperado |
|---|---|
| `message.created` | Insere/atualiza mensagem; atualiza `lastActivityAt` da conversa |
| `message.updated` | Atualiza mensagem existente |
| `conversation.created` | Insere conversa na lista + contato |
| `conversation.updated` | Atualiza conversa + contato |
| `conversation.status_changed` | Atualiza status |
| `conversation.read` | Atualiza contadores de leitura |
| `assignee.changed` | Atualiza responsável |
| `conversation.typing_on` / `typing_off` | Liga/desliga indicador; timer de expiração de **30 s** |
| `contact.updated` | Atualiza contato |
| `notification.created` / `notification.deleted` | Insere/remove da inbox de notificações |
| `presence.update` | Atualiza presença de contatos e disponibilidade do usuário atual |

**Eventos ainda não tratados** (comentados na baseline, candidatos à Fase 2): `conversation.contact_changed`, `contact.deleted`, `conversation.mentioned`, `first.reply.created`.

**Requisitos nativos adicionais:**
- **RF-RT-01:** reconexão automática com backoff exponencial e *jitter*; hoje não há estratégia de reconexão explícita.
- **RF-RT-02:** ao reconectar, buscar o delta de mensagens (`after={último id conhecido}`) em vez de assumir continuidade.
- **RF-RT-03:** desconectar ao ir para background além de ~30 s e reconectar no foreground, respeitando o ciclo de vida do iOS.

---

## 8. Modelo de dados

Entidades e campos conforme `src/types/`. Todos os timestamps são **Unix (segundos)**.

**Conversation** — `id`, `accountId`, `inboxId`, `uuid`, `status` (open/pending/snoozed/resolved), `priority` (urgent/high/medium/low/null), `unreadCount`, `muted`, `canReply`, `labels[]`, `customAttributes`, `additionalAttributes` (navegador, SO, referer, iniciada em, idioma), `agentLastSeenAt`, `assigneeLastSeenAt`, `contactLastSeenAt`, `createdAt`, `lastActivityAt`, `firstReplyCreatedAt`, `waitingSince`, `snoozedUntil`, `slaPolicyId`, `appliedSla`, `slaEvents[]`, `meta` (sender, assignee, team, channel, hmacVerified), `messages[]`, `lastNonActivityMessage`.

**Message** — `id`, `conversationId`, `inboxId`, `content`, `contentType` (text, input_text, input_textarea, input_email, input_select, cards, form, article, incoming_email, input_csat, integrations), `messageType` (0–3), `private`, `status`, `sender` (Agent | User | AgentBot | Contact), `senderId`, `senderType`, `sourceId`, `echoId`, `createdAt`, `attachments[]`, `contentAttributes` (inReplyTo, deleted, email{subject,from,to,cc,bcc,htmlContent,textContent}, externalError, isUnsupported, translations), e os flags de agrupamento visual.

**Attachment (ImageMetadata)** — `id`, `messageId`, `fileType` (image/video/audio/file/ig_reel), `extension`, `dataUrl`, `thumbUrl`, `fallbackTitle`, `coordinatesLat`, `coordinatesLong`.

**Contact**, **Agent**, **User**, **AgentBot**, **Team**, **Inbox**, **Label**, **CannedResponse**, **Macro**, **CustomAttribute**, **DashboardApp**, **Notification**, **SLA/SLAEvent**, **Account** — conforme os arquivos correspondentes em `src/types/`.

**Canais suportados** (`INBOX_TYPES`): WebWidget, FacebookPage, TwitterProfile, TwilioSms, Whatsapp, Api, Email, Telegram, Line, Sms, Instagram, Tiktok. Cada um tem ícone próprio e algumas regras de resposta (janela de 24 h, `canReply`).

**Nota de migração:** `User` chega do servidor em snake_case e assim é mantido na baseline (`account_id`, `avatar_url`, `identifier_hash`). Na versão nativa, normalizar para camelCase na camada de mapeamento como todo o resto.

---

## 9. Notificações push

**Baseline:** Firebase Cloud Messaging + Notifee, com `UIBackgroundModes: [fetch, remote-notification]` e `aps-environment: production`.

**Decisão para o nativo:** o backend Chatwoot envia push para iOS via APNs (o registro em `notification_subscriptions` carrega o token). Recomenda-se **APNs direto**, eliminando o SDK Firebase. Isso exige confirmar com a equipe de backend que a instalação-alvo está configurada para APNs sem intermediação do FCM; se houver dependência de FCM em instalações self-hosted, manter o registro dual por uma versão de transição.

| ID | Requisito |
|---|---|
| RF-PUSH-01 | Solicitar permissão de notificação após o login (não no primeiro launch) |
| RF-PUSH-02 | Registrar device em `POST notification_subscriptions` a cada boot logado e a cada rotação de token |
| RF-PUSH-03 | Remover device em `DELETE notification_subscriptions` no logout |
| RF-PUSH-04 | Tocar na notificação abre a conversa correta — em cold start, background e foreground |
| RF-PUSH-05 | Badge do app espelha as notificações não lidas |
| RF-PUSH-06 | Limpar notificações entregues ao abrir o app |
| RF-PUSH-07 | Respeitar as preferências por tipo definidas em `notification_settings` |

---

## 10. Requisitos não funcionais

| Área | Alvo |
|---|---|
| Cold start até lista de conversas utilizável | < 1,5 s com cache local |
| Scroll no histórico de mensagens | 120 fps em ProMotion, sem quedas de frame com anexos |
| Envio de mensagem — feedback visual | < 100 ms (otimista) |
| Tamanho do app | < 40 MB (a remoção do ffmpeg-kit já responde por boa parte) |
| Segurança | Tokens no **Keychain** (`kSecAttrAccessibleAfterFirstUnlock`), nunca em `UserDefaults`; ATS habilitado; sem log de tokens |
| Acessibilidade | VoiceOver completo, Dynamic Type até XXL, contraste AA — a baseline é fraca neste ponto |
| Idiomas | 42 locales, mesmo conjunto de chaves; RTL correto para árabe, farsi e hebraico |
| Offline | Leitura completa do cache; fila de envio com retry |
| Crash-free sessions | ≥ 99,8% |
| Cobertura de testes | ≥ 70% em `Domain` e `Data`; smoke E2E dos 5 fluxos críticos |

---

## 11. Roadmap

### Fase 0 — Fundação (3 semanas)
Projeto, CI (build + testes + TestFlight), design system (tokens de cor claro/escuro, tipografia Inter, espaçamento), `APIClient` com os interceptors da §2.3, Keychain, camada GRDB com migrations, i18n com os 42 locales importados, Sentry.
**Saída:** app que valida URL, faz login e exibe o perfil.

### Fase 1A — Núcleo de leitura (4 semanas)
Épicos A, B, D e o histórico do C (C.1, C.2). ActionCable completo. Push básico (RF-PUSH-01 a 04).
**Saída:** dogfooding interno — dá para acompanhar conversas, mas não responder.

### Fase 1B — Núcleo de escrita (4 semanas)
C.3 (composição completa), C.4 (ações de mensagem), C.5 (ações de conversa), Épico F.
**Saída:** beta fechado — paridade operacional para o dia a dia do agente.

### Fase 1C — Paridade total (3 semanas)
Épicos E (busca), G (configurações), H (dashboard apps), C.6 (Copilot), todos os transversais, acessibilidade, RTL.
**Saída:** TestFlight público, lado a lado com a versão RN.

### Fase 1D — Endurecimento (2 semanas)
Performance, E2E, revisão de segurança, migração de sessão da versão RN (ver §13), submissão à App Store.

**Total Fase 1: ~16 semanas** com 2 engenheiros iOS + 1 designer parcial.

### Fase 2 — Evolução (backlog priorizado)

| Prioridade | Item | Valor |
|---|---|---|
| Alta | **Notificações acionáveis** — responder, resolver e atribuir direto da notificação | Reduz drasticamente o tempo até a primeira resposta |
| Alta | **Widgets** (Home/Lock Screen) — conversas não lidas, SLA em risco | Visibilidade sem abrir o app |
| Alta | **Offline real com fila de envio** | Atendimento em deslocamento |
| Média | **iPad + Mac Catalyst / Designed for iPad** — layout de três colunas | O RN atual declara `supportsTablet` mas não tem layout de tablet |
| Média | **Live Activity** para SLA em risco de estouro | Diferencial claro para times com SLA |
| Média | **Siri / App Intents** — "responder à última conversa", "definir disponibilidade como ocupado" | Mãos livres |
| Média | **Handoff** entre iPhone, iPad e o dashboard web | Continuidade |
| Média | Eventos WebSocket hoje ignorados (`conversation.mentioned`, `first.reply.created`, `contact.deleted`, `conversation.contact_changed`) | Fecha lacunas de consistência |
| Baixa | Apple Watch — triagem e respostas prontas | Nicho |
| Baixa | Rascunhos por conversa sincronizados | Conveniência |
| Baixa | Transcrição local de áudio recebido (`Speech`) | Acessibilidade e velocidade |

---

## 12. Critérios de aceite da Fase 1

1. Todo requisito `RF-*` da §5 implementado e verificado contra uma instalação Chatwoot 4.1+ real.
2. Nenhuma chamada de API fora do contrato da §6 — nenhuma mudança exigida no servidor.
3. Contra a versão RN 4.7.0, executando o mesmo roteiro de 40 passos manuais: mesmos resultados, mesmos textos (mesmas chaves i18n).
4. Todos os 12 tipos de canal renderizam corretamente na lista e no chat.
5. Todos os tipos de anexo (imagem, vídeo, áudio, arquivo, localização, e-mail) exibem e reproduzem/abrem corretamente.
6. Deep link e push abrem a conversa correta nos três estados do app (frio, background, foreground).
7. Troca de conta e logout não deixam dados residuais da conta anterior — verificado por inspeção do banco.
8. VoiceOver percorre lista de conversas, chat e composição sem armadilhas de foco.
9. Crash-free ≥ 99,8% em duas semanas de TestFlight com ≥ 100 usuários.
10. Nenhum token em `UserDefaults`, em logs ou em relatórios do Sentry.

---

## 13. Riscos e mitigações

| Risco | Impacto | Mitigação |
|---|---|---|
| **Sessão do usuário se perde na migração** RN → nativo (bundle id igual, storages diferentes) | Alto — todo mundo é deslogado no update | Ler uma vez o AsyncStorage legado (arquivo do `RCTAsyncLocalStorage`) no primeiro launch nativo, migrar tokens e `installationUrl` para o Keychain/GRDB e apagar o legado. **Precisa ser prototipado na Fase 0.** |
| Divergência de comportamento entre a versão RN e a nativa durante a coexistência | Médio | Roteiro de paridade da §12.3 executado a cada release candidate |
| Instalações self-hosted em versões antigas do Chatwoot | Médio | Manter a checagem de versão (RF-SYS-02) e degradar por feature (ex.: esconder Copilot quando o endpoint `captain/*` retornar 404) |
| Push via APNs direto pode não estar configurado em todas as instalações | Alto | Validar com o backend na Fase 0; manter registro dual FCM+APNs numa versão de transição, se necessário |
| Reprodução de áudio ogg (Opus) sem ffmpeg | Médio | `AVAudioConverter` cobre AAC/M4A/WAV/MP3; Opus exige decodificador — validar com amostras reais de WhatsApp na Fase 0 e, se necessário, embarcar apenas `libopus` (bem menor que o ffmpeg-kit) |
| Rodapé de tradução Crowdin desalinhado com as chaves nativas | Baixo | Reutilizar os mesmos JSONs, convertidos para `.xcstrings` por script, mantendo as chaves idênticas |
| Copilot depende do Captain, que é recurso pago/opcional | Baixo | Detectar disponibilidade e ocultar a UI quando indisponível |

---

## 14. Questões em aberto

1. **Versão mínima do iOS** — iOS 17 (recomendado) ou iOS 16? (§4.2)
2. **APNs direto ou manter FCM** para instalações self-hosted? (§9)
3. A nova versão substitui o app existente (mesmo bundle `com.chatwoot.app`) ou é publicada em paralelo durante o beta?
4. O widget de suporte "Fale conosco" (§RF-SET-07) deve ser reimplementado nativamente ou pode continuar como WebView?
5. Há intenção de manter a paridade Android após a reescrita iOS, ou a base RN permanece como app Android?

---

## Anexo A — Mapa de origem

| Assunto | Arquivos de referência na baseline |
|---|---|
| Rede e interceptors | `src/services/APIService.ts`, `src/store/storeAccessor.ts` |
| Estado e persistência | `src/store/index.ts`, `src/store/reducers.ts` |
| Tempo real | `src/utils/baseActionCableConnector.ts`, `src/utils/actionCable.ts` |
| Navegação, deep link, SSO, push | `src/navigation/index.tsx`, `src/navigation/tabs/AppTabs.tsx`, `src/utils/pushUtils.ts`, `src/utils/ssoUtils.ts` |
| Contrato de API | `src/store/*/*Service.ts`, `src/screens/search/config.ts` |
| Modelo de dados | `src/types/`, `src/utils/camelCaseKeys.ts` |
| Catálogo de funcionalidades e textos | `src/i18n/en.json`, `src/constants/index.ts` |
| Permissões | `src/constants/permissions.ts`, `src/utils/permissionUtils.ts` |
| Design tokens | `src/theme/`, `.cursor/rules/about.mdc` |
| Configuração nativa | `app.config.ts`, `with-ffmpeg-pod.js`, `eas.json` |
