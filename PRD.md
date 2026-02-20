# PRD - Case-me

## 1. VISÃO DO PRODUTO

O **Case-me** é uma plataforma SaaS tipo marketplace para o setor de casamentos, conectando noivos a fornecedores qualificados (fotógrafos, buffets, decoradores, etc.) com um ecossistema completo de organização. A plataforma oferece dashboards personalizados para cada perfil (noivos, fornecedores, assessores), um sistema de afiliados para assessores de casamento, página pública personalizada do casamento, lista de presentes, RSVP digital e ranking transparente de fornecedores.

## 2. OBJETIVOS DE NEGÓCIO

- **Marketplace funcional:** Criar o principal marketplace de casamentos do Brasil, conectando noivos e fornecedores de forma premium e escalável.
- **Monetização via assinaturas:** Gerar receita recorrente através de planos de assinatura para fornecedores (Free, Pro, Premium) com benefícios progressivos de visibilidade.
- **Rede de afiliados:** Escalar a aquisição de fornecedores e receita com um programa de assessores-afiliados com comissões automáticas e gamificação por selos.
- **Plataforma emocional para noivos:** Reter noivos oferecendo ferramentas completas e gratuitas de organização (lista de presentes, RSVP, página pública do casamento).
- **Preparação para escala e investimento:** Arquitetura multi-tenant robusta, pronta para internacionalização, expansão nacional e captação de investimento.

## 3. PERSONAS

### 👰 Noivos (Casais)
- Casais em fase de planejamento de casamento (6 a 24 meses antes da data).
- Jovens adultos (25-40 anos), digitais e exigentes com design e experiência.
- Buscam praticidade, organização e inspiração em um só lugar.
- **Necessidade principal:** Centralizar toda a organização do casamento em uma plataforma confiável e bonita, com busca fácil de fornecedores, lista de presentes e RSVP digital.

### 📸 Fornecedores (Empresas do setor)
- Profissionais e empresas de casamentos: fotógrafos, buffets, floristas, DJs, decoradores, cerimonialistas, etc.
- Micro a pequenas empresas buscando visibilidade e leads qualificados.
- Valorizam métricas de performance, portfólio visual e reputação.
- **Necessidade principal:** Ganhar visibilidade, receber leads qualificados e construir uma reputação transparente baseada em avaliações reais.

### 💼 Assessores (Afiliados)
- Assessores e cerimonialistas que possuem rede ativa de fornecedores.
- Buscam renda extra indicando fornecedores para a plataforma.
- Motivados por gamificação (selos) e comissões recorrentes.
- **Necessidade principal:** Monetizar sua rede de contatos indicando fornecedores para a plataforma, com tracking transparente de comissões e reconhecimento.

### 🛡️ Administrador da Plataforma
- Equipe interna da Case-me.
- Responsável por moderação, aprovação de fornecedores, gestão de planos e análise de métricas.
- **Necessidade principal:** Visão completa da plataforma, gestão de usuários/organizações, moderação de avaliações e controle financeiro.

## 4. FUNCIONALIDADES CORE

### 4.1 Landing Page

**Descrição:**
Página de apresentação pública com design premium, sofisticado e emocional, utilizando paleta clara com tons champagne, branco, dourado suave e rosé. Deve transmitir confiança, sofisticação e profissionalismo.

**Requisitos:**
- Hero section com headline: *"O ecossistema completo para organizar seu casamento."*
- CTA duplo: "Sou Noivo(a)" e "Sou Fornecedor"
- Seção "Como funciona para Noivos" (passo a passo visual)
- Seção "Como funciona para Fornecedores" (passo a passo visual)
- Seção "Programa para Assessores (Afiliados)" com benefícios
- Seção de depoimentos (simulados no MVP)
- Seção de planos e monetização (tabela comparativa)
- Footer com links institucionais
- Totalmente responsiva (mobile-first)
- SEO otimizado

**Fluxo do usuário:**
1. Usuário acessa a landing page
2. Explora as seções e entende a proposta do Case-me
3. Clica em "Sou Noivo(a)" ou "Sou Fornecedor" → Direcionado ao cadastro/login

