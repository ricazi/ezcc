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
- Manter uma lista de bloqueio (nomes que já se provaram errados em sessões anteriores) e nunca
  reautorizar esse match sem confirmação explícita:
  - ELIAS RIBEIRO FERNANDES
  - SAMARA GOTTEMS DONATTI
  - ASSOC DOS PRODUTORES DE HORTIGRANJEIROS DE SANTA ROSA
- Nomes de aba que já causaram dúvida antes e foram resolvidos manualmente pelo Ricardo (não
  reabrir a dúvida, usar o nome de aba já correto):
  - "TARIFA ZERO TELECOM LTDA" (no extrato) = aba "TARIFA ZERO TELECOM LTDA" (renomeada em
    2026-09; antes era "TARIFA ZERO CONSULTORIA LTDA")
  - "ASSOCIACAO DE AMIGOS BALNEARIO PILATTI DE PORTO MAUA - RS" (no extrato) = aba
    "ASSOCIAÇÃO BALNEÁRIO PILATTI"

## Passo 3 — Lógica de reconciliação por aba ("janela" + duplicatas)

Para cada aba de cliente afetada pelo extrato (ou toda aba, se for uma varredura completa):

### 3a. Reconciliação por janela (window zip)
1. Identificar todas as linhas "REC HONORARIOS MM/AAAA" cuja **vencimento** (competência + 1
   mês) caia dentro do período coberto pelo extrato atual (ex.: se o extrato cobre
   julho–setembro/2026, a janela é vencimento em jul/ago/set 2026).
2. Pegar os pagamentos do extrato casados com confiança a essa aba, ordenados por data.
3. Ordenar as linhas REC da janela por ordem de competência (mais antiga primeiro).
4. Fazer o "zip" 1-para-1: o pagamento mais antigo vai na linha REC de competência mais antiga
   dentro da janela, e assim por diante — sobrescrevendo qualquer coisa que já estivesse lá.
5. Se sobrarem linhas REC da janela sem pagamento correspondente (menos pagamentos que linhas):
   deixar essas linhas em branco (cliente ainda não pagou aquela competência) — não inventar
   valor.
6. Se sobrarem pagamentos sem linha REC correspondente (mais pagamentos que linhas da janela):
   **não aplicar automaticamente** — separar esse cliente para revisão manual com o Ricardo,
   mostrando as datas/valores do extrato e o estado atual das linhas daquela aba. Foi o caso do
   VINICIUS FINGER LTDA (dois pagamentos no mesmo dia 13/07 com valores diferentes de 444,00,
   sem bater exatamente com a mensalidade padrão — só o Ricardo sabia a qual competência cada
   um se referia).

### 3b. Duplicatas em toda a aba (fora da janela também)
Independente da janela acima, varrer TODAS as linhas REC da aba e procurar pares com a MESMA
data e MESMO valor (duplicata exata). Quando encontrar:
- Manter a linha de menor número (a primeira).
- Limpar (data E valor) as demais linhas duplicadas, mesmo que estejam fora da janela do
  extrato atual — esse é o bug sistêmico original (um valor de pagamento colado por engano em
  uma linha de competência futura ainda não vencida).

### 3c. Quando os dois processos colidem
Sempre rodar 3a e 3b juntos por aba (não são excludentes). Se depois da limpeza de duplicatas
sobrar uma linha REC "solta" com data/valor que na verdade pertence a outra competência (ex.:
um pagamento de maio lançado por engano na linha de setembro, sem duplicata exata em outro
lugar), NÃO tentar realocar sozinho — é o mesmo tipo de caso do Vinicius Finger. Perguntar ao
Ricardo a que competência aquele valor se refere antes de mover.

## Passo 4 — Antes de aplicar: montar o plano
Montar uma lista completa de mudanças (aba, linha, data antiga → nova, valor antigo → novo)
cobrindo tanto os preenchimentos de janela quanto as limpezas de duplicata. Não é necessário
mostrar linha por linha ao Ricardo — ele já autorizou aplicar diretamente — mas é bom dar um
resumo rápido antes de começar (quantas abas, quantos lançamentos) e sinalizar de antemão quais
clientes ficaram de fora por ambiguidade (nome incerto, mais pagamentos que linhas, etc.).

## Passo 5 — Aplicar direto na planilha ao vivo
Ricardo quer as correções feitas diretamente na planilha (não um arquivo à parte, não só um
relatório). Método que funciona de forma confiável:

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
Depois de aplicar um lote de edições, baixar uma cópia fresca da planilha via
`Google_Drive__download_file_content` (exportMimeType xlsx) e comparar com o plano do Passo 4
(célula por célula) para confirmar que tudo aplicou certo — não confiar só na ausência de erro
do `browser_batch`. Isso também serve para detectar se o Ricardo editou algo manualmente ao
mesmo tempo (nesse caso, nunca sobrescrever o que ele fez — perguntar).

## Passo 7 — Relatório final
Depois de aplicar tudo, gerar a lista de clientes que ainda não pagaram a competência mais
recente coberta pelo extrato (a que vence no início do mês seguinte ao fim do período do
extrato). Buscar em todas as abas (exceto as que começam com `*`, que são modelos/templates)
as linhas REC daquela competência com valor em branco. Excluir clientes já sabidamente
inativos/encerrados se o Ricardo tiver indicado isso antes.

Entregar essa lista como um arquivo .xlsx simples (Cliente | Competência | Vencimento) via
SendUserFile.

## Regras gerais
- Nunca aplicar uma mudança automática num cliente com ambiguidade real (nome incerto,
  pagamentos sobrando, valor não batendo com a mensalidade padrão). Separar e perguntar.
- Nunca fazer "substituir a planilha inteira" — always fazer edições célula a célula na
  planilha viva, para não conflitar com edições manuais simultâneas do Ricardo.
- Se o Ricardo disser que editou algo manualmente enquanto o trabalho está em andamento,
  confiar nele e não desfazer.
- Sempre que tiver dúvida genuína sobre a competência/alocação certa de um valor, perguntar —
  nunca adivinhar em cima de dinheiro real de cliente.
