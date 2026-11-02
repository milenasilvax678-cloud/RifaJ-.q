# RifaJá

Next.js 15 + TypeScript + Tailwind 4 + Motion + Supabase (projeto `hrwpsbltcdlbabzzdvxg`, São Paulo).

## Rodar
`npm install`, copie `.env.example` para `.env.local`, preencha e `npm run dev`.

## Pagamento (PIX com Mercado Pago): FALTA CONFIGURAR
O botão "Continuar para pagamento" só libera quando estas variáveis existem NO SERVIDOR (Vercel > Settings > Environment Variables):
- `NEXT_PUBLIC_SITE_URL` = endereço público do site (https://...)
- `SUPABASE_SERVICE_ROLE_KEY` = chave secreta do Supabase (Settings > API Keys). Nunca no navegador.
- `MERCADOPAGO_ACCESS_TOKEN` = credencial de produção (ou de teste) da sua conta Mercado Pago
- `MERCADOPAGO_WEBHOOK_SECRET` = "assinatura secreta" do webhook
No Mercado Pago (Suas integrações > Webhooks), cadastre a URL `https://SEU-DOMINIO/api/webhooks/mercadopago` e marque o evento **Pagamentos**.
Sem isso o botão fica desativado e NENHUM número é reservado. O adaptador (`lib/payments/mercadopago.ts`) não foi testado
contra a conta real: teste com credenciais de teste antes de vender. Para outro provedor, troque só esse arquivo e a rota do webhook.

## Como funciona (marcação automática)
1. Cliente escolhe números > `reserve_numbers` (banco) confere tudo, calcula o valor e reserva por tempo limitado.
2. O servidor cria o PIX no provedor (chave de idempotência = id do pedido) e guarda o QR Code.
3. O provedor chama o webhook. O servidor valida a assinatura, consulta o pagamento direto no provedor e chama `confirm_payment`.
4. Só então os números viram "pago" e o pedido "confirmado". Voltar ao site ou abrir a página de sucesso não confirma nada.
Proteções no banco: número único por rifa (nunca vendido duas vezes), evento repetido ignorado, valor divergente ou reserva vencida
vão para "conferir" (reembolso manual no provedor), reservas vencidas liberadas a cada minuto (pg_cron).
Ao vivo: a tela de números e o painel (tabela de números, Compras, Pagamentos) atualizam na hora via Supabase Realtime
(tabela pública `number_status`, só número e estado, sem dados de pessoas), com consulta a cada 10 s como reforço.

## Painel (/admin, só quem está em administradores)
Produtos (cards, editar, publicar, pausar), Rifas (tabela), Tabela de números, Compras, Pagamentos (consulta + histórico),
Cupons, Resultados, Financeiro, Cargos, Suporte, Auditoria, Configurações.
Confirmação manual de pagamento: só com a permissão `payments.manual_confirm` (dono), justificativa obrigatória e registro na auditoria.
Reembolso: o painel registra, mas o dinheiro precisa ser devolvido no painel do provedor.

## Suporte e redes sociais
- Central de suporte: `/suporte` (FAQ, abrir chamado, meus chamados, conversa). Chamados e mensagens ficam no banco; cada usuário só enxerga os próprios (RLS).
  Equipe: Admin > Central de Suporte (busca, filtros, resposta, notas internas, status). Permissões: `support.read` (ver) e `support.write` (responder).
- Redes sociais: Admin > Configurações. Logos pela biblioteca Simple Icons. Rede sem link ou desativada não aparece no site.

## Painel (visual escuro, como no modelo enviado)
Barra lateral, visão geral com indicadores reais, gráfico de 7 dias, últimos pedidos e atividades (cada bloco aparece conforme a permissão do cargo).
Rifas: tabela com foto, prêmio, preço, vendidos e botões Editar, Editar números, Trocar foto, Editar prêmio, Ver detalhes e Excluir (só o dono; o banco recusa se houver pedidos).
Editar números: Reservar (bloqueia), Liberar (só os bloqueados pela equipe) e Marcar como pago (venda manual: permissão `payments.manual_confirm`, justificativa e auditoria).

## Chave de acesso do painel
Além do login de administrador, o painel pede uma **chave secreta** (variável `ADMIN_ACCESS_KEY` na Vercel, mínimo 12 caracteres).
Sem a variável o painel fica fechado. A chave vale por 8 horas, é conferida no servidor (nunca fica no código) e 5 erros bloqueiam por 15 minutos.
Botão "Bloquear painel" no topo encerra o acesso na hora. Escolha uma chave longa e diferente da sua senha.

## Para o cliente
Minhas compras (`/minhas-compras`): pedidos, status e os números de cada compra. Só aparece no menu para quem está logado, e o banco só entrega os dados da própria conta.
No painel, Compradores mostra cada pessoa com os números que comprou.

## Aviso legal
Rifas no Brasil exigem autorização prévia. O sistema só deixa a rifa ativa depois da validação jurídica marcada pelo dono.

## Pendente
Credenciais e webhook do provedor, testes ponta a ponta com PIX de teste, textos jurídicos, deploy com HTTPS, MFA.

## Publicar o site (resumo)
1. Zip > repositório privado no GitHub (sem `.env.local`) > Vercel: Add New > Project.
2. Variáveis na Vercel: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` e `NEXT_PUBLIC_SITE_URL`.
3. Supabase > Authentication > URL Configuration: Site URL e Redirect URLs com o endereço do site.
4. Pagamento: sem Mercado Pago configurado, o site usa o **PIX direto na chave do dono** (Admin > Configurações > Recebimento por PIX).
   O cliente vê QR Code e copia e cola com o valor exato; **você confere e confirma** em Admin > Pagamentos > Confirmação manual (permissão do dono, justificativa e auditoria).
   Para confirmação automática, configure o Mercado Pago (variáveis `MERCADOPAGO_*` e `SUPABASE_SERVICE_ROLE_KEY`).
5. O link "Painel administrativo" aparece no menu só para contas de administrador.
