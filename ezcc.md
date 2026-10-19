# ezcc — Reconciliação de recebimentos (Asaas → EZ Conta Corrente Clientes)

## Quando usar
Ricardo (contador, EZ Contabilidade, Santa Rosa/RS) vai anexar um extrato do Asaas (lista de
transações "Cobrança recebida") e pedir para rodar o prompt "ezcc". Ao ver esse pedido, siga
integralmente este documento sem pedir para o usuário reexplicar o processo.

## A planilha
Google Sheets "EZ CONTA CORRENTE CLIENTES", fileId:
`16W5fnw-4eN3bUjNLmDyZQ4NsNGRpkEILlKt2UXnF3w0`

Cada cliente tem sua própria aba (tab). Dentro de cada aba, colunas:
- A = DATA
- B = HISTÓRICO (texto como "HONORARIOS MM/YYYY" ou "REC HONORARIOS MM/YYYY")
- C = A RECEBER (valor da cobrança)
- D = RECEBIDO (valor efetivamente pago)
- E = SALDO (fórmula `=E(linha anterior)+D(linha atual)-C(linha atual)` — nunca mexer)
- F = OBS

Padrão de lançamento: uma linha "HONORARIOS MM/YYYY" (a cobrança da competência) é seguida por
uma linha "REC HONORARIOS MM/YYYY" (o recebimento daquela competência). É a linha REC que
recebe a DATA do pagamento (coluna A) e o VALOR recebido (coluna D).

**Competência vs vencimento**: a mensalidade de honorários da competência MM/AAAA vence no mês
seguinte (MM+1), normalmente por volta do dia 10.

## Passo 1 — Ler o extrato
O extrato do Asaas anexado traz linhas "Cobrança recebida" com data, descrição (contém o nome
do cliente, número da fatura e da nota fiscal) e valor. Extrair: data, valor, nome do cliente
(da descrição), número da fatura, número da NF.

## Passo 2 — Casar cliente do extrato com a aba certa
Os nomes no extrato raramente batem letra a letra com o nome da aba (razão social completa vs.
nome fantasia abreviado, por exemplo). Usar fuzzy matching (ex.: `difflib.SequenceMatcher`) do
nome extraído contra a lista de nomes de abas.

- Confiança ≥ 0.80: considerar casado, mas checar manualmente os casos limítrofes (0.80–0.90)
  olhando o valor e o histórico de pagamentos daquele cliente antes de aplicar.
- Confiança < 0.80: não aplicar automaticamente. Reunir esses casos e perguntar ao Ricardo
  (nome do extrato + os 2–3 candidatos mais prováveis de aba).
- Nomes do extrato que **não têm aba e devem ser ignorados** (o controle deles já fica
  centralizado no próprio Asaas — confirmado pelo Ricardo em 19/10/2026):
  - ELIAS RIBEIRO FERNANDES
  - SAMARA GOTTEMS DONATTI
- Mapeamentos já confirmados pelo Ricardo (usar direto, sem perguntar e sem depender do fuzzy):

  | Nome no extrato Asaas | Aba da planilha |
  |---|---|
  | TARIFA ZERO TELECOM LTDA | TARIFA ZERO TELECOM LTDA (renomeada em 2026-09; antes "TARIFA ZERO CONSULTORIA LTDA") |
  | ASSOCIACAO DE AMIGOS BALNEARIO PILATTI DE PORTO MAUA - RS | ASSOCIAÇÃO BALNEÁRIO PILATTI |
  | ASSOC DOS PRODUTORES DE HORTIGRANJEIROS DE SANTA ROSA | APRHOROSA (confirmado em 19/10/2026 — antes estava na lista de bloqueio porque o fuzzy casava errado) |
  | ASSOCIACAO DE FAM. E AMIGOS DE PESS. NECES. ESPECIAIS | AFAPENE |
  | ASSOCIACAO SANTA-ROSENSE DE ORNITOLOGOS AMADORES | ASOA |
  | ASSOCIACAO CASA ROTARIA DOS CLUBES DE ROTARY DE SANTA ROSA - RS | ASSOCIAÇÃO CASA ROTÁRIA |
  | ZENZEN INDUSTRIA DE ESQUADRIAS DE PVC LTDA | ZENZEN ESQUADRIAS DE PVC LTDA (fuzzy dá 0,79) |
  | STARTSYNC PRODUTORA DE FILMES LTDA | 604 MEDIA HOUSE LTDA (nome antigo da mesma empresa) |
  | JOAO C. JUNGES, JOELMIR G. WINCK, VANDO K. HARTMANN E VOLMIR F. PASCH LTDA | aba de mesmo nome. A aba **LEGVITA** é uma cópia antiga do mesmo cliente e não é atualizada — não lançar nela e não incluí-la no relatório |

