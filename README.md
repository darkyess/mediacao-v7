# MediAção — V7

Projeto escolar: assistente de apoio ao atendimento em saúde. Não faz diagnóstico.

## Como executar
- **Sem IA:** abra `index.html` no navegador. Tudo funciona, exceto o passo opcional "descreva com suas palavras".
- **Com IA:** publique no Cloudflare Pages (veja `LEIA-ME-IA.md`).
- **Desenvolvimento local com IA:** configure `.dev.vars` a partir de `.dev.vars.example` e rode `npx wrangler pages dev .` (veja `LEIA-ME-IA.md`).

## Estrutura
- `index.html`: telas e conteúdo.
- `style.css`: visual responsivo e estilo de impressão do resumo.
- `script.js`: fluxo do chatbot, regras de sinais de alerta e geração do resumo.
- `functions/api/organizar.js`: função do servidor que chama a IA (guarda a chave).
- `functions/api/config.js`: entrega ao navegador a configuração pública (chave pública do Turnstile, se houver).
- `_headers`: cabeçalhos de segurança (CSP etc.) usados pelo Cloudflare Pages.
- `favicon.svg`: ícone da aba.

## Privacidade
- Por padrão, os dados ficam somente no navegador e não são salvos.
- Se a pessoa escolher o passo com IA (e aceitar o aviso), **o texto que ela escreve é enviado ao Google (Gemini, plano gratuito)**, que pode usá-lo para melhorar seus produtos. Por isso o site pede para não escrever dados pessoais reais.
- Sinais de alerta são detectados por regras fixas; a IA só pode acrescentar cautela, nunca retirá-la.

## Problemas com a IA ou com a chave Gemini

A chave não fica no navegador nem no código: no Cloudflare Pages, configure-a como Secret `GEMINI_API_KEY` para o ambiente correto (Production e, se usar, Preview). Depois de alterar um Secret, faça um novo deploy. No computador, use `.dev.vars` local; não compartilhe nem envie esse arquivo ao GitHub.

Se a IA parar, gerar outra chave de imediato pode esconder a causa. O servidor registra o status e o código de erro retornados pelo Google, sem registrar a chave nem o texto enviado. Consulte os logs da função Pages no horário da tentativa e use este guia:

| Status/código nos logs | O que costuma indicar | O que conferir |
|---|---|---|
| `429` / `RESOURCE_EXHAUSTED` | Limite de requisições ou cota do projeto esgotada; trocar a chave do mesmo projeto normalmente não aumenta a cota. | Uso e limites no Google AI Studio/Google Cloud; aguarde a renovação ou ajuste o projeto/plano. |
| `401` / `UNAUTHENTICATED` | Chave inválida, revogada, copiada incompleta ou restrita para outro uso. | Secret `GEMINI_API_KEY`, projeto de origem e restrições aplicadas à chave. |
| `403` / `PERMISSION_DENIED` | Projeto sem acesso à API/modelo, API desabilitada ou política/restrição de chave. | Projeto associado à chave, acesso à Gemini API e restrições da chave. |
| `404` / `NOT_FOUND` | Modelo ou endpoint não encontrado. | `GEMINI_MODEL` e nome do modelo; isso não é, por si só, sinal de chave inválida. |
| `5xx`, `504` ou falha de rede | Indisponibilidade temporária do provedor, timeout ou problema de conexão. | Logs e nova tentativa mais tarde; trocar a chave geralmente não ajuda. |
| Sem chamada ao provedor / `config` | Secret ausente no ambiente publicado ou deploy antigo. | Nome exato `GEMINI_API_KEY`, ambiente e novo deploy. |

O código padrão do projeto é `gemini-3.5-flash-lite`. O nome consta como modelo estável na documentação do Gemini; portanto, uma falha recorrente não deve ser atribuída automaticamente a uma chave expirada. A causa concreta só pode ser determinada pelos logs da execução que falhou e pelo estado do projeto/chave no Google.

