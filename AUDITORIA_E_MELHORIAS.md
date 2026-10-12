# Auditoria técnica e funcional — ApexImob CRM

**Data:** 10 de outubro de 2026  
**Público-alvo informado:** uso em uma imobiliária  
**Escopo:** projeto recebido em ZIP; React/TypeScript/Vite, servidor Express, Firebase Authentication/Firestore, regras, PWA, CRM, relatórios, documentos, backup, IA e e-mail.

## Resumo executivo

O projeto já tinha uma interface visual extensa e uma boa base para operação imobiliária: dashboard, leads, imóveis, pipeline, corretores, comissões, visitas, tarefas, documentos, relatórios, backup e recursos de IA. **Preservei os módulos existentes** e fiz melhorias sem reescrever a aplicação do zero.

O código atual passa no TypeScript e no build de produção. A autenticação Firebase e as regras de associação à agência foram integradas no código, e as rotas privadas do servidor agora validam o token e a associação. Isso **não equivale a uma implantação pronta**: o Firebase, Vercel, e-mail, IA e armazenamento documental precisam ser configurados e testados com as contas e políticas reais da imobiliária.

### Decisão de liberação

**Recomendação: liberar somente para homologação, não para dados reais de clientes ainda.** O recorte por carteira está implementado no cliente e nas regras, mas ainda exige teste com contas distintas no Firebase da imobiliária. Também faltam armazenamento seguro de documentos, configuração do primeiro diretor, MFA e verificação da instância Firebase usada.

## Melhorias aplicadas

### Fluxo, interface e qualidade de uso

- Corrigida a inicialização independente das coleções para que uma coleção vazia não seja confundida com uma coleção ainda não inicializada.
- Corrigida a ordem de hooks no modal de perfil, que podia causar erro ao abrir/fechar.
- Mantida a navegação e os módulos preexistentes; atalhos de imóvel/visita e destinos do menu mobile foram completados.
- Busca global normaliza acentos e telefones formatados; métricas de meta passam a respeitar corretor selecionado e valores zero.
- Smart Match deixa de promover leads ganhos/perdidos ou candidatos incompatíveis em orçamento/localização.
- As tarefas deixaram de nascer com visitas e nomes fictícios. Agora podem ser compartilhadas em tempo real pelo Firestore, com tratamento de erro, confirmação de exclusão, cache local e inclusão no backup/restauração.
- Reduzido o carregamento inicial com módulos sob demanda e chunks separados; o build final não gerou chunk individual acima de 500 KB.
- Zoom e gestos mobile foram preservados; tema armazenado é aplicado antes da montagem da interface.

### Segurança, dados e integridade

- Login local/demonstrativo foi substituído por Firebase Authentication, sessão observada pelo SDK, verificação do membro ativo e associação ao `orgId` da agência.
- As regras Firestore foram substituídas por regras autenticadas para membros ativos, incluindo a coleção `crm_tasks`; convites e perfis administrativos têm restrições mais estritas.
- O endpoint de snapshot JSON aberto foi removido; rotas privadas do Express exigem token Firebase e verificação de vínculo/papel.
- Credenciais AWS saíram do navegador. SES só usa variáveis secretas do servidor e respostas de erro explícitas; HTML de e-mail é validado/escapado.
- Chamadas de IA foram reduzidas para enviar contexto mínimo, sem telefone e sem coleções completas; prompts não devem inventar disponibilidade, preço, urgência ou condições comerciais.
- Efeitos automáticos de backup que apenas simulavam uma cópia foram removidos. A interface agora descreve backup manual.
- Backups usam senha individual, AES-256-GCM e verificação SHA-256; a restauração faz mesclagem não destrutiva e inclui tarefas. Senha perdida não pode ser recuperada.
- Flags locais que simulavam MFA/senha padrão foram removidas. Textos de interface que alegavam assinatura digital, imutabilidade jurídica ou “documento oficial” foram corrigidos.
- Modelos de e-mail demonstrativos foram convertidos em rascunhos neutros/inativos e os indicadores de abertura deixaram de parecer métricas reais.
- Impressão de documentos agora escapa HTML para evitar injeção de script; saída TXT está identificada como TXT, e modelos impressos incluem aviso de rascunho/revisão.
- Exportações CSV protegem contra fórmula maliciosa e escapam aspas/campos fornecidos por usuários.

## Riscos e requisitos antes de operar com clientes

### Alto — arquivos e documentos ainda são locais

A tela **Gerenciador de Documentos & Contratos** salva conteúdo/arquivos em `localStorage` do navegador. Não usa Firebase Storage, não sincroniza entre membros e **não entra no backup criptografado**. Apesar do aviso e do limite inseridos na interface, não use essa tela para CPF, contratos assinados, documentos pessoais ou arquivos sensíveis antes de configurar armazenamento corporativo, autenticação, regras por agência, retenção e auditoria.

### Alto — homologação das regras e dados preexistentes

