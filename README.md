# WhatsApp para Sea

Biblioteca em **Sea** para a **WhatsApp Business Cloud API** da Meta (Graph API). Com ela você envia mensagens de todos os tipos, gerencia mídia, templates, números e o perfil comercial, e recebe webhooks com a verificação de assinatura.

Ela é Sea puro e usa a lib `http` do [sea_vm](../sea_vm/libs/http), com HTTPS via `libs/ssl`.

```sea
local wa = require("./whatsapp")

local client = wa.new({token = os.getenv("WA_TOKEN"), phone_number_id = os.getenv("WA_PHONE_ID")})?
local r = client:send_text("5511999999999", "Olá do Sea!")?
print(wa.message_id(r))   -- wamid.HBgM...
```

---

## 1. Requisitos

- `../sea_vm` ao lado deste diretório, com `bin/sea` compilado (`make`) e as libs nativas `http` e `ssl` (`make -C libs/http`, `make -C libs/ssl`).
- Um app na Meta com o produto WhatsApp. Em *WhatsApp › API Setup* você encontra o **token** de acesso, o **Phone number ID** e o **WhatsApp Business Account ID**. Em produção, use o token de um *System User*.

## 2. Cliente

```sea
local client = wa.new({
    token = "...",              -- obrigatório
    phone_number_id = "...",    -- mensagens, mídia, perfil, registro
    waba_id = "...",            -- templates e lista de números
    version = "v23.0",          -- versão da Graph API (padrão)
    timeout = 30000,            -- ms por operação de rede
    retries = 2,                -- novas tentativas em falha de rede, 429 e 5xx (padrão 0)
})?
```

Cada chamada retorna a resposta da Graph API já decodificada ou um **valor de erro**:

```sea
local r = client:send_text("000", "oi")
if is_error(r) then
    print(r.message)                -- whatsapp: (#131030) Recipient phone number not in allowed list: ... (HTTP 400, OAuthException, fbtrace_id ...)
    print(wa.error_code(r))         -- 131030
    print(client.last_error.status, client.last_error.details)
end
```

Números de telefone são normalizados (`"+55 (11) 98765-4321"` vira `"5511987654321"`).

## 3. Enviando mensagens

Todas as funções `send_*` aceitam um `opts` final com `reply_to` (id da mensagem citada) e `biz_opaque_callback_data`.

```sea
client:send_text(to, "Veja https://sea.dev", {preview_url = true, reply_to = msg.id})

-- mídia: URL, id de mídia enviada por upload_media, ou {id|link}
client:send_image(to, "https://exemplo.com/foto.jpg", {caption = "legenda"})
client:send_video(to, media_id, {caption = "..."})
client:send_audio(to, media_id)
client:send_document(to, media_id, {filename = "relatorio.pdf", caption = "..."})
client:send_sticker(to, media_id)

client:send_location(to, {latitude = -23.55, longitude = -46.63, name = "Paulista", address = "Av. Paulista"})
client:send_contacts(to, [{name = {formatted_name = "Ana", first_name = "Ana"}, phones = [{phone = "+5511999999999", type = "CELL"}]}])
client:send_reaction(to, message_id, "👍")      -- "" remove a reação

-- interativas
client:send_buttons(to, "Confirma o pedido?", [{id = "sim", title = "Sim"}, {id = "nao", title = "Não"}],
    {header = "Pedido #12", footer = "Loja Sea"})         -- até 3 botões; strings também servem
client:send_list(to, "Escolha um sabor", "Ver cardápio", [
    {title = "Pizzas", rows = [{id = "mus", title = "Mussarela", description = "Clássica"}]},
])                                                          -- até 10 linhas
client:send_cta_url(to, "Acompanhe sua entrega", "Rastrear", "https://exemplo.com/r/12")
client:send_location_request(to, "Envie sua localização para calcular o frete")
client:send_flow(to, "Complete seu cadastro", {id = "FLOW_ID", cta = "Abrir", token = "sessao-1", screen = "START"})
client:send_interactive(to, {...})        -- objeto interactive bruto
client:send({to = to, type = "...", ...}) -- qualquer mensagem no formato da Cloud API

-- leitura e "digitando..."
client:mark_read(msg.id)
client:typing(msg.id)                     -- marca como lida e mostra "digitando..." (até 25 s)
```

### Templates

Fora da janela de 24 h desde a última mensagem do usuário, só templates aprovados são entregues (erro `131047`).

```sea
local tpl = wa.template
client:send_template(to, "hello_world", "en_US")
client:send_template(to, "pedido_enviado", "pt_BR", [
    tpl.header(tpl.image("https://exemplo.com/logo.png")),
    tpl.body("Ana", "#1234", tpl.currency("BRL", 99.9), tpl.date_time("22/10/2026")),
    tpl.button_url(0, "1234"),                 -- sufixo dinâmico da URL do botão 0
    tpl.button_quick_reply(1, "PARAR"),
])
```

Builders: `text(v [, nome])` (parâmetros nomeados), `currency`, `date_time`, `image`, `video`, `document`, `location`, `body(...)`, `header(...)`, `button_url`, `button_quick_reply`, `button_copy_code` e `button_otp`. Uma string ou um número entre os parâmetros vira um parâmetro de texto.

## 4. Mídia

```sea
local up = client:upload_media("foto.png")?                         -- {id}; o tipo vem da extensão
local up2 = client:upload_media("nota.pdf", {data = bytes, mime_type = "application/pdf"})?
local info = client:get_media(up.id)?                               -- {url, mime_type, sha256, file_size}
local bytes = client:download_media(msg.media.id)?                  -- aceita id ou URL
client:download_media(msg.media.id, "/tmp/recebida.jpg")?           -- grava em arquivo
client:delete_media(up.id)
```

