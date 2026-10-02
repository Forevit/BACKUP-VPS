# Monitoramento de Internet — MikroTik + n8n

Sistema de monitoramento de conectividade baseado em **heartbeat enviado pelo MikroTik para o n8n**, com armazenamento do estado em Data Table e notificações via Telegram.

O projeto foi desenvolvido para monitorar a disponibilidade do link PPPoE sem depender de ping externo.

---

## Visão geral

O MikroTik envia periodicamente um `HTTP POST` para o n8n.

```text
MikroTik
   │
   │ HTTP POST
   ▼
n8n — Heartbeat
   │
   ▼
Data Table
monitor_internet
   │
   │ verificação a cada 5 segundos
   ▼
n8n — Monitoramento
   │
   ├── Link offline > 90s
   │       │
   │       └── Telegram 🚨
   │
   └── Link voltou
           │
           └── Telegram 🟢
```

O sistema possui dois workflows:

```text
MikroTik - Heartbeat
MikroTik - Monitoramento
```

---

# 1. Componentes

## MikroTik

Equipamento monitorado:

```text
Modelo: hEX
Interface WAN: pppoe-opennet
```

O MikroTik é responsável apenas por informar ao n8n que o link está funcionando.

---

## n8n

URL:

```text
https://n8n.eduardoferreira.space
```

O n8n possui dois workflows:

### Workflow 1

```text
MikroTik - Heartbeat
```

Recebe o heartbeat e atualiza o último horário de comunicação.

### Workflow 2

```text
MikroTik - Monitoramento
```

Verifica periodicamente o último heartbeat e controla os alertas de queda e recuperação.

---

## Telegram

O Telegram é utilizado exclusivamente para enviar as notificações de:

* queda do link;
* recuperação do link.

O token do bot deve permanecer armazenado nas **Credentials do n8n**.

> Nunca armazenar o token do bot diretamente neste README, em scripts ou no GitHub.

---

# 2. Workflow — MikroTik - Heartbeat

## Objetivo

Receber o heartbeat enviado pelo MikroTik e registrar o horário da última comunicação.

Estrutura:

```text
Webhook
   ↓
Data Table — Update row(s)
```

---

## Webhook

Método:

```text
POST
```

Path:

```text
mikrotik-heartbeat
```

URL:

```text
https://n8n.eduardoferreira.space/webhook/mikrotik-heartbeat
```

Resposta:

```text
Immediately
```

---

## Payload enviado pelo MikroTik

```json
{
  "router": "hEX",
  "interface": "pppoe-opennet"
}
```

---

## Data Table

Tabela:

```text
monitor_internet
```

O workflow atualiza somente:

```text
lastHeartbeat = $now
status = UP
```

Não alterar neste workflow:

```text
alertSent
wasOffline
outageStartedAt
```

Esses campos são controlados pelo workflow de monitoramento.

---

# 3. Configuração do MikroTik

O MikroTik utiliza o seguinte comando para enviar o heartbeat:

```routeros
/tool fetch url="https://n8n.eduardoferreira.space/webhook/mikrotik-heartbeat" \
http-method=post \
http-header-field="Content-Type: application/json" \
http-data="{\"router\":\"hEX\",\"interface\":\"pppoe-opennet\"}" \
output=none
```

Esse comando deve ser executado periodicamente pelo MikroTik.

---

## Exemplo de Scheduler

O heartbeat deve ser executado em um intervalo menor que o limite de 90 segundos utilizado pelo monitoramento.

Exemplo:

```text
Intervalo: 60 segundos
```

A ideia é:

```text
Heartbeat
    ↓
60s
    ↓
Heartbeat
    ↓
60s
    ↓
Heartbeat
```

Se os heartbeats deixarem de chegar, o n8n detectará a ausência de comunicação.

---

# 4. Data Table

Nome:

```text
monitor_internet
```

Tabela utilizada pelo sistema para armazenar o estado atual do monitoramento.

## Estrutura

| Campo             | Tipo    | Função                                        |
| ----------------- | ------- | --------------------------------------------- |
| `failures`        | Number  | Contador de falhas                            |
| `status`          | String  | Estado atual do link                          |
| `lastHeartbeat`   | String  | Data/hora do último heartbeat                 |
| `alertSent`       | Boolean | Indica se o alerta de queda já foi enviado    |
| `wasOffline`      | Boolean | Indica se o link estava anteriormente offline |
| `outageStartedAt` | String  | Data/hora em que a queda foi registrada       |

---

## Estado inicial

A tabela deve possuir uma linha com:

