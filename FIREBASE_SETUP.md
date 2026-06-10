# Configurar o Firebase (sincronização online em tempo real)

O portal já está com o código de nuvem pronto. Enquanto o `FIREBASE_CONFIG` no `index.html` estiver com
`"COLE_AQUI"`, o site roda em **modo local** (igual antes: `localStorage` + login demo). Quando você colar
o config do seu projeto e der `push`, o **modo nuvem liga sozinho**: tudo que o consultor editar passa a
ficar **online em tempo real** no link do GitHub Pages, para todos.

> ⏱️ Tempo estimado: ~10 minutos. Não precisa cartão de crédito (plano gratuito "Spark" basta).

---

## Passo 1 — Criar o projeto
1. Acesse **https://console.firebase.google.com** e faça login com sua conta Google.
2. **Adicionar projeto** → dê um nome (ex.: `plano-amapa`) → pode **desativar** o Google Analytics → **Criar**.

## Passo 2 — Ativar o Firestore (banco)
1. No menu lateral: **Criar** (Build) → **Firestore Database** → **Criar banco de dados**.
2. Escolha **Modo de produção** → escolha a localização (ex.: `southamerica-east1`) → **Ativar**.
3. Aba **Regras** (Rules) → apague tudo e cole o conteúdo abaixo, **trocando o e-mail** pelo e-mail que
   você vai usar como consultor/admin:
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /plano/state {
         allow read: if true;
         allow write: if request.auth != null
           && request.auth.token.email == 'SEU_EMAIL_ADMIN@exemplo.com';
       }
     }
   }
   ```
4. **Publicar** (Publish).

> 🔒 Isso significa: **qualquer um pode ler** o plano (é o que o cliente vê), mas **só o seu e-mail
> consegue escrever**. Ninguém mais altera o conteúdo.

## Passo 3 — Ativar o login do consultor (Authentication)
1. Menu: **Criar** → **Authentication** → **Vamos começar**.
2. Aba **Sign-in method** → **E-mail/senha** → **Ativar** (a primeira opção) → **Salvar**.
3. Aba **Users** → **Adicionar usuário** → digite o **mesmo e-mail** das regras + uma **senha** → **Adicionar**.
   - Esse e-mail/senha é o login do consultor no portal.

## Passo 4 — Pegar o config do app web
1. Engrenagem (⚙️) ao lado de "Visão geral do projeto" → **Configurações do projeto**.
2. Role até **Seus apps** → clique no ícone **`</>`** (Web) → dê um apelido (ex.: `portal`) → **Registrar app**.
3. O Console mostra um objeto `firebaseConfig`. **Copie os valores.** Exemplo:
   ```js
   const firebaseConfig = {
     apiKey: "AIzaSy...",
     authDomain: "plano-amapa.firebaseapp.com",
     projectId: "plano-amapa",
     storageBucket: "plano-amapa.appspot.com",
     messagingSenderId: "123456789",
     appId: "1:123...:web:abc..."
   };
   ```

## Passo 5 — Colar no portal e publicar
1. Abra o `index.html` e localize, logo no começo do `<script>`, o bloco:
   ```js
   const FIREBASE_CONFIG = {
     apiKey: "COLE_AQUI",
     ...
   };
   ```
2. Substitua cada `"COLE_AQUI"` pelos valores do seu `firebaseConfig`.
3. Publique:
   ```bash
   git add index.html
   git commit -m "Conecta o portal ao Firebase"
   git push
   ```
4. Em ~1 minuto o site atualiza. Pronto: **modo nuvem ligado.**

> O `apiKey` aparecer no código é **normal e seguro** — ele é um identificador público, não uma senha. A
> proteção é a regra do Passo 2.

---

## Como testar
1. Abra o link do GitHub Pages, entre como **consultor** (seu **e-mail** + senha) → edite um título.
2. Abra o mesmo link numa **aba anônima** (faz o papel do cliente: `cliente` / `amapa`) → a edição deve
   aparecer em **poucos segundos**, sozinha.
3. Mude o status de uma ação e mova um widget na grade → reflete no cliente ao vivo.

## Perguntas comuns
- **O cliente precisa de conta?** Não. Ele só lê (entra com `cliente`/`amapa`, que é só uma portinha).
- **Funciona offline?** Não mais — o modo nuvem exige internet.
- **Onde ficam os dados?** Num único documento `plano/state` no seu Firestore (textos, status, links de
  vídeo e layouts de grade). As imagens em base64 continuam no `index.html`.
- **E upload de vídeo/foto?** É a próxima etapa (Firebase Storage), ainda não incluída.
- **Esqueci a senha do admin:** Authentication → Users → reset, ou crie outro usuário com o mesmo e-mail
  das regras.

---

# Fase 3b — equipe, papéis e convites (multi-usuário)

A interface de **Conta & Equipe** (menu no seu nome, no rodapé do menu lateral) já existe: você cria
papéis (Gerente, Colaborador…), marca permissões, lista a equipe e registra convites. Para que outras
pessoas **realmente entrem e editem conforme a permissão**, faça os passos abaixo.

> Você (o e-mail admin) é **superusuário fixo** nas regras — nunca se tranca pra fora.

## 1. Habilitar login por link de e-mail (para convites)
1. **Authentication → Sign-in method → Add new provider →** ative **E-mail/senha** (já feito) e marque
   também **"Link de e-mail (login sem senha)"** → **Salvar**.
2. **Authentication → Settings → Authorized domains:** confirme que **`thiago-akira.github.io`** está na
   lista (o GitHub Pages). Se não estiver, **Add domain**.

## 2. Trocar as regras do Firestore (papéis + permissões)
**Firestore → Regras**, apague e cole (troque `SEU_EMAIL_ADMIN@exemplo.com` pelo seu e-mail admin):
```
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    function admin(){ return request.auth != null && request.auth.token.email == 'SEU_EMAIL_ADMIN@exemplo.com'; }
    function team(){ return get(/databases/$(db)/documents/plano/team).data; }
    function mem(){ return team().members[request.auth.uid]; }
    function can(p){ return admin() || (request.auth != null && mem() != null
      && team().roles[mem().roleId].perms[p] == true); }
    match /plano/state { allow read: if true; allow write: if can('editContent') || can('editLayout'); }
    match /plano/team  { allow read: if true; allow write: if admin() || can('manageUsers'); }
  }
}
```
**Publicar.**

## 3. (Opcional) Storage para foto de perfil
Hoje a foto é por **URL** (cole o link de uma imagem). Para upload de arquivo:
**Storage → Começar → modo de produção**; depois me avise que eu ligo o upload no app.

## Como funciona o convite
1. No app: **menu do seu nome → Equipe → Convidar** (e-mail + papel). O Firebase envia um link de acesso.
2. A pessoa abre o link → entra → vira membro com o papel que você definiu → define uma senha para os
   próximos acessos.
3. Você pode trocar o papel de alguém ou cancelar convites a qualquer momento na aba **Equipe**.

> Permissões: **Editar conteúdo** (textos/status), **Organizar widgets** (mover/redimensionar/cores),
> **Exportar**, **Gerenciar usuários**. Só quem tem *Gerenciar usuários* (ou você) convida e mexe em papéis.