### 4.2 Autenticação e Multi-Tenancy (Clerk + Organizations)

**Descrição:**
Sistema de autenticação robusto com suporte multi-tenant via Clerk Organizations. Cada fornecedor opera como uma organização independente com dados isolados.

**Requisitos:**
- Login/Cadastro via OAuth (Google)
- Multi-tenant via Clerk Organizations
- Roles: `noivo`, `fornecedor`, `assessor`, `admin`
- Redirecionamento automático baseado no perfil:
  - Noivo → Dashboard do Casamento (`/dashboard/casamento`)
  - Fornecedor → Dashboard Empresarial (`/dashboard/fornecedor`)
  - Assessor → Dashboard Afiliado (`/dashboard/afiliado`)
  - Admin → Painel Administrativo (`/admin`)
- Webhook Clerk → Supabase para sincronização de users e organizations
- Onboarding diferenciado por perfil

**Fluxo do usuário:**
1. Clica em "Sou Noivo(a)" ou "Sou Fornecedor" na landing page
2. Tela de Sign Up/Sign In do Clerk (OAuth Google)
3. Após autenticação, verifica se possui perfil:
   - Se não → Onboarding do perfil
   - Se sim → Redireciona ao dashboard adequado

### 4.3 Dashboard do Fornecedor

**Descrição:**
Painel moderno e orientado à performance para fornecedores gerenciarem seu negócio na plataforma.

**Requisitos:**
- **Estatísticas:** Visualizações do perfil, leads recebidos, taxa de conversão
- **Plano atual:** Exibição do plano (Free, Pro, Premium) com botão de upgrade
- **Portfólio:** Upload e gestão de imagens via Supabase Storage (galeria visual)
- **Avaliações:** Lista de avaliações recebidas com nota média
- **Ranking:** Posição no ranking público (baseado exclusivamente em nota, não manipulável por plano)
- **Leads:** Lista de leads recebidos com dados de contato dos noivos interessados
- **Perfil público:** Edição do perfil público (descrição, categorias, localização, preços)

**Fluxo do usuário:**
1. Fornecedor acessa seu dashboard
2. Visualiza métricas de performance no topo
3. Gerencia portfólio e perfil público
4. Responde a leads recebidos
5. Visualiza e responde avaliações

### 4.4 Dashboard dos Noivos

**Descrição:**
Interface emocional e organizada, centrada na experiência dos noivos com ferramentas completas de planejamento.

**Requisitos:**
- **Contador regressivo:** Contagem regressiva para a data do casamento
- **Lista de desejos (Wishlist):** Favoritar fornecedores para considerar depois
- **Lista de presentes:** Criar lista de presentes com itens personalizados
- **Lista convertível em dinheiro:** Opção de converter presentes em contribuições financeiras (PIX/link de pagamento)
- **Página pública personalizada:** Landing page do casamento com informações, fotos e RSVP
- **RSVP digital:** Confirmação de presença online integrada à página pública
- **Gestão de convidados:** Cadastro e controle de convidados com status de confirmação

**Fluxo do usuário:**
1. Noivo acessa o dashboard após login
2. Vê o contador regressivo e cards de ações rápidas
3. Busca e favorita fornecedores no marketplace
4. Monta e gerencia lista de presentes
5. Personaliza página pública do casamento
6. Envia link de RSVP para convidados
7. Acompanha confirmações de presença

### 4.5 Marketplace (Busca de Fornecedores)

**Descrição:**
Página pública de busca e descoberta de fornecedores com filtros avançados.

**Requisitos:**
- Busca com filtros: categoria, localização, faixa de preço, nota mínima
- Grid de cards de fornecedores com foto, nome, categoria, nota e selo de plano
- Página de perfil público do fornecedor com portfólio, avaliações e botão de contato/lead
- Ranking transparente baseado em avaliações (posição não influenciada pelo plano pago)
- Destaque visual (badge) para fornecedores Pro/Premium (sem alterar posição no ranking)

