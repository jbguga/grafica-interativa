# Gráfica Interativa — Sistema de Gestão

Este arquivo é o ponto de partida para qualquer sessão do Claude Code neste projeto. Leia isto primeiro.

## Sobre o negócio

- **Gráfica Interativa**: gráfica rápida com 15 anos de mercado, em Araripina/PE. Atende a região e envia pelos Correios para o Brasil todo.
- Produtos/serviços: impressão rápida (banners, adesivos, cartões) e personalização de objetos (carimbos, copos, brindes).
- Contato: WhatsApp +55 87 99116-6737 · contato@interativa.art.br · Av. Florentino Alves Batista, 143 - Centro, Araripina/PE · Instagram @interativa.pe · Facebook graficainterativa.
- O dono (Gustavo) usava um sistema de gestão de terceiros, genérico e engessado, que não se encaixava no fluxo real da gráfica. Este projeto nasceu pra resolver isso, sob medida.

## Estrutura do projeto

Repositório Git em `github.com/jbguga/grafica-interativa` (público), publicado via GitHub Pages a partir da pasta `docs/`:

```
CLAUDE.md              → notas internas (não fica na pasta publicada, de propósito)
LICENSE                → licença proprietária
docs/                   → tudo que é publicado pelo GitHub Pages
├── CNAME                → domínio customizado (graficainterativa.com.br, ver notas de arquitetura)
├── index.html            → site institucional (público)
├── loja/index.html       → loja virtual (placeholder, ainda não construída de verdade)
└── adm/index.html        → painel de gestão completo (o "coração" do sistema)
```

Os três HTML se linkam entre si por caminhos relativos (`loja/index.html`, `adm/index.html`, `../index.html`) — continuam funcionando porque os três se moveram juntos pra dentro de `docs/`.

**`adm/index.html` é um app single-file**: todo HTML, CSS e JS estão num único arquivo (~2.900 linhas). Não há build step, bundler, nem dependências de npm. É intencional — o projeto inteiro foi construído assim, arquivo por arquivo, para rodar em qualquer navegador sem servidor de aplicação.

## Banco de dados (Supabase)

- **Project URL**: `https://hbbxxzlzkyofecjgfepc.supabase.co`
- **Chave pública (publishable/anon)**: `sb_publishable_M4SBqC9VpiiacTxgaXbUpQ_tMSsJmK8` (já embutida no código — é seguro estar no client-side, a proteção real vem das políticas RLS)
- **Tabela única**: `dados_sistema` (id fixo = 1, coluna `payload` do tipo `jsonb`, guarda o sistema inteiro como um único objeto JSON)
- **Autenticação**: Supabase Auth nativo (e-mail + senha). O usuário Master hoje é `gustavo` (criado direto no painel do Supabase, não pelo app). Não existe cadastro de novo usuário pelo app ainda — ver pendências abaixo.
- Todo acesso de leitura/escrita exige um usuário autenticado (`role = authenticated`) — ver políticas RLS criadas na tabela.

### Estrutura do `payload` (JSON)

```
{
  pedidos: [...],           // pedidos/orçamentos — ver estrutura abaixo
  produtos: [...],          // catálogo de estoque
  categoriasProdutos: [...],// categorias de produto (nome)
  contas: [...],            // contas bancárias/caixa cadastradas
  lancamentos: [...],       // movimentações financeiras (entrada/saída)
  creditos: [...],          // saldo de crédito por cliente
  clientesCadastro: [...],  // cadastro formal de clientes (PF/PJ)
  historicoEtapas: [...],   // log de toda mudança de etapa no kanban
  financeiro: {...}         // dados mensais do PGV (custo fixo, variável etc., por "YYYY-MM")
}
```

**`pedido`**:
```
{
  id, os, criadoEm, cliente, telefone, prazo, desconto,
  valorTotal,      // calculado: soma dos itens - desconto (NÃO editar direto)
  valorPago,       // calculado: soma de pagamentos[]
  arquivado, dataArquivado,   // orçamento arquivado (abandonado/não decidido)
  pagamentos: [{id, data, valor, contaId, lancamentoId, credito}],
  itens: [{id, descricao, quantidade, valorUnitario, status, prazo, produtoId}]
}
```
`itens[].status` é uma das 6 etapas do kanban: `orcamento`, `aguardando_pagamento`, `arte`, `producao`, `aguardando_retirada`, `finalizado` (ver array `COLS` no início do `<script>`).

## O que já está pronto e funcionando