```text
id: 1
failures: 0
status: UP
lastHeartbeat: vazio
alertSent: false
wasOffline: false
outageStartedAt: vazio
```

Depois que o MikroTik enviar o primeiro heartbeat, `lastHeartbeat` será preenchido automaticamente.

---

# 5. Workflow — MikroTik - Monitoramento

## Objetivo

Verificar continuamente se o MikroTik continua enviando heartbeat.

Estrutura:

```text
Schedule Trigger
       ↓
Get row(s)
       ↓
Verificar Heartbeat
       ↓
Verificar se está offline
       │
       ├── TRUE
       │     ↓
       │  Alerta ainda não enviado?
       │     │
       │     ├── TRUE
       │     │    ↓
       │     │ Montar Alerta de Queda
       │     │    ↓
       │     │ Telegram
       │     │    ↓
       │     │ Update row(s)
       │
       └── FALSE
              ↓
           WasOffline?
              │
              ├── TRUE
              │    ↓
              │ Montar Recuperação
              │    ↓
              │ Telegram
              │    ↓
              │ Update row(s)
              │
              └── FALSE
                   fim
```

---

# 6. Schedule Trigger

Intervalo atual:

```text
5 segundos
```

O workflow consulta a tabela a cada 5 segundos.

Isso permite detectar a ausência de heartbeat sem precisar alterar o intervalo utilizado pelo MikroTik.

---

# 7. Verificar Heartbeat

O node Code calcula quanto tempo passou desde o último heartbeat.

```javascript
const data = $input.first().json;

const agora = new Date();
const ultimoHeartbeat = new Date(data.lastHeartbeat);

const diferencaMs = agora.getTime() - ultimoHeartbeat.getTime();
const segundosSemHeartbeat = Math.floor(diferencaMs / 1000);

// Considera offline após 90 segundos sem heartbeat
const offline = segundosSemHeartbeat > 90;

return [
  {
    json: {
      ...data,
      offline,
      segundosSemHeartbeat
    }
  }
];
```

---

## Limite de queda

O sistema considera o link offline quando:

```text
> 90 segundos
```

sem receber heartbeat.

Esse valor deve ser mantido em conjunto com o intervalo configurado no MikroTik.

---

# 8. Verificar se está offline

Condição:

```text
{{ $json.offline }}
```

Verificação:

```text
is true
```

### TRUE

O link está offline.

O fluxo continua para:

```text
Alerta ainda não enviado?
```

### FALSE

O link está funcionando.

Essa saída não precisa ser conectada diretamente a nenhum node.

Ela segue para a lógica de recuperação através do fluxo:

```text
WasOffline?
```

---

# 9. Controle de alerta

O campo:

```text
alertSent
```

impede que o Telegram receba dezenas de mensagens durante uma única queda.

Exemplo:

```text
Internet cai
     ↓
90s
     ↓
Telegram 🚨
     ↓
alertSent = true
```

Enquanto o link continuar offline:

```text
alertSent = true
```

Portanto, nenhuma nova mensagem de queda será enviada.

---

# 10. Alerta de queda

Mensagem enviada:

```text
🚨 ALERTA DE INTERNET

🔴 LINK OFFLINE

🔌 Interface: pppoe-opennet
🖥️ Router: hEX

❌ O MikroTik parou de enviar o heartbeat.

🕐 Último heartbeat: ...
🕐 Verificação: ...

⏱️ Sem comunicação: ...

⚠️ Verifique o link PPPoE.
```

O tempo sem comunicação é calculado automaticamente.

---

# 11. Registro da queda

Após o envio do alerta, a tabela é atualizada:

```text
alertSent = true
wasOffline = true
status = DOWN
outageStartedAt = $now
```

Exemplo:

```text
status: DOWN
alertSent: true
wasOffline: true
outageStartedAt: 2026-10-01T10:30:00
```

O campo `outageStartedAt` é importante porque será utilizado para calcular a duração da indisponibilidade.

---

# 12. Recuperação

Quando um novo heartbeat chega, o sistema volta a considerar o link online.

Porém, se:

```text
wasOffline = true
```

o sistema entende que houve uma queda anterior e envia uma mensagem de recuperação.

---

# 13. Montar Recuperação

Código utilizado:

