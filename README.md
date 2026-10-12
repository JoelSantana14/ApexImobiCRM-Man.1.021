# ApexImob CRM

CRM web para operação imobiliária, desenvolvido com React, TypeScript, Vite, Firebase Authentication/Firestore e endpoints Express. A interface original foi preservada e os fluxos foram revisados para uso por uma equipe imobiliária.

> **Estado desta entrega:** código compilado e checado; não publicado e não conectado a credenciais reais da imobiliária. Antes de operar com clientes, conclua o checklist de configuração abaixo.

App no Google AI Studio: https://ai.studio/apps/63492fc5-c363-44ff-a3dd-3e399ad2b3f3

## Executar localmente

**Requisito:** Node.js 20 ou superior.

```bash
npm install
cp .env.example .env.local
# Configure as chaves somente no servidor; não use VITE_ para segredos.
npm run dev
```

O servidor Express de desenvolvimento serve o frontend Vite e as rotas `/api`. A variável `VITE_DEMO_MODE` deve permanecer `false` (padrão) para um ambiente real. `true` é apenas para uma demonstração isolada, nunca para o banco da imobiliária.

Checagens locais:

```bash
npm run lint
npm run build
```

## Configuração necessária antes de produção

### 1. Firebase

1. Confirme que `firebase-applet-config.json` aponta para o projeto Firebase correto. Os dados de configuração do cliente identificam o projeto; **não substituem** as credenciais administrativas do servidor.
2. No Firebase Console, habilite **Authentication → Email/Password** e adicione o domínio de produção em Authorized domains.
3. Crie/habilite o banco Firestore cujo ID corresponde ao indicado em `firebase-applet-config.json`.
4. Publique `firestore.rules` no **banco correto** antes do primeiro login. As regras autorizam membros ativos, restringem consultas de corretores pela carteira e reservam a visão global para gerente/diretor.
5. Provisione a primeira conta diretora conforme o procedimento abaixo. O papel inicial não pode ser criado pela interface pública.
6. Convide os demais membros pela área de equipe. O backend valida token Firebase, vínculo e papel de diretor. A validade do link de redefinição segue o Firebase; o CRM não anuncia um prazo de expiração próprio.

#### Provisionamento inicial do diretor

1. No Firebase Authentication do projeto de produção, crie a conta do diretor com o e-mail que será usado no acesso e copie o UID gerado.
2. No Firestore do mesmo projeto e database, crie o documento `agency_members/{UID}` (o ID do documento deve ser exatamente o UID) com os campos: `uid` (string, mesmo UID), `orgId` (string `apeximob`), `active` (boolean `true`), `role` (string `diretor`), `name` (string), `email` (string em minúsculas, igual ao e-mail da conta) e `createdAt` (string ISO 8601). `status: "ativo"` **não substitui** `active: true`.
3. Não crie esse vínculo pelo navegador nem dê papel de diretor a uma conta sem confirmar sua identidade. Use o Firebase Console com uma conta administrativa protegida ou um processo interno confiável com Firebase Admin SDK. Não compartilhe nem inclua a chave de serviço no ZIP/Git.
4. Um diretor pode operar sem `brokerId`. Se optar por vinculá-lo a uma carteira, crie também `brokers/{brokerId}` e confirme que o documento representa a carteira correta. Para usuários com papel de corretor, `brokerId` e o e-mail no documento `brokers/{brokerId}` precisam corresponder ao membro autenticado.
5. Faça login com a conta diretora e valide o acesso; depois convide gerente e corretores pela tela de equipe. Use contas distintas para homologar os três papéis antes de carregar dados reais.

**Permissões:** corretores consultam e alteram apenas leads atribuídos à sua carteira, imóveis captados por ela, comissões vinculadas, tarefas atribuídas e logs relacionados à própria carteira. Gerentes e diretores têm visão operacional global na imobiliária. As regras do Firestore também aplicam o recorte no servidor, portanto o filtro não depende apenas da interface. A autorização remota ainda precisa ser homologada no projeto real com contas separadas.