- Para o fuzzy, normalizar antes de comparar (tirar acentos, pontuação e sufixos LTDA/ME/EPP/
  EIRELI/CIA). Com isso quase todos os nomes dão ≥ 0,90.

## Passo 3 — Lógica de reconciliação por aba ("janela" + duplicatas)

Para cada aba de cliente afetada pelo extrato (ou toda aba, se for uma varredura completa):

### 3a. Reconciliação por janela (window zip)
0. **Competência de cada linha REC = a da linha de cobrança logo acima dela**, não o texto da
   própria linha REC — os rótulos REC às vezes estão trocados (ex.: "REC HONORARIOS 082026"
   logo abaixo de "HONORARIOS 072026"). Uma linha totalmente em branco logo abaixo de uma linha
   de cobrança de honorários também conta como linha REC (preencher o rótulo
   "REC HONORARIOS MMAAAA" junto com data e valor — a fórmula de saldo já existe nela).
   Linhas REC abaixo de cobranças que não são honorários (ABERTURA, ALTERAÇÃO CNPJ, serviço
   avulso) não entram na janela.
1. Identificar todas as linhas "REC HONORARIOS MM/AAAA" cuja **vencimento** (competência + 1
   mês) caia dentro do período coberto pelo extrato atual (ex.: se o extrato cobre
   julho–setembro/2026, a janela é vencimento em jul/ago/set 2026).
2. Pegar os pagamentos do extrato casados com confiança a essa aba, ordenados por data.
   **Antes do zip, tirar os pagamentos que já estão lançados** em qualquer linha REC da aba
   (mesmo valor com tolerância de ~2%, data até 7 dias de diferença) — inclusive em linhas de
   serviço (ex.: parcela de alteração de CNPJ da 604 MEDIA, R$ 301,49 no Asaas x R$ 300,00
   lançado). Esses ficam como estão.
3. Ordenar as linhas REC da janela por ordem de competência (mais antiga primeiro). Só são
   "vagas" as linhas REC da janela que estão **vazias** ou que contêm uma **duplicata** (mesma
   data e valor de outra linha REC acima — é o bug sistêmico). Linhas da janela já preenchidas
   corretamente (ex.: pagamento de agosto feito antes do período do extrato) **não** são
   sobrescritas.
4. Fazer o "zip" 1-para-1: o pagamento restante mais antigo vai na vaga de competência mais
   antiga, e assim por diante.
   Se sobrar uma vaga que era duplicata e não recebeu pagamento, limpar (data e valor).
5. Se sobrarem linhas REC da janela sem pagamento correspondente (menos pagamentos que linhas):
   deixar essas linhas em branco (cliente ainda não pagou aquela competência) — não inventar
   valor.
6. Se sobrarem pagamentos sem linha REC correspondente (mais pagamentos que linhas da janela):
   **não aplicar automaticamente** — separar esse cliente para revisão manual com o Ricardo,
   mostrando as datas/valores do extrato e o estado atual das linhas daquela aba. Foi o caso do
   VINICIUS FINGER LTDA (dois pagamentos no mesmo dia 13/07 com valores diferentes de 444,00,
   sem bater exatamente com a mensalidade padrão — só o Ricardo sabia a qual competência cada
   um se referia).

### 3b. Duplicatas em competências futuras
Varrer as linhas REC da aba e procurar pares com a MESMA data e MESMO valor (duplicata exata).
- **Limpar** (data E valor) a duplicata quando ela está numa competência **futura** (ainda não
  vencida, ex.: 10/2026 num extrato até 18/10/2026) **e** esse pagamento (data+valor) aparece
  no extrato do Asaas — é o bug sistêmico (pagamento colado por engano na competência
  seguinte). Manter a linha de menor número.
- **Não limpar** duplicatas cujo pagamento não está no extrato do Asaas: podem ser
  adiantamento pago pelo banco. Ex.: AFAPENE pagou adiantado em 25/09/2026 pelo Sicredi e o
  valor aparece repetido nas competências 10, 11 e 12/2026 — está **correto** (confirmado).