```javascript
const data = $input.first().json;

const agora = new Date();
const inicioQueda = new Date(data.outageStartedAt);

// Evita NaN caso o timestamp esteja inválido
const inicioValido = !Number.isNaN(inicioQueda.getTime());

const inicio = inicioValido
  ? inicioQueda
  : new Date(data.lastHeartbeat);

const tempoOfflineMs = Math.max(
  0,
  agora.getTime() - inicio.getTime()
);

const totalSegundos = Math.floor(tempoOfflineMs / 1000);

const horas = Math.floor(totalSegundos / 3600);
const minutos = Math.floor((totalSegundos % 3600) / 60);
const segundos = totalSegundos % 60;

let tempoOffline;

if (horas > 0) {
  tempoOffline = `${horas}h ${minutos}min ${segundos}s`;
} else if (minutos > 0) {
  tempoOffline = `${minutos}min ${segundos}s`;
} else {
  tempoOffline = `${segundos}s`;
}

const inicioFormatado = inicio.toLocaleString('pt-BR', {
  timeZone: 'America/Fortaleza'
});

const restabelecimento = agora.toLocaleString('pt-BR', {
  timeZone: 'America/Fortaleza'
});

const mensagem =
`🟢 *INTERNET RESTABELECIDA*

🔌 *Interface:* \`pppoe-opennet\`
🖥️ *Router:* \`hEX\`

🕐 *Início da queda:* ${inicioFormatado}
🕐 *Restabelecimento:* ${restabelecimento}

⏱️ *Tempo offline:* ${tempoOffline}

✅ O link voltou a responder normalmente.`;

return [
  {
    json: {
      ...data,
      mensagem,
      tempoOffline
    }
  }
];
```

---

# 14. Alerta de recuperação

Mensagem enviada:

```text
🟢 INTERNET RESTABELECIDA

🔌 Interface: pppoe-opennet
🖥️ Router: hEX

🕐 Início da queda: ...
🕐 Restabelecimento: ...

⏱️ Tempo offline: ...

✅ O link voltou a responder normalmente.
```

---

# 15. Reset após recuperação

Depois do envio da mensagem de recuperação, a tabela deve voltar para o estado normal:

```text
status = UP
alertSent = false
wasOffline = false
outageStartedAt = vazio
```

Isso permite que uma nova queda gere um novo alerta.

---

# 16. Máquina de estados

O funcionamento pode ser resumido em quatro estados:

```text
NORMAL
  │
  │ heartbeat para
  ▼
OFFLINE
  │
  │ alerta enviado
  ▼
OFFLINE + ALERTA ENVIADO
  │
  │ heartbeat retorna
  ▼
RECUPERADO
  │
  │ estado resetado
  ▼
NORMAL
```

---

# 17. Comportamento esperado

## Internet funcionando

```text
MikroTik → heartbeat → n8n