### 2. Variáveis de ambiente do servidor

Configure em **Vercel → Project Settings → Environment Variables** (ou em um gerenciador de segredos equivalente). Consulte `.env.example`.

- `FIREBASE_ADMIN_PROJECT_ID` e `FIREBASE_SERVICE_ACCOUNT_JSON`: Firebase Admin, somente no servidor.
- `GEMINI_API_KEY`: geração de IA, somente no servidor. O código não contém chave real.
- `APP_URL`: URL pública para links de autenticação.
- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION` e `AWS_SENDER_EMAIL`: opcionais, para AWS SES. Valide domínio/remetente e política de envio no SES.

Nunca coloque esses segredos no código, em `VITE_*`, no navegador ou no Git. O envio de e-mail permanece manual; gatilhos automáticos e métricas de abertura não estão integrados.

### 3. Publicação

O projeto contém `vercel.json` e a rota serverless `api/[...path].ts`. Importe o repositório no Vercel, configure as variáveis acima e publique. Verifique os logs das funções, as regras do Firestore e o domínio autorizado antes de convidar a equipe. Este pacote não executou publicação remota.

## Segurança e operação

- O login usa Firebase Authentication; não há senha mestra/local ou credenciais padrão no projeto.
- O cache local é vinculado ao UID autenticado; ao trocar de conta, os caches de CRM são limpos. Itens offline com UID de outro usuário bloqueiam a troca até serem sincronizados pela conta proprietária. Eventos offline antigos sem UID são isolados em quarentena e não são enviados automaticamente para outra carteira.
- As rotas privadas exigem token Firebase e associação ativa a `orgId: "apeximob"`.
- **MFA/passkeys não estão configurados pelo aplicativo.** Habilite as opções disponíveis no provedor e faça um teste de recuperação antes do uso real.
- O histórico gera hashes SHA-256 para verificações de integridade. Isso não é assinatura digital, blockchain, certificação jurídica nem prova de imutabilidade absoluta.
- O cofre oferece **exportação manual** criptografada AES-256-GCM. Use uma senha exclusiva de pelo menos 12 caracteres e guarde-a separadamente: não há recuperação se a senha for perdida. A restauração mescla registros sem apagar os ausentes e não altera convites administrativos.
- **Documentos e arquivos da tela de documentos ficam somente no `localStorage` deste navegador**; não sincronizam, não fazem parte do backup e não são protegidos pelas regras do Firestore. Não armazene CPF, contratos assinados, documentos de clientes ou outros dados sensíveis nessa área até configurar Firebase Storage (ou armazenamento corporativo equivalente), regras por agência, limites e política de retenção.
- Os modelos de contrato/recibo são rascunhos editáveis para revisão profissional. Não são documentos oficiais nem substituem validação jurídica.
- As chamadas de IA usam contexto reduzido, mas ainda podem conter dados comerciais fornecidos pela equipe. Revise a política de privacidade e não inclua dados pessoais desnecessários.
- Em uma implantação preexistente, confira se dados demonstrativos do protótipo chegaram ao Firestore. A configuração nova inicia sem leads/imóveis de demonstração; `VITE_DEMO_MODE=true` é opt-in.

## O que foi verificado nesta auditoria

- TypeScript (`npm run lint`) e compilação de produção (`npm run build`).
- Smoke test HTTP local da interface e das rotas protegidas, sem envio de e-mail ou escrita em uma conta Firebase real.
- Teste isolado da troca de UID no cache local e de retry com ID estável, incluindo bloqueio de fila pertencente a outra conta e quarentena de itens legados sem proprietário.
- O ID `gemini-3.8-flash` foi conferido em documentação oficial; nenhuma chamada à Gemini foi realizada sem chave configurada. Consulte [VALIDACAO_FONTES_OFICIAIS.md](./VALIDACAO_FONTES_OFICIAIS.md).

A aprovação operacional ainda depende de teste com Firebase/Vercel da imobiliária, teste de permissões com contas de papéis diferentes, configuração de armazenamento documental e validação da política de privacidade/retensão de dados.