O código e as regras restringem corretores a leads atribuídos, imóveis captados, comissões vinculadas, tarefas atribuídas e logs relacionados à própria carteira. Gerentes e diretores mantêm visão operacional global dentro da agência. Essa separação foi validada por inspeção do código e compilação, mas não por leituras/escritas contra Firebase com contas reais separadas. Faça esses testes antes do uso com clientes e verifique a atribuição dos registros antigos: dados sem `assignedBrokerId`, `brokerCaptatorId`, `brokerIds` ou `assignedBrokerId` não devem ser atribuídos por suposição.

### Alto — identidade inicial e ambiente Firebase

A primeira conta diretora e seu registro `agency_members/{uid}` devem ser provisionados por meio administrativo seguro. É necessário garantir que `firebase-applet-config.json`, o banco Firestore e as credenciais Firebase Admin do servidor apontem para o mesmo projeto/database, e publicar `firestore.rules` nesse banco.

### Alto — possível tentativa de sincronização durante a inspeção inicial

Antes da migração para Firebase Authentication, houve uma abertura local de teste da versão original do projeto no navegador. O código original fazia sync/semeadura no início. **Não foi possível confirmar se essa instância conseguiu gravar no Firestore configurado no ZIP.** Por precaução, confira `system_meta`, `leads`, `properties`, `brokers` e `activity_logs` no projeto configurado, compare com seus dados e backups e não apague registros por suposição. A versão final exige sessão autenticada para carregar os dados.

### Médio/alto — controles de segurança operacional

- MFA/passkeys não são configurados pelo app; habilite no provedor e valide recuperação de conta.
- O histórico gera hashes SHA-256 no cliente, úteis para verificação de integridade, mas não são assinatura digital, prova oficial, blockchain ou imutabilidade absoluta.
- Backups são manuais e externos; convites administrativos não são alterados na restauração. Armazene cópia e senha em locais separados e controle quem pode restaurar.
- As APIs não têm rate limiting robusto por membro; configure limites/alertas de custo para Gemini e políticas antiabuso para SES antes de abrir a instância publicamente.
- Email SES e geração Gemini não foram chamados com credenciais reais; a chave Gemini não estava configurada localmente.
- E-mail tem envio manual; gatilhos automáticos de funil e rastreamento de abertura não estão ligados.
- O protótipo possuía exemplos de nomes, imóveis, leads e logs. Novas instalações em modo normal iniciam sem esses registros; confira se dados fictícios foram sincronizados em um projeto anterior antes de convidar usuários.
- Defina política de privacidade/LGPD, retenção, consentimento, exportação/exclusão e contratos com os provedores antes de usar PII real.

## Validação realizada

| Verificação | Resultado |
|---|---|
| `npm run lint` (`tsc --noEmit`) | **Passou na validação final**, sem erros TypeScript |
| `npm run build` | **Passou na validação final** (Vite + PWA); maior chunk JS: 476,91 KB |
| JSON/configuração Vercel | Validado em etapa anterior |
| Smoke test HTTP local | `GET /` retornou **200**; `POST /api/ai/qualify-lead` sem token retornou **401** e não chamou provedor nem alterou dados |
| Idempotência do log offline | Teste isolado passou: dois appends com o mesmo ID deixaram um único registro local; retries reutilizam o ID da fila/log e usam `setDoc` no mesmo ID na nuvem |
| Troca de usuário/cache | Teste isolado de `CRMStorage` com `localStorage` simulado passou: bloqueou fila pendente de outro UID, limpou cache na troca, descartou item sincronizado estrangeiro e colocou item antigo sem UID em quarentena. Isso não substitui teste com sessões Firebase reais |
| Navegação visual inicial | Página abriu localmente na versão original; não houve fluxo autenticado real |
| Login real, regras remotas, email SES, Gemini e deploy | Não testados: dependem de credenciais/conta da imobiliária |
| Suite de testes automatizados | Não existe suíte unitária/e2e configurada no projeto |

Para o Gemini, o ID `gemini-3.8-flash` foi conferido na documentação oficial do Google; não houve chamada real por falta de chave. Fontes: [VALIDACAO_FONTES_OFICIAIS.md](./VALIDACAO_FONTES_OFICIAIS.md).

## Checklist de homologação

1. Configure Firebase Auth, Firestore e Admin SDK para o mesmo projeto e banco.
2. Publique `firestore.rules` no banco correto e provisione a primeira conta diretora conforme “Provisionamento inicial do diretor” no README.
3. Teste diretor, gerente e pelo menos dois corretores com contas separadas. Confirme que cada corretor não consegue consultar nem alterar registros de outra carteira, inclusive por chamadas diretas ao Firestore.
4. Verifique se a versão original sem login deixou dados de teste no Firestore.
5. Configure Gemini e SES em segredos server-side; teste somente com dados e destinatários de homologação.
6. Teste backup, recuperação com senha, restauração não destrutiva e logout em dispositivo compartilhado.
7. Configure armazenamento corporativo para documentos antes de carregar dados pessoais ou contratos.
8. Habilite MFA, rate limit, alertas de custo, política LGPD e rotina de backup externo.
9. Só então publique para a equipe e inclua clientes reais.