status = UP
alertSent = false
wasOffline = false
```

Nenhum alerta é enviado.

---

## Internet cai

O MikroTik deixa de enviar heartbeat.

Após mais de 90 segundos:

```text
status = DOWN
alertSent = true
wasOffline = true
outageStartedAt = horário da detecção
```

Telegram:

```text
🚨 LINK OFFLINE
```

---

## Internet continua offline

O n8n continua verificando.

Nenhum novo alerta é enviado.

Isso evita spam no Telegram.

---

## Internet retorna

O MikroTik volta a enviar heartbeat.

O n8n identifica:

```text
offline = false
wasOffline = true
```

Telegram:

```text
🟢 INTERNET RESTABELECIDA
```

Depois:

```text
status = UP
alertSent = false
wasOffline = false
outageStartedAt = vazio
```

---

# 18. Segurança

## Nunca colocar no GitHub

Não versionar:

* token do Telegram;
* senhas;
* credenciais do n8n;
* credenciais do MikroTik;
* cookies;
* API keys;
* secrets;
* arquivos `.env` contendo credenciais.

As credenciais devem permanecer nas **Credentials do n8n** ou em mecanismos apropriados de secret management.

---

# 19. Reconstrução da VPS

Caso seja necessário reconstruir a VPS do zero, seguir esta ordem.

## 1 — Preparar o servidor

Instalar:

```text
Docker
Docker Compose
```

---

## 2 — Subir o n8n

Restaurar o container do n8n utilizando a configuração adotada pelo ambiente.

Depois acessar:

```text
https://n8n.eduardoferreira.space
```

---

## 3 — Configurar domínio

Garantir que:

```text
n8n.eduardoferreira.space
```

aponte para a VPS.

Também garantir HTTPS válido.

---

## 4 — Configurar n8n

Restaurar:

* Credentials;
* Data Tables;
* workflows.

---

## 5 — Criar Data Table

Criar:

```text
monitor_internet
```

Com os campos:

```text
failures
status
lastHeartbeat
alertSent
wasOffline
outageStartedAt
```

Criar a linha:

```text
id = 1
```

---

## 6 — Importar Workflow Heartbeat

Criar:

```text
MikroTik - Heartbeat
```

Configurar o webhook:

```text
POST /webhook/mikrotik-heartbeat
```

---

## 7 — Importar Workflow Monitoramento

Criar:

```text
MikroTik - Monitoramento
```

Configurar:

```text
Schedule: 5 segundos
Limite: 90 segundos
```

---

## 8 — Configurar Telegram

Criar/restaurar a credencial do bot no n8n.

Configurar o Chat ID correto.

Nunca colocar o token diretamente nos nodes ou no GitHub.

---

## 9 — Configurar MikroTik

Restaurar o scheduler responsável pelo heartbeat.

Endpoint:

```text
https://n8n.eduardoferreira.space/webhook/mikrotik-heartbeat
```

---

# 20. Teste após reconstrução

Antes de considerar o sistema restaurado, verificar:

### Teste 1 — Heartbeat

Executar manualmente o comando no MikroTik.

Verificar se:

```text
lastHeartbeat
```

foi atualizado.

---

### Teste 2 — Monitoramento

Confirmar que o workflow:

```text
MikroTik - Monitoramento
```

está executando a cada 5 segundos.

---

### Teste 3 — Queda

Interromper temporariamente o envio de heartbeat.

Aguardar mais de:

```text
90 segundos
```

Deve chegar:

```text
🚨 LINK OFFLINE
```

---

### Teste 4 — Recuperação

Restaurar o heartbeat.

Deve chegar:

```text
🟢 INTERNET RESTABELECIDA
```

---

### Teste 5 — Anti-spam

Durante uma queda prolongada, deve existir somente:

```text
1 alerta de queda
```

e não uma mensagem a cada execução do workflow.

---

# 21. Troubleshooting

## Não chega alerta de queda

Verificar:

```text
1. MikroTik está enviando heartbeat?
2. lastHeartbeat está sendo atualizado?
3. Monitoramento está ativo?
4. offline está ficando true?
5. alertSent está false?
6. Telegram está configurado?
```

---

## Telegram não envia

Verificar:

```text
1. Credential do Telegram
2. Chat ID
3. Node Telegram
4. Execução do workflow
```

---

## Alerta é enviado várias vezes

Verificar:

```text
alertSent
```

Após o primeiro alerta deve ficar:

```text
true
```

---

## Recuperação não é enviada

Verificar:

```text
wasOffline
```

Durante a queda deve estar:

```text
true
```

Quando o heartbeat voltar:

```text
offline = false
wasOffline = true
```

Isso dispara o fluxo de recuperação.

---

## Tempo offline aparece incorreto

Verificar:

```text
outageStartedAt
```

Esse campo deve ser preenchido no momento em que a queda é detectada.

Ele é utilizado para calcular a duração da indisponibilidade.

---

# 22. Checklist de reconstrução

```text
[ ] VPS instalada
[ ] Docker instalado
[ ] n8n funcionando
[ ] DNS configurado
[ ] HTTPS funcionando
[ ] Credentials restauradas
[ ] Data Table monitor_internet criada
[ ] Linha ID 1 criada
[ ] Workflow MikroTik - Heartbeat criado
[ ] Workflow MikroTik - Monitoramento criado
[ ] Telegram configurado
[ ] MikroTik configurado
[ ] Heartbeat recebido
[ ] lastHeartbeat atualizado
[ ] Teste de queda realizado
[ ] Alerta de queda recebido
[ ] Teste de recuperação realizado
[ ] Alerta de recuperação recebido
[ ] Anti-spam validado
```

---

# 23. Informações importantes

### Endpoint

```text
https://n8n.eduardoferreira.space/webhook/mikrotik-heartbeat
```

### Router

```text
hEX
```

### Interface

```text
pppoe-opennet
```

### Data Table

```text
monitor_internet
```

### ID da linha

```text
1
```

### Intervalo de monitoramento

```text
5 segundos
```

### Limite de offline

```text
90 segundos
```

### Timezone

```text
America/Fortaleza
```

---

# 24. Objetivo do projeto

Este projeto existe para fornecer um monitoramento simples e independente da disponibilidade do link de Internet.

A arquitetura foi mantida propositalmente simples:

```text
MikroTik
   ↓
HTTP Heartbeat
   ↓
n8n
   ↓
Data Table
   ↓
Telegram
```

A principal vantagem é que, em caso de reconstrução da VPS, toda a lógica necessária para restaurar o monitoramento está documentada neste README.