- **Não limpar** duplicatas históricas (as duas linhas em competências já vencidas — havia 52
  casos entre 2017 e jul/2026): muitas são dois meses pagos no mesmo dia. Só listar se o
  Ricardo pedir.

### 3c. Quando os dois processos colidem
Sempre rodar 3a e 3b juntos por aba (não são excludentes). Se depois da limpeza de duplicatas
sobrar uma linha REC "solta" com data/valor que na verdade pertence a outra competência (ex.:
um pagamento de maio lançado por engano na linha de setembro, sem duplicata exata em outro
lugar), NÃO tentar realocar sozinho — é o mesmo tipo de caso do Vinicius Finger. Perguntar ao
Ricardo a que competência aquele valor se refere antes de mover.

### 3d. Regras de clientes específicos (confirmadas, não perguntar de novo)
- **Serviço avulso pago numa fatura separada sem linha de cobrança na aba** (ex.: DE LA VEGA,
  R$ 1.621 em 15/10/2026): criar a cobrança e o recebimento — inserir 2 linhas logo depois da
  última linha REC já vencida, com "SERVIÇO AVULSO" (C = valor) e "REC SERVIÇO AVULSO"
  (D = valor), ambas com a data do pagamento; escrever a fórmula de saldo nas 2 linhas novas
  e **corrigir a fórmula de saldo da linha seguinte**, que continua apontando para a linha
  antiga depois da inserção.
- **FAEGLE ARTIGOS ESPORTIVOS**: a abertura de CNPJ (R$ 1.300) é paga em parcelas de ~R$ 433;
  não precisa lançar as parcelas — o saldo da aba vai diminuindo. Lançar só os honorários.
- **LP ASSESSORIA EM SISTEMAS**: honorário mensal de R$ 299,00. Se faltarem linhas de
  cobrança para competências novas, criar "HONORÁRIOS MMAAAA" (data = último dia do mês,
  C = 299) + "REC HONORÁRIOS MMAAAA" até dezembro.
- Se faltarem linhas de cobrança de competência em outro cliente, perguntar o valor antes de
  criar.

## Passo 4 — Antes de aplicar: montar o plano
Montar uma lista completa de mudanças (aba, linha, data antiga → nova, valor antigo → novo)
cobrindo tanto os preenchimentos de janela quanto as limpezas de duplicata. Não é necessário
mostrar linha por linha ao Ricardo — ele já autorizou aplicar diretamente — mas é bom dar um
resumo rápido antes de começar (quantas abas, quantos lançamentos) e sinalizar de antemão quais
clientes ficaram de fora por ambiguidade (nome incerto, mais pagamentos que linhas, etc.).

## Passo 5 — Aplicar direto na planilha ao vivo
Ricardo quer as correções feitas diretamente na planilha (não um arquivo à parte, não só um
relatório).

### Método preferido (validado em 19/10/2026): conector Google Sheets
- **Ler tudo de uma vez**: `get_spreadsheet` com `includeGridData: true` e `fields`
  `["sheets.properties.title","sheets.properties.sheetId","sheets.properties.hidden",
  "sheets.data.rowData.values.formattedValue"]`. O resultado (~1,4 MB) é salvo em arquivo —
  processar em Python. Os nomes de aba vêm completos (sem o problema de 31 caracteres do .xlsx).
- **Gravar em lote**: `update_spreadsheet` com requests `updateCells` célula a célula
  (`start: {sheetId, rowIndex (0-based), columnIndex}`, `fields: "userEnteredValue"`).
  ~130 requests por chamada funciona bem. Data como número serial
  (dias desde 30/12/1899; ex. 09/10/2026 = 46304) — a coluna A já tem formato dd/MM/yyyy.
  Valor como número. Para limpar: `values: [{}]` com `fields: "userEnteredValue"`.
- Para inserir linhas: `insert_dimension` (ROWS, `inheritFromBefore: true`) e depois
  `update_formulas` com as fórmulas de saldo (ver 3d).
- Antes de gravar, reler a planilha e confirmar que não mudou desde a leitura do plano
  (comparar o JSON); se mudou, refazer o plano.

### Alternativa (se o conector não estiver disponível): Chrome
- Usar o **Chrome real do usuário** (ferramentas `mcp__claude-in-chrome__*`), não o browser
  pane interno (`Claude_Browser`) — o pane interno já se mostrou pouco confiável (screenshots
  travando, cliques errando, `type` não funcionando). Carregar as tools via ToolSearch se
  estiverem deferidas:
  `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__browser_batch`