**Fluxo do usuário:**
1. Noivo acessa a página do marketplace
2. Aplica filtros de busca (categoria, cidade, etc.)
3. Visualiza grid de resultados ordenados por ranking
4. Clica em um fornecedor → Página de perfil público
5. Envia mensagem/lead ou favorita

### 4.6 Sistema de Avaliações

**Descrição:**
Sistema de reviews e ratings para fornecedores, com proteções antifraude.

**Requisitos:**
- Somente noivos que tiveram contato real (lead aceito) podem avaliar
- Nota de 1 a 5 estrelas + comentário
- Moderação por admin (denúncia e revisão)
- Proteção antifraude: limite de avaliações por noivo/fornecedor, validação de lead
- Ranking público calculado pela média ponderada de avaliações

**Fluxo do usuário:**
1. Noivo contrata/interage com fornecedor
2. Após evento, recebe convite para avaliar
3. Envia avaliação (nota + comentário)
4. Avaliação aparece no perfil público do fornecedor
5. Ranking se recalcula automaticamente

### 4.7 Sistema de Afiliados (Assessores)

**Descrição:**
Programa de afiliados para assessores de casamento com link exclusivo, tracking de indicações, comissões e gamificação.

**Requisitos:**
- **Link exclusivo de indicação:** URL com código de tracking por assessor
- **Dashboard de comissões:** Histórico de comissões geradas, pendentes e pagas
- **Total de fornecedores ativos indicados:** Contador de fornecedores que se cadastraram via link
- **Selos por performance:**
  - 🥉 Bronze (1-5 indicações ativas)
  - 🥈 Prata (6-15 indicações ativas)
  - 🥇 Ouro (16-30 indicações ativas)
  - 💎 Elite (31+ indicações ativas)
- **Comissão:** Percentual sobre a primeira assinatura paga do fornecedor indicado

**Fluxo do usuário:**
1. Assessor se cadastra e recebe link exclusivo
2. Compartilha link com fornecedores da sua rede
3. Fornecedor se cadastra via link → Tracking automático
4. Fornecedor assina plano pago → Comissão gerada
5. Assessor acompanha comissões e selos no dashboard

### 4.8 Planos e Monetização

**Descrição:**
Sistema de assinaturas para fornecedores com planos progressivos.

**Requisitos:**
- **Free:** Perfil básico, até 5 fotos no portfólio, listagem no marketplace
- **Pro:** Perfil destacado, até 20 fotos, selo Pro, prioridade em leads, analytics avançado
- **Premium:** Tudo do Pro + fotos ilimitadas, selo Premium, banner destacado, suporte prioritário
- Integração com gateway de pagamento (Stripe ou similar, definir na V2)
- Gestão de assinaturas (upgrade, downgrade, cancelamento)
- Período de trial para planos pagos (opcional V2)

**Fluxo do usuário:**
1. Fornecedor acessa página de planos
2. Compara benefícios de cada plano
3. Seleciona plano e realiza pagamento
4. Benefícios aplicados imediatamente ao perfil/dashboard

### 4.9 Emails Transacionais (Resend)

**Descrição:**
Comunicação automatizada por email para todos os eventos críticos da plataforma.

**Requisitos:**
- Templates React Email com design alinhado à marca Case-me
- Emails obrigatórios:
  - **Boas-vindas Noivo:** Após cadastro de casal
  - **Boas-vindas Fornecedor:** Após cadastro de organização fornecedora
  - **Novo lead recebido:** Notifica fornecedor quando noivo demonstra interesse
  - **Nova avaliação recebida:** Notifica fornecedor sobre nova review
  - **Comissão gerada:** Notifica assessor quando comissão é contabilizada
  - **Confirmação de pagamento:** Após assinatura de plano

### 4.10 Painel Administrativo

**Descrição:**
Dashboard para a equipe interna da Case-me gerenciar a plataforma.

