# PROMPT BASE --- CASE-ME

Atue como um Especialista Sênior em Engenharia de Software, Arquitetura
SaaS Multi-Tenant, UX/UI para Marketplaces Digitais e Especialista em
Modelos de Monetização Escaláveis.

Crie a base de uma Plataforma SaaS chamada **Case-me**, um marketplace
para o setor de casamentos, utilizando:

-   Next.js 14+ (App Router, TypeScript)
-   Tailwind CSS
-   shadcn/ui
-   Clerk (com Organizations - obrigatório multi-tenant)
-   Supabase (PostgreSQL + Storage + Realtime)
-   Drizzle ORM
-   Resend (emails transacionais)
-   Deploy via Vercel

O sistema deve ser multi-tenant e preparado para escalar nacionalmente.

------------------------------------------------------------------------

## 1. LANDING PAGE "CASE-ME"

-   Design elegante, sofisticado e premium (estética casamento moderno).
-   Paleta clara com tons champagne, branco, dourado suave ou rosé.
-   Seção Hero com headline forte:
    -   "O ecossistema completo para organizar seu casamento."
-   CTA duplo:
    -   "Sou Noivo(a)"
    -   "Sou Fornecedor"

Seções obrigatórias: - Como funciona para Noivos - Como funciona para
Fornecedores - Programa para Assessores (Afiliados) - Depoimentos
simulados - Planos e monetização

------------------------------------------------------------------------

## 2. AUTENTICAÇÃO (CLERK - ORGANIZATIONS)

Configurar:

-   Login/Cadastro com OAuth (Google)
-   Multi-tenant via Organizations
-   Redirecionamento baseado em perfil:
    -   Noivo → Dashboard do Casamento
    -   Fornecedor → Dashboard Empresarial
    -   Assessor → Dashboard Afiliado
    -   Admin → Painel Administrativo

------------------------------------------------------------------------

## 3. DASHBOARD DO FORNECEDOR (ORGANIZATION)

Criar um painel moderno contendo:

-   Estatísticas de visualização do perfil
-   Leads recebidos
-   Plano atual (Free, Pro, Premium)
-   Botão para upgrade de plano
-   Gestão de portfólio (imagens via Supabase Storage)
-   Avaliações recebidas
-   Ranking público baseado em nota (não manipulável por plano)

------------------------------------------------------------------------

## 4. DASHBOARD DOS NOIVOS

Interface emocional e organizada com:

-   Contador regressivo para o casamento
-   Lista de desejos (favoritar fornecedores)
-   Lista de presentes
-   Lista convertível em dinheiro
-   Página pública personalizada (landing page do casamento)
-   Controle de RSVP digital
-   Gestão de convidados

------------------------------------------------------------------------

## 5. SISTEMA DE AFILIADOS (ASSESSORES)

Criar sistema completo de afiliado:

-   Link exclusivo de indicação
-   Dashboard de comissões
-   Total de fornecedores ativos indicados
-   Sistema de selos por performance:
    -   Bronze
    -   Prata
    -   Ouro
    -   Elite

Comissão: - Percentual sobre a primeira assinatura paga do fornecedor
indicado

------------------------------------------------------------------------

## 6. MODELO DE DADOS INICIAL (DRIZZLE + SUPABASE)

Tabelas obrigatórias:

-   users
-   organizations
-   vendor_profiles
-   couples
-   wedding_pages
-   gift_lists
-   gift_items
-   wishlists
-   reviews
-   affiliate_links
-   commissions
-   subscriptions
-   plans
-   leads
-   badges
-   rankings

Todas as tabelas empresariais devem conter: - organization_id -
created_at - updated_at - deleted_at (soft delete)

RLS obrigatório baseado em organization_id.

------------------------------------------------------------------------

## 7. EMAILS TRANSCIONAIS (RESEND)

Criar templates para:

-   Boas-vindas noivo
-   Boas-vindas fornecedor
-   Novo lead recebido
-   Nova avaliação recebida
-   Comissão gerada
-   Confirmação de pagamento

------------------------------------------------------------------------

## 8. EXPERIÊNCIA DO USUÁRIO (UX)

Priorizar:

-   Sensação premium e emocional para noivos
-   Sensação de autoridade e performance para fornecedores
-   Clareza e transparência no ranking
-   Sistema antifraude para avaliações

O sistema deve transmitir:

-   Confiança
-   Sofisticação
-   Organização
-   Profissionalismo

------------------------------------------------------------------------

## 9. ARQUITETURA

Fluxo multi-tenant:

Request → Clerk Auth → Extract Org ID → Supabase Query com RLS

Todas as queries empresariais devem obrigatoriamente filtrar por
organization_id.

------------------------------------------------------------------------

## 10. OBJETIVO DO MVP

Focar em:

-   Marketplace funcional
-   Assinatura fornecedor
-   Dashboard noivos
-   Sistema afiliado básico
-   Página pública de casamento

Deve estar pronto para:

-   Escalar
-   Internacionalizar
-   Receber investimento
