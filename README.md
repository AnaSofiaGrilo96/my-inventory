# Inventário da casa — especificação e regras do projeto

Documento de continuidade: descreve o que a app é, como está montada e as regras
acordadas, para que qualquer pessoa (ou o Claude numa conversa nova) saiba
exatamente por onde pegar. **Não contém segredos** — o URL do script e o código
de acesso ficam fora do repositório (ver secção "Segurança").

## 1. O que é

App pessoal da Ana para inventariar o recheio da casa (artigos, detalhes,
localização, data de compra, valor, fatura, fotos), acessível no telemóvel e no
computador, com os dados guardados na sua Google Drive. Utilizadora única
(só a Ana). Interface em português de Portugal.

Campos por artigo: nome*, detalhes (texto livre), divisão, categoria, marca,
modelo, nº de série, data de compra, valor (€), loja, garantia em meses
(por defeito 36 = 3 anos legais em PT), fatura (1 ficheiro PDF ou imagem),
fotos (várias). A app calcula o fim da garantia e avisa quando falta pouco.

## 2. Arquitetura (decidida e a manter)

```
Telemóvel / PC  ──►  index.html (GitHub Pages, estático, público)
                          │  fetch POST (JSON, Content-Type text/plain)
                          ▼
                 Google Apps Script (Code.gs) — API protegida por código
                          │  DriveApp
                          ▼
                 Google Drive / pasta "Inventário Casa"
```

Porquê assim:
- Supabase ficou de fora (sem espaço na BD gratuita). A Drive é gratuita e já é
  "a cloud" da Ana, por isso **não há backup à parte: os dados nascem na Drive**.
- A interface **não** é servida pelo Apps Script (HtmlService) porque o Google
  mostra sempre o banner "esta aplicação foi criada por outro utilizador" em
  contas Gmail, sem forma de o desligar. Por isso a página vive no GitHub Pages
  e o script é só uma API JSON.
- Uma página publicada no claude.ai foi descartada: os dados ficariam numa BD
  interna que um timer do Apps Script não consegue ler.

### Componentes

| Onde | O quê |
|---|---|
| Repositório GitHub `AnaSofiaGrilo96/my-inventory` (público) | `index.html` (toda a UI: HTML+CSS+JS num só ficheiro), `manifest.webmanifest`, `icon-192.png`, `icon-512.png`, `apple-touch-icon.png`, `.nojekyll`, `.github/workflows/deploy-pages.yml`, este `README.md` |
| GitHub Pages | Fonte: **GitHub Actions** (workflow `deploy-pages.yml`: checkout → upload-pages-artifact → deploy-pages). Cada push para `main` publica sozinho. URL: `https://anasofiagrilo96.github.io/my-inventory/` |
| Google Apps Script (conta Google pessoal da Ana) | Projeto com `Code.gs`. Implementação como **Aplicação Web — Executar como: Eu; Acesso: Qualquer pessoa** (obrigatório para a página conseguir chamar o script; a proteção é o código). O `Index.html` antigo que lá exista já não é usado. |
| Google Drive | Pasta `Inventário Casa` com `inventario.json`, `Fotos/`, `Faturas/`, `Backups/` (cópia diária do JSON, últimas 30), `inventario.csv` (só quando se exporta) |

### Dados (`inventario.json`)

```json
{ "version": 1, "updatedAt": "ISO", "rooms": [...], "categories": [...],
  "items": [ { "id": "…", "name": "…", "details": "…", "room": "…", "category": "…",
               "brand": "…", "model": "…", "serial": "…", "purchaseDate": "YYYY-MM-DD",
               "price": 0, "warrantyMonths": 36, "store": "…",
               "invoice": { "id": "<id Drive>", "name": "…", "mime": "…" } | null,
               "photos": [ { "id": "<id Drive>", "name": "…" } ],
               "thumb": "data:image/jpeg;base64,… (capa 128px)",
               "createdAt": "ISO", "updatedAt": "ISO" } ] }
```

Divisões por defeito: Sala, Cozinha, Quarto, Casa de banho, Escritório, Hall,
Arrumos, Garagem, Exterior. Categorias: Eletrodomésticos, Eletrónica,
Mobiliário, Decoração, Roupa e têxteis, Ferramentas, Desporto e lazer,
Livros e media, Joias e relógios, Outro. Ambas editáveis nas Definições
(renomear propaga aos artigos).

### API do script (`doPost`, corpo JSON `{token, fn, args}` → `{result}` ou `{error}`)

