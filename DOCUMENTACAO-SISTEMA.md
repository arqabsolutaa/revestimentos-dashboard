# Documentação Técnica — Dashboard Essence Residence

> Este documento existe para ser colado no início de uma conversa nova com o Claude, junto com o `index.html`, o `Code.gs` e a autorização de acesso ao GitHub, para que a nova conversa entenda o sistema inteiro sem precisar reexplicar tudo do zero.
>
> Ele é atualizado sempre que uma mudança relevante é feita no sistema. Última atualização: **26/09/2026** (tarde).

## 1. Quem usa e para quê

- **Bárbara** (arquiteta/coordenadora de projetos na Absoluta Construtora e Incorporadora) é a usuária principal e quem toma todas as decisões de produto.
- **Fernanda** (diretora administrativa) recebe relatórios gerados pelo sistema.
- O sistema é o **Dashboard Essence Residence**: gerencia a personalização de acabamentos (revestimentos, vinílico, marmoraria, banheiras, metais, civil, ar-condicionado) de um empreendimento residencial de 30 unidades em São Bernardo do Campo/SP, além de controle financeiro (entradas, saídas, contas, previsão de recebimento), vagas/depósitos e instalação em obra.
- Bárbara se comunica de forma direta e informal, frequentemente por voz transcrita (mensagens longas, às vezes fragmentadas). Ela dá liberdade de decisão de implementação, mas exige que a lógica seja confirmada com ela antes de mudanças grandes/novas ("me explica antes de fazer").

## 2. Arquitetura

- **Frontend**: `index.html` — arquivo único (~11.700 linhas), hospedado no **GitHub Pages**. HTML + CSS + JS puro (sem framework), PDF gerado no cliente via **jsPDF**.
- **Backend**: `Code.gs` — Google Apps Script, publicado como Web App. Bárbara cola manualmente no editor do Apps Script e reimplanta (Implantar → Gerenciar implantações → Editar → Nova versão → Implantar) sempre que o backend muda.
- **Banco de dados**: Google Sheets. O `Code.gs` expõe `doGet(e)` como dispatcher de ações (`action` no parâmetro `p`), lendo/gravando nas abas da planilha.
- **Sincronização de leitura**: depois de qualquer escrita, `Code.gs` roda `atualizarGithub()`, que regenera um arquivo `data.json` no repositório GitHub — esse arquivo é usado como cache rápido de leitura pelo frontend (evita chamadas JSONP lentas o tempo todo).
- **Repositório**: `arqabsolutaa/revestimentos-dashboard` (GitHub). Site publicado em `https://arqabsolutaa.github.io/revestimentos-dashboard/`.
- **Planilha Google Sheets ID**: `1flPYLA-cceH5D94pKN9PieN3tKsj2YCqClRQDvxQVY0`.

### Fluxo de deploy — MUITO IMPORTANTE

Toda mudança se encaixa em uma (ou ambas) destas categorias:
1. **Frontend (`index.html`)** → o Claude sobe direto no GitHub (com autorização de push habilitada), o GitHub Pages atualiza sozinho em ~1 min.
2. **Backend (`Code.gs`)** → o Claude entrega o arquivo `.gs` completo (nunca parcial), e **Bárbara precisa colar manualmente no Apps Script e reimplantar** — isso não é automatizável, é passo manual dela.

Sempre que uma mudança envolve os dois lados, isso deve ser dito explicitamente a ela, e o `Code.gs` enviado é sempre o arquivo **cumulativo completo**, nunca um trecho.

## 3. Autenticação de acesso ao GitHub (contexto para sessões novas)

- O Claude precisa de acesso de **push** ao repositório para subir o `index.html` diretamente.
- Esse acesso é feito via **GitHub App do Claude**, vinculado à conta do GitHub da Bárbara (não é mais feito com token colado à mão — o ambiente de execução do Claude bloqueia isso por segurança).
- Se numa conversa nova o Claude não conseguir dar push, o caminho é: Configurações do Claude (claude.ai) → Connectors → GitHub → conferir se "Instalar o Claude GitHub App" está feito e autorizado para o repositório `revestimentos-dashboard`.
- Isso só precisa ser refeito se a autorização expirar ou for removida — não é algo rotineiro.