- Abrir/achar a aba do navegador já na planilha (`tabs_context_mcp`), navegar para
  `https://docs.google.com/spreadsheets/d/16W5fnw-4eN3bUjNLmDyZQ4NsNGRpkEILlKt2UXnF3w0/edit` se
  necessário.
- Para editar uma célula: clicar na Caixa de Nome (Name Box) em aproximadamente `[45, 127]`,
  digitar a referência `'Nome Da Aba'!D19` (nome da aba entre aspas simples se tiver espaço ou
  caractere especial), apertar Return para navegar, apertar Delete para limpar o conteúdo
  atual, e se for para preencher (não só limpar), digitar o novo valor e apertar Return de
  novo. Data no formato `DD/MM/AAAA`; valor decimal com vírgula (ex.: `250,9`).
- Agrupar várias edições (∼80 ações ≈ 8 lançamentos) por chamada de `browser_batch` para ir
  mais rápido — mas o `browser_batch` para no primeiro erro, então depois de qualquer erro
  verificar o que já tinha sido aplicado antes de continuar (não assumir que nada aplicou).
- **Cuidado com nomes de aba truncados em 31 caracteres**: se a lista de nomes de aba tiver
  sido obtida a partir de uma exportação em .xlsx (via `Google_Drive__download_file_content`
  com `exportMimeType=...spreadsheetml.sheet`), nomes de aba com mais de 31 caracteres vêm
  cortados (limite do Excel), o que quebra a navegação pela Caixa de Nome. Antes de editar,
  sempre conferir a lista de nomes REAL diretamente do Google Sheets (`read_page` na barra de
  abas, filter="all", pegando o texto "Página do quadro" de cada aba) e montar um mapeamento de
  correção nome-truncado → nome-real para qualquer nome de aba suspeito (≥30 caracteres).

## Passo 6 — Verificar
Depois de aplicar, reler a planilha inteira (mesmo `get_spreadsheet` do Passo 5) e comparar com
o plano do Passo 4 célula por célula, **e** comparar com a leitura anterior para confirmar que
nenhuma outra célula das colunas A–D mudou — não confiar só no "success" da chamada. Isso também serve para detectar se o Ricardo editou algo manualmente ao
mesmo tempo (nesse caso, nunca sobrescrever o que ele fez — perguntar).

## Passo 7 — Relatório final
Depois de aplicar tudo, gerar a lista de clientes que ainda não pagaram a competência mais
recente coberta pelo extrato (a que vence no início do mês seguinte ao fim do período do
extrato). Buscar em todas as abas (exceto as que começam com `*`, que são modelos/templates)
as linhas REC daquela competência com valor em branco. Excluir clientes já sabidamente
inativos/encerrados se o Ricardo tiver indicado isso antes.

Excluir também: abas ocultas (clientes encerrados) e a aba LEGVITA (duplicada).

Entregar essa lista como um arquivo .xlsx (Cliente | Competência | Vencimento | Valor |
Saldo na aba | Observação, com total no fim) via SendUserFile. Na Observação, marcar os
clientes que costumam pagar **direto no banco/caixa, fora do Asaas** (APADA, JAQUELINE RIBAS,
LUIZA RIGO, MARCENARIA FITZ, LETÍCIA PARISE, ALINE SERAFIM, WURSTER, entre outros — ver a
lista do ezmensal) — eles aparecem "em aberto" aqui só porque esses pagamentos não estão no
extrato do Asaas.

## Fórmula de saldo (coluna E)
Padrão: `=E<linha anterior>-C<linha>+D<linha>` (cobrança diminui, recebimento aumenta; saldo
negativo = cliente devendo). Se uma aba aparecer com saldo positivo depois de uma cobrança,
conferir se a fórmula está invertida (`+C-D`) — foi o caso da SABRINA LA BELLA SILVA,
corrigido em 19/10/2026.

## Regras gerais
- Nunca aplicar uma mudança automática num cliente com ambiguidade real (nome incerto,
  pagamentos sobrando, valor não batendo com a mensalidade padrão). Separar e perguntar.
- Nunca fazer "substituir a planilha inteira" — always fazer edições célula a célula na
  planilha viva, para não conflitar com edições manuais simultâneas do Ricardo.
- Se o Ricardo disser que editou algo manualmente enquanto o trabalho está em andamento,
  confiar nele e não desfazer.
- Sempre que tiver dúvida genuína sobre a competência/alocação certa de um valor, perguntar —
  nunca adivinhar em cima de dinheiro real de cliente.
