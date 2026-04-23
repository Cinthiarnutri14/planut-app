# 🏥 PLANUT – GUIA DE PUBLICAÇÃO
## Do zero ao app no ar em ~30 minutos

---

## PASSO 1 – Criar conta no GitHub
1. Acesse https://github.com/signup
2. Crie sua conta (gratuita)
3. Clique em **"New repository"**
4. Nome: `planut-app` → **Create repository**
5. Faça upload de todos os arquivos desta pasta

---

## PASSO 2 – Criar projeto no Firebase

1. Acesse https://console.firebase.google.com
2. Clique em **"Criar projeto"** → Nome: `planut-app`
3. Desative o Google Analytics (opcional) → Criar projeto

### Ativar Login com Google:
4. No menu esquerdo: **Authentication** → **Começar**
5. Aba **"Método de login"** → Google → **Ativar** → Salvar

### Ativar banco de dados (Firestore):
6. Menu esquerdo: **Firestore Database** → **Criar banco de dados**
7. Modo: **Produção** → Localização: `southamerica-east1` → Concluído

### Configurar regras do Firestore:
8. Aba **"Regras"** → cole o código abaixo → **Publicar**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /autorizados/{email} {
      allow read: if request.auth != null && request.auth.token.email == email;
      allow write: if false;
    }
  }
}
```

### Pegar as credenciais do Firebase:
9. **Configurações do projeto** (ícone ⚙️) → **Seus apps** → **Web** (ícone </>)
10. Nome: `planut-web` → **Registrar app**
11. Copie o objeto `firebaseConfig` que aparecer
12. Cole no arquivo **`src/firebase.js`** substituindo os campos `COLE_AQUI_...`

---

## PASSO 3 – Publicar no Vercel

1. Acesse https://vercel.com/signup
2. **Entrar com GitHub**
3. Clique em **"Add New Project"**
4. Selecione seu repositório `planut-app`
5. Clique em **Deploy** → aguarde ~2 min
6. ✅ Seu app estará em: `planut-app.vercel.app`

---

## PASSO 4 – Integrar com o Hotmart (liberar acesso automático)

Quando alguém compra no Hotmart, um webhook é enviado automaticamente.
Você precisa de uma **Cloud Function** para processar isso:

1. No Firebase → **Functions** → **Começar**
2. Instale Firebase CLI no seu computador:
   ```
   npm install -g firebase-tools
   firebase login
   firebase init functions
   ```
3. Cole este código em `functions/index.js`:

```javascript
const functions = require("firebase-functions");
const admin = require("firebase-admin");
admin.initializeApp();

exports.hotmartWebhook = functions.https.onRequest(async (req, res) => {
  // Verificação básica de segurança
  const token = req.headers["x-hotmart-hottok"];
  if (token !== "SEU_TOKEN_HOTMART") {
    return res.status(401).send("Não autorizado");
  }

  const body = req.body;
  const evento = body?.event;
  const email = body?.data?.buyer?.email;

  if (!email) return res.status(400).send("Email não encontrado");

  const db = admin.firestore();

  // Compra aprovada → liberar acesso
  if (evento === "PURCHASE_APPROVED" || evento === "PURCHASE_COMPLETE") {
    await db.collection("autorizados").doc(email).set({
      email,
      liberado_em: new Date().toISOString(),
      status: "ativo"
    });
    console.log(`✅ Acesso liberado: ${email}`);
  }

  // Reembolso ou cancelamento → revogar acesso
  if (evento === "PURCHASE_REFUNDED" || evento === "PURCHASE_CANCELLED") {
    await db.collection("autorizados").doc(email).delete();
    console.log(`❌ Acesso revogado: ${email}`);
  }

  return res.status(200).send("OK");
});
```

4. Deploy: `firebase deploy --only functions`
5. Copie a URL gerada (ex: `https://southamerica-east1-planut-app.cloudfunctions.net/hotmartWebhook`)

### No Hotmart:
6. **Ferramentas** → **Webhooks** → **Adicionar webhook**
7. Cole a URL da Cloud Function
8. Eventos: `PURCHASE_APPROVED` e `PURCHASE_REFUNDED`
9. Copie o **Hottok** gerado e cole no código acima (`SEU_TOKEN_HOTMART`)

---

## PASSO 5 – Liberar acesso manual (se precisar)

Se precisar liberar ou revogar manualmente:
1. Firebase → **Firestore** → Coleção `autorizados`
2. **Adicionar documento** → ID = email do comprador
3. Adicione o campo: `status: "ativo"`

---

## ✅ RESUMO DO FLUXO FINAL

```
Compra no Hotmart
      ↓
Hotmart envia webhook automático
      ↓
Cloud Function registra email no Firestore
      ↓
Comprador acessa planut-app.vercel.app
      ↓
Login com Google (Gmail)
      ↓
Sistema verifica email no Firestore
      ↓
✅ ACESSO LIBERADO  ou  ❌ "Sem licença ativa"
```

---

## 💡 DICAS FINAIS

- **Domínio personalizado**: Em Vercel → Settings → Domains → compre `planut.com.br` no Registro.br (~R$40/ano)
- **Reembolso automático**: O webhook já revoga o acesso automaticamente
- **Acesso pelo celular**: O app funciona como PWA – a pessoa pode "instalar" no celular
- **Custo**: Tudo GRATUITO até ~10.000 usuários/mês (Firebase + Vercel free tier)

---

📧 Dúvidas técnicas: guarde este guia com cuidado!