Para passos detalhados de configuração e proteção de cota, consulte [`LEIA-ME-IA.md`](LEIA-ME-IA.md).

## Recursos de segurança e acessibilidade incluídos desde a V4
- Pergunta de sinais de alerta mais completa (dor no peito, AVC, convulsão, pensamentos de se machucar com CVV 188) e presente também em "Organizar meus sintomas".
- Detecção de alertas no texto livre ampliada.
- "Voltar uma pergunta" na confirmação da IA volta para a caixa de texto.
- Progresso mostra "Etapa opcional" no passo da IA e não salta.
- Avisos de emergência anunciados imediatamente por leitores de tela (`role="alert"`).
- Verificação anti-robô opcional (Cloudflare Turnstile) para proteger a cota da IA.
- Cabeçalhos de segurança (`_headers`) e remoção de `onclick` do HTML.
- Botão "Imprimir / salvar PDF" e estilo de impressão.
- Hierarquia de títulos corrigida, `<noscript>`, descrição, favicon e prévia para redes sociais.
- No celular, o aviso "não diagnostica" aparece antes dos botões.

## Próximas versões
- persistência local opcional;
- painel para profissionais;
- base de informações oficiais.

## Revisão por profissional de saúde

O conteúdo de saúde do site (sinais de alerta, mensagens de emergência, checklists, orientações por tipo de atendimento
e instruções dadas à IA) pode ser exportado em um único documento para um profissional revisar:

```
node tools/exportar-revisao.mjs
```

Isso gera `docs/REVISAO-CLINICA.md`, sempre a partir do código atual. O documento também testa a rede de segurança do
texto livre com frases de exemplo e **mostra as lacunas**. Depois da revisão, atualize o status na seção
"Fontes e revisão" (tela Sobre, `id="reviewStatus"` no `index.html`) com nome, profissão e data, **somente com o consentimento do revisor**.

## Perguntas para o profissional e serviços por perto

No resumo, abaixo das orientações, o site mostra:

- **Perguntas para fazer ao profissional**: lista fixa (sem IA), conforme o tipo de consulta e o motivo informado, com caixinhas
  para marcar. Entra no texto copiado e na impressão. Não dá diagnóstico nem indica remédio.
- **Serviços perto de você**: botões que abrem o Google Maps em outra aba (UBS, UPA 24h, pronto-socorro, CEO, farmácia,
  laboratório, conforme a situação). O link leva **só o tipo de serviço** (e a cidade, se a pessoa digitar). Nenhum sintoma,
  resposta ou texto do resumo vai para o mapa.

Quando a pessoa marca um sinal de alerta, **nenhum dos dois blocos aparece**: o foco é ligar para o SAMU (192) ou o CVV (188).
As perguntas e os serviços também são exportados para o documento de revisão (`node tools/exportar-revisao.mjs`).

## Diário de sintomas

Função **opcional** (`diario.js`): a pessoa anota como está ao longo dos dias (sintoma, data e hora, como está, intensidade de 0 a 10,
temperatura e observações) e leva o histórico para a consulta, copiando o texto ou imprimindo.

- **Privacidade:** os registros ficam só no `localStorage` do navegador e **só depois que a pessoa ativa o diário** marcando um
  consentimento. Nada é enviado a servidor nem à IA; o arquivo `diario.js` não tem nenhuma chamada de rede (há teste automático para isso).
- **Controle:** apagar um registro ou o diário inteiro sempre pede confirmação. Dados ilegíveis nunca são apagados sozinhos.
- **Sem interpretação:** o diário só guarda, mostra e exporta. Não calcula tendência nem sugere condutas.
- **A partir do resumo:** o botão "Salvar no diário" aparece só nos fluxos de sintomas e **nunca** quando há sinal de alerta.
- Os campos e os textos do diário também vão para o documento de revisão (`node tools/exportar-revisao.mjs`, Anexo C).