`loadData`, `saveItem(item)`, `deleteItem(id)`, `saveLists({rooms, categories, renames})`,
`uploadFile(kind, itemId, name, mime, base64)`, `uploadFiles([...])` (vários num pedido),
`getFileData(id)`, `getFilesData([ids])` (devolve data URLs), `deleteFile(id)`, `exportCsv()`.
`doGet` responde apenas com uma mensagem de estado. Pedidos sem o código certo
recebem `{"error":"Código de acesso incorreto."}`. Gravações usam `LockService`.

## 3. Regras de funcionamento (acordadas)

- Fotos são reduzidas no browser antes de subir (máx. 1600 px, JPEG ~0,82) e
  gera-se uma miniatura de 128 px guardada no JSON como capa. Faturas em imagem
  são reduzidas a 2000 px; PDFs sobem tal como estão (máx. 10 MB).
- Fotos e fatura de um artigo sobem **num único pedido** (`uploadFiles`); as
  fotos de uma vista carregam-se num único pedido (`getFilesData`). O Apps
  Script demora 1–3 s por pedido, por isso minimizar pedidos é regra.
- A página mostra logo a última lista guardada no dispositivo (`localStorage`
  `inv-cache`) e atualiza em segundo plano; ao voltar à app (visibilitychange)
  recarrega a lista.
- Apagar um artigo envia as fotos e a fatura para o lixo da Drive.
- Garantia: verde = em garantia, laranja = acaba em ≤ 60 dias, vermelho =
  expirada. O Resumo lista as que acabam em ≤ 90 dias.
- Resumo: valor total, nº de artigos, sem fatura, sem fotos, totais por
  divisão e por categoria (útil para seguro). Exportar CSV (separador `;`,
  BOM UTF-8) cria `inventario.csv` na pasta da Drive.
- Lista: pesquisa (nome, detalhes, marca, modelo, nº série, loja, divisão,
  categoria) + chips de divisão e categoria; ordenação alfabética.
- Formulário: "+ Nova…" nos selects abre um campo de texto (nada de `prompt()`,
  não funciona em alguns contextos).
- Instalação no telemóvel: "Adicionar ao ecrã principal" (manifest + ícones).

## 4. Segurança

- O repositório é público (obrigatório para GitHub Pages gratuito) mas **nunca**
  pode conter o URL `/exec` do script, o `API_TOKEN` nem o link de ligação
  `#cfg=…`. Confirmar antes de cada commit.
- O código de acesso vive em `Code.gs` (`var API_TOKEN = '…'`) e, nos
  dispositivos ligados, em `localStorage` (`inv-cfg` = `{url, token}`).
- Ligar um dispositivo novo: Definições → "Ligar outro dispositivo" → copiar o
  link (contém URL + código no fragmento `#cfg=`, que nunca é enviado ao
  servidor; a página guarda-o e limpa-o do endereço). Guardar esse link só em
  sítios privados.
- Se o código for comprometido: mudar `API_TOKEN` no Code.gs → Implementar →
  Gerir implementações → nova versão → voltar a ligar os dispositivos.

## 5. Como atualizar

- **Interface**: substituir `index.html` no repositório (upload com o mesmo
  nome, ou editar no GitHub). O Actions publica em ~30 s. Ver o estado em
  Actions → "Deploy to GitHub Pages". Pode ser preciso fechar e reabrir a app
  no telemóvel para apanhar a versão nova.
- **Script**: colar o `Code.gs` novo no Apps Script → Implementar → Gerir
  implementações → ✎ → Versão: Nova versão → Implementar. O URL `/exec`
  mantém-se; os dados nunca são tocados por uma reimplementação.
- Mudar o modo de execução/acesso não perde dados: tudo está em ficheiros da
  Drive e o script procura a pasta pelo nome se os IDs guardados falharem.

## 6. Histórico de decisões

1. Primeira versão: web app inteira no Apps Script (HtmlService). Funcionou, mas
   banner do Google impossível de remover em conta Gmail, e lentidão (sem cache
   local no iframe do Apps Script).
2. Migração: UI para GitHub Pages + script como API com código de acesso;
   uploads em lote; cache local. Dados preservados.
3. Pages passou de "Deploy from a branch" para "GitHub Actions" (workflow
   próprio), a pedido da Ana.
4. "Enviar ligação" (partilha nativa) substituído por link sempre visível nas
   Definições com botão Copiar, porque a partilha falhava em silêncio no PC.

## 7. Ideias ainda não feitas (opcionais)

- Pesquisa/ordenação por valor ou data; filtro "sem fatura".
- Lembrete (e-mail via Apps Script com timer) quando uma garantia está a acabar.
- Exportação em PDF para a seguradora.
- Vários utilizadores (hoje a app é só da Ana; o código é único).