- **Autenticação real** via Supabase Auth (login por e-mail/senha, recuperação de senha por e-mail).
- **Kanban de produção** com 6 etapas, drag-and-drop, múltiplos itens por pedido, prazo por item.
- **Pedidos**: valor calculado automaticamente pela soma dos itens menos desconto; pagamentos parciais registrados individualmente (data + valor + conta); saldo restante em tempo real; uso de crédito do cliente; conversão de excedente pago em crédito.
- **Orçamentos arquivados**: pedidos abandonados podem ser arquivados (não excluídos) e reativados depois; usado para calcular taxa de conversão real via `historicoEtapas`.
- **Financeiro**: contas cadastradas com saldo calculado, transferência entre contas, lançamentos (entrada/saída, pago/pendente), régua de cobrança automática, PGV (painel de gestão à vista) mensal.
- **Estoque**: produtos com categoria (caixa suspensa), baixa/estorno automático de estoque ao criar/editar/excluir item de pedido vinculado a produto.
- **Clientes**: cadastro PF/PJ com máscaras (CPF, CNPJ, telefone com +55), busca de cliente ao criar pedido (autopreenche), histórico de pedidos por cliente, clientes "esfriando" (sem comprar há 60+ dias) com botão de WhatsApp.
- **Relatórios**: visão geral por período, vendas últimos 12 meses, comparação ano a ano, conversão de orçamentos, produtos mais vendidos, clientes novos/recorrentes, financeiro entradas x saídas, estoque abaixo do mínimo — tudo com botão "Gerar relatório em PDF" (usa `window.print()`, sem lib externa).
- **Layout**: menu lateral estilo dashboard (inspirado num app de referência que o usuário mostrou), recolhe para uma barra fina de ícones e expande ao passar o mouse (desktop) ou por hambúrguer (mobile). Paleta: preto, cinza escuro, amarelo queimado (cor da logo), fundo pastel.
- Todos os modais/popups têm botão "✕" explícito para fechar — clicar fora **não** fecha (decisão deliberada, pra evitar perda de dados de formulário por clique acidental).

## Pendências conhecidas (o usuário já pediu para anotar)

1. **Cadastro de usuários com permissão por módulo** (aba "Equipe", hoje só um placeholder): o Master precisa poder criar login de atendente/designer/etc. com checkboxes habilitando/desabilitando módulos — ex.: atendente vê contas a receber (pra cobrança) mas não vê saldo de caixa nem projeções; designer só vê o kanban.
2. **Conferência de lançamentos pela atendente**: quando o financeiro estiver mais completo, lançamentos feitos por uma atendente devem ter uma checkbox "lançar no financeiro" que só o Master marca. Antes disso, o lançamento aparece só no "movimento do dia" dela, sem contar nas contas gerais — evita que conta errada (ex. Pix da maquineta lançado numa conta poupança por engano) bagunce o saldo real até o Master conferir e autorizar no fim do expediente.
3. **Loja virtual de verdade**: hoje é só uma página de espera. O plano é: produto vendido na loja gera pedido automaticamente no mesmo sistema (kanban, financeiro, baixa de estoque), como se fosse lançado por uma atendente.
4. **Site institucional**: conteúdo já escrito (história, produtos, qualidade, contato), falta a seção de depoimentos reais (espaço já reservado no layout) e possivelmente uma galeria de trabalhos.

## Notas de arquitetura / decisões importantes

- **Sem framework, sem bundler.** Se for reestruturar em múltiplos arquivos/componentes, isso é uma decisão nova a discutir com o usuário — ele é iniciante em programação (parou "há muito tempo" em páginas simples), então prefira manter simplicidade sobre sofisticação técnica, e explique mudanças de arquitetura em termos práticos.
- **`salvarTudo()`** salva o payload inteiro de uma vez (PATCH na linha única da tabela) — não é por-registro. Funciona bem no volume atual (uma gráfica pequena/média); se crescer muito, pode precisar normalizar em tabelas separadas no futuro.
- Datas armazenadas como string `YYYY-MM-DD` (não `Date` objects) em todo o código, por simplicidade de serialização/comparação.
- O usuário tem 2 domínios: `interativa.art.br` (em uso, aponta hoje para o sistema antigo de terceiros — só migra quando este sistema for validado e for pra valer) e `graficainterativa.com.br` (domínio reserva, mais curto, usado como CNAME do GitHub Pages para testar este sistema publicado enquanto isso). Quando aprovado, ele pretende trocar o DNS do `interativa.art.br` pra apontar pra cá também — ainda não decidiu se vai manter os dois domínios ativos ou cancelar o antigo.
- Todo o desenvolvimento até aqui foi feito por chat com o Claude (claude.ai), editando o arquivo com `str_replace`/reescritas pontuais. Esta é a primeira vez que o projeto entra num ambiente com sistema de arquivos real e Git — vale considerar inicializar um repositório Git logo no início da sessão, se ainda não existir.