## 4. Áreas do dashboard (navegação principal)

Áreas protegidas por senha:
- **Personalizar** — senha `ABS4027@`
- **Vagas e Depósitos** — senha `ABS4002@`
- **Obra** — senha `arquitetura1@`
- **Financeiro** — senha `financeiro1@`

Dentro de **Personalizar**, os módulos de orçamento/personalização são (cada um com seu próprio fluxo de criação, status de pagamento Pendente/Parcial/Pago, e geração de PDF no estilo visual "Essence" — marrom/dourado #43331e/#8c8467, nunca azul):

| Módulo | O que faz |
|---|---|
| **Revestimentos** | Catálogo principal de pisos/revestimentos cerâmicos; personalização por apartamento/parede |
| **Vinílico** | Personalização de piso vinílico |
| **Marmoraria** | Orçamentos de mármore/granito (catálogo Sabbia com imagens, tamanho, m²/chapa, custo, preço) |
| **Banheiras** | Catálogo Sabbia (378 produtos: modelo/material/acabamento/tipo_sifão/lado); preço = custo × 1.5 |
| **Metais** | Orçamento item a item (lista de peças) |
| **Civil** | Orçamentos de serviços civis avulsos (serviço + valor + condição de pagamento) |
| **Ar-Condicionado** | Orçamento "máquina por máquina": cada ponto adicional tem seu próprio BTU (9.000/12.000/24.000) e valor, somado ao total |

## 5. Módulo de Estoque / Lotes / Instalações (fluxo reformulado em 26/09/2026)

Esse é o módulo mais reformulado recentemente — importante entender bem:

- **Aba "Lotes"** (dentro de Estoque): registro de lotes de fabricação por revestimento (data de fabricação + quantidade de caixas + m²/caixa). Um revestimento pode ter **vários lotes/datas simultâneos**.
  - Quando um lote é criado manualmente ali (`addLote`), isso **soma ao estoque geral** do revestimento (`somarEstoqueRevestimento`). Esse é o fluxo de "entrada de estoque" tradicional — continua existindo, sem mudanças.
- **Aba "Instalações"**: registra o que foi efetivamente instalado em obra, apartamento por apartamento.
  - **Fluxo antigo (descontinuado em 26/09/2026)**: o usuário pré-cadastrava o lote (data+quantidade) na aba Lotes, e ao criar uma instalação apenas *selecionava* qual lote pré-existente estava usando, o que descontava a quantidade daquele lote.
  - **Fluxo atual**: revestimentos grandes (ex: 20x270) vêm paletizados, não em caixa — a etiqueta com data de fabricação só é vista na hora da instalação, não antes. Por isso o fluxo foi **invertido**: agora é a tela "Nova Instalação" que **é a fonte** da informação de data/lote, não o contrário.
    - Campos do modal: **Apartamento** (seleção, não mais texto livre), **Revestimento** (seleção — mostra todos, não só os que já têm lote), **Data do lote/fabricação** (calendário — a data da etiqueta da caixa/pallet visualizada em obra), **Qtd de caixas desse lote**, **Qtd de peças desse lote**, Cliente, Data da instalação, Observações.
    - Ao salvar, a informação de data/lote **é formalizada automaticamente na aba Lotes** (soma a um lote já existente com mesmo revestimento+data, ou cria um novo) — **sem descontar nem somar ao estoque geral** do revestimento. É puramente um registro histórico de "esse lote com essa data foi usado nesse apartamento".
    - Essa decisão foi confirmada explicitamente pela Bárbara: a instalação só registra data/lote, não mexe em estoque; e sim, pode haver vários lotes/datas simultâneos por revestimento.
  - Função backend responsável: `addInstalacao(d)` chama `formalizarLoteViaInstalacao(d)` (não confundir com `addLote`, que é o fluxo antigo de entrada manual de estoque e continua existindo e funcionando como antes).

## 6. Bug histórico mais importante já corrigido (para não reintroduzir)

**Sintoma**: dado aparecia na tela e sumia sozinho poucos segundos depois, ou dados apareciam desatualizados/errados só às vezes.

**Causa raiz**: `lerDados()` sem forçar (`lerDados()`, sem `true`) lê o `data.json` via `raw.githubusercontent.com`, que tem cache de CDN com atraso imprevisível (às vezes 30–90+ segundos, às vezes muito mais). Várias telas liam esse cache uma única vez e nunca se autocorrigiam. O caso mais grave era o **bootstrap global da página**, que sobrescrevia dados já corretos com a versão desatualizada do cache, sem nenhuma tela de segurança.

**Padrão de correção aplicado (usar sempre que houver uma tela nova)**: renderizar rápido com o que tiver em cache/`data.json`, e **sempre** disparar em paralelo um `lerDados(true)` em segundo plano que force a leitura direta da planilha (bypassa o cache) e re-renderiza silenciosamente se os dados vieram diferentes. Isso já foi aplicado em praticamente todas as funções de carregamento de tela (`loadPersonalizar`, `loadPagamentos`, `loadEstoque`, `loadVisaoGeral`, `loadRevestimentos`, o bootstrap global, etc.).

## 7. Convenções e regras fixas (não quebrar)

- **"Sem mexer no que já funciona"** — regra número um da Bárbara. Mudanças novas nunca devem regredir funcionalidade existente.
- `Code.gs` sempre entregue como arquivo **completo e cumulativo**, nunca parcial.
- Sempre buscar a versão mais recente real do arquivo antes de editar (nunca confiar em cache).
- Mudanças de frontend vs. backend devem ser **claramente distinguidas** para a Bárbara (ela precisa saber se precisa reimplantar o Apps Script ou não).
- Módulos novos de orçamento devem ser completamente independentes dos módulos de revestimentos existentes.
- Modais fecham **só pelo botão X**, nunca clicando fora.
- Paredes adicionais/personalizadas sempre aparecem no final da aba Paredes.
- A marca Essence **nunca usa azul** — paleta oliva + dourado/marrom.
- UI otimista é o padrão preferido: itens aparecem na tela imediatamente, sem esperar confirmação do GitHub/planilha (com reversão em caso de erro).
- Detecção de tablet usa `isMobileDevice()`/`navigator.maxTouchPoints`, nunca `window.innerWidth` (iPads modernos se identificam como "Macintosh").
- Nomes de arquivo de PDF passam por `nomeArquivoLimpo()` (remove acentos/espaços).

## 8. Helpers e padrões de código reaproveitados

- `_apartamentosConhecidos()` — retorna lista `{ap, cliente}` de todos os apartamentos já cadastrados em `Personalizar`, usada para popular selects de apartamento em vários módulos (Civil, Ar-Condicionado, Instalações, etc.).
- `_labelCondicaoPagamento(c)` — formata código de condição de pagamento em texto legível.
- `calcularTotalComPagamento(valorAvista, condicao)` — calcula total com juros conforme a condição escolhida.
- Helpers de PDF compartilhados: `_pdfCapa`, `_pdfInfoCliente`, `_pdfTabelaHeader`, `_pdfLinha`, `_pdfTotal` — usados por Revestimentos, Vinílico, Marmoraria, Banheiras, Metais, Civil e Ar-Condicionado, cada um com sua paleta em `ESSENCE_PALETAS`.
- `api(action, params)`, `apiPost(action, dados)`, `apiAction(action, id)` — wrappers de chamada ao backend (JSONP).
- `lerDados(forcarFresco)` — leitura de dados; **sempre preferir `lerDados(true)`** para qualquer coisa que precise estar correta na hora (ver seção 6).

## 9. Itens em aberto / próximos passos conhecidos

- Confirmar que o `Code.gs` mais recente (com o módulo de Instalações reformulado — incluindo o campo de quantidade de peças — e o schema `itens` do Ar-Condicionado) já foi colado e reimplantado no Apps Script pela Bárbara.
- Botão "Baixar documentação técnica" foi adicionado à tela de **Configurações** — baixa este mesmo arquivo direto do GitHub. Deve ser mantido atualizado a cada mudança relevante no sistema.