**Requisitos:**
- Visão geral: Total de usuários, fornecedores, noivos, assessores
- Gestão de fornecedores: Aprovar, suspender, gerenciar perfis
- Gestão de avaliações: Moderação e resolução de denúncias
- Gestão de planos: Configuração de preços e benefícios
- Gestão de comissões: Aprovação e controle de pagamentos a assessores
- Métricas e relatórios: Receita, crescimento, engajamento

## 5. REQUISITOS NÃO-FUNCIONAIS

- **Performance:** First Contentful Paint < 1.5s, Time to Interactive < 3s, Lighthouse Score > 90
- **Segurança:** RLS em todas as tabelas, auth() validado em todas as Server Actions, Zod validation em todos os formulários, CORS configurado, rate limiting via Vercel
- **Escalabilidade:** Arquitetura multi-tenant com RLS escalável até 10k+ organizações, preparada para expansão nacional e internacionalização
- **Responsividade:** Suporte completo a desktop, tablet e mobile (mobile-first)
- **Disponibilidade:** Deploy via Vercel com Edge Functions para alta disponibilidade
- **Acessibilidade:** WCAG 2.1 nível AA

## 6. FORA DO ESCOPO V1

❌ Integração com gateway de pagamento real (Stripe) — usar mock/manual no MVP  
❌ Chat em tempo real entre noivos e fornecedores (Supabase Realtime — V2)  
❌ App mobile nativo (React Native — V3)  
❌ Internacionalização (i18n — V2)  
❌ Sistema de agenda/calendário para fornecedores  
❌ Integração com redes sociais (Instagram API para importar portfólio)  
❌ Sistema de convites com QR Code  
❌ Notificações push  
❌ Trial period para planos  
❌ Sistema de cupons e promoções  

## 7. ONBOARDING

### Fluxo Noivo(a):
1. Clica em "Sou Noivo(a)" na landing page
2. Sign Up via Clerk (Google OAuth)
3. Tela de onboarding: Nome do casal, data do casamento, cidade
4. Dashboard do Casamento com checklist de primeiros passos

**Checklist de Primeiros Passos (Noivos):**
- [ ] Definir data do casamento
- [ ] Buscar e favoritar primeiros fornecedores
- [ ] Criar lista de presentes
- [ ] Personalizar página pública do casamento
- [ ] Convidar primeiros convidados

### Fluxo Fornecedor:
1. Clica em "Sou Fornecedor" na landing page
2. Sign Up via Clerk (Google OAuth) → Cria Organization
3. Tela de onboarding: Nome da empresa, categoria, cidade, descrição breve
4. Dashboard Empresarial com checklist de primeiros passos

**Checklist de Primeiros Passos (Fornecedores):**
- [ ] Completar perfil (descrição, fotos, preços)
- [ ] Subir portfólio (mínimo 3 fotos)
- [ ] Definir categorias de atuação
- [ ] Escolher plano (Free, Pro, Premium)
- [ ] Responder primeiro lead

### Fluxo Assessor:
1. Acessa link do programa de afiliados
2. Sign Up via Clerk (Google OAuth)
3. Tela de onboarding: Nome, cidades de atuação, rede de contatos estimada
4. Dashboard Afiliado com link de indicação gerado

**Checklist de Primeiros Passos (Assessores):**
- [ ] Compartilhar link de indicação com 5 fornecedores
- [ ] Acompanhar primeiro cadastro via link
- [ ] Verificar primeira comissão gerada

## 8. MÉTRICAS DE SUCESSO

- **Fornecedores cadastrados (3 meses):** 200+
- **Noivos ativos (3 meses):** 500+
- **Assessores ativos (3 meses):** 30+
- **Taxa de conversão Free → Pro/Premium:** > 15%
- **Leads gerados por fornecedor/mês:** > 10
- **NPS de noivos:** > 70
- **NPS de fornecedores:** > 60
- **Taxa de retenção mensal de fornecedores pagos:** > 85%
- **Receita mensal recorrente (MRR) em 6 meses:** R$ 20.000+
- **Avaliações por fornecedor (média):** > 3 avaliações em 90 dias