## 5. Gerenciamento

```sea
-- templates (waba_id)
client:list_templates({status = "APPROVED", limit = 50})   -- uma página: {data, paging}
client:all_templates()                                       -- todas as páginas
client:create_template({name = "boas_vindas", language = "pt_BR", category = "UTILITY",
    components = [{type = "BODY", text = "Olá {{1}}!", example = {body_text = [ ["Ana"] ]}}]})
client:edit_template(template_id, {components = [...]})
client:delete_template("boas_vindas" [, template_id])

-- números
client:list_phone_numbers()
client:get_phone_number()            -- número configurado: nome verificado, quality_rating, ...
client:register(123456)              -- PIN de 6 dígitos da verificação em duas etapas
client:deregister()
client:request_code("SMS", "pt_BR")
client:verify_code("123456")
client:set_two_step_pin(654321)

-- perfil comercial
client:get_business_profile()        -- {about, address, description, email, websites, vertical, ...}
client:update_business_profile({about = "Atendimento 24h", websites = ["https://sea.dev"]})

-- qualquer outro endpoint
client:get("123/fields", query)
client:post("123/edge", body)
client:delete("123")
client:next_page(resposta)
client:all_pages(path, query)
```

## 6. Webhooks

`wa.webhook.handler(opts)` devolve um handler para `http.listen`, `http.serve` ou uma rota de `http.router()`. Ele responde ao GET de verificação, confere o `X-Hub-Signature-256` quando `app_secret` é informado e transforma cada notificação em eventos planos.

```sea
local http = require("../sea_vm/libs/http/http")
local wa = require("./whatsapp")

http.serve(8080, wa.webhook.handler({
    verify_token = os.getenv("WA_VERIFY_TOKEN"),
    app_secret = os.getenv("WA_APP_SECRET"),
    async = true,                              -- responde 200 na hora e processa em uma goroutine
    on_message = function(msg)
        client:typing(msg.id)
        client:send_text(msg.from, `Olá ${msg.name}, você disse: ${msg.text}`)
    end,
    on_status = function(st) print(st.id, st.status) end,   -- sent, delivered, read, failed
    on_event = function(e) end,                -- todos os eventos (inclusive "change" e "error")
}))
```

Evento de mensagem (`kind = "message"`):

| Campo | Conteúdo |
|-------|----------|
| `type` | `text`, `image`, `video`, `audio`, `document`, `sticker`, `location`, `contacts`, `interactive`, `button`, `reaction`, `order`, `system`, ... |
| `id`, `from`, `name`, `timestamp` | id `wamid`, número do remetente, nome no perfil, unix (int) |
| `text` | corpo do texto, legenda da mídia ou título do botão/linha escolhida |
| `reply` | `{type, id, title}` de respostas a botões, listas e botões de template; `{type = "flow", data}` de um Flow |
| `media` | `{id, mime_type, sha256, caption, filename}` |
| `location`, `contacts`, `reaction`, `order` | conforme o tipo |
| `reply_to` | id da mensagem citada pelo usuário |
| `phone_number_id`, `raw` | número que recebeu e a mensagem original |

Evento de status (`kind = "status"`): `id`, `status`, `recipient_id`, `timestamp`, `errors`, `conversation`, `pricing`. Outros campos assinados (`message_template_status_update`, `account_update`, ...) chegam como `{kind = "change", field, value}`.

As peças também podem ser usadas separadamente: `wa.webhook.verify(query, token)`, `wa.webhook.valid_signature(body, header, secret)` e `wa.webhook.parse(body)`.

## 7. Exemplos

```sh
export WA_TOKEN=... WA_PHONE_ID=... WA_WABA_ID=...
../sea_vm/bin/sea examples/01_send_text.sea 5511999999999          # texto, imagem e botões
../sea_vm/bin/sea examples/02_send_template.sea 5511999999999      # template hello_world
WA_VERIFY_TOKEN=segredo ../sea_vm/bin/sea examples/03_echo_bot.sea 8080   # bot via webhook
../sea_vm/bin/sea examples/04_media_and_account.sea 5511999999999 relatorio.pdf
```

Para o bot, exponha a porta via HTTPS (por exemplo `cloudflared tunnel --url http://localhost:8080`), cadastre `https://.../webhook` como *Callback URL* e assine o campo `messages`.

## 8. Testes

```sh
../sea_vm/bin/sea test
```

`whatsapp_test.sea` testa o cliente contra uma Graph API falsa em loopback: o formato de cada mensagem, os erros, o upload multipart, o download, a paginação e os retries. `webhook_test.sea` cobre a verificação, a assinatura, o parsing e o handler HTTP. Nenhum dos dois precisa de rede nem de conta na Meta.

## 9. Arquivos

| Arquivo | Conteúdo |
|---------|----------|
| `whatsapp.sea` | Entrada: `wa.new`, `wa.message_id`, `wa.error_code`, `wa.template`, `wa.webhook` |
| `client.sea` | Configuração, requisições autenticadas, erros, paginação, retries |
| `messages.sea` | `send_*`, `mark_read`, `typing` |
| `media.sea` | Upload, consulta, download e exclusão de mídia |
| `manage.sea` | Templates, números de telefone, perfil comercial |
| `template.sea` | Builders de componentes de template |
| `webhook.sea` | Verificação, assinatura, parsing e handler |
| `util.sea` | Normalização de números, objetos de mídia, multipart, tipos MIME |
